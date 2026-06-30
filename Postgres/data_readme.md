# Postgres Data README (D2-Postgres)

## Dataset Location

Primary dataset:

```text
/lp-dev/yw2399/Data_Backend/Dataset/Postgres/
```

Repository access path (via the `Dataset` symlink):

```text
/home/yw2399/Data_Backbone_Apps/Dataset -> /lp-dev/yw2399/Data_Backend/Dataset
/home/yw2399/Data_Backbone_Apps/Dataset/Postgres/   (resolves to the path above)
```

Metadata (under `Dataset/`):

```text
/lp-dev/yw2399/Data_Backend/Dataset/INDEX.csv               one row per Bug-ID (all apps)
/lp-dev/yw2399/Data_Backend/Dataset/_qc_report.{json,md}    per-dp deterministic v1.5.5 QC
/lp-dev/yw2399/Data_Backend/Dataset/_single_shot_build.json single-shot per-stage log/metric counts
/lp-dev/yw2399/Data_Backend/Dataset/backup/vlm_pre_review_20260628.tar.gz   pre-review VLM targets+text backup
```

## Dataset Purpose

Postgres is a multimodal RCA dataset for an HA relational OLTP backbone. It trains and evaluates
models that read metric **plots**, **logs**, and topology, then (Stage 1) *triage* and (Stage 2)
*diagnose*. It emphasizes database-backed-service failures: primary failover (with a deliberate
headless-DSN-no-failover handling bug), replica loss, network/DNS on the DB edge, caller-side
connection-pool exhaustion, lock contention, deadlocks, statement timeouts, vacuum/bloat, and
replication-slot loss — and, distinctively, **caller-vs-callee attribution** (the symptom surfaces
at the API while the root cause is not the database engine).

## Counts

Final merged package (trainable):

- Total datapoints: **80**
- Incident families: **16** (9 `PE*` external + 7 `PI*` internal; `PE09` disk-I/O deferred)
- Stage 1 VLM records: 80 | Stage 2 VLM records: 80 | total stage records: 160
- Source: 16 original (one curated run per fault) + 64 augmentation (strict ×5 repeats)

Issue-level distribution (0–4 scale):

- L0: 0
- L1 `resource anomaly / bounded single-event, no service incident`: 10  (PI02 statement-timeout, PI06 deadlock)
- L2 `component degradation, writes unaffected`: 10  (PE02/PE03 replica loss)
- L3 `service degradation`: 50
- L4 `hard service failure`: 10  (PE01 primary-kill DSN-stale; PE10 primary-kill + standby partition)

Stage-2 packet distribution (9 packets):

```text
db_lock_packet 20 | network_edge_packet 15 | pod_availability_packet 15
pod_resource_memory_packet 5 | storage_path_packet 5 | compound_incident_packet 5
connection_pool_packet 5 | pod_resource_cpu_packet 5 | replication_packet 5
```

Localization distribution (9 classes): `db_lock` 20, `kubernetes_availability` 15, `network_path`
15, `cluster_resource_cpu` 5, `cluster_resource_memory` 5, `compound` 5, `connection_pool` 5,
`replication` 5, `storage_path` 5.

## Layout

Three views of the same datapoints, keyed by `Bug-ID = postgres-<fault-kebab>__NN`
(e.g. `postgres-pi03-connection-pool-flood__03`):

```text
Postgres/RAW/<Bug-ID>/          (629M) full UNPURGED run archive — metric CSVs + windowed jsonl logs + manifests
Postgres/Single_Shot/<Bug-ID>/  (306M) all/ stage1/ stage2/  — raw logs+metrics concatenated, LEAK-PURGED
Postgres/VLM/<Bug-ID>/          ( 14M) stage1/ stage2/  — input_text.md + plot.png + target.json (v1.5.5, model-facing)
```

- **RAW/** — archival source of truth; unpurged. **Do not train on RAW.**
- **VLM/** — multimodal training/eval surface; injected fault never named in the input (answer only
  in `target.json`).
- **Single_Shot/** — for raw-log/raw-metric LLM baselines (`all` / `stage1` / `stage2`, each with
  `input.txt` + `output.txt`, stages also with `plot.png`). Metrics serialized under `=== METRIC:
  <stem> ===`, logs under `=== LOG: <file> ===`.

## The two-stage task

Same Stage-1/Stage-2 contract as the Kafka app (see `../Kafka/data_readme.md`). The 5 canonical
Stage-1 signals are sourced from Postgres metrics: `latency_badness ←
inventory_api_request_duration_seconds_p95{route="/orders"}`; `error_badness ←
inventory_api_requests_total{status=5xx}`; `throughput_drop_badness ← {status=2xx, route="/orders"}`
rate; `saturation_or_backlog_badness ← inventory_api_pool_in_use / pool_max (cap 20)`;
`availability_badness ← pg_up` (MIN across instances) + `deployment_replicas_available`. Thresholds
trace to a 30-minute steady-state baseline (`/orders` p95 ≈ 8.8 ms, 5xx = 0, ~10 req/s, pool ≈
0.18, `pg_up` = 1). Stage-2 adds three data-platform packets (`db_lock_packet`,
`connection_pool_packet`, `replication_packet`).

## Fault catalog (Bug-ID → meaning)

Strip the `__NN`; the `PE0x`/`PI0x` code maps to a run-spec in
`Orchestrator/experiments/fault_catalog/apps/postgres/` and a (localization, packet) in
`fault_mapping.py`. External `PE*`: primary pod kill (DSN-stale), replica/multi-replica kill,
app→primary network latency/loss, primary DNS failure, inventory-api CPU stress, primary memory
pressure (log-only), primary-kill+standby-partition (compound). Internal `PI*`: lock contention,
statement timeout, connection-pool flood, vacuum starvation, long-running transaction, deadlock,
replication-slot drop. Deferred (absent): `PE09` disk-I/O (Bitnami read-only mounts block clean
IoChaos). See the implementation README for full descriptions.

## How the dataset was produced

Identical pipeline to the Kafka app: live 12-step orchestrator runs; pinned localization/packet +
data-driven severity with a **log-aware floor** (critical here — `PE08`/`PI06`/`PI07` are
log-evident); plots + per-run mechanical log summaries from each run's real logs; ~5× strict-repeat
augmentation. The postgres load generator auto-runs at 10 req/s (no control API).

**Leak policy.** Model-facing surfaces are scrubbed of injection identity/mechanism (`fault_started`,
`fault_type=PIxx`, `PI03_pool_flood`, `/faults/`, chaos events, the `fault-injector-api`
component-role line, injector metric rows). The **explicit Postgres operator messages** (`deadlock
detected`, `canceling statement due to statement timeout`, `could not receive data from WAL stream`,
`cannot execute … in a read-only transaction`, `gaierror`) are *real evidence* and are **kept** —
only injection-control statements were removed. Verified **0** leaks across model-facing files. Note
for the paper: the explicit error strings are a potential **log-label shortcut**; consider a
log-redacted ablation track.

## Splits

No frozen split committed. Recommended: **group by `fault`** (all 16 families have exactly 5
executions → uniformly splittable), stratify by `localization` + `issue_level`; `source`-based
provenance split available (original 16 as a clean reference). Cross-app (train Postgres → test
Kafka) is the strongest generalization test, enabled by the shared schema.

## Caveats

- **Candidate, not frozen.**
- **PE09 (disk-I/O) is intentionally absent** — deferred for the Bitnami read-only-mount reason.
- Many `PI*` faults are log-driven: canonical metric panels can be flat while the LOG evidence
  carries the signal (by design; the log-aware floor scores them).
- `issue_level` in `INDEX.csv` is the per-run labeled severity. The agent review returned 16/16
  families PASS; the only target fixes were removing banned `confidence_reasons` (pe06, pi04) and
  the injection-framing actions on `pe07`.

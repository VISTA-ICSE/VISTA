# Kafka Data README (D1-Kafka)

## Dataset Location

Primary dataset:

```text
/lp-dev/yw2399/Data_Backend/Dataset/Kafka/
```

Repository access path (via the `Dataset` symlink):

```text
/home/yw2399/Data_Backbone_Apps/Dataset -> /lp-dev/yw2399/Data_Backend/Dataset
/home/yw2399/Data_Backbone_Apps/Dataset/Kafka/   (resolves to the path above)
```

Metadata (under `Dataset/`):

```text
/lp-dev/yw2399/Data_Backend/Dataset/INDEX.csv               one row per Bug-ID (all apps): bug_id,app,fault,issue_level,localization,source,raw_present,dp_hash
/lp-dev/yw2399/Data_Backend/Dataset/_qc_report.{json,md}    per-dp deterministic v1.5.5 QC
/lp-dev/yw2399/Data_Backend/Dataset/_single_shot_build.json single-shot per-stage log/metric counts
/lp-dev/yw2399/Data_Backend/Dataset/backup/vlm_pre_review_20260628.tar.gz   pre-review VLM targets+text backup
```

## Dataset Purpose

Kafka is a multimodal RCA dataset for an event-streaming order pipeline. It trains and evaluates
models that read time-series metric **plots**, **logs**, and topology, then (Stage 1) *triage* the
incident (severity + localization + which evidence to pull next) and (Stage 2) *diagnose* the root
cause from a focused evidence packet. It emphasizes streaming-platform failures: quorum-bounded
broker availability, producer-edge network degradation, broker storage-path (fsync) latency,
consumer-group rebalance/offset/backlog dynamics, and data-quality (poison/schema/DLQ) faults —
including many cases where every canonical metric is flat and the signal lives in the logs.

## Counts

Final merged package (trainable):

- Total datapoints: **93**
- Incident families: **20** (11 `KE*` external + 7 `KI*` internal + 2 `C*` compound)
- Stage 1 VLM records: 93 | Stage 2 VLM records: 93 | total stage records: 186
- Source: 20 original (one curated run per fault) + 73 augmentation (strict ×~5 repeats)

Issue-level distribution (0–4 scale, after the v1.5.5 review relabels):

- L0: 0
- L1 `resource anomaly, no service incident`: 10  (ke03 broker-memory, ke10 processor-OOM — both absorbed)
- L2 `component degradation`: 15  (poison/schema DLQ-absorbed, offset-commit, etc.)
- L3 `service degradation`: 53
- L4 `hard service failure / corroborated outage`: 15  (multi-broker kill, pod flap, retry amplification)

Stage-2 packet distribution (11 packets):

```text
pod_availability_packet 15 | pod_resource_cpu_packet 15 | network_edge_packet 11
pod_resource_memory_packet 10 | interstage_queue_backlog_packet 10 | consumer_group_packet 10
storage_path_packet 5 | pipeline_stage_latency_packet 5 | schema_validation_packet 5
data_quality_packet 5 | compound_incident_packet 2
```

Localization distribution (9 classes): `cluster_resource_cpu` 20, `kubernetes_availability` 15,
`network_path` 11, `backlog` 10, `cluster_resource_memory` 10, `consumer_group` 10,
`data_quality_path` 10, `storage_path` 5, `compound` 2.

## Layout

Three views of the **same** datapoints, keyed by a readable `Bug-ID = kafka-<fault-kebab>__NN`
(e.g. `kafka-ke05-producer-network-latency__03`; `NN` = per-fault run index by artifact timestamp):

```text
Kafka/RAW/<Bug-ID>/          (1.3G) full UNPURGED run archive — metrics CSVs + windowed jsonl logs (app/pods/namespaces/cluster) + manifests/snapshots
Kafka/Single_Shot/<Bug-ID>/  (666M) all/ stage1/ stage2/  — raw logs+metrics concatenated, LEAK-PURGED
Kafka/VLM/<Bug-ID>/          ( 15M) stage1/ stage2/  — input_text.md + plot.png + target.json (v1.5.5, model-facing)
```

- **RAW/** — archival source of truth; nothing scrubbed (intentionally retains `fault_active`,
  injector logs, chaos events for audit + reconstruction). **Do not train on RAW.**
- **VLM/** — the multimodal training/eval surface. The injected fault is **never named** in
  `input_text.md` or the plot; the answer lives only in `target.json`.
- **Single_Shot/** — for "dump unprocessed logs+metrics into an LLM" baselines:
  - `all/` = every (purged) component log + every (purged) metric CSV → `output.txt` = Stage-2 target
  - `stage1/` = full-topology raw logs + the 5 canonical-signal metric CSVs + `plot.png` → Stage-1 target
  - `stage2/` = packet-scoped raw logs + packet panel metric CSVs + `plot.png` → Stage-2 target

  Each metric CSV is serialized under a `=== METRIC: <stem> ===` header; each log under `=== LOG: <file> ===`.

## The two-stage task

**Stage 1 (triage).** Input = a fixed 5-panel canonical-"badness" plot + text (topology,
canonical-signal table, mechanical log/event summary). Target = `{issue_level (0–4), localization,
suspected_components, suspected_edges, recommended_stage2_focus, rationale_brief}`.

**Stage 2 (root cause).** Input = the selected packet's 2×2 metric plot + focused text (Metric
Summary, Selected Normalized Event Evidence with real verbatim lines). Target = `{issue_level,
root_cause_type, root_cause_scope, root_cause_target, resource_root_cause, service_incident_root_cause,
evidence_for, evidence_against_alternatives, recommended_debugging_actions, recommended_recovery_actions}`
(compound dps add `compound_root_causes`/`compound_interaction`).

The 5 canonical Stage-1 signals (normalized `[-1.1,1.1]`, 0=baseline, higher=worse), Kafka sources:
`latency_badness ← orders_processing_latency_seconds_p95`; `error_badness ←
orders_processing_errors_total`; `throughput_drop_badness ← orders_consumed_total{status=success}`
(clamped ≥0 + noise floor); `saturation_or_backlog_badness ← kafka_consumergroup_lag`;
`availability_badness ← deployment_replicas_available` (MIN across deployments).

## Fault catalog (Bug-ID → meaning)

Strip the `__NN` to get the fault id; the `KE0x`/`KI0x` code maps to a run-spec in
`Orchestrator/experiments/fault_catalog/apps/kafka/` and a (localization, packet) in
`Orchestrator/orchestrator/labeling/fault_mapping.py`. External `KE*`: broker pod kill, CPU
(retargeted to DB), broker memory, producer net latency/loss, broker disk-IO, processor/producer
CPU, processor pod flap, multi-broker kill, processor OOM. Internal `KI*`: poison message,
processing slowdown, offset-commit suppression, rebalance storm, back-pressure, schema mismatch,
DLQ saturation. Compound `C1`/`C2`. See the implementation README for full descriptions.

## How the dataset was produced

Each datapoint is a **live** 12-step orchestrator run (baseline → inject → fault → recovery →
finalize). Localization + Stage-2 packet are pinned per fault; severity is **data-driven** from the
canonical signals + a log-aware floor (FATAL/ERROR counts). Plots + per-run mechanical log summaries
are regenerated from each run's own logs. Augmentation re-runs each fault ~5× (strict repeats);
only the per-fault answer is inherited.

**Leak policy.** Model-facing surfaces (VLM `input_text.md`/`plot.png`; Single_Shot `input.txt`)
are scrubbed of the injected-fault identity and mechanism: `fault_active` arrays emptied,
fault-naming `experiment_id` nulled, `fault-injector-api` excluded, `/faults/`/chaos/`poison_burst`/
`fault_enabled` lines dropped, the `fault-injector-api` component-role line removed. Organic
operator symptoms are **kept** (broker fence, ISR shrink, consumer lag, DLQ growth, poison-prefixed
order ids, network timeouts). Verified **0** leaks across all model-facing files.

## Splits

No frozen train/val/test split is committed; `INDEX.csv` carries the dimensions to split safely.
Recommended: **group by `fault`** (never split a fault's ~5 repeats across train/test — they share
the inherited answer), stratify by `localization` + `issue_level`. The `source` column allows a
provenance split (original 20 as a clean reference, augmentation for training). Cross-app
(train Kafka → test Postgres) is the strongest generalization test, enabled by the shared
canonical-signal + packet schema.

## Caveats

- **Candidate, not frozen** (awaiting human end-to-end freeze review).
- A few `ki01-poison` Single_Shot `all/input.txt` are large — the fault's genuinely verbose DLQ
  logs, faithfully concatenated (truncated at 20 MB/log).
- Broker logs live only in `RAW/<id>/logs/namespaces/`; Single_Shot uses that superset.
- `issue_level` in `INDEX.csv` is the per-run labeled severity; where the design "expected" level
  and the run's data disagreed, the data won (see the four review relabels in the impl README).

# Redis Data README (D3-Redis)

## Dataset Location

Primary dataset:

```text
/lp-dev/yw2399/Data_Backend/Dataset/Redis/
```

Repository access path (via the `Dataset` symlink):

```text
/home/yw2399/Data_Backbone_Apps/Dataset -> /lp-dev/yw2399/Data_Backend/Dataset
/home/yw2399/Data_Backbone_Apps/Dataset/Redis/   (resolves to the path above)
```

Important metadata:

```text
/lp-dev/yw2399/Data_Backend/Dataset/Redis/_mapping.json       EXP_ID -> built_dp / raw_run / family / source / packet / issue_level
/lp-dev/yw2399/Data_Backend/Dataset/Redis/_vlm_qc.json        per-dp deterministic v1.5.5 QC (88/88 PASS)
/lp-dev/yw2399/Data_Backend/Dataset/Redis/_qc_report.md       narrative QC report (build + repairs + verification)
/lp-dev/yw2399/Data_Backend/Dataset/Redis/_rephrase_log.json  the 17 stage-2 target v1.5.5 regenerations
/lp-dev/yw2399/Data_Backend/Dataset/Redis/_build_log.json     single-shot per-stage purge counts
```

## Dataset Purpose

Redis is a multimodal RCA dataset for an in-memory cache tier (primary + async replica). It trains and evaluates models that read logs, metrics, plots, and structured context, then infer issue level, localization, and root cause. It emphasizes cache-tier engineering failures that are mechanistically distinct from a message log (Kafka) or a relational store (Postgres): single-threaded **event-loop head-of-line blocking**, **cache eviction / OOM-on-write**, **server-side connection-slot exhaustion**, **persistence forks**, **pod/network faults**, and — distinctively — faults whose signal is *not* where you'd first look (event-loop blocks show in throughput, not latency; a maxclients flood is client-invisible; async replication absorbs replica/replication faults).

## Counts

Final merged package (trainable):

- Total datapoints: 88
- Incident families: 15 (14 fault + 1 healthy baseline; +2 internal faults excluded — see below)
- Stage 1 VLM records: 88 · Stage 2 VLM records: 88 · Total stage records: 176
- Single-pass datapoints: 14 (one per fault family + initial set)
- Augmentation datapoints: 74 (5 independent live runs per fault family + healthy baselines)

Issue-level distribution (0–4 scale):

- L0 `no_fault` / `no_impact_observed`: 22  (4 healthy baseline + 18 absorbed)
- L1 `resource_anomaly_no_service_incident`: 8  (CPU/memory pressure contrastives)
- L2 `component_degradation`: 14  (OOM-on-write, maxclients, replica kill)
- L3 `service_degradation`: 30  (event-loop block, eviction storm, primary kill, edge latency)
- L4 `service_failure`: 0  (redis faults here are bounded/recoverable)

Packet distribution (10 packets):

- `event_loop_packet`: 18
- `cache_eviction_packet`: 12
- `pod_availability_packet`: 12
- `network_edge_packet`: 12
- `connection_pool_packet`: 6
- `pod_resource_cpu_packet`: 6
- `pod_resource_memory_packet`: 6
- `replication_packet`: 6
- `persistence_packet`: 6
- `healthy_baseline_packet`: 4

## Directory Layout

```text
RAW/<EXP_ID>/                 full copy of the source run (unpurged archive)        129M
Single_Shot/<EXP_ID>/{all,stage1,stage2}/                                            83M
VLM/<EXP_ID>/{stage1,stage2}/                                                         15M
_mapping.json  _vlm_qc.json  _qc_report.md  _rephrase_log.json  _build_log.json
```

`EXP_ID = dp_<hex>` (the neutral datapoint id; unique across the 14 single-pass + 74 augmented runs). The fault family is in `_mapping.json` only — it is **not** placed inside any model-facing input. Use `_mapping.json` for the audit-only EXP_ID → built_dp / raw-run / family mapping.

## RAW

`RAW/<EXP_ID>/` is a full byte copy of the source orchestrator run, for audit and reconstruction:

```text
RAW/<EXP_ID>/logs/{app,namespaces,cluster,pods}/   per-pod kubectl+loki logs, kubernetes events
RAW/<EXP_ID>/metrics/*.csv                          32 Prometheus CSVs (incl. control-plane injector rows)
RAW/<EXP_ID>/rendered/                              rendered chaos CRD (RE*) / executor inputs
RAW/<EXP_ID>/faults/                                fault start/stop records (name the injected fault)
RAW/<EXP_ID>/{summary,metadata,verification_*,cleanup_*}.json
```

RAW intentionally retains control-plane provenance (the `faults/` records, `redis-fault-injector` logs, chaos-mesh events, injector metric rows) that is removed from the model-facing views. **Do not train from RAW.**

## VLM Records

`VLM/<EXP_ID>/stage1/`:

```text
input_text.md   plot.png   target.json
```

Stage 1 is broad triage: five canonical badness panels (one 5-panel image) + a canonical-signal summary table + a deterministic mechanical diagnostic summary (incl. non-canonical resource evidence) + a categorized log/event summary. Target = `issue_level`, `localization`, `suspected_components`, `suspected_edges`, `recommended_stage2_focus`, `rationale_brief`.

`VLM/<EXP_ID>/stage2/`:

```text
input_text.md   plot.png   target.json
```

Stage 2 is focused RCA. The component log evidence is **inlined into `input_text.md`** (one text per stage — there is **no** separate `input_logs.md`), under a `## Component Logs (raw, windowed, capped)` section. The packet plot shows the chosen packet's panels. Target (audit contract **v1.5.5**) = `issue_level`, `root_cause_type`, `root_cause_scope`, `root_cause_target`, `resource_root_cause`, `service_incident_root_cause`, `evidence_for`, `evidence_against_alternatives`, `recommended_debugging_actions`, `recommended_recovery_actions`. There is **no** `confidence`/`confidence_reasons` field (banned by v1.5.5).

Each stage's `## Task` is the app5-style block: a one-line instruction + the exact JSON object shape with per-field guidance (Redis localization / packet / root_cause_type / scope enums) + "Do not include prose outside the JSON object."

Model-facing prompts carry **no** mechanism vocab (chaos kinds, `redis-fault-injector`, fault ids, `injected`, config-knob-as-mechanism), no raw absolute paths, no control-plane identity. Organic symptoms (`OOM command not allowed`, `max number of clients reached`, `EVAL`/slow-Lua, eviction/latency rises, pod `Killing`/`Started`, "no OOMKilled / ruled out" reasoning) are retained as legitimate evidence.

## Single-Shot Records

`Single_Shot/<EXP_ID>/`:

```text
all/input.txt      all/output.txt
stage1/input.txt   stage1/plot.png   stage1/output.txt
stage2/input.txt   stage2/plot.png   stage2/output.txt
```

`input.txt` concatenates the **raw** kubectl logs and serialized metric CSVs (`=== LOG: … ===` / `=== METRIC: … ===`), purged of answer-leaks (below) but otherwise unprocessed. `output.txt` is the stage target (stage1 = Stage-1 target; stage2/all = the v1.5.5 Stage-2 target). Per-stage scope:

- `all`: every component log (minus the injector) + cluster events (purged) + all 32 metric CSVs (injector rows dropped).
- `stage1`: the redis-primary/replica/load-generator logs + the canonical-panel metric stems (+ non-canonical resource stems cited in the stage-1 text).
- `stage2`: the suspected-component logs + the gold packet's plotted metric stems.

Use Single_Shot for experiments testing whether an LLM can reason over less-processed operational evidence.

### Single-Shot answer-leak purge

Single_Shot inputs are purged of injection-mechanism leaks while keeping organic signal (verified: **0 mechanism leaks** across all 264 input files):

- Dropped entirely: the `redis-fault-injector` component logs; the `rendered/` chaos CRDs / `faults/` records (not logs).
- **Metric CSV rows for the `redis-fault-injector` control-plane deployment are dropped** (kube-state series name it) — redis-primary/replica/load-generator rows are kept.
- Line-filtered from `kubernetes_events`: chaos-mesh lines (`podchaos`/`networkchaos`/…, "Successfully apply/recover chaos", `desiredPhase`, finalizers, chaos resource names, `RIxx`/`RExx`, "inject") — while organic pod-lifecycle (`Killing`/`Started`/`Created`/`Scheduled`) is **kept**.
- Domain symptoms are **not** purged: `OOM command not allowed`, `max number of clients reached`, replication sync, eviction/latency complaints survive.

## Incident Families

Internal Redis faults (control-plane injector, real mechanisms):

- `RI01_maxmemory_eviction_storm`, `RI02_noeviction_oom_on_write`, `RI03_on_command_block`, `RI04_big_key_operation`, `RI05_slow_lua_script`, `RI06_maxclients_saturation`

External Chaos Mesh faults:

- `RE01_primary_pod_kill`, `RE02_replica_pod_kill`, `RE03_primary_memory_oomkill` (contrastive), `RE04_primary_cpu_stress` (contrastive), `RE05_app_to_redis_network_latency`, `RE06_master_replica_network_latency` (absorbed), `RE07_persistence_io_latency` (absorbed), `RE08_redis_dns_failure` (absorbed)

Healthy control: `BASELINE_healthy_control`.

Excluded / deferred (not in the 88): `RI07_cache_stampede_mass_expiry` (no backend behind the cache → not exercisable), `RI08_persistence_fork_stall` (fork too fast on the tiny dataset → covered by RE07).

## Canonical Signals (Stage 1)

Redis's five canonical signals are **workload-outcome sourced** (client-experienced where possible), normalized to `[-1, 1]` (0 = baseline, higher = worse):

- `latency_badness` ← `redis_lg_op_latency_seconds_p95` (load-gen client p95; dual-clause ratio AND absolute-delta so a sub-ms baseline can't false-fire)
- `error_badness` ← `redis_lg_error` as a **fraction** of total client ops (a high error *rate* with most ops succeeding is partial degradation, not failure — RI02)
- `throughput_drop_badness` ← `redis_lg_success_rate` drop vs baseline
- `saturation_or_backlog_badness` ← `redis_evicted_keys_rate` (eviction is the symptom; a full cache is normal)
- `availability_badness` ← `deployment_replicas_available` (+ `redis_up`, `redis_master_link_up`); **control plane excluded** (`plot_exclude_control_plane`)

This sourcing is *why* event-loop blocks show in throughput (not latency p95), a maxclients flood is client-invisible, and absorbed replica/replication faults stay flat.

## Metrics and Logs

Metrics (32 CSVs/run, 5s step) include the load-gen client signals (`redis_lg_op_latency_seconds_p95`, `redis_lg_ops_total{status}`, `redis_lg_errors_total{type}`), redis_exporter server series (memory used/max, evicted-keys, connected/rejected clients, keyspace hit/miss, replication offsets, persistence status), and cgroup/kube-state series (`container_cpu_utilization`, `container_memory_working_set_bytes`, `deployment_replicas_available`, `pod_restarts_total`). Columns: `window,timestamp,value,metric_name,metric_group,query` (+ label columns for kube-state).

Logs are real pod stdout (kubectl + Loki), windowed baseline/fault/recovery. Stage 1 uses a categorized severity/event summary; Stage 2 inlines the suspected-component log snippets. The control-plane injector logs are excluded from all model-facing surfaces.

## Severity / Label Policy

- Event-loop / eviction / network-edge / primary-kill faults that cross the canonical threshold → **L3**; partial/client-invisible component degradation (OOM-on-write, maxclients, replica kill) → **L2**; resource anomaly with no service impact (StressChaos cpu/memory) → **L1**; absorbed faults → **L0 `no_impact_observed`**; healthy baseline → **L0 `no_fault`**.
- A `connection_pool` L2 floor fires on a positive `rejected_connections` counter delta when no canonical signal moves (RI06 is client-invisible).
- No L4 is present (redis faults here are bounded/recoverable).

## Schema-Fix Provenance (v1.5.5)

All 88 Stage-2 targets are v1.5.5-conformant (recorded in `_rephrase_log.json` + `_qc_report.md`):

- 71 were already compliant (both action fields, no confidence field).
- 17 were brought to v1.5.5 via the leak-safe `rephrase_stage2` "rephrase-known-facts" transform: dropped `confidence_reasons`, added `recommended_debugging_actions` + `recommended_recovery_actions` (grounded on each run's own structural + numerical + mechanical facts; cite that dp's numbers).
- The 3 RE03 targets had their `resource_root_cause` corrected from `primary_container_memory_oomkill` → `primary_container_memory_pressure` (StressChaos can't OOMKill redis; their own evidence shows restarts=0 / OOMKilled=0) — fixed at the source structural labeler.
- 5 old dps' stale stage-2 carried rationale (`<packet> fault`) were synced to the clean stage-1 target rationale.

## Validation and Audits

Deterministic QC (`qc_redis_vlm.py`) over all 88 VLM dps — **88/88 PASS**:

- no `confidence` key; both action fields present; full v1.5.5 required set
- 0 mechanism-leak hits on any input/target surface; `redis-fault-injector` absent everywhere
- plot↔text metric correspondence (canonical sources in the 5-panel; stage-2 metric rows match the packet panels)
- carried Stage-1-rationale consistent with the Stage-1 target
- no misleading `oomkill` resource labels

An AI agent additionally eyeballed 1 representative per fault family (both stages) + every build-warned datapoint for plot↔text match, sufficiency-to-triage, and no-answer-disclosure; the single-shot purge was agent-verified across every chaos (RE*) family.

See: `_vlm_qc.json`, `_qc_report.md`, `_rephrase_log.json`, `_build_log.json`, and the questionnaire at `/home/yw2399/Data_Backbone_Apps/docs/Paper/Q_Sec_3/Redis/`.

## Recommended Use

For standard VLM training:

```text
VLM/<EXP_ID>/stage1/
VLM/<EXP_ID>/stage2/
```

For raw-evidence LLM baselines:

```text
Single_Shot/<EXP_ID>/
```

For audit/reconstruction:

```text
RAW/<EXP_ID>/   (+ _mapping.json for EXP_ID -> family/raw-run)
```

Do not expose the fault-family name or RAW control-plane provenance to the model during normal training unless the experiment explicitly studies leakage. The dataset is a **candidate** pending human freeze review.

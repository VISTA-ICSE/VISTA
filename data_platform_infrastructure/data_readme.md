# Data-Platform Infrastructure — Data And Specification README (`data_platform_infrastructure`)

This README explains the data-like assets owned by the data-platform infrastructure. Unlike the app
deliverables (`Kafka`, `Postgres`, `Redis`, `Boutique`), this is **not** a model-training dataset.
It is the fault-catalog, metric-pack, app-profile, contract, and raw-artifact-shape layer that
*produces and validates* those datasets.

## Locations

- Source repo: `/home/yw2399/Data_Backbone_Apps`
- Published code snapshot: `/lp-dev/yw2399/VISTA_Publish/code/data_platform_infrastructure`
- Implementation README: `/lp-dev/yw2399/VISTA_Land/data_platform_infrastructure/README.data`
- Handoff: `/home/yw2399/Demo_Apps/docs/Handoffs/data_platform_infrastructure.md`

The datasets this layer produces are stored separately:

```text
/lp-dev/yw2399/Data_Backend/Dataset/Kafka      (D1)
/lp-dev/yw2399/Data_Backend/Dataset/Postgres   (D2)
/lp-dev/yw2399/Data_Backend/Dataset/Redis      (D3)
/lp-dev/yw2399/Data_Backend/Dataset/Boutique   (W1)
```

## Specification Inventory

The snapshot includes these specification families:

- Chaos Mesh reusable templates: `21` (`experiments/fault_catalog/templates/chaos_mesh/`)
- Data-platform app fault bindings: **kafka `18`, postgres `17`, redis `15`, boutique `27`**
- Metric packs (per-app scrape lists + canonical-signal sources): `5` (`kafka`, `postgres`,
  `redis`, `boutique`, `default`)
- Audit contract files (v1.5.5): `6` (spec, checklist, changelog, audit prompt, supporting note, README)
- Dataset build/QC/review scripts: `~21`
- Orchestrator Python modules: `~194`

## Fault Catalog Layout

Fault specifications live under `orchestrator_repo/experiments/fault_catalog/`:

- `schema/` — YAML schema for fault templates + bindings.
- `templates/chaos_mesh/` — reusable single-fault + ramp templates (PodChaos / StressChaos /
  NetworkChaos / IoChaos / DNSChaos).
- `templates/compound/` — compound (multi-fault) templates.
- `apps/<app>/` — app-specific bindings mapping a template onto concrete targets
  (`apps/kafka/KE01_broker_pod_kill.yaml`, `apps/postgres/PE04_app_to_primary_network_latency.yaml`, …).
  For the data-platform apps the bindings are largely **self-contained run specs**.
- `index.yaml`, `README.md` — catalog index + guidance.

The catalog is declarative so a reviewer can inspect exactly what was *supposed* to happen before
reading any collected artifact.

## App Profiles

Each `apps/<app>/app_profile.yaml` (in the source repo; referenced by the engine) describes:
components + roles, dependency edges, the five `canonical_stage1_signals` with per-app source
metrics + thresholds, `stage2_packet_types` + `stage2_panel_renderers`, the `log_normalizer`
(component log patterns + per-packet log selection), and `fault_to_packet_mapping`. It is the
bridge between raw infrastructure evidence and deterministic model-facing datapoints.

The five canonical Stage-1 "badness" signals are universal in meaning (latency / error /
useful-throughput-drop / saturation-or-backlog / availability), normalized to `[-1.1, 1.1]`
(0=baseline, higher=worse), but each signal's **source metric is chosen per app** in the profile
(e.g. Kafka `saturation ← kafka_consumergroup_lag`, Postgres `saturation ← inventory_api_pool_in_use/
pool_max`). Plot-contract v1.4 fixes the derivation (type-aware aggregation, MIN for boolean health,
throughput clamp + floor, availability-as-replica-loss).

## Generated Raw-Artifact Shape

When the infrastructure runs an experiment, the raw artifact directory (archived as the dataset's
`RAW/<Bug-ID>/`) typically contains:

- `metrics/` (or `performance_metrics/`) — Prometheus range-query CSVs, columns
  `window,timestamp,value,metric_name,…`; windows = `baseline` / `fault` / `recovery`.
- `logs/{app,pods,namespaces,cluster}/` — windowed pod logs (`*kubectl.jsonl` + `*loki.jsonl`) +
  cluster `kubernetes_events`. (Note: `namespaces/` is the complete superset incl. StatefulSet pods.)
- `metadata.json`, `summary.json`, `config_snapshot.yaml`, `collection_manifest.json`.
- `root_cause/`, `state_transitions.jsonl`, snapshots.
- `verification_report.{json,md}`, `cleanup_report.{json,md}`, `artifact_verification.json` —
  evidence that the intended signal appeared and that cleanup was clean.

The invariant: evidence + cleanup/verification status are stored beside the run. The RAW view is
the **unpurged** archive (retains injector logs, `fault_active`, chaos events for audit) and is not
model-facing.

## Dataset Views The Builders Produce

For each datapoint the app-specific builder emits three views (`Bug-ID = <app>-<fault-kebab>__NN`):

- `RAW/<Bug-ID>/` — full unpurged run archive (above).
- `Single_Shot/<Bug-ID>/{all,stage1,stage2}/` — raw logs+metrics concatenated and **leak-purged**
  (`=== LOG: … ===` / `=== METRIC: … ===` headers; `output.txt` = the stage target).
- `VLM/<Bug-ID>/stage{1,2}/` — `input_text.md` + `plot.png` + `target.json`, conforming to v1.5.5.

## The v1.5.5 Contract (`audit_contracts/v1_5_5/`)

The cross-app datapoint contract validated for every dataset: Stage-1 (6 fields) + Stage-2 (10
fields) schemas, badness semantics, the **plot-contract v1.4** amendment (§4.7 / §5.2.1), leakage/
redaction rules (no Chaos-Mesh/PodChaos/fault-id/injection vocabulary in model-facing surfaces;
incident-window wording), §16 resource-localization visibility, §17 mechanical log/event-evidence
categories, §23 log-summary truthfulness. The checklist file is the auditor's gate list.

## Leakage And Provenance Rules

For model-facing surfaces, direct-answer leakage is removed; legitimate symptoms are kept:

- **Remove:** chaos controller/event lines (podchaos/networkchaos/…/apply chaos/desiredPhase),
  rendered fault manifests, the `fault-injector` control plane (logs, metric rows, component-role
  listing), `fault_active` arrays, fault-naming `experiment_id`, `/faults/` calls, `fault_enabled/
  started`, template/fault names, absolute local paths.
- **Keep:** timeouts, HTTP 5xx, readiness failures, pod-lifecycle events, broker fences, consumer
  lag, DLQ growth, deadlock/lock-wait/gaierror/WAL-stream errors, raw metrics + probes, app logs
  that describe observed behavior rather than the injection plan.

## Validation Data

The infrastructure emits validation at several levels: fault-library validation, app-profile
validation, campaign preflight, per-experiment sanity verification, cleanup reports, artifact-shape
verification, the deterministic v1.5.5 dataset QC (`scripts/qc_dataset_review.py` →
`Dataset/_qc_report.{json,md}`), and single-shot purge logs (`Dataset/_single_shot_build.json`).
These reports are as important as the datapoints — they explain exclusions, recollections, weak
effects, deferred faults, and relabels.

## How To Cite Or Reuse This Layer

For paper writing, cite this as the experimental-control + reproducibility layer for the
data-platform RCA datasets: executable fault definitions, app bindings, collection contracts,
verification + cleanup guarantees, deterministic datapoint builders, and the v1.5.5 audit contract.
For model training, use the per-app dataset roots. To reproduce or extend, use the published code
snapshot + the per-app dataset/source manifests.

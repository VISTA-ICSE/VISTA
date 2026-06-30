# Boutique Data README (W1-Boutique)

## Dataset Location

Primary dataset:

```text
/lp-dev/yw2399/Data_Backend/Dataset/Boutique/
```

Repository access path (via the `Dataset` symlink):

```text
/home/yw2399/Data_Backbone_Apps/Dataset -> /lp-dev/yw2399/Data_Backend/Dataset
/home/yw2399/Data_Backbone_Apps/Dataset/Boutique/   (resolves to the path above)
```

Important metadata:

```text
/lp-dev/yw2399/Data_Backend/Dataset/Boutique/_mapping.json            EXP_ID -> dp/binding/packet/issue_level/is_ti/raw_run
/lp-dev/yw2399/Data_Backend/Dataset/Boutique/INDEX.csv                one row per EXP_ID
/lp-dev/yw2399/Data_Backend/Dataset/Boutique/_schema_fix_report.json  per-dp v1.5.5 schema-fix class/source/TI-coherence
/lp-dev/yw2399/Data_Backend/Dataset/Boutique/_qc_report.json          per-dp deterministic QC (84/84 PASS)
/lp-dev/yw2399/Data_Backend/Dataset/Boutique/_build_log.json          single-shot per-stage purge counts
```

## Dataset Purpose

Boutique is a multimodal RCA dataset for an 11-service Online Boutique gRPC mesh. It trains and evaluates models that read logs, metrics, plots, and structured context, then infer issue level, localization, and root cause. It emphasizes microservice-engineering failures: dependency-edge latency/errors/timeouts, retry amplification, resource pressure, pod availability loss — and, distinctively, **business-correctness violations** (duplicate charge, orphaned order, unfinalized cart, price drift) that are invisible to performance metrics and only provable from the application's own transactional records.

## Counts

Final merged package (trainable):

- Total datapoints: 84
- Incident families: 18 (+2 deferred, no substrate)
- Stage 1 VLM records: 84
- Stage 2 VLM records: 84
- Total stage records: 168
- Single-pass datapoints: 18
- Augmentation datapoints: 66 (57 main + 9 low-intensity)

Issue-level distribution (0–4 scale):

- L0 `no_fault` / `no_impact_observed`: 5  (2 healthy baseline + 3 absorbed pod-kill)
- L1 `resource_anomaly`: 0  (no execution landed in this band)
- L2 `component_degradation`: 13
- L3 `service_degradation` / confirmed correctness violation: 66
- L4 `service_failure`: 0  (boutique faults are bounded/recoverable)

Packet distribution (9 packets):

- `transaction_integrity_packet`: 24
- `service_dependency_packet`: 14
- `network_edge_packet`: 12
- `pod_availability_packet`: 9
- `service_saturation_packet`: 7
- `service_runtime_packet`: 7
- `pod_resource_cpu_packet`: 6
- `pod_resource_memory_packet`: 3
- `healthy_baseline_packet`: 2

## Directory Layout

```text
RAW/<EXP_ID>/                 full copy of the source run (unpurged archive)
Single_Shot/<EXP_ID>/{all,stage1,stage2}/
VLM/<EXP_ID>/{stage1,stage2}/
_mapping.json  INDEX.csv  _schema_fix_report.json  _qc_report.json  _build_log.json
```

`EXP_ID = boutique-<binding-kebab>__NN` (e.g. `boutique-ii10-double-charge__00`); `NN` is the per-binding run index by raw-run timestamp. The binding name is in the directory only — it is **not** placed inside any model-facing input. Use `_mapping.json` for the audit-only EXP_ID → binding/raw-run mapping.

## RAW

`RAW/<EXP_ID>/` is a full byte copy of the source orchestrator run, intended for audit and reconstruction:

```text
RAW/<EXP_ID>/logs/{app,namespaces,cluster,pods}/   per-pod kubectl+loki logs, kubernetes events
RAW/<EXP_ID>/metrics/*.csv                          ~59 Prometheus/Linkerd CSVs (incl. injector_fault_*)
RAW/<EXP_ID>/faults/                                fault specs (name the injected fault directly)
RAW/<EXP_ID>/{summary,metadata,verification_*}.json
```

RAW intentionally retains control-plane provenance (the `faults/` specs, `payment-injector` logs, injector metric CSVs, chaos-mesh events) that is removed from the model-facing views. **Do not train from RAW.**

## VLM Records

`VLM/<EXP_ID>/stage1/`:

```text
input_text.md   plot.png   target.json
```

Stage 1 is broad triage: five canonical RED panels + numerical/exact-value tables + auxiliary log/event summary + dynamic topology snapshot. Target = `issue_level`, `localization`, `suspected_components`, `suspected_edges`, `recommended_stage2_focus`, `rationale_brief`.

`VLM/<EXP_ID>/stage2/`:

```text
input_text.md   plot.png   target.json
```

Stage 2 is focused RCA. The component log evidence is **inlined into `input_text.md`** (one text per stage — there is no separate `input_logs.md`). The packet plot is shape-aware (transaction-integrity charge panels, service_runtime memory-working-set, edge p95/rps for the dependency packets, etc.). Target (audit contract **v1.5.5**) = `issue_level`, `root_cause_type`, `root_cause_scope`, `root_cause_target`, `resource_root_cause`, `service_incident_root_cause`, `evidence_for`, `evidence_against_alternatives`, `recommended_debugging_actions`, `recommended_recovery_actions`. There is **no** `confidence`/`confidence_reasons` field (banned by v1.5.5).

Model-facing prompts: explicit JSON-only output schema; Stage-1 summarized evidence + Stage-2 inlined packet logs; **no** mechanism vocab (chaos kinds, `payment-injector`, `fault_name`, `apply_mode`, `BOUTIQUE_FAULT`), no raw absolute paths, no control-plane identity. Organic symptoms (`duplicate_charge_detected`, `UNAVAILABLE`, latency rises, pod restarts) are retained as legitimate evidence.

## Single-Shot Records

`Single_Shot/<EXP_ID>/`:

```text
all/input.txt      all/output.txt
stage1/input.txt   stage1/plot.png   stage1/output.txt
stage2/input.txt   stage2/plot.png   stage2/output.txt
```

`input.txt` concatenates the **raw** logs and serialized metric CSVs (`=== LOG: … ===` / `=== METRIC: … ===`), purged of answer-leaks (see below) but otherwise unprocessed. `output.txt` is the stage target (stage1 = Stage-1 target; stage2/all = the v1.5.5 Stage-2 target). Per-stage scope:

- `all`: every per-pod kubectl log (minus injector) + cluster events (purged) + all metric CSVs (minus injector).
- `stage1`: the same logs + the 5 canonical metric stems.
- `stage2`: only the suspected-component pod logs + the gold packet's plotted metric stems.

Use Single_Shot for experiments testing whether an LLM can reason over less-processed operational evidence.

### Single-Shot answer-leak purge

Single_Shot inputs are purged of injection-mechanism leaks while keeping organic signal:

- Dropped entirely: the `faults/` specs, `payment-injector` component logs, `injector_fault_*` metric CSVs.
- Line-filtered from `kubernetes_events`: chaos-mesh lines (`StressChaos`/`PodChaos`/…, "Successfully apply/recover", `desiredPhase`, finalizers, chaos resource names) — while organic pod-lifecycle (`Killing`/`Started`/`Pulled`/`Created`) is **kept**.
- Metric CSV rows for `payment-injector` dropped.
- A residual mechanism-vocab regex over the assembled text (chaos kinds, `fault_name`, `apply_mode`, `BOUTIQUE_FAULT`, `envoy_apply_mode`, `FracMilli`).
- Domain symptoms are **not** purged: `duplicate_charge_detected`, `UNAVAILABLE`, `DeadlineExceeded`, latency complaints survive.

Verified: 0/252 single-shot inputs contain mechanism leaks; a BE03 pod-kill view keeps 4391 organic `Killing`/`Started` lines with 0 chaos lines.

## Incident Families

Internal application correctness/runtime faults (in-process Shape-B toggles):

- `II10_double_charge`, `II11_orphaned_order`, `II12_cart_not_emptied`, `II13_price_drift`
- `BI09_productcatalog_memory_leak`, `BI07_productcatalog_cpu_fanin`

Edge faults reproduced via the Envoy payment-injector (checkout→payment):

- `BI01_payment_induced_latency`, `BI02_payment_induced_errors`, `BI03_payment_dependency_timeout`, `BI04_payment_slow_dependency`, `BI05_payment_retry_amplification`

External Chaos Mesh faults:

- `BE01_payment_cpu_stress`, `BE02_productcatalog_mem_stress`, `BE03_payment_pod_kill` (held), `BE04_productcatalog_pod_kill` (one-shot → absorbed L0), `BE04B_productcatalog_pod_failure` (held), `BE05_payment_net_delay`

Healthy control: `B00_healthy_baseline`.

Deferred (no substrate; not in the 84): `BI10_checkout_pool_saturation`, `BI11_checkout_cpu_hotpath`.

## Canonical Signals (Stage 1)

Boutique's five canonical signals are **frontend-RED-only** (the user-facing Linkerd mesh edge):

- `latency_badness` ← `frontend_latency_p95` (histogram_quantile p95 of `response_latency_ms`, frontend inbound; baseline ~20 ms)
- `error_badness` ← `frontend_error_fraction` (failure / total of `response_total`, frontend inbound)
- `throughput_drop_badness` ← `frontend_request_rps` drop vs baseline
- `saturation_or_backlog_badness` ← in-flight concurrency / backlog estimate
- `availability_badness` ← `deployment_replicas_available` (data-plane, control-plane excluded)

This frontend-RED design is *why* transaction-integrity and masked-saturation faults are RED-invisible and need the record-driven floor + Stage-2 evidence.

## Metrics and Logs

Metrics (~59 CSVs/run) include frontend RED (p95, error fraction, useful rps, in-flight), per-edge latency/error/rps (e.g. `edge_checkout_payment_*`), callee inbound metrics, container CPU/memory working-set, `deployment_replicas_available`, `pod_restarts_total`, and the injector telemetry (RAW-only).

Logs are real pod stdout (kubectl + Loki), windowed baseline/fault/recovery. The patched services' **business-event** stream (charge/ship/cart_empty/order_done + `*_detected` symptom events) is the primary transaction-integrity evidence. Stage 1 uses summarized severity counts; Stage 2 inlines the suspected-component log snippets. Identifiers (order_id, txn_id) are anonymized to `<id-N>` tokens on model-facing surfaces; control-plane identity is redacted.

## Severity / Label Policy

- Transaction-integrity faults are floored to **L3** by a deterministic record-violation detector over the app's own business-event logs (fires only on a confirmed violation; never on the label alone). RED remains flat by design.
- `B00` healthy baseline → **L0** `no_fault`; an absorbed one-shot pod-kill (`BE04`) → **L0** `no_impact_observed` (honest contrastive).
- No L1 and no L4 are present (a real property of the frontend-RED signal model — see the implementation README's Findings).

## Schema-Fix Provenance (v1.5.5)

All 84 Stage-2 targets were brought to v1.5.5 in-place (recorded in `_schema_fix_report.json`):

- Class A (36): already compliant — no-op.
- Class B (26, non-TI): four narrative fields regenerated via the leak-safe `rephrase_stage2` transform; `confidence_reasons` dropped.
- Class C (22, transaction-integrity): regenerated via a record-bound, G6-coherence-gated path (evidence_for[0] cites the confirmed violation; RED stated normal); `confidence_reasons` dropped, both action fields added.
- 15 Chaos-Mesh `BE*` targets had a null `root_cause_type` (original build gap); filled deterministically from packet → boutique vocab (`cpu_pressure`/`memory_pressure`/`service_unavailable`/`network_path_latency`).

Original targets are backed up as `*.pre_v155_bak` / `*.pre_rctfill_bak` in the source dp dirs.

## Validation and Audits

Deterministic QC (`qc_boutique_vlm.py`) over all 84 VLM dps — **84/84 PASS**:

- no `confidence` key; both action fields present; full v1.5.5 required set
- 0 mechanism-leak hits on any input/target surface; `payment-injector` absent everywhere
- plot↔text/panel parity; carried Stage-1-result block consistent with the Stage-1 target
- every transaction-integrity target passes the G6 record-coherence gate

An AI agent additionally eyeballed 18 representatives (all families, both stages) for plot↔text match, sufficiency-to-triage, no-answer-disclosure, and same-edge separability.

See: `_qc_report.json`, `_schema_fix_report.json`, `_build_log.json`, and the questionnaire at `/home/yw2399/Data_Backbone_Apps/docs/Paper/Q_Sec_3/Boutique/`.

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
RAW/<EXP_ID>/   (+ _mapping.json for EXP_ID -> binding/raw-run)
```

Do not expose the binding name or RAW control-plane provenance to the model during normal training unless the experiment explicitly studies leakage. The dataset is a **candidate** pending human freeze review.

# AI_3 Data README

## Dataset Purpose

AI_3 is a VLM root-cause-analysis dataset for a single-GPU LLM summarization service. It provides multimodal incident records with:

- Stage 1 global triage prompts;
- Stage 2 packet-specific RCA prompts;
- raw source artifacts for audit/replay;
- single-shot raw-log/raw-metric prompts for alternate experiments.

The dataset is designed to evaluate whether a model can infer RCA from operational evidence, not from fault names or injection logs.

## Dataset Location

Final package:

```text
/lp-dev/yw2399/Dataset/AI_3
```

Repo symlink:

```text
/home/yw2399/Demo_Apps/Dataset/ai_3
```

## Package Layout

```text
Raw/<BUG_ID>/
VLM/<BUG_ID>/stage1/
VLM/<BUG_ID>/stage2/
Single_Shot/<BUG_ID>/all/
Single_Shot/<BUG_ID>/stage1/
Single_Shot/<BUG_ID>/stage2/
MANIFESTS/
LEAKAGE_REPORTS/
CHECKSUMS/
README.md
```

### Raw

`Raw/<BUG_ID>/` contains datapoint-local provenance:

- raw source artifact subtree;
- normalized evidence;
- labels and datapoint summaries;
- Stage 1/Stage 2 candidate files;
- workflow/campaign reports;
- metric query payloads/CSVs;
- Kubernetes events/status/log snapshots;
- source manifests and validation reports.

Raw artifacts may include provenance terms that reveal how the incident was created. They are not model-facing prompts.

### VLM

`VLM/<BUG_ID>/stage1/` contains:

```text
input_text.md
plot.png
target.json
metadata.json
stage1_record.json
```

Stage 1 asks the model to produce:

```text
issue_level
localization
suspected_components
suspected_edges
recommended_stage2_focus
rationale_brief
```

`VLM/<BUG_ID>/stage2/` contains:

```text
input_text.md
plot.png
target.json
metadata.json
stage2_record.json
```

Stage 2 asks the model to produce:

```text
issue_level
root_cause_type
root_cause_scope
root_cause_target
resource_root_cause
service_incident_root_cause
evidence_for
evidence_against_alternatives
recommended_debugging_actions
recommended_recovery_actions
```

### Single_Shot

`Single_Shot/<BUG_ID>/` provides raw-log/raw-metric prompts for non-VLM or single-shot experiments:

```text
all/input.txt
all/output.txt
stage1/input.txt
stage1/plot.png
stage1/output.txt
stage2/input.txt
stage2/plot.png
stage2/output.txt
```

Inputs are purged for direct-answer leakage, including Chaos Mesh control-plane logs, fault-control endpoints, raw fault mode names, binding IDs, split labels, absolute local paths, and raw GPU UUIDs.

## Counts

Source of truth:

```text
MANIFESTS/final_dataset_manifest.json
```

Final counts:

| Item | Count |
|---|---:|
| Incident families | 16 |
| Datapoints | 80 |
| Stage 1 VLM records | 80 |
| Stage 2 VLM records | 80 |
| VLM stage records total | 160 |
| Main RCA records | 57 |
| Framework diagnostics | 18 |
| Negative/control records | 5 |

Issue levels:

| Issue level | Count | Meaning |
|---:|---:|---|
| 0 | 5 | healthy/control |
| 1 | 13 | weak impact/resource anomaly/diagnostic |
| 2 | 27 | component degradation or limited service effect |
| 3 | 15 | service degradation |
| 4 | 20 | service failure/no-success or severe impact |

## Runs Included

The package merges five runs, each with 16 datapoints:

```text
app3_patch14_20260531T050000Z
app3_aug01_20260626T034240Z
app3_aug02_20260626T052619Z
app3_aug03_20260626T071036Z
app3_aug04_20260626T085453Z
```

BUG_IDs preserve provenance:

```text
app3_patch14_20260531T050000Z__app3_dp_0001
app3_aug01_20260626T034240Z__app3_dp_0001
...
```

## Incident Families

Each family has five executions.

| Family | Stage 1 localization | Stage 2 packet | Notes |
|---|---|---|---|
| `baseline_no_fault` | `insufficient_evidence` | `insufficient_evidence_packet` | healthy control |
| `vllm_pod_restart` | `pod_availability` | `pod_availability_packet` | backend pod availability |
| `api_pod_restart` | `pod_availability` | `pod_availability_packet` | API pod availability |
| `api_to_vllm_network_latency` | `network_edge` | `network_edge_packet` | API-backend dependency latency |
| `api_cpu_pressure` | `cpu` | `pod_resource_cpu_packet` | mixed main/diagnostic depending on impact |
| `vllm_cpu_pressure` | `cpu` | `pod_resource_cpu_packet` | backend CPU pressure |
| `api_memory_pressure` | `memory` | `pod_resource_memory_packet` | diagnostic memory anomaly |
| `vllm_memory_pressure` | `memory` | `pod_resource_memory_packet` | diagnostic memory anomaly |
| `long_context_prefill_overload` | `pipeline_stage` | `model_prefill_overload_packet` | LLM prefill |
| `bad_gateway_batch_wait` | `backlog` | `request_queue_backlog_packet` | API queue/backlog |
| `low_backend_timeout` | `pipeline_timeout` | `timeout_policy_packet` | API timeout policy |
| `slow_tokenization_cpu` | `cpu` | `tokenizer_cpu_packet` | API tokenizer CPU |
| `decode_heavy_generation` | `pipeline_stage` | `decode_latency_packet` | LLM decode/completion |
| `premature_readiness` | `pod_availability` | `readiness_warmup_packet` | readiness/warmup policy |
| `gateway_queue_delay` | `backlog` | `request_queue_backlog_packet` | API gateway queue delay |
| `kv_cache_pressure_oom` | `gpu` | `kv_cache_gpu_memory_packet` | high-severity diagnostic with missing GPU causal attribution |

## Evidence Collected

### Workload Evidence

- per-request JSONL;
- request start/finish timestamps;
- success/error/timeout;
- useful summary flag;
- latency summaries;
- useful throughput per second.

### Application Metrics

- request duration;
- backend request duration;
- errors and timeouts;
- inflight requests;
- queue depth and queue wait;
- tokenization duration;
- prefill delay;
- decode delay;
- readiness/warmup mismatch.

### Kubernetes And Resource Metrics

- pod readiness;
- pod restarts;
- deployment availability;
- container CPU utilization;
- CPU throttling;
- memory working set/utilization;
- pod phase/status;
- Kubernetes events.

### Backend/GPU Metrics

- vLLM request/latency metrics where available;
- running/waiting requests;
- vLLM GPU/KV-cache usage;
- GPU memory/utilization metrics where available.

## Model-Facing Redaction Policy

The model-facing `VLM/` and `Single_Shot/` inputs should not contain direct-answer leakage. The final leakage scan passed and removes or excludes:

- Chaos Mesh and specific chaos resource logs;
- `/faults/enable`, `/faults/status`, and fault-control traces;
- raw fault names and internal mode IDs;
- binding IDs;
- split labels;
- raw GPU UUIDs;
- absolute local paths;
- answer-bearing scenario names.

Symptom evidence is preserved. For example, a prompt may say that requests timed out, pod readiness dropped, or CPU utilization rose; it should not say that a specific injected fault was enabled.

## Quality Reports

Important report files:

```text
MANIFESTS/final_dataset_manifest.json
MANIFESTS/bug_id_mapping.csv
MANIFESTS/target_semantics_audit.json
LEAKAGE_REPORTS/final_leakage_report.json
LEAKAGE_REPORTS/vlm_quality_review.json
LEAKAGE_REPORTS/vlm_prompt_contract_repair_report.json
LEAKAGE_REPORTS/target_semantics_repair_report.json
CHECKSUMS/PACKAGE_SHA256SUMS.txt
CHECKSUMS/checksum_report.json
```

Known final results:

- Target semantics audit passed with no violations.
- Final leakage report passed.
- VLM quality review passed.
- `checksum_report.json` records 71,766 files.

## Recommended Usage

For VLM stage-wise training/evaluation:

```text
VLM/<BUG_ID>/stage1/input_text.md + stage1/plot.png -> stage1/target.json
VLM/<BUG_ID>/stage2/input_text.md + stage2/plot.png -> stage2/target.json
```

For raw single-shot experiments:

```text
Single_Shot/<BUG_ID>/all/input.txt -> all/output.txt
Single_Shot/<BUG_ID>/stage1/input.txt + stage1/plot.png -> stage1/output.txt
Single_Shot/<BUG_ID>/stage2/input.txt + stage2/plot.png -> stage2/output.txt
```

For reproducibility/audit:

```text
Raw/<BUG_ID>/
MANIFESTS/
LEAKAGE_REPORTS/
CHECKSUMS/
```

## Limitations

- Controlled faults are synthetic/chaos-induced rather than organic production outages.
- The topology is one API, one backend, and one GPU.
- There are five executions per family.
- Stage 2 packet names reveal broad debugging area, which can be a shortcut risk.
- Some diagnostics intentionally contain visible symptoms but are not main RCA because causal proof is incomplete.

## Paper Support

Section 3 questionnaire support:

```text
/home/yw2399/Demo_Apps/docs/Paper/Q_Sec_3/AI_3/AI_3_report.md
/home/yw2399/Demo_Apps/docs/Paper/Q_Sec_3/AI_3/AI_3_stats.json
```

Implementation guide:

```text
/lp-dev/yw2399/VISTA_Land/AI_3/README.data
```

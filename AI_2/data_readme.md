# AI_2 Data README

## Dataset Location

Primary dataset:

```text
/lp-dev/yw2399/Dataset/AI_2/
```

Repository symlink:

```text
/home/yw2399/Demo_Apps/Dataset/ai_2 -> /lp-dev/yw2399/Dataset/AI_2
```

Important metadata:

```text
/lp-dev/yw2399/Dataset/AI_2/manifest.json
/lp-dev/yw2399/Dataset/AI_2/provenance/datapoint_mapping.csv
/lp-dev/yw2399/Dataset/AI_2/provenance/datapoint_mapping.json
/lp-dev/yw2399/Dataset/AI_2/Audits/merge_audit.json
/lp-dev/yw2399/Dataset/AI_2/Reports/merge_validation_report.md
```

## Dataset Purpose

AI_2 is a multimodal RCA dataset for a GPU-backed support-ticket triage pipeline. It is designed to train and evaluate models that read logs, metrics, plots, and structured context, then infer root cause and affected component. The dataset emphasizes software-engineering failures in AI pipelines: malformed intermediate output, validator failure surfaces, retry storms, queues, model backend latency, Kubernetes availability, CPU/memory pressure, dependency edges, and GPU resource contention.

## Counts

Final merged package:

- Total datapoints: 85
- Main RCA datapoints: 80
- Negative/control datapoints: 5
- Incident families: 17
- Stage 1 VLM records: 85
- Stage 2 VLM records: 85
- Total stage records: 170
- Frozen seed datapoints: 17
- Augmentation datapoints: 68

Issue-level distribution:

- L1 `RESOURCE_ANOMALY`: 5
- L2 `COMPONENT_DEGRADATION`: 10
- L3 `SERVICE_DEGRADATION`: 50
- L4 `SERVICE_FAILURE`: 20

Packet distribution:

- `gpu_resource_packet`: 10
- `interstage_queue_backlog_packet`: 5
- `malformed_intermediate_output_packet`: 20
- `model_backend_timeout_packet`: 5
- `network_edge_packet`: 5
- `pipeline_stage_latency_packet`: 5
- `pod_availability_packet`: 15
- `pod_resource_cpu_packet`: 10
- `pod_resource_memory_packet`: 5
- `retry_storm_packet`: 5

## Directory Layout

```text
RAW/<EXP_ID>/
Single_Shot/<EXP_ID>/
VLM/<EXP_ID>/
Dataset/<EXP_ID>/
provenance/
Audits/
Reports/
```

`EXP_ID` is a neutral datapoint identifier such as `ai2_dp_0001`. The true incident family is intentionally not exposed in VLM paths. Use `provenance/datapoint_mapping.csv` for audit-only mapping from neutral IDs to fault families.

## RAW

`RAW/<EXP_ID>/` contains the copied source evidence for each datapoint:

```text
RAW/<EXP_ID>/source_artifact/
RAW/<EXP_ID>/source_baseline/
RAW/<EXP_ID>/provenance.json
```

The raw source artifact may include workflow reports, workload records, metric/log coverage summaries, app-fault status, campaign artifacts, Kubernetes evidence, GPU telemetry, Prometheus-derived metrics, and logs. RAW is intended for audit and reconstruction. It may preserve control-plane provenance that is intentionally removed from VLM-facing prompts.

## VLM Records

`VLM/<EXP_ID>/stage1/` contains:

```text
input_text.md
plot.png
target.json
```

Stage 1 is broad triage. The input asks for issue level, broad localization, suspected components/edges, recommended Stage 2 packet, and evidence-grounded rationale.

`VLM/<EXP_ID>/stage2/` contains:

```text
input_text.md
plot.png
target.json
```

Stage 2 is focused RCA. The input asks for root-cause type, scope, target, resource/service root-cause fields, supporting evidence, alternatives, debugging actions, and recovery actions.

Training-visible prompts were repaired to:

- state explicit JSON-only output schemas;
- include Stage 1 summarized log evidence;
- include Stage 2 packet-relevant log snippets;
- remove answer-bearing artifact IDs and exact fault IDs;
- avoid raw ticket bodies, absolute local paths, Chaos/control wording, and secrets.

## Single-Shot Records

`Single_Shot/<EXP_ID>/` contains raw-evidence variants for LLM-only tests:

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

The inputs concatenate raw logs and serialized metric readings. They are not the same as the curated VLM prompts. Use these for experiments that intentionally test whether an LLM can reason over less-processed operational evidence.

## Prepared Datapoints

`Dataset/<EXP_ID>/prepared_datapoint/` stores the structured material used to build VLM and single-shot views. It includes Stage 1 summaries, Stage 2 packet summaries, normalized events, targets, checks, and metadata.

## Incident Families

Internal application/model-contract faults:

- `classifier_invalid_category`
- `summarizer_malformed_json`
- `summarizer_slow_decode`
- `validator_retry_storm`
- `batch_queue_saturation`
- `model_backend_timeout`
- `priority_invalid_team`
- `classifier_confidence_collapse`

External Kubernetes/Chaos/Orchestrator faults:

- `classifier_to_summarizer_network_latency`
- `classifier_pod_restart`
- `validator_pod_unavailable`
- `summarizer_reasoner_pod_restart`
- `summarizer_cpu_pressure`
- `summarizer_memory_pressure`
- `noncritical_stage_cpu_pressure_control`

Same-GPU GPU resource faults:

- `summarizer_gpu_compute_saturation_same_gpu`
- `summarizer_gpu_memory_pressure_same_gpu`

Excluded/no-effect GPU helper diagnostics are not part of the main trainable dataset:

- `summarizer_gpu_compute_saturation`
- `summarizer_gpu_memory_pressure`
- `summarizer_gpu_throttling_proxy`

## Metrics and Logs

Metrics include:

- request latency;
- useful throughput;
- stage latency;
- model backend latency;
- invalid output counters;
- validator rejection counters;
- retry counters;
- queue depth/wait/inflight;
- Kubernetes readiness/replica/restart signals;
- CPU utilization and throttling where available;
- memory working set/utilization;
- dependency edge latency;
- GPU utilization and memory telemetry for same-GPU resource datapoints.

Logs include sanitized application/stage events such as request received/completed, stage start/end, model output invalid, validator rejected, retry activity, backend timeout, queue/backlog symptoms, readiness symptoms, and resource/backend context. Stage 1 uses summarized counts; Stage 2 uses packet-relevant snippets.

## Useful-Throughput Policy

Useful throughput means valid tickets that complete with a usable final triage output. It excludes:

- health/admin/probe requests;
- raw attempts;
- retry attempts;
- malformed-control tickets;
- invalid final outputs;
- validator-rejected outputs.

This matters because retry storms and invalid outputs can make raw request counters look healthy while user-facing useful outputs collapse.

## Split Policy

- `main_rca_train`: 80 datapoints. These are intended for RCA training/evaluation.
- `negative_or_control`: 5 datapoints. These are resource-anomaly controls and should be used for contrastive evaluation or audit, not as main RCA service-failure examples.

The negative/control family is `noncritical_stage_cpu_pressure_control`, labeled L1 `RESOURCE_ANOMALY`.

## Augmentation

The package combines:

- 17 frozen seed datapoints from the accepted GPU-backed App #2 release;
- 4 additional GPU-backed augmentation rounds, each repeating the same 17 accepted family templates with fresh evidence.

The final merge into neutral IDs is recorded in:

```text
provenance/datapoint_mapping.csv
manifest.json
Reports/merge_validation_report.json
```

## Ablation Copies

Related ablation packages live outside the primary AI_2 root:

```text
/lp-dev/yw2399/Dataset/AI_2_No_Image
/lp-dev/yw2399/Dataset/AI_2_No_Metric
/lp-dev/yw2399/Dataset/AI_2_Patch
```

The no-image and no-metric copies have prompt text adjusted so they do not ask the model to use missing inputs.

## Validation and Audits

The merged package passed:

- manifest checks;
- missing file checks;
- VLM reference checks;
- plot checks after time-series repair;
- leakage/path/raw-ticket scans;
- target/evidence consistency checks;
- visual/evidence review;
- prompt schema checks;
- log-evidence prompt checks.

See:

```text
Audits/merge_audit.json
Reports/merge_validation_report.md
Reports/merge_validation_report.json
```

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
RAW/<EXP_ID>/
Dataset/<EXP_ID>/prepared_datapoint/
provenance/
```

Do not expose provenance fault IDs to the model during normal training unless the experiment explicitly studies leakage.

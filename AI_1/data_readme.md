# AI_1 Dataset README

AI_1 is the RAG application dataset for VISTA. It contains raw experiment records, single-shot LLM inputs, VLM-facing two-stage records, aggregate training JSONL, Qwen-VL SFT export files, provenance, reports, and paper-facing summaries.

## Dataset Location

- Canonical dataset root: `/lp-dev/yw2399/Dataset/AI_1`
- Repository symlink: `/home/yw2399/Demo_Apps/Dataset/ai_1`
- Published implementation notes: `/lp-dev/yw2399/VISTA_Land/AI_1/README.data`
- Published code: `/lp-dev/yw2399/VISTA_Publish/code/AI_1`

## Top-Level Layout

- `RAW/<datapoint_id>/`: raw collected artifacts for each execution.
- `Single_Shot/<datapoint_id>/{all,stage1,stage2}/`: unprocessed log/metric concatenations for single-shot diagnosis tests.
- `VLM/<datapoint_id>/{stage1,stage2}/`: per-datapoint VLM-facing inputs, plots, and outputs.
- `VLM_RECORDS/`: aggregate JSONL for trainable main RCA datapoints.
- `VLM_sft_qwen/`: Qwen-VL-style SFT export with train/val/test split.
- `reports/`: merge, leakage, layout, and review reports.
- `docs/`: paper-facing and engineering audit reports.
- `provenance/`: original frozen release, augmentation provenance, and failed/unusable attempts.
- `source_index.json`: authoritative datapoint index with split, trainability, root cause, packet type, and source paths.
- `SHA256SUMS.txt`: checksums for key dataset files.

## Dataset Counts

- Total datapoint packages: `82`
- Trainable main RCA datapoints: `75`
- Negative/control datapoints: `6`
- Framework diagnostic datapoints: `1`
- Stage 1 aggregate VLM records: `75`
- Stage 2 aggregate VLM records: `75`
- Combined aggregate VLM records: `150`
- Single-shot input files: `246`
- VLM input markdown files: `164`

Trainable issue-level distribution:

- `RESOURCE_ANOMALY`: `28`
- `COMPONENT_DEGRADATION`: `29`
- `SERVICE_DEGRADATION`: `18`
- `SERVICE_FAILURE`: `0`

Trainable packet distribution:

- `pod_resource_memory_packet`: `14`
- `compound_incident_packet`: `11`
- `rag_retrieval_packet`: `9`
- `network_edge_packet`: `9`
- `rag_retrieval_quality_packet`: `6`
- `pod_resource_cpu_packet`: `6`
- `pod_availability_packet`: `6`
- `rag_context_construction_packet`: `3`
- `rag_schema_contract_packet`: `3`
- `rag_generation_packet`: `3`
- `gpu_resource_packet`: `3`
- `storage_path_packet`: `2`

## What Is In RAW

Each `RAW/<datapoint_id>/` directory may contain:

- application logs,
- workload request logs,
- pod and namespace logs from kubectl/Loki collection,
- Kubernetes events,
- known-good pre/post gate logs,
- Prometheus-style metric CSVs,
- network/service probe outputs,
- collection manifests,
- state snapshots and metadata.

These raw files are intended for reproducibility and for single-shot experiments. They are not the cleaned VLM training prompt by themselves.

## What Is In Single_Shot

Each `Single_Shot/<datapoint_id>/` directory has:

- `all/input.txt`: raw-ish concatenation across the available logs and serialized metrics for the whole execution.
- `all/output.txt`: final root-cause answer.
- `stage1/input.txt`: raw logs and metrics aligned with Stage 1 evidence.
- `stage1/plot.png`: Stage 1 plot.
- `stage1/output.txt`: Stage 1 triage target.
- `stage2/input.txt`: raw logs and metrics aligned with Stage 2 evidence.
- `stage2/plot.png`: Stage 2 packet plot.
- `stage2/output.txt`: Stage 2 RCA target.

These inputs are for testing whether a model can diagnose from less-processed evidence. They were purged for direct fault-injection leakage, but they intentionally preserve operational symptoms such as timeouts, readiness failures, HTTP errors, retrieval/generation/schema symptoms, pod lifecycle events, and raw metrics.

## What Is In VLM

Each `VLM/<datapoint_id>/` directory has:

- `stage1/input_text.md`
- `stage1/plot.png`
- `stage1/output.txt`
- `stage2/input_text.md`
- `stage2/plot.png`
- `stage2/output.txt`
- `dataset_manifest.json`

Stage 1 is a broad triage view. Stage 2 is generated from the Stage 1 output and focuses on one packet type. The VLM-facing text and plots are deterministic renderings of telemetry and numeric evidence.

## Aggregate Training Records

Use these for canonical trainable records:

- `/lp-dev/yw2399/Dataset/AI_1/VLM_RECORDS/stage1_vlm_train.jsonl`
- `/lp-dev/yw2399/Dataset/AI_1/VLM_RECORDS/stage2_vlm_train.jsonl`
- `/lp-dev/yw2399/Dataset/AI_1/VLM_RECORDS/combined_vlm_train.jsonl`

Only trainable `main_rca_train` datapoints are included in these aggregate JSONL files. Non-training controls and framework diagnostics remain available as per-datapoint folders but are excluded from aggregate VLM training records.

## Qwen-VL SFT Export

Use `/lp-dev/yw2399/Dataset/AI_1/VLM_sft_qwen` for a Qwen-VL-style supervised fine-tuning export.

Important files:

- `README.md`
- `sft_manifest.json`
- `metadata/dataset_summary.json`
- `metadata/validation_report.json`
- `jsonl/stage1_train.jsonl`
- `jsonl/stage1_val.jsonl`
- `jsonl/stage1_test.jsonl`
- `jsonl/stage2_train.jsonl`
- `jsonl/stage2_val.jsonl`
- `jsonl/stage2_test.jsonl`

The SFT export uses grouped splitting to avoid leaking repeated executions or duplicate target bundles across train/validation/test.

## Fault Families

AI_1 includes external/system faults and internal RAG faults.

External/system examples:

- CPU saturation,
- memory pressure,
- GPU pressure,
- pod kill/restart/availability disruption,
- network latency on `api->qdrant` and `api->vllm`,
- vector-store storage/query slowdown,
- compound incidents combining resource and dependency effects.

Internal RAG examples:

- empty retrieval,
- retriever timeout,
- wrong top-k or poor retrieval quality,
- context too long after retrieval,
- schema mismatch between retrieval and context construction,
- vector database latency spike,
- stale or corrupted index,
- generator waiting on retriever.

## Metrics Collected

Representative metrics include:

- request latency,
- error rate,
- useful successful `/query` throughput,
- request rate as diagnostic-only attempt rate,
- CPU utilization and throttling,
- memory working set and utilization,
- GPU utilization and memory,
- pod readiness, replicas available, restarts, and phase,
- retrieval latency and retrieved-document count,
- empty retrieval and retrieval timeout counters,
- generation latency,
- context token pressure,
- schema mismatch counter,
- vector-store query latency and slow-query counters,
- network probe latency.

## Logs And Events Collected

Representative log/event evidence includes:

- API application logs,
- Qdrant logs,
- vLLM/generation logs,
- workload request logs,
- Kubernetes pod and namespace logs,
- Kubernetes events,
- readiness/unhealthy/killing/restart events,
- service-probe evidence.

Direct Chaos Mesh/operator/fault-resource leakage was purged from model-facing files. Legitimate symptoms were preserved.

## Validation Status

Final merged dataset validation:

- Merge report: `PASS`
- Leakage review: `PASS`
- Remaining direct-answer leakage findings: `0`
- Layout validation: `PASS`
- Empty directories remaining: `0`
- Trainable datapoint review: `75 PASS`
- Non-training/control/diagnostic review: `7 WARN`, expected and non-blocking.

Important report files:

- `/lp-dev/yw2399/Dataset/AI_1/reports/merge_report.md`
- `/lp-dev/yw2399/Dataset/AI_1/reports/leakage_review.md`
- `/lp-dev/yw2399/Dataset/AI_1/reports/final_layout_validation.md`
- `/lp-dev/yw2399/Dataset/AI_1/reports/datapoint_review_cards.md`
- `/lp-dev/yw2399/Dataset/AI_1/docs/final_quality_coverage_review_final.md`
- `/lp-dev/yw2399/Dataset/AI_1/docs/throughput_signal_audit.md`
- `/lp-dev/yw2399/Dataset/AI_1/docs/value_semantics_audit.md`
- `/lp-dev/yw2399/Dataset/AI_1/docs/profile_pattern_coverage_audit_after_tuning.md`

## Known Limitations

- No trainable hard `SERVICE_FAILURE` examples are present.
- GPU coverage is narrower than CPU, memory, dependency-network, retrieval, and schema coverage.
- Original frozen C04 is retained only as a framework diagnostic because its pre-fault workload was baseline-contaminated.
- Augmented E08 failed CPU verification and is retained only under provenance failed attempts.
- The frozen VLM product materializes the selected/gold Stage 2 packet, not all candidate packets for every datapoint.

## Recommended Use

- Use `VLM_RECORDS/` for canonical Stage 1 and Stage 2 training/evaluation.
- Use `VLM_sft_qwen/` for Qwen-VL SFT training.
- Use `Single_Shot/` for unprocessed-log LLM experiments.
- Use `RAW/` and `provenance/` for reproducibility, debugging, and paper audits.

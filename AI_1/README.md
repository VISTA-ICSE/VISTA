# AI_1 Implementation README

AI_1 is the VISTA RAG application: a Kubernetes-deployed retrieval-augmented generation system used to build root-cause-analysis data for multimodal models. The application deliberately combines ordinary distributed-system behavior with AI-pipeline-specific failures. It is useful as an RCA benchmark because it forces a model to reason across service topology, resource pressure, dependency latency, vector retrieval quality, context construction, schema contracts, and model generation.

For the dataset itself, see `/lp-dev/yw2399/VISTA_Land/AI_1/data_readme.md`.

## Big Picture

The project asks whether a VLM can diagnose incidents from realistic operational evidence: time-series plots, numeric summaries, logs, events, and scoped packet views. The RAG application was chosen because it is both familiar to software engineers and distinctively AI-native. It has normal microservice failure modes, such as CPU saturation, memory pressure, pod replacement, and network latency. It also has RAG-specific semantic failures, such as empty retrieval, bad top-k retrieval, context overrun, schema mismatch, stale indexes, and generator waiting on retrieval.

That mixture is valuable for software engineering research. It moves beyond generic AIOps dashboards into AI application pipelines where correctness can fail even when the service is technically up.

## Application Architecture

The deployed application has three primary components:

- `api`: FastAPI service that handles `/query`, orchestrates retrieval, builds prompts, calls the generator, exposes metrics, and records structured RAG-stage events.
- `qdrant`: vector database used for retrieval over SQuAD-derived document chunks.
- `vllm`: OpenAI-compatible model-serving backend used for generation when GPU-backed mode is enabled.

The two important dependency edges are:

- `api->qdrant`: retrieval and vector-store query path.
- `api->vllm`: generation path.

The normal request path is:

1. A workload client calls API `/query`.
2. The API embeds or looks up the query representation.
3. The API queries Qdrant for top-k chunks.
4. The API constructs a context/prompt.
5. The API calls vLLM for generation.
6. The API returns a final answer.

## Technology Choices

- Kubernetes is used to expose realistic pod, deployment, readiness, event, and resource behavior.
- Qdrant was selected because vector-store behavior is central to RAG systems and it has operationally meaningful memory, CPU, and query-latency failure modes.
- vLLM was selected because it represents production-style model serving and provides GPU-sensitive behavior.
- Prometheus-style metrics, kubectl/Loki logs, Kubernetes events, workload traces, and service probes were collected because these are the observability streams engineers actually use.
- Chaos Mesh was used for external/system perturbations because it gives repeatable Kubernetes fault injection.
- Deterministic Stage 1 and Stage 2 preparers were used so the VLM-facing records are reproducible and not dependent on LLM-generated labels.

## Implementation Details

The published code snapshot is at `/lp-dev/yw2399/VISTA_Publish/code/AI_1`.

Important subdirectories:

- `rag_app/`: RAG service implementation, app config, RAG pipeline, Qdrant client, vLLM client, observability code, tests, and Kubernetes/Helm manifests.
- `orchestrator_ai1/`: dataset generation and validation logic copied from the Orchestrator for AI_1.
- `orchestrator_ai1/orchestrator/app_profiles/rag-demo-v2.yaml`: app profile defining components, topology, canonical signals, packet mappings, log parsers, and metric semantics.
- `orchestrator_ai1/orchestrator/dataset/v2.py`: deterministic Stage 1 and Stage 2 datapoint preparation.
- `orchestrator_ai1/orchestrator/dataset/log_normalizer_v2.py`: normalized log/event extraction and severity handling.
- `orchestrator_ai1/orchestrator/dataset/rag_v2_datapoint_review.py`: automated datapoint review cards.
- `orchestrator_ai1/scripts/create_app1_qwen_vl_sft.py`: Qwen-VL SFT export.

## Stage 1 And Stage 2 Design

Stage 1 is broad triage. It gives the model:

- topology snapshot,
- five canonical signal panels,
- canonical numeric table,
- mechanical metric warnings,
- selected log/event summaries,
- task output schema for issue level, localization, suspected components/edges, and recommended Stage 2 packet.

The five canonical Stage 1 panels are:

- latency badness,
- error badness,
- throughput-drop badness,
- saturation/backlog badness,
- availability badness.

Badness is normalized so `0` means normal or pre-fault, positive means worse, and negative means improved.

Stage 2 is packet-specific debugging. It is generated from Stage 1 output, not from private target metadata. Packet types include pod CPU, pod memory, pod availability, network edge, storage path, GPU, compound incident, retrieval, retrieval quality, context construction, schema contract, generation, and insufficient evidence. Stage 2 includes a packet-specific plot, metric table, numeric probe summaries, scoped logs/events, and the target RCA schema.

## Interesting Findings

Several findings shaped the final implementation:

- Throughput must mean useful successful `/query` throughput, not raw request attempts or health/admin endpoints. The throughput signal audit confirmed AI_1 uses successful workload query records and lower-is-worse semantics.
- Boolean health and availability metrics must be inverted into badness. A ready value of `1` means availability badness `0`; ready value `0` means badness `1`.
- Pod replacement can create pod-identity discontinuity. Availability and resource signals need component/deployment-level aggregation where possible, while old/new pod names remain secondary evidence.
- INFO logs must not be promoted to warning merely because the message contains text such as `UserWarning`. Severity must come from the parsed log level first.
- Compound packet mapping must be constituent-aware. C01 should not get irrelevant GPU/vLLM panels; C03 should keep GPU/vLLM evidence because its constituent involves vLLM/GPU.
- Stage 2 plot axes should be time-series relative to fault start, not three categorical bars for baseline/fault/recovery.
- Path-only evidence such as `artifact_ref=...csv` is not good VLM evidence. The final records render numeric probe summaries instead.

## Validation And Reports

The final merged AI_1 dataset reports live under `/lp-dev/yw2399/Dataset/AI_1/reports` and `/lp-dev/yw2399/Dataset/AI_1/docs`.

High-signal reports:

- `reports/merge_report.md`: merge status and counts.
- `reports/leakage_review.md`: direct-answer leakage scan.
- `reports/final_layout_validation.md`: layout and file-count validation.
- `reports/datapoint_review_cards.md`: per-datapoint review cards.
- `docs/final_quality_coverage_review_final.md`: final frozen-quality review.
- `docs/throughput_signal_audit.md`: useful-throughput validation.
- `docs/value_semantics_audit.md`: unit/value-semantics validation.
- `docs/profile_pattern_coverage_audit_after_tuning.md`: profile-pattern coverage audit.

## Known Limitations

- AI_1 has no trainable hard `SERVICE_FAILURE` examples.
- GPU coverage is useful but narrower than CPU/memory/network/RAG-stage coverage.
- Controls and diagnostics are retained for audit but excluded from aggregate trainable JSONL.
- Some full candidate-packet statistics are not materialized because the frozen VLM corpus stores only the selected/gold Stage 2 packet.

## Reproduction Pointers

Use `/lp-dev/yw2399/Dataset/AI_1/VLM_RECORDS` for canonical trainable Stage 1/Stage 2 JSONL. Use `/lp-dev/yw2399/Dataset/AI_1/VLM_sft_qwen` for Qwen-VL SFT formatting. Use `/lp-dev/yw2399/Dataset/AI_1/RAW` for raw observability and source artifacts.

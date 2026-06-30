# AI_3 Implementation README

## Overview

AI_3 is the VISTA application and dataset package for a single-GPU LLM summarization serving system. The application represents a production-style inference service:

```text
workload_driver -> summarizer_api -> vllm_engine -> assigned GPU
```

The implementation combines a FastAPI summarization API, a vLLM OpenAI-compatible backend, Kubernetes/Helm deployment, Prometheus-style metrics, workload request logs, Kubernetes events, and controlled external/internal faults. The dataset asks a VLM to perform root-cause analysis in two stages:

- Stage 1: global triage from a five-panel plot and concise system-wide evidence.
- Stage 2: packet-specific RCA from a focused plot and packet evidence.

The data package is documented separately in:

```text
/lp-dev/yw2399/VISTA_Land/AI_3/data_readme.md
```

## Why This Application Is Interesting

LLM serving systems are now common software systems, but they do not fail exactly like older web services. They still have classical incidents: pod restarts, dependency latency, CPU pressure, memory pressure, request errors, timeouts, and queueing. They also have LLM-specific issues: long-context prefill, decode-heavy generation, tokenizer CPU, readiness before backend warmup, vLLM cache pressure, and GPU attribution questions.

AI_3 is interesting because it forces RCA across these layers. A user-visible failure may originate from:

- Kubernetes availability;
- the API-to-backend network edge;
- API request admission or queueing;
- timeout policy;
- tokenization/preprocessing;
- model prefill or decode stage;
- backend CPU or memory;
- GPU/KV-cache pressure with incomplete causal proof.

This makes the application useful for software engineering research: it tests whether a model can reason from heterogeneous operational evidence rather than memorizing a fault ID.

## Technology Selection

Runtime choices:

```text
API service: FastAPI
Backend: vLLM OpenAI-compatible server
Model: Qwen/Qwen2.5-7B-Instruct
Served model name: app3-qwen25-7b-summarizer
GPU: one assigned A100-class GPU
Tensor parallel size: 1
Deployment: Kubernetes + Helm
Observability: Prometheus-style metrics, Kubernetes events/status, workload JSONL, app/backend logs, GPU metrics where available
External fault mechanism: Chaos Mesh / approved Orchestrator workflows
Internal fault mechanism: application fault API through approved Orchestrator workflows
```

The one-GPU design is deliberate. It avoids turning GPU-related diagnostics into multi-GPU placement or tensor-parallel coordination problems, while still preserving real model-serving behavior.

## Implementation Components

Publication code bundle:

```text
/lp-dev/yw2399/VISTA_Publish/code/AI_3/
```

Important subdirectories:

- `llm_summarization_serving_app/`: FastAPI service, vLLM client, metrics, schemas, workload generator, fault state, Helm chart, tests.
- `orchestrator_ai3/orchestrator/baseline/`: baseline workflow.
- `orchestrator_ai3/orchestrator/external/`: external/Chaos-style campaign workflow.
- `orchestrator_ai3/orchestrator/internal/`: internal app-fault campaign workflow.
- `orchestrator_ai3/orchestrator/dataset/`: normalizer, labels, Stage 1/Stage 2 builders, candidate builder, verifier, audit, final review, action templates.
- `orchestrator_ai3/orchestrator/llm_summarization_timeseries.py`: raw time-series collection/preservation logic.
- `orchestrator_ai3/experiments/`: app profile, metric pack, baseline/campaign configs, fault catalog.
- `orchestrator_ai3/scripts/`: build, validate, campaign, audit, freeze, export, and review entry points.
- `paper_support/`: Section 3 questionnaire report and stats script.

## Dataset Generation Pipeline

The implementation uses approved Orchestrator workflows:

1. Deploy the App 3 service through Helm on Kubernetes.
2. Verify runtime guardrails: correct context, vLLM backend, one GPU, tensor parallel size 1, `local_stub=false`.
3. Run baseline/reference workloads.
4. Run external campaigns for pod availability, network latency, CPU pressure, and memory pressure.
5. Run internal campaigns for LLM-serving/application faults.
6. Preserve raw time-series samples, workload request JSONL, Kubernetes events/status, logs, and workflow reports.
7. Normalize source evidence into datapoint-local evidence bundles.
8. Build Stage 1 global plots/text/targets.
9. Build Stage 2 selected packet plots/text/targets.
10. Run target/evidence, leakage, visual, checksum, and package validations.
11. Export final `Raw`, `VLM`, and `Single_Shot` package views.

## Internal Faults

Internal fault families were invented to cover LLM-serving and API policy failures not expressible as pure infrastructure faults:

- `long_context_prefill_overload`
- `bad_gateway_batch_wait`
- `low_backend_timeout`
- `slow_tokenization_cpu`
- `decode_heavy_generation`
- `premature_readiness`
- `gateway_queue_delay`
- `kv_cache_pressure_oom`

The `kv_cache_pressure_oom` family is intentionally diagnostic. It can show service failure and GPU/cache pressure, but it remains outside main RCA unless same-process/same-physical-GPU and model-stage causal attribution are established.

## External Faults

External/Chaos-style families cover infrastructure and resource failures:

- `vllm_pod_restart`
- `api_pod_restart`
- `api_to_vllm_network_latency`
- `api_cpu_pressure`
- `vllm_cpu_pressure`
- `api_memory_pressure`
- `vllm_memory_pressure`

Some attempted ideas, such as packet loss or GPU helper pressure without sufficient attribution, were not accepted as final main-RCA examples. They informed the diagnostic policy and audit gates.

## Interesting Findings

The App 3 construction process produced several important findings:

- Honest phase summaries were not enough. Human review required true, readable time-series for target-critical evidence.
- Prometheus query timeouts must fail collection gates. They cannot be silently converted into "source unavailable" limitations.
- Workload/request outcomes are the most reliable user-impact evidence. They must override sparse, reset, NaN, or contradictory app counters.
- Diagnostic records should still show concrete observed anomalies.
- Severity is not the same as RCA sufficiency. A high-severity diagnostic can exist when causal proof is incomplete.
- GPU/KV-cache RCA needs special attribution policy. Correlated GPU/cache pressure is not enough for main GPU RCA.
- VLM prompts must be learning prompts, not audit reports. Audit-internal checks and target-action echoes were removed from final prompts.

## Validation Summary

Final package reports:

```text
Dataset/ai_3/MANIFESTS/final_dataset_manifest.json
Dataset/ai_3/MANIFESTS/target_semantics_audit.json
Dataset/ai_3/LEAKAGE_REPORTS/final_leakage_report.json
Dataset/ai_3/LEAKAGE_REPORTS/vlm_quality_review.json
Dataset/ai_3/CHECKSUMS/checksum_report.json
```

Final status:

- 80 datapoints.
- 160 VLM stage records.
- 57 main RCA, 18 framework diagnostics, 5 controls.
- Target semantics audit passed.
- Leakage scan passed.
- VLM quality review passed.
- Per-file checksum manifest generated.

## Known Limitations

- The application has one API, one backend, and one GPU; it does not cover multi-GPU tensor parallelism or large service graphs.
- Faults are controlled and synthetic/chaos-induced, not organic production outages.
- There are five repetitions per family.
- Stage 2 packet names reveal broad debugging area; stricter studies should evaluate a packet-neutralized variant.
- Some diagnostic examples deliberately withhold main RCA despite visible symptoms; evaluation must account for diagnostic policy.

## Pointers

Dataset guide:

```text
/lp-dev/yw2399/VISTA_Land/AI_3/data_readme.md
```

Final data:

```text
/lp-dev/yw2399/Dataset/AI_3
```

Repo symlink:

```text
/home/yw2399/Demo_Apps/Dataset/ai_3
```

Paper support:

```text
/home/yw2399/Demo_Apps/docs/Paper/Q_Sec_3/AI_3/AI_3_report.md
/home/yw2399/Demo_Apps/docs/Paper/Q_Sec_3/AI_3/AI_3_stats.json
```

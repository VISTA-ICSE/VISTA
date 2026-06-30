# AI_4 Implementation README: Distributed Elastic Training

## Project Overview

AI_4 is the Distributed Elastic Training application. It was built as a GPU-first, production-inspired ML training workload for VISTA/SE-Paper RCA research. The workload intentionally resembles the operational surface of a real distributed training platform: Kubernetes scheduling, GPU allocation, PyTorch `torchrun`/DDP ranks, checkpoint storage, runtime health/readiness endpoints, Prometheus metrics, DCGM GPU telemetry, rank-local logs, and controlled fault hooks.

The goal was not to train a high-quality model. The goal was to collect evidence-rich, source-traceable incidents that let a VLM practice two-stage root-cause analysis:

1. Stage 1: triage the broad broken area and choose the next debugging packet.
2. Stage 2: use packet-specific evidence to identify the exact faulty component, rank, resource, path, edge, or training stage.

Dataset details are in:

- `/lp-dev/yw2399/VISTA_Land/AI_4/data_readme.md`

Published code is in:

- `/lp-dev/yw2399/VISTA_Publish/code/AI_4/`

## Technology Selection

Runtime:

- Python/FastAPI for runtime control endpoints.
- PyTorch DDP with `torchrun --standalone --nnodes=1 --nproc_per_node=4`.
- Kubernetes Deployment for the first stable single-pod 4-GPU runtime.
- Prometheus metrics and DCGM GPU metrics for raw sampled evidence.
- Structured JSONL logs, one stream per rank.
- File-backed fault state so worker ranks can observe runtime-enabled fault hooks.

Dataset pipeline:

- Repo-native Orchestrator profiles, metric packs, fault catalog, Stage 1 preparer, Stage 2 preparer, deterministic labels, verifier, universal audit, final review, freeze/export, and Qwen SFT exporter.
- Audit contract: universal VLM RCA datapoint audit package v1.5.5.

Training/evaluation:

- Qwen3-VL-8B LoRA baseline.
- Qwen3.6-27B LoRA stronger baseline.
- Separate Stage 1 and Stage 2 adapters.
- Validation-selected checkpoints, with test evaluation run only after selection.

## Runtime Architecture

The live App 4 runtime is a single large Kubernetes pod with four GPUs. The supervisor process starts the HTTP runtime service and launches four DDP ranks. Each rank owns one CUDA device and emits structured logs/metrics. Rank 0 writes checkpoints to the configured checkpoint path.

Key endpoints:

- `/healthz`
- `/readyz`
- `/metrics`
- `/runtime/status`
- `/faults/enable`
- `/faults/status`
- `/faults/reset`

The runtime deliberately fails loudly if CUDA, four visible GPUs, or required torchrun/DDP rank configuration is missing. Tests may use mocks, but live data must be GPU-backed.

## Internal Faults

The internal faults were chosen because they are common real distributed-training failure modes and because they give clear rank/resource/stage evidence:

- Dataloader stall: all-rank batch wait, step-time increase, throughput drop.
- Checkpoint write failure: rank 0 checkpoint path permission/capacity/availability failure.
- NaN/overflow: nonfinite loss/gradient or finite-flag changes in training dynamics.
- Worker crash: controlled rank process death and torchrun reaction.
- CUDA OOM: rank/device memory pressure and CUDA OOM event.
- NCCL/rendezvous failure: process-group startup/rendezvous failure.
- Elastic recovery failure: missing/bad checkpoint manifest or resume failure.

Important lesson: internal faults are often the best source of main RCA datapoints for GPU/rank/checkpoint evidence because they can prove the exact target rank/device/path.

## External And General Faults

External/general campaigns used repo-native chaos/fault mechanisms against the App 4 pod or node:

- CPU saturation: accepted main RCA, aligned with step-time and throughput degradation.
- Pod restart/readiness: accepted main RCA, aligned with Kubernetes lifecycle and endpoint/rank interruption.
- Memory pressure: rejected weak/wrong signal, even though source collection passed.
- GPU helper compute canary: diagnostic/control only; no same physical GPU proof.
- Storage IOChaos canary: diagnostic/control only; checkpoint writes stayed successful and baseline-like.
- Network latency canary: diagnostic/control only; single-pod torchrun had no real inter-pod DDP/NCCL edge proof.

The important finding is that a fault mechanism is not enough. A datapoint becomes main RCA only when target-critical evidence proves causal impact. This is why diagnostic/control containment matters.

## Dataset Preparation Method

The canonical comparison baseline is `app4-baseline-no-fault-long1`. The earlier `app4-baseline-no-fault-clean1` is source-gate provenance only and is not used for candidate comparison.

Stage 1 uses broad canonical signals:

- `latency_badness`
- `error_badness`
- `throughput_drop_badness`
- `saturation_or_backlog_badness`
- `availability_badness`

All Stage 1 badness signals follow higher-worse semantics, with normal/baseline near zero. Duplicate per-rank x-values are aggregated by logical component. Pre-incident plot context uses neutral baseline/reference values rather than fault-run degradation.

Stage 2 uses packet-specific raw metrics and event/timeline evidence. It must not simply repeat Stage 1 canonical badness. Examples:

- Checkpoint packet: checkpoint status, bytes, duration, rank 0 errors.
- Worker packet: rank status, worker exit reason, torchrun reaction, pod restart count.
- CUDA packet: rank/device memory, DCGM GPU evidence, OOM marker.
- Rendezvous packet: startup/rank/world-size/process-group state.
- CPU packet: container CPU rate, step time, tokens/sec, dataloader wait.
- Pod packet: readiness/status/restart/endpoint failures and Kubernetes events.

## Data Augmentation

The final data used repeated real runs, not synthetic duplicates. The first accepted set had 12 records. The first augmentation raised the combined set to 30 records. Two additional real augmentation batches added 36 more main-RCA records, producing the final 66-datapoint package.

Final count:

- 66 datapoints.
- 63 main RCA datapoints.
- 3 diagnostic/control datapoints.
- 1 skipped/rejected memory-pressure ledger entry.

The Qwen SFT split is deterministic:

- train: 46
- validation: 10
- test: 10
- seed: `20260616`

## Model Training Results

Two LoRA baselines were trained and exported under `check_point/ai_4/`.

Qwen3-VL-8B:

- bundle: `qwen3vl8b_lora_baseline_20260617T000000Z`
- separate Stage 1 and Stage 2 adapters
- validation-selected checkpoints

Qwen3.6-27B:

- bundle: `qwen36_27b_lora_baseline_20260618T000000Z`
- model revision: `6a9e13bd6fc8f0983b9b99948120bc37f49c13e9`
- Stage 1 test core success: 5/10
- Stage 2 test core success: 9/10
- JSON parse rate: 10/10 for both stages

## Interesting Findings

The strongest lesson is that automated schemas are not enough. App 4 passed early mechanical checks while still having human-obvious prompt/plot problems. The dataset became useful only after manual review forced repairs to text format, answer leakage, Stage 2 metric selection, and plot semantics.

Stage 2 needed the most care. A VLM cannot learn the two-stage workflow if Stage 2 repeats Stage 1 badness panels. The repaired Stage 2 packet plots expose raw packet evidence in the right units and use timelines for state transitions.

Diagnostic/control datapoints are valuable when represented honestly. GPU helper, storage, and network canaries teach insufficient-evidence behavior rather than pretending every anomaly is main RCA.

The final App 4 package is a useful reference for future applications because it shows how to go from live distributed-system evidence to VLM-facing examples, Qwen SFT JSONL, LoRA training, checkpoint export, and test inference.

## Code Map

Published App 4 code contains:

- `distributed_training_app/`: FastAPI runtime, DDP training loop, fault hooks, checkpoint helpers, metrics, chart, Dockerfile, runtime tests.
- `orchestrator_dataset/`: Stage 1, Stage 2, candidate assembly, labels, verifier, audit, final review, and layout modules.
- `orchestrator_scripts/`: baseline/fault runners, Stage 1/Stage 2 preparation, candidate assembly, audit, final review, export, freeze, SFT creation, and model training scripts.
- `orchestrator_experiments/`: app profile, baseline/campaign scaffolds, and App 4 fault catalog YAMLs.
- `orchestrator_metric_packs/`: App 4 metric pack.
- `orchestrator_tests/`: App 4 static, Stage 1, Stage 2, candidate, audit, layout, and final-review tests.
- `docs/`: planning, plot-contract, and experience notes.

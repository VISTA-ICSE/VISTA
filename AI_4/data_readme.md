# Data README: AI_4 Distributed Elastic Training

## Dataset Roots

Primary final package:

```text
Dataset/ai_4/dataset_aug66/
```

Qwen/VLM SFT package:

```text
Dataset/ai_4/VLM/
```

Raw source artifacts:

```text
Dataset/ai_4/dataset_aug66/raw_source_artifacts/
```

Single-shot export:

```text
Dataset/ai_4/dataset_aug66/single_shot/
```

Candidate datapoints:

```text
Dataset/ai_4/dataset_aug66/dataset_candidate/app4_candidate_datapoint_set_augmented_66/
```

## Workload

The dataset comes from a GPU-first PyTorch distributed-training workload. The runtime is one Kubernetes-managed pod running `torchrun`/DDP with four local ranks and four GPUs. The workload uses deterministic synthetic training data so evidence collection focuses on system behavior, not external downloads or model quality.

The only candidate comparison baseline is:

```text
app4-baseline-no-fault-long1
```

The shorter baseline `app4-baseline-no-fault-clean1` is provenance only.

## Counts

- Candidate datapoints: 66
- Main RCA datapoints: 63
- Diagnostic/control datapoints: 3
- Skipped/rejected ledger entries: 1
- Stage 1 VLM records: 66
- Stage 2 VLM records: 66
- Combined Stage records: 132
- Single-shot datapoints: 66

Qwen SFT split:

- train: 46
- validation: 10
- test: 10
- random seed: `20260616`

## Main RCA Families

Each main RCA family has seven accepted real executions:

1. `dataloader_stall`
2. `checkpoint_write_failure`
3. `nan_loss`
4. `worker_crash`
5. `cuda_oom`
6. `nccl_rendezvous_failure`
7. `elastic_recovery_failure`
8. `cpu_saturation`
9. `pod_restart_readiness`

## Diagnostic/Control Records

The dataset contains three diagnostic/control records:

- `gpu_helper_compute_canary`: GPU helper evidence exists, but same physical GPU proof is absent.
- `storage_iochaos_canary`: checkpoint-path canary evidence exists, but checkpoint writes stayed successful and duration stayed baseline-like.
- `network_latency_canary`: network canary evidence exists, but single-pod torchrun has no real inter-pod DDP/NCCL edge proof.

These records are not main RCA datapoints. They teach insufficient-causal-proof behavior.

## Rejected Ledger

Memory pressure is preserved only as skipped/rejected evidence:

```text
app4-ef-memory-pressure-a1
```

It is rejected because visible memory pressure did not produce source-traceable training degradation, pod condition change, restart, OOMKill, or rank availability change.

## Evidence Sources

Raw source artifacts include:

- per-rank structured JSONL logs;
- runtime endpoint snapshots;
- Kubernetes pod status, events, descriptions, and container logs;
- Prometheus app metrics;
- DCGM/GPU metrics;
- container CPU/memory/network/disk metrics;
- checkpoint manifests and checkpoint artifacts;
- fault-gate reports and source collection validation.

Important App 4 metrics:

- `app4_training_step_time_seconds`
- `app4_training_dataloader_wait_seconds`
- `app4_training_batch_load_seconds`
- `app4_training_tokens_per_second`
- `app4_training_samples_per_second`
- `app4_training_loss`
- `app4_training_grad_norm`
- finite/overflow indicators
- `app4_training_checkpoint_duration_seconds`
- `app4_training_checkpoint_bytes`
- `app4_training_checkpoint_status`
- `app4_training_rank_status`
- `app4_runtime_state`
- `app4_training_world_size`
- DCGM GPU utilization and memory
- Kubernetes/container CPU, memory, readiness, restart, and status series

## VLM-Facing Stage 1

Stage 1 is broad system triage. It uses a five-panel plot with canonical normalized signals:

- latency badness
- error badness
- throughput-drop badness
- saturation/backlog badness
- availability badness

Stage 1 text contains:

- topology;
- workload summary;
- canonical signal summary;
- component and edge evidence;
- resource and workload evidence;
- summarized event/log evidence;
- missing signals only when important;
- a JSON-only task asking for localization and next packet focus.

Stage 1 must not leak the exact Stage 2 answer.

## VLM-Facing Stage 2

Stage 2 is focused packet RCA. It includes:

- Stage 1 triage output for continuity;
- packet-specific raw metric readings;
- selected event/log evidence;
- evidence against alternatives;
- unavailable or low-quality evidence notes when relevant;
- a JSON-only task asking for exact root cause and concrete actions.

Stage 2 plots use raw packet metrics and event timelines. They intentionally do not duplicate Stage 1 canonical badness panels.

## Single-Shot Layout

Each `single_shot/<datapoint_id>/` directory has:

```text
all/input.txt
all/output.txt
stage1/input.txt
stage1/plot.png
stage1/output.txt
stage2/input.txt
stage2/plot.png
stage2/output.txt
manifest.json
```

Single-shot inputs are sanitized. They omit fault-control/injection/chaos artifacts and remove direct answer-bearing lines.

## Audit And Review

The final package passed:

- source time-series audit;
- visual evidence audit;
- mechanical log/event traceability audit;
- diagnostic/control containment audit;
- memory-pressure exclusion check;
- VLM-facing leakage scan;
- final review.

Final review decision for the 66-point package was pass. Earlier freeze-blocking protected-path state was unrelated to App 4 and later resolved by owner approval for App 4 packaging.

## Model Artifacts

Clean inference bundles:

```text
check_point/ai_4/qwen3vl8b_lora_baseline_20260617T000000Z/
check_point/ai_4/qwen36_27b_lora_baseline_20260618T000000Z/
```

Test outputs:

```text
test/ai_4/qwen36_27b_lora_baseline_20260618T000000Z/stage_1/
test/ai_4/qwen36_27b_lora_baseline_20260618T000000Z/stage_2/
```

The 27B model achieved:

- Stage 1 test core success: 5/10
- Stage 2 test core success: 9/10
- JSON parse: 10/10 for both stages

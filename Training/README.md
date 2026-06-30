# VISTA Training Notes

This page records the training work I carried out for the VISTA VLM experiments: what was trained, what worked, what failed, and what I would keep in mind when continuing the project.

## Role I Played

I acted as the training operator and babysitter for a sequence of Qwen VLM supervised fine-tuning runs. In practice that meant:

- validating SFT exports before model loading;
- creating Qwen-compatible JSONL packages when needed;
- launching LoRA SFT jobs on A100 GPUs;
- monitoring GPU memory, logs, validation metrics, checkpoint writes, and recovery actions;
- selecting checkpoints by validation-only core metrics;
- running held-out test inference after selection;
- exporting best adapters under `check_point/` and later publication staging;
- writing reports, sidecar telemetry, mismatch files, and result summaries.

The most useful habit was to treat training as an operations job, not just a modeling job. Many failures were not model-quality failures: they were prompt length, CUDA memory, label masking, stale split assumptions, path layout, or artifact-management problems.

## Main Data Profile

The current VISTA corpus spans several incident domains:

| Dataset | Domain | Datapoints | Stage Records | Notes |
| --- | --- | ---: | ---: | --- |
| AI_1 | RAG pipeline | 82 packages, 75 trainable main RCA | 164 VLM inputs, 150 trainable aggregate records | RAG retrieval, context, vector DB, generator, GPU/resource, network. |
| AI_2 | GPU-backed support-ticket triage | 85 | 170 | Strong 8B results; also used for no-image/no-metric/patch ablations. |
| AI_3 | Single-GPU summarization service | 80 | 160 | Stage 1 learned very well; Stage 2 was partial. |
| AI_4 | Distributed elastic training | 66 | 132 | App 4 was the first strong proof that Qwen3-VL-8B LoRA could learn the task. |
| AI_5 | Tool-using operations agent | 81 | 162 | Strong Stage 1, weaker but usable Stage 2 partial fields. |
| Boutique | Online Boutique service mesh | 84 | 168 | Stage 2 prompts were too long in the original form. |
| Kafka | Streaming order pipeline | 93 | 186 | Good Stage 1 partial/selected performance; Stage 2 weak. |
| Postgres | HA relational backend | 80 | 160 | Stage 1 and Stage 2 both hard in current specialist run. |
| Redis | Cache tier | 88 | 176 | Stage 1 strong, Stage 2 required assistant-only-loss recovery. |

Most datasets have `RAW`, `Single_Shot`, and `VLM` views. Training used the `VLM_sft_qwen` exports where available, with grouped splits so Stage 1 and Stage 2 of the same incident stayed in the same split.

## Models And Training Setup

The main model families were:

- `Qwen/Qwen3-VL-8B-Instruct`
- `Qwen/Qwen3.6-27B`

The configuration that became the practical default was:

- BF16 LoRA SFT;
- LoRA rank `8`, alpha `16`, dropout `0.05`;
- language linear layers only;
- frozen vision encoder/projector/merger and frozen base weights;
- per-device batch size `1`;
- gradient accumulation `4`;
- AdamW, cosine schedule, warmup ratio `0.05`;
- gradient clipping `1.0`;
- maximum `30` epochs with early stopping;
- deterministic generation for validation/test;
- checkpoint selection by validation core success, average core-field accuracy, JSON validity, then validation loss.

The 8B model usually gave the best cost/results tradeoff. The 27B model was much heavier operationally and did not consistently beat the 8B LoRA specialists on these datasets.

## Evaluation Policy

I used core-field success as the primary metric.

Stage 1 core fields:

- `localization`
- `recommended_stage2_focus`
- `suspected_components`
- `suspected_edges`

Stage 2 core fields:

- `resource_root_cause`
- `root_cause_scope`
- `root_cause_target`
- `root_cause_type`
- `service_incident_root_cause`

Full exact JSON match was kept as a secondary diagnostic only. It is useful for checking whether the model reproduced the full target object, but it is too strict for judging RCA usefulness because non-core text fields and wording can differ.

## Headline Results

Selected held-out test results for the Qwen3-VL-8B LoRA specialists:

| Dataset | Stage 1 Core Success | Stage 1 Avg Core Field | Stage 2 Core Success | Stage 2 Avg Core Field |
| --- | ---: | ---: | ---: | ---: |
| AI_1 retrain | 0/12 (0.0%) | 12.5% | 0/12 (0.0%) | 20.0% |
| AI_2 | 12/13 (92.3%) | 96.2% | 6/13 (46.2%) | 76.9% |
| AI_3 | 12/12 (100.0%) | 100.0% | 3/12 (25.0%) | 75.0% |
| AI_5 | 12/12 (100.0%) | 100.0% | 3/12 (25.0%) | 81.7% |
| Boutique | 4/12 (33.3%) | 39.6% | not trained | context-window blocked |
| Kafka | 6/14 (42.9%) | 73.2% | 0/14 (0.0%) | 44.3% |
| Postgres | 1/12 (8.3%) | 58.3% | 0/12 (0.0%) | 48.3% |
| Redis | 9/13 (69.2%) | 80.8% | 3/13 (23.1%) | 63.1% |

Important context:

- AI_4 Qwen3-VL-8B LoRA was very strong in the earlier App 4 run: Stage 1 `10/10`, Stage 2 `6/10`.
- AI_4 Qwen3.6-27B LoRA had a different profile: Stage 1 `5/10`, Stage 2 `9/10`.
- Prompt-only baselines generally produced valid JSON sometimes, but did not achieve reliable core success.
- AI_1 remained difficult even after the later 8B retrain. JSON validity was good, but core-field matches stayed low.

## AI_2 Ablation Results

The AI_2 ablation/generalization runs used explicit self-describing names under:

```text
/lp-dev/yw2399/Experiments/AI_2/ai2_ablation_qwen3vl_8b_lora/
```

Selected held-out test highlights:

| Variant | Stage | Modality | Core Success | Avg Core Field |
| --- | --- | --- | ---: | ---: |
| `ai_2_no_image` | stage1 | text only | 12/13 (92.3%) | 98.1% |
| `ai_2_no_image` | stage2 | text only | 0/13 (0.0%) | 29.2% |
| `ai_2_no_metric` | stage1 | text without metric block | 7/13 (53.8%) | 88.5% |
| `ai_2_no_metric` | stage2 | text without metric block | 5/13 (38.5%) | 64.6% |
| `ai_2_patch` | stage1 | image + text, external new category | 4/10 (40.0%) | 82.5% |
| `ai_2_patch` | stage2 | image + text, external new category | 0/10 (0.0%) | 40.0% |

My read: AI_2 Stage 1 is robust; Stage 2 is more sensitive to representation and category shift.

## Operational Lessons

### 1. Stage 1 is learnable; Stage 2 needs better evidence compression

The strongest Stage 1 runs reached near-perfect or perfect held-out core success on AI_2, AI_3, AI_5, and App 4. Stage 2 often had good partial field accuracy but low all-core success. The model frequently learned scope/target better than the exact root-cause type or resource/service cause field.

For the next round, I would prioritize compact, structured Stage 2 packets over simply scaling model size.

### 2. Qwen3-VL-8B at `1e-5` was the most reliable default

Across the later specialist experiments, Qwen3-VL-8B LoRA with learning rate `1e-5` was easier to run, faster to iterate, and often better than the larger 27B runs. It became the default specialist setup.

### 3. 27B was expensive and not automatically better

The Qwen3.6-27B runs required careful GPU layout decisions. Early probes over-sharded the model; later probes showed smaller layouts could be viable in some cases, but the 27B path remained operationally heavy. On these data sizes, the 27B model did not reliably dominate the 8B specialists.

### 4. Assistant-only loss masking matters

The Redis Stage 2 jobs originally OOMed on long prompts using the ordinary full causal-LM loss path. The successful recovery used assistant-only cross-entropy/loss masking, preserving the same examples, targets, images, LoRA setup, and optimizer while avoiding unnecessary prompt-token loss materialization. That became the safer training path for long-prompt VLM SFT.

### 5. Prompt length is a dataset problem, not just a GPU problem

Boutique Stage 2 was blocked because many Stage 2 prompts were too large, with one warning around 784k tokens against a 262k model limit. Throwing GPUs at that would not be the right fix. The right fix is prompt compacting or structured evidence reduction while preserving target semantics.

### 6. Artifact hygiene became part of the experiment

I repeatedly had to manage checkpoint size, base-model duplication, stale run folders, symlinks, and publication staging. The final storage policy is:

- keep selected LoRA adapters;
- keep reports, configs, metrics, and split manifests;
- keep a single base model reference;
- do not duplicate full base model shards inside every checkpoint bundle.

## Important Artifact Roots

Training and reports:

```text
/lp-dev/yw2399/Experiments/Demo_Apps/
/lp-dev/yw2399/Experiments/AI_1/VLM/qwen3vl_8b_lora_retrain_winner/
/lp-dev/yw2399/Experiments/AI_2/ai2_ablation_qwen3vl_8b_lora/
```

Canonical test outputs:

```text
/home/yw2399/Demo_Apps/test/
```

Curated checkpoints:

```text
/home/yw2399/Demo_Apps/check_point/
/lp-dev/yw2399/VISTA_Publish/check_points/
```

Publication staging:

```text
/lp-dev/yw2399/VISTA_Publish/
```

Selected summary reports:

```text
/lp-dev/yw2399/Experiments/Demo_Apps/final_summary/test_success_rate_report.md
/lp-dev/yw2399/Experiments/Demo_Apps/qwen3vl_8b_lora_ai3_redis/final_summary/ai3_redis_qwen3vl_8b_diagnostic_report.md
/lp-dev/yw2399/Experiments/Demo_Apps/qwen3vl_8b_lora_boutique_kafka_postgres_report.md
/lp-dev/yw2399/Experiments/AI_2/ai2_ablation_qwen3vl_8b_lora/final_summary/ai2_ablation_qwen3vl_8b_lora_report.md
```

## What I Would Do Next

1. Compact Stage 2 prompts for Boutique, Kafka, Postgres, Redis, and other long/weak Stage 2 datasets.
2. Re-run 8B LoRA Stage 2 after compacting before spending more time on 27B.
3. Keep reporting both strict core success and per-field accuracy; the latter is often more diagnostic.
4. Keep a mismatch CSV for every selected test run so human reviewers can see whether failures are categorical, naming, null-semantics, or evidence-selection failures.
5. Preserve the current sidecar/telemetry pattern. It made later auditing and handoff much less painful.


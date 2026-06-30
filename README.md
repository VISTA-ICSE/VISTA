# VISTA

VISTA is a multimodal benchmark and training corpus for operational root-cause analysis in modern AI, data, and distributed systems. Each application dataset captures real or orchestrated incidents with raw execution artifacts, model-facing evidence, plots, targets, leakage audits, and split/export metadata.

The central task is deliberately two-stage:

1. **Stage 1 triage**: read broad incident evidence and identify the severity, localization, suspected components/edges, and the evidence packet that should be inspected next.
2. **Stage 2 root-cause analysis**: read a focused packet and produce a structured RCA object with root-cause type, scope, target, resource/service cause fields, evidence, and recovery/debugging actions.

The project is built around the idea that a useful RCA model should not memorize injected fault names. It should reason from operational symptoms: logs, events, metrics, dependency edges, workload behavior, and time-series plots.

## Repository Map

| Area | What It Covers | Entry Point |
| --- | --- | --- |
| AI_1 | RAG application incidents: retrieval, vector DB, generator, context/schema, GPU/resource, network, and compound failures. | [AI_1/README.md](AI_1/README.md) |
| AI_2 | GPU-backed support-ticket triage pipeline with malformed outputs, validator/retry failures, queueing, backend latency, resource pressure, and GPU contention. | [AI_2/README.md](AI_2/README.md) |
| AI_2 flat-test scoring | Frontier-model scoring subset for AI_2 single-shot stage outputs. | [AI_2_flat_test_scoring/README.md](AI_2_flat_test_scoring/README.md) |
| AI_3 | Single-GPU LLM summarization service RCA dataset. | [AI_3/README.md](AI_3/README.md) |
| AI_4 | Distributed elastic training workload with DDP/torchrun failures. | [AI_4/README.md](AI_4/README.md) |
| AI_5 | Tool-using AI operations agent incidents. | [AI_5/README.md](AI_5/README.md) |
| Boutique | Online Boutique microservice mesh, including service faults and business-correctness violations. | [Boutique/README.md](Boutique/README.md) |
| Kafka | Event-streaming order pipeline with broker, producer, consumer, storage, schema, backlog, and compound incidents. | [Kafka/README.md](Kafka/README.md) |
| Postgres | HA relational OLTP backbone with failover, replica, network, pool, lock, deadlock, timeout, and replication faults. | [Postgres/README.md](Postgres/README.md) |
| Redis | Cache-tier incidents: event-loop stalls, eviction/OOM, maxclients, persistence, pod/network faults, and absorbed failures. | [Redis/README.md](Redis/README.md) |
| Infrastructure | Fault catalogs, campaign definitions, app profiles, reusable templates, and artifact schemas. | [INFRASTRUCTURE/README.md](INFRASTRUCTURE/README.md) |
| Data-platform infrastructure | Fault catalogs, metric packs, app profiles, and audit contracts for Kafka, Postgres, Redis, and Boutique builders. | [data_platform_infrastructure/README.md](data_platform_infrastructure/README.md) |
| Training | Fine-tuning runs, checkpoint-selection policy, model results, and operator notes. | [Training/README.md](Training/README.md) |

## Dataset Shape

Most application datasets use three complementary views of the same incidents:

- `RAW/<id>/`: full audit/reconstruction artifacts. These may include control-plane provenance and should not be used as model-facing training input unless a leakage experiment explicitly calls for it.
- `Single_Shot/<id>/`: purged raw-log/raw-metric text inputs for LLM-only or less-curated experiments.
- `VLM/<id>/stage1` and `VLM/<id>/stage2`: curated multimodal evidence with `input_text.md`, `plot.png`, and structured targets.

Many datasets also include `VLM_sft_qwen/`, a Qwen-compatible supervised fine-tuning export with deterministic train/validation/test splits and metadata reports.

Across the current landing pages, VISTA covers roughly **739 datapoint directories** and about **1,464 stage-level VLM records** across AI applications, distributed training, microservices, Kafka, Postgres, and Redis.

## Core Evaluation Fields

The training/evaluation pipeline reports strict JSON validity and full-output exact match, but the primary task metric is core-field success.

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

Arrays are compared as normalized order-independent sets. Stage 2 null semantics are preserved exactly.

## Leakage And Provenance Policy

The datasets separate reconstruction evidence from model-facing evidence:

- RAW archives intentionally retain enough provenance to recreate and audit the incident.
- VLM and Single_Shot inputs are scrubbed of direct fault IDs, injection mechanisms, Chaos/control-plane labels, raw local paths, and other answer-bearing provenance.
- Organic operational symptoms are preserved, even when they are strong signals. Examples include pod restarts, error strings, latency spikes, backlog metrics, database lock/deadlock messages, cache eviction/OOM symptoms, and service-level failures.

This distinction is important: VISTA is not trying to hide the symptoms. It is trying to prevent models from learning the answer from the experiment machinery.

## Publication Artifacts

The current publication staging area is:

```text
/lp-dev/yw2399/VISTA_Publish/
```

The first staged deliverable is AI_1:

- data archive: `/lp-dev/yw2399/VISTA_Publish/data/AI_1/`
- lightweight LoRA checkpoint bundle: `/lp-dev/yw2399/VISTA_Publish/check_points/AI_1/`

Checkpoint publication follows the storage policy used during training: keep LoRA adapters and metadata, keep one base-model reference, and avoid duplicating base model shards.

## Recommended Reading Order

1. Start with this page.
2. Read one application page, such as [AI_2](AI_2/README.md), to understand the common dataset layout.
3. Read [INFRASTRUCTURE](INFRASTRUCTURE/README.md) to understand how fault catalogs, campaigns, and app profiles produce model-facing datapoints.
4. Read [Training](Training/README.md) for the fine-tuning workflow, model families, and empirical lessons.

## Current Limitations

- Stage 1 is generally easier than Stage 2. Stage 2 requires precise root-cause attribution and remains the main modeling challenge.
- Some app/dataset families have long Stage 2 prompts that require compacting before reliable fine-tuning.
- Some datasets contain healthy/absorbed/control incidents; these are valuable for evaluation but should be handled intentionally in training.
- Full-output exact match is too harsh for many experiments. Use it as a secondary diagnostic, not as the main success metric.

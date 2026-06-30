# Kafka Implementation README: Event-Streaming Order-Pipeline RCA Application (D1-Kafka)

## Project Introduction

Kafka is the **event-streaming data-backbone** application for the VISTA multimodal
root-cause-analysis dataset effort (paper-facing ID `D1-Kafka`). It was chosen because an
enormous fraction of production systems sit on a streaming backbone (commerce order flows,
telemetry/CDC ingestion, ledger fan-out, ML feature pipelines), and because streaming systems
fail in ways a plain request/response service cannot: a broker can be lost but **absorbed by
quorum**; a producer edge can slow without the consumer noticing; bad data can be quarantined to a
dead-letter queue with *no* error surfacing; consumer-group rebalancing can stall throughput.

The application is a faithful order pipeline:
`load-generator → order-producer → orders.v1 (3-broker Strimzi/KRaft cluster) → order-processor →
Postgres`, with `analytics-consumer` as a second subscriber and `orders.dlq` for quarantine. It is
small (5 services + a 3-broker cluster) but it runs the real Strimzi operator, real
`min.insync.replicas=2` quorum durability, a real DLQ, real librdkafka producer semantics, and
full Prometheus/Loki/JMX observability — so the faults we inject and the signals they produce are
representative operational incidents, not toy artifacts.

The topic is valuable for reviewers and dataset users because the **severity is earned by the
system's own mechanics**, not assumed: killing one of three brokers is a Level-1/2 absorbed event
(quorum holds, producer never sees an ack failure); killing two is a Level-4 hard failure
(`NotEnoughReplicasException`). A poison message is Level-2 because the DLQ correctly absorbs it.
The only way to tell these apart is to *read the evidence*, including cases where every canonical
metric is flat and the signal lives entirely in the logs.

For dataset details, see `/lp-dev/yw2399/VISTA_Land/Kafka/data_readme.md` and the live dataset root
`/lp-dev/yw2399/Data_Backend/Dataset/Kafka`.

## Runtime Architecture

```text
load-generator -> order-producer -> kafka-broker(orders.v1: 3 partitions x RF3, min.insync=2) -> order-processor -> postgres
                                            \-> analytics-consumer
                                   order-processor -> orders.dlq   (poison/schema quarantine)
```

Component roles (kafka profile):

- `order-producer`: serializes orders and publishes to `orders.v1` with `acks=all`; hot path is
  C-level network I/O via **librdkafka**.
- `kafka-broker` (`orders-cluster`, ×3): Strimzi/KRaft brokers; `orders.v1` 3×RF3 `min.insync=2`,
  `orders.dlq` for quarantine.
- `order-processor`: consumes `orders.v1`, validates, **writes each order to Postgres** (I/O-bound),
  routes poison/schema-invalid records to the DLQ.
- `analytics-consumer`: independent consumer group doing in-memory CPU-bound aggregation.
- `postgres`: the processor's write sink (lets us model cross-tier propagation).
- `fault-injector-api`: role `control_plane` — arms the internal `KI*` faults; **excluded from
  every model-facing surface**.
- `load-generator`: synthetic order traffic (default 2 orders/s; FastAPI control on port 8003).

Kubernetes runs one Deployment per app service plus a Strimzi `Kafka` CR on a minikube-in-docker
cluster (profile `data-backbone`). Prometheus scrapes PodMonitors + the kafka-exporter + JMX; Loki/
kubectl collect logs; the Orchestrator drives everything externally.

## Technology Selection

Strimzi/KRaft over a hand-rolled broker (production-grade operator, real controller/ISR semantics,
first-class metrics). `acks=all` + `min.insync.replicas=2` over weaker durability — this is what
converts "kill a broker" into a *defensible severity gradient* (L2 for one, L4 for two). A real
downstream Postgres sink rather than a stub — real cross-tier propagation and an I/O-bound consumer.
A deliberately low 2 orders/s workload so a resource fault must be *genuinely severe* to register,
which is what makes the absorbed/no-impact negatives trustworthy.

The dataset targets **vision-language models**: Stage-1/Stage-2 inputs are a rendered plot image +
text, the target is structured JSON; a Single-Shot flat-text view supports text-only baselines and
plot-modality ablations. Intended target tier is the latest multimodal Claude models; nothing is
model-specific.

Key implementation pieces (in `/home/yw2399/Data_Backbone_Apps`):

- App package: `apps/kafka/` (services, `app_profile.yaml`, `k8s/`, `shared/schema.py`)
- Profile: `apps/kafka/app_profile.yaml` — `canonical_stage1_signals`, `components`/roles,
  `stage2_packet_types`, `stage2_panel_renderers`, `log_normalizer`, `fault_to_packet_mapping`
- Metric pack: `Orchestrator/orchestrator/metric_packs/kafka.yaml`
- Labeling engine: `Orchestrator/orchestrator/labeling/` (`fault_mapping`, `structural_labeler`,
  `rephrase_narrative`, `renarrate_v2`, `leakage_gate`)
- Rendering: `Orchestrator/orchestrator/training_data/` (`plot_renderer_spec`, `plot_renderer`)
- Run-specs: `Orchestrator/experiments/fault_catalog/apps/kafka/` (20 bindings)
- Build/QC tooling: `scripts/` (`build_augmentation_dp.py`, `build_single_shot_purged.py`,
  `qc_dataset_review.py`, `fix_stage2_schema.py`, `regen_*`, `apply_task_block.py`, `q2_evidence_stats.py`)

The publishable code bundle is copied to `/lp-dev/yw2399/VISTA_Publish/code/Kafka`.

## Orchestrator Integration

Kafka reuses the shared Orchestrator without forking it: the app profile drives canonical-signal
extraction, packet dispatch, and panel rendering; self-contained fault bindings + the 12-step
lifecycle (`parse → collectors → baseline → inject → fault → recovery → finalize → export →
verify → cleanup → summary`) produce each artifact; Chaos Mesh + the in-process app-fault API
execute faults; `fault_mapping.py` pins each fault's (localization, packet) answer so the two
stages can never diverge. The shared labeling/rendering code is byte-identical across the frozen
data-platform apps (Kafka, Postgres, Redis).

## Internal Faults (`KI*`)

Application-level streaming-semantics faults that Chaos Mesh cannot express, armed through the
`fault-injector-api` and executed under lifecycle control. Each was chosen to span the
severity/observability space and exercise a *different* evidence channel:

- `KI01_poison_message` — non-deserializable payloads; DLQ absorbs them → useful-throughput drop
  with **zero service errors** (L2; the hardest "looks healthy but isn't" case).
- `KI02_processing_slowdown` — per-message handler delay → direct service-visible latency (L3).
- `KI03_offset_commit_suppression` — consumer stops committing → latent until restart/rebalance (L2).
- `KI04_rebalance_storm` — forced consumer-group churn → throughput stalls (L3).
- `KI05_back_pressure` — consumer can't drain → monotonic lag growth, drains in recovery (L3).
- `KI06_schema_mismatch` — producer emits wrong-schema records; DLQ absorbs (L2).
- `KI07_dlq_saturation` — ~100% routed to DLQ → useful-throughput collapse (L3).

**Lessons (designed but absorbed):** single poison/schema messages were *absorbed* (one bad record
and the pipeline shrugs) — the faults only register as a high-rate burst, and even then manifest as
throughput-drop + DLQ growth, not errors. `KI05` first didn't move the canonical envelope because
consumer-group lag wasn't collected; re-specified to a consumer-side slowdown whose true signature
is lag growth. These near-misses became the **non-bite policy** (escalate ~3×, keep genuine
no-impact runs as labeled negatives).

## External Faults (`KE*`)

Standard infrastructure chaos (Chaos Mesh `PodChaos`/`StressChaos`/`NetworkChaos`/`IoChaos` + stress
sidecars), targeted at meaningful points in the data flow and labeled against the system's
mechanics: `KE01` single broker kill (L2, quorum-absorbed), `KE02` CPU stress retargeted to the
downstream DB (L3), `KE03` broker memory pressure (L1, absorbed), `KE05/KE05B` producer→broker
network latency/loss (L3), `KE06` broker disk-I/O fsync latency (L3, biggest latency lever),
`KE07/KE07B` processor/producer CPU (L1 contrastive — the processor is I/O-bound and librdkafka
releases the GIL), `KE08` processor pod flap (L4), `KE09` multi-broker kill (L4,
`NotEnoughReplicasException`), `KE10` processor OOM induction. Compound: `C1`, `C2`.

**Unsuccessful/excluded trials (honest):** `KE03/KE05/KE05B/KE07B/KE10` were absorbed on first
attempt (a 60 s window at 2 orders/s is mild against a quorum/batched pipeline) and only entered the
dataset after escalation; `C3` (network+poison) was not promoted; `KE04` does not exist (numbering
gap). Broker kills don't move `deployment_replicas_available` (Strimzi StatefulSet) — they triage
via lag/latency/broker-fence logs.

## Dataset Preparation

Each datapoint is a **live** 12-step run capturing a baseline → fault → recovery window. The 5
canonical Stage-1 "badness" signals (latency / error / useful-throughput / consumer-group backlog /
availability) are normalized to `[-1,1]` (0=baseline, higher=worse) and sourced per-app from the
profile (e.g. `orders_processing_latency_seconds_p95`, `orders_consumed_total{status=success}`,
`kafka_consumergroup_lag`, `deployment_replicas_available` MIN-aggregated). Stage-2 selects one
**evidence packet** (one of 11) that defines the 2–4 diagnostic metrics to plot (one per panel,
blanks allowed, plot↔text parity enforced) and which components' full windowed logs to include.
Localization + packet are pinned per fault; **severity is data-driven** from the canonical signals
plus a log-aware floor (FATAL/ERROR counts). Augmentation re-runs each fault ~5× (strict repeats);
only the per-fault answer is inherited — plots, metric tables, and **per-run mechanical log
summaries** are regenerated from each run's real logs.

## Interesting Findings

- **Quorum absorption is a clean, defensible severity gradient.** KE01 (one broker) is L2/log-only;
  KE09 (two brokers) is L4 with `NotEnoughReplicasException`. The model must learn that the *same*
  action class (broker kill) maps to very different severities depending on the surviving ISR.
- **librdkafka releases the GIL**, so CPU stress on the producer barely moves latency — KE07B is a
  deliberate Level-1 contrastive "no-impact" sample, not a failed experiment.
- **The old labeling path over-labeled absorbed faults.** The v1.5.5 review found 4 faults marked
  L4 whose evidence showed absorption: `ke03` (only memory elevated), `ki01` (all service signals
  normal, DLQ-absorbed), `ki05` (severe degradation but no availability loss), and most starkly
  `ke10` — labeled "OOMKilled hard failure" but the OOM **never fired** (restarts flat, 0 OOMKilled
  in logs, memory only briefly touched the 512 MiB cgroup ceiling). All four were relabeled
  (→L1/L2/L3/L1) with grounded, production-framed narratives.
- **Plot-contract v1.4** (derived during this corpus): type-aware multi-series aggregation (SUM
  rates / MAX gauges / **MIN boolean health**), throughput clamp + noise floor, availability-as-
  replica-loss not restart-count, data-fit y-axis, and a hard plot↔text parity gate.
- **Leak surface is subtle:** the injected fault is named not only in obvious places but in *every*
  app log line (`fault_active: [...]`), in `experiment_id`, in the `fault-injector-api`
  component-role listing, and in cluster chaos events — all scrubbed from model-facing surfaces while
  keeping organic symptoms (broker fence, consumer lag, DLQ growth, poison-prefixed order ids).

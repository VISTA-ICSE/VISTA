# Data-Platform Infrastructure Implementation README (`data_platform_infrastructure`)

This document describes the shared VISTA infrastructure as used to build the **data-platform** RCA
datasets — Kafka (`D1`), Postgres (`D2`), Redis (`D3`), and Boutique (`W1`): the experiment
controller, fault catalog, injection mechanism, collectors, cleanup/verification layer, the
deterministic labeling/rendering engine, and the v1.5.5 datapoint contract.

It is the same shared control plane as the generic `INFRASTRUCTURE` bundle, snapshotted from the
data-platform repo (`/home/yw2399/Data_Backbone_Apps`) and scoped to the data-platform apps, plus
the audit contracts and dataset-construction toolchain. For a data/spec inventory, see
`/lp-dev/yw2399/VISTA_Land/data_platform_infrastructure/data_readme.md`.

## Big Picture

Realistic root-cause-analysis datasets cannot be produced by labels alone — they need controlled
incidents, real operational evidence, reproducible transformations, and defensible decisions about
whether a run is clean enough for training. The infrastructure therefore treats experiment
execution as a **12-step lifecycle**:

1. parse + validate the experiment spec → create the output directory
2. start log (kubectl/Loki) + metric (Prometheus) collectors
3. collect a **baseline** window
4. **inject the fault** — Chaos Mesh CRD or app-fault API or SQL/engine action
5. collect the **fault** window
6. **stop the fault** — delete the CRD / disable via API
7. collect the **recovery** window
8. stop collectors, finalize CSVs/JSONL
9. export artifacts (manifests, data files, snapshots)
10. **sanity verification** — confirm the fault left evidence in metrics/logs
11. **cleanup + health verification** — remove residual K8s resources, confirm recovery
12. write JSON + Markdown summary

The shared engine gives each data-platform application a common way to manufacture grounded RCA
examples while still allowing app-specific internal faults and app-specific metric semantics. A
second, equally important layer turns the raw run artifact into deterministic Stage-1/Stage-2
datapoints under a single cross-app contract (v1.5.5).

## Why This Infrastructure Matters For Software-Engineering Research

- **Faults are executable and auditable, not hand-written labels.** Every incident is a declarative
  fault template + an app binding, rendered into a real Chaos Mesh CRD / app-fault call and run
  under lifecycle control with verification + cleanup reports beside it.
- **Severity is earned by the system's mechanics**, not assumed: the same action (kill a broker /
  exhaust a pool) maps to different severities depending on quorum, synchronous standbys, GIL
  behavior, DLQ absorption — so the datasets teach *correct* reasoning, including "absorbed".
- **The dataset-building path has correctness checks** for value units, plot semantics, leakage,
  target/evidence consistency, and cleanup contamination — codified in the v1.5.5 contract and a
  deterministic QC pass.

## Runtime Architecture

The orchestrator runs **externally** (outside the cluster) and drives everything via `kubectl` and
the Chaos Mesh / app HTTP APIs. One Kubernetes cluster (minikube profile `data-backbone`) hosts the
app namespaces (`app-kafka`, `app-postgres`, …), a shared `chaos-mesh` namespace, and a shared
`observability` namespace (Prometheus `obs-prometheus-prometheus`, Loki `obs-loki`). For each
experiment the controller renders the fault, collects three windows of evidence, verifies, cleans
up, and finalizes a raw artifact directory; an app-specific dataset builder then converts that
artifact into the three model-facing views (RAW, Single_Shot, VLM).

## Technology Selection

- **Chaos Mesh** for infrastructure faults (`PodChaos`/`StressChaos`/`NetworkChaos`/`IoChaos`/
  `DNSChaos`) — declarative, auditable, with a real apply/recover lifecycle. Application-semantic
  faults that Chaos Mesh cannot express (poison/schema/offset/back-pressure for Kafka; lock/
  deadlock/timeout/pool/vacuum/replication for Postgres) live in in-process app-fault APIs / SQL
  actions, deliberately split so the chaos path gives realistic infra evidence without touching app
  semantics.
- **Prometheus + Loki + kubectl** for collection; per-app **metric packs** define the scrape lists
  and the canonical-signal source metrics.
- **Deterministic labeling**: localization + Stage-2 packet are pinned per fault
  (`fault_mapping.py`); severity is data-driven from the five canonical badness signals + a
  log-aware floor; a **leak-safe LLM rephrase** (`rephrase_stage1/2`, grounded only on
  structural+numerical+mechanical facts) writes the narrative fields. No app forks the engine —
  app-specific behavior is gated behind opt-in, default-off profile flags.

Key pieces (in `/home/yw2399/Data_Backbone_Apps`): `Orchestrator/orchestrator/` (the package),
`Orchestrator/experiments/fault_catalog/` (templates + bindings), `apps/<app>/app_profile.yaml`
(the per-app contract), `docs/audit_contracts/v1_5_5/` (the datapoint contract), `scripts/` (the
build/QC/review toolchain). The publishable snapshot is at
`/lp-dev/yw2399/VISTA_Publish/code/data_platform_infrastructure`.

## The App Profile (the bridge)

Each `apps/<app>/app_profile.yaml` is the contract between raw infrastructure evidence and
model-facing datapoints. It declares: components + roles (incl. the `control_plane`
fault-injector, excluded from every model-facing surface), dependency edges, the five
`canonical_stage1_signals` and their per-app source metrics + thresholds, the `stage2_packet_types`
+ `stage2_panel_renderers`, the `log_normalizer` (component log patterns + per-packet log
selection), and the `fault_to_packet_mapping`. Frozen apps carry none of the opt-in enhancement
flags, so the shared code is byte-identical for them.

## Labeling and Rendering

- `labeling/fault_mapping.py` — the authoritative fault → (localization, Stage-2 packet) table;
  both stages derive from it so they cannot diverge.
- `labeling/structural_labeler.py` — binding → `root_cause_type/scope/target` + resource/service
  cause. (Caveat the review surfaced: a constant can name a mechanism a given run's evidence shows
  did *not* fire, e.g. `oomkilled` — always verify against the run.)
- `labeling/rephrase_narrative.py` + `renarrate_v2.py` — the **rephrase-known-facts** transform
  (`rephrase_stage2` returns evidence_for/against + both action fields; never emits
  `confidence_reasons`; leak-checked). This is the schema-correct regeneration path; the old
  `classifier`/`build_primary_dp` path is avoided (it re-emits the legacy schema for low-severity
  dps).
- `training_data/plot_renderer_spec.py` + `plot_renderer.py` — **plot-contract v1.4**: 5-panel
  canonical badness (Stage 1) + packet 2×2 (Stage 2), with type-aware aggregation, MIN for boolean
  health, throughput clamp + noise floor, data-fit y-axis, and a hard plot↔text parity gate.

## The v1.5.5 Datapoint Contract

`audit_contracts/v1_5_5/` is the cross-app universal VLM RCA datapoint spec: Stage-1 (6 fields) +
Stage-2 (10 fields) schemas, badness semantics, the plot-contract v1.4 amendment, leakage/redaction
rules (no Chaos-Mesh/PodChaos/fault-id/injection vocabulary in model-facing surfaces;
incident-window wording), §16 resource-localization visibility, §17 mechanical log/event-evidence
categories, and §23 log-summary truthfulness. The checklist is the auditor gate; the changelog
records the plot-contract-v1.4 data-platform consensus.

## Interesting Findings

- **The leak surface is multi-layered.** Beyond chaos-event logs, the injected fault is named in
  *every* app log line (`fault_active: [...]`), in `experiment_id`, in the `fault-injector-api`
  component-role listing, in injector metric-CSV rows, and in cluster kubernetes-events — all of
  which the purge handles while keeping organic operator symptoms.
- **Per-pod log collection is not uniform.** kubectl `app/`/`pods/` views can omit StatefulSet pods
  (e.g. Kafka brokers); only the `namespaces/` view is the complete superset — the single-shot
  builder uses it for exactly this reason.
- **Absorbed faults are first-class data.** A fault that fires but is absorbed (quorum, batching,
  DLQ, no-OOMKill-at-the-ceiling) is a real, learnable signal — but the old labeling path
  over-labeled several to L4; the v1.5.5 review relabeled them with grounded, production-framed
  narratives.
- **Plot determinism matters.** Rendering multiple datapoints in one process could collapse panels
  to `legit_na`; the builders render one datapoint per process and verify the PNG.

# Redis Implementation README: In-Memory Cache RCA Application (D3-Redis)

## Project Introduction

Redis is the in-memory-cache application for the VISTA multimodal root-cause-analysis dataset effort (paper-facing ID `D3-Redis`). It was chosen to widen the *data-backbone* tier with a failure-physics class that neither Kafka (a partitioned durable log) nor Postgres (a relational store) can structurally exhibit: a **single-threaded event loop** where one O(N) command, big-key operation, or slow Lua script performs head-of-line blocking on *every* client at once; an **in-memory cache with an eviction policy** (eviction storms, OOM-on-write); **server-side connection-slot exhaustion**; and **asynchronous replication** that absorbs faults invisibly to clients.

The application is a Redis 7.2 primary + asynchronous replica (each with a `redis_exporter` sidecar), driven by a pooled mixed-read/write client, on Kubernetes (minikube profile `data-backbone`). It is a compact but expressive testbed for the failures an SRE actually debugs on a cache tier — and it is deliberately distinctive in that several of its faults are **not visible in the obvious signal**: an event-loop block shows up in *throughput and server-ops collapse*, not in latency p95; a maxclients flood is *client-invisible* to a pooled client and surfaces only in a rejected-connection counter and an organic log line; a replica kill or a 100 ms replication delay is *absorbed* by async replication.

The topic is valuable for reviewers and dataset users because the same component (redis-primary) fails in mechanistically different ways — eviction vs OOM-on-write vs event-loop block vs connection exhaustion vs pod loss — and the only way to tell them apart is to *read the right evidence panel and the logs*, not the packet name. It also contributes a healthy set of **honest contrastives**: faults that were injected but absorbed (async replica kill, DNS keepalive, persistence on a tiny dataset) or that register only as a resource anomaly with no service impact (StressChaos memory/CPU), labeled as such rather than dressed up as incidents.

For dataset details, see [`data_readme.md`](./data_readme.md) and the live dataset root `/lp-dev/yw2399/Data_Backend/Dataset/Redis`.

## Runtime Architecture

The application topology:

```text
load-generator -> redis-primary -> redis-replica        (async replication)
redis-fault-injector ..> redis-primary                  (control plane, scrubbed)
redis-primary/replica -> redis_exporter -> Prometheus
```

Component roles (redis profile):

- `redis-primary` (`cache_primary`): the single-threaded command event loop; serves all client reads/writes; the target of every internal fault and the focal RCA component.
- `redis-replica` (`cache_replica`): asynchronous replica; serves no client traffic in this workload (so a stressed/killed replica is client-invisible).
- `redis_exporter`: Prometheus exporter sidecar, one per redis pod (itself a redis client — self-blinds under a maxclients flood).
- `load-generator` (`load_generator`): pooled mixed ~70/30 GET/SET at ~100 ops/s; the source of the client-observed canonical signals; not a fault target.
- `redis-fault-injector`: control-plane FastAPI service that applies the internal (RI) faults on demand and restores state; role `control_plane`, **excluded from every model-facing surface**.

Kubernetes runs one Deployment+Service per redis instance (deliberately Deployments, not a StatefulSet, so `deployment_replicas_available` is a valid availability source for pod kills). Prometheus scrapes metrics; Loki/kubectl collect logs.

## Technology Selection

- **Redis 7.2**, primary + async replica, `redis_exporter` v1.62.0 sidecars — the canonical cache-tier topology.
- **Deployments, not a StatefulSet** (deliberate): gives a clean kube-state replicas gauge for availability; later required a component-level pod-identity-transition detector (a Deployment changes pod *name* on replacement, not in-place uid).
- **A logging load generator** (`loadgen-redis:0.1.1`): logs each client error with the verbatim organic reply (OOM, timeout, `max number of clients reached`) + a 10s `op_summary` heartbeat — without this, error faults were log-poor by construction.
- **A real-mechanism control-plane injector**: applies RI01-RI08 with genuine Redis operations (CONFIG SET of eviction policy / maxclients + a workload; O(N) SORT; server-side Lua `HGETALL`-discard for big-key; a CPU-busy-loop Lua calibrated to a target block for slow-Lua), then restores config + clears fixtures. Never `FLUSHDB`; never touches the load-gen keyset.
- **Chaos Mesh** for the external faults (PodChaos, StressChaos, NetworkChaos, IOChaos, DNSChaos).

Key implementation pieces (in `/home/yw2399/Data_Backbone_Apps`):

- App package: `apps/redis/` (manifests, `load-generator/`, `fault-injector/`, `k8s/`, `app_profile.yaml`, ledgers)
- Profile: `apps/redis/app_profile.yaml` (v1.0.7)
- Metric pack: `Orchestrator/orchestrator/metric_packs/redis.yaml`
- Labeling engine: `Orchestrator/orchestrator/labeling/` (`structural_labeler`, `rephrase_narrative`, `renarrate_v2`, `classifier_v3`, `stage2_label`, `leakage_gate`, `fault_mapping`)
- Rendering: `Orchestrator/orchestrator/training_data/` (`plot_renderer`, `plot_data_prep`, `narrative_validator`, `sample_writer_v2`)
- Run-specs: `Orchestrator/experiments/fault_catalog/apps/redis/` (15 specs)
- Build/QC tooling: `scripts/build_redis_dataset.py`, `build_redis_input_text.py`, `build_redis_baseline_dp.py`, `rephrase_redis_targets.py`, `qc_redis_vlm.py`, `patch_redis_vlm_tasks_logs.py`, `build_primary_dp.py`

The publishable code bundle is copied to `/lp-dev/yw2399/VISTA_Publish/code/Redis`.

## Orchestrator Integration

Redis reuses the shared Orchestrator without forking it. It plugs into existing conventions through the app profile (`canonical_stage1_signals`, `stage2_packet_types`, `stage2_panel_renderers`, `fault_to_packet_mapping`, `components`/roles); fault bindings + run-spec YAML; the Prometheus metric pack; Chaos Mesh + the in-process app-fault API for fault execution; the shared Stage-1/Stage-2 dataset writer and deterministic structural labeler; and the shared leakage/path checks. Every redis-specific enhancement to the **shared** rendering/labeling code is gated behind an opt-in, default-off profile flag (e.g. `availability_badness.plot_exclude_control_plane`) so the frozen sibling apps (Kafka/Postgres/Boutique) are byte-identical — a repeatedly-verified constraint.

## Internal Faults

Application-level Redis faults, armed through the control-plane injector and executed under Orchestrator lifecycle control:

- `RI01_maxmemory_eviction_storm`: low `maxmemory` + write flood under `allkeys-lru` → sustained eviction (rate 0 → ~20–26k keys/s, hit-ratio 1.0 → ~0.005). L3.
- `RI02_noeviction_oom_on_write`: `noeviction` + fill → SET returns OOM; ~29% of ops error while reads succeed → **partial degradation (L2)**, not a failure (error is scored as a *fraction* of total, not an absolute rate).
- `RI03_on_command_block`: repeated O(N) `SORT` over a 500k set stalls the single thread → server-ops 100→51, throughput 97→48; **p95 barely moves** (the slow ops are <5% of completed ops). L3.
- `RI04_big_key_operation`: repeated server-side Lua full-read of a big hash; at 300k fields p95 0.5 ms → 485 ms. L3. (A direct HGETALL OOM-kills the injector, hence the Lua-discard mechanism.)
- `RI05_slow_lua_script`: a CPU-busy-loop Lua auto-calibrated to a ~500 ms block (DEBUG SLEEP is disabled in Redis 7). Cleanest event-loop demonstrator. L3.
- `RI06_maxclients_saturation`: hold connections at maxclients → server rejects *new* connections (rejected_connections +78…280) but the pooled client keeps its connections → **client-invisible (L2)**, diagnosable via the rejected-connection counter + the `max number of clients reached` log.

Deferred / excluded: `RI07_cache_stampede_mass_expiry` (no backend behind the cache → the stampede recompute cost is not exercisable; would be contrived) and `RI08_persistence_fork_stall` (BGSAVE/AOF fork is ~25 ms on the tiny dataset → log-only; the reliable persistence bite is the external `RE07`).

## External Faults

Chaos Mesh perturbations:

- `RE01_primary_pod_kill` (PodChaos): command blackout during primary recreation; **L3 via event-sourcing** (pod-identity transition from kube events) — the coarse replicas gauge missed the ~10–15s recreation gap.
- `RE02_replica_pod_kill` (PodChaos): async-absorbed at the client; **L2**, service=None (role-scoped severity — replica kill ≠ primary kill).
- `RE03_primary_memory_oomkill` (StressChaos memory): **L0/L1 contrastive** — StressChaos cannot OOMKill redis (the cgroup OOM-killer takes the ~600 MB stressor, not the ~38 MB redis; restarts=0). The memory climb registers only as a resource anomaly.
- `RE04_primary_cpu_stress` (StressChaos cpu): **L1 contrastive** — saturates the 1-core limit but at ~97 ops/s redis is not cpu-bound; CFS fair-shares.
- `RE05_app_to_redis_network_latency` (NetworkChaos): per-op RTT added (p95 ~200–500×). **L3**, clean. (Netem applied on the reply leg to beat ClusterIP DNAT.)
- `RE06_master_replica_network_latency` (NetworkChaos): async replication absorbs 100 ms → **L0 absorbed**.
- `RE07_persistence_io_latency` (IOChaos): absorbed (`appendonly no` + tiny dataset → main thread never blocks on disk). **L0 absorbed**.
- `RE08_redis_dns_failure` (DNSChaos): pooled client holds its connection through the outage (keepalive). **L0 absorbed**.

`BASELINE_healthy_control` is the no-fault control (L0 `no_fault` / `healthy_baseline`, built deterministically without an LLM).

## Dataset Preparation Methods

The pipeline produces two VLM stages plus a raw single-shot view (see the [data README](./data_readme.md) for the on-disk layout).

**Stage 1 (broad triage):** five canonical badness panels (latency / error / throughput-drop / saturation / availability), a canonical-signal summary table, a deterministic mechanical diagnostic summary (incl. non-canonical resource evidence when the canonical service signals stay flat), a categorized log/event summary, and a JSON-only target (issue level, localization, suspected components/edges, recommended Stage-2 packet, rationale).

**Stage 2 (focused RCA):** a packet-specific plot, a scoped metric summary (exactly the plotted metrics, per window, reduced the way each panel is read), **inlined component-log evidence**, and a v1.5.5 target (root_cause_type/scope/target, resource/service root cause, evidence_for, evidence_against_alternatives, recommended_debugging_actions, recommended_recovery_actions).

Canonical signals are **workload-outcome-sourced**: latency/error/throughput from the load generator (what a client experiences), saturation from the server's eviction rate (a full cache is normal — eviction is the symptom), availability from `deployment_replicas_available` (+ `redis_up`, `master_link_up`) with the **control plane excluded**. Each run is the 12-step orchestrator lifecycle: 60s baseline / 60s fault / 60s recovery, Prometheus at 5s step (32 metric CSVs), kubectl+Loki logs.

The combined dataset is assembled by `scripts/build_redis_dataset.py` (phased: inventory → raw → singleshot → vlm), which unions the single-pass + augmentation runs into RAW/Single_Shot/VLM views and purges the single-shot raw evidence of injection-mechanism leaks while keeping organic symptoms. The Stage-2 schema fix (`rephrase_redis_targets.py`) and the format passes (`patch_redis_vlm_tasks_logs.py`) bring the targets to v1.5.5 and to the one-text-per-stage layout.

## Interesting Findings

- **The signal isn't always where you'd look.** Event-loop blocking (RI03/04/05) collapses *server-ops and throughput* while latency p95 can stay flat (RI03's 2.3s SORTs are <5% of completed ops). A model trained to expect a latency spike misses it; the packet foregrounds ops+throughput, with p95 sufficient-not-necessary.
- **A maxclients flood blinds its own meter.** `redis_up` is the exporter's scrape-success gauge, and the exporter is itself a redis client subject to maxclients — so it reports `redis_up=0` mid-fault and every primary-sourced metric goes stale. The diagnosable evidence is the `rejected_connections` counter delta (surfacing in the recovery window) + the organic `max number of clients reached` log. We added a `connection_pool` L2 floor because no canonical signal moves.
- **External stress ≠ application OOMKill.** StressChaos memory cannot OOMKill redis (the cgroup OOM-killer takes the biggest process — the stressor — not the ~38 MB redis). RE03 is therefore an honest L0/L1 memory-pressure contrastive, and the structural resource label was corrected from `..._oomkill` to `..._memory_pressure` to match the evidence (restarts=0, OOMKilled=0).
- **Async replication absorbs faults silently.** A replica kill (RE02), a 100 ms master↔replica delay (RE06), a DNS outage with pooled keepalive (RE08), and IOChaos on a tiny RDB dataset (RE07) all leave the client-facing signals flat. These are kept as confident absorbed/`no_impact_observed` contrastives — the boundary between "a fault was injected" and "the service was impacted."
- **Organic domain vocabulary must be kept, not purged.** `EVAL`/`SCRIPT KILL`/`lua-time-limit` (slow-Lua), the `OOM command not allowed` reply, `max number of clients reached`, and "no OOMKilled / ruled out" reasoning are exactly what an operator sees in a real incident — they are evidence, not leaks. Only the injection *mechanism* (chaos kinds, fault ids, `fault-injector`, "injected", config-knob-as-mechanism) is banned. (Over-banning `EVAL` once false-flagged 5 good datapoints.)
- **Schema drift is per-datapoint, and the legacy generator is the wrong fix.** ~19% of the stage-2 targets carried the old schema (`confidence_reasons`, no action fields). Re-running the legacy narrative generator **non-deterministically re-emits the old schema** for low-severity/absorbed cases (it uses the pre-rephrase prompt). The contract-correct path is the leak-safe `rephrase_stage2` "rephrase-known-facts" transform (grounded only on structural + numerical + mechanical facts), which produced 17/17 conformant targets that cite each run's own numbers.
- **Control-plane leakage hides inside metric CSVs.** kube-state series (`deployment_replicas_available`, `pod_restarts_total`, `container_*`) carry a row for the `redis-fault-injector` deployment, naming the control plane. The single-shot purge drops those rows (and the injector logs, and chaos-mesh event lines) while preserving organic pod-lifecycle events.

## Limitations of Realism

Single small minikube; ~100 ops/s pooled client; short windows (60/60/60s); injected rather than naturally-occurring faults; a small topology (1 primary + 1 replica + client); no L4 hard failure (redis faults here are bounded/recoverable); several externals absorbed at this scale (RE03/04 L1, RE06/07/08 L0); `RI07` not exercisable (no backend behind the cache) and `RI08` absorbed on a tiny dataset. The internal faults are deterministic injector-driven mechanisms rather than emergent bugs — their *evidence* is real telemetry, but their *trigger* is scripted.

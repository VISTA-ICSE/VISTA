# Postgres Implementation README: HA Relational OLTP-Backbone RCA Application (D2-Postgres)

## Project Introduction

Postgres is the **relational data-backbone** application for the VISTA multimodal
root-cause-analysis dataset effort (paper-facing ID `D2-Postgres`). It was chosen as the
deliberate counterpart to the streaming Kafka app: almost every transactional service sits on a
relational database, and its failure modes — primary failover, replica loss, network/DNS on the DB
edge, connection-pool exhaustion, lock contention, deadlocks, statement timeouts, vacuum/bloat,
replication-slot loss — are among the most universally relevant in all of software operations. It
also contributes a failure class the Kafka app cannot: **caller-vs-callee attribution**, where the
symptom surfaces at the API but the root cause is *not* the database engine.

The application is a faithful HA stack:
`load-generator → inventory-api (bounded pool) → postgres-primary (writes) / postgres-replica
(reads)`, with `analytics-worker` reading a replica and WAL streaming primary→replica. It runs the
real Bitnami PostgreSQL-HA chart (primary + 2 streaming replicas + pgpool + postgres-exporter), a
real async FastAPI/asyncpg API with a bounded connection pool, and full Prometheus/Loki
observability.

The topic is valuable for reviewers and dataset users because the failures are **application-
handling bugs surfaced by infrastructure faults**, the most interesting and realistic RCA class:
the API connects via a per-pod *headless DSN that does not auto-failover*, so killing the primary
produces hundreds of consecutive 500s + `cannot execute … in a read-only transaction` on the
demoted node; a connection-pool flood degrades the API while the DB engine stays perfectly healthy.
Telling these apart requires reading the evidence and resisting the urge to blame the component that
reports the error.

For dataset details, see `/lp-dev/yw2399/VISTA_Land/Postgres/data_readme.md` and the live dataset
root `/lp-dev/yw2399/Data_Backend/Dataset/Postgres`.

## Runtime Architecture

```text
load-generator -> inventory-api(bounded pool 20, per-pod headless DSN) -> postgres-primary  (writes: /orders, /inventory/adjust)
                                                                       \-> postgres-replica (reads: /products, /analytics)
analytics-worker -> postgres-replica
postgres-primary -> postgres-replica   (WAL streaming replication)
```

Component roles (postgres profile):

- `inventory-api`: async FastAPI/asyncpg; bounded **connection pool (max 20)** to the primary via a
  per-pod headless DSN; the user-facing RED surface.
- `postgres-primary`: write path; storage-bound (fsync dominates commit; WAL caps throughput).
- `postgres-replica` (×2): streaming-replication read replicas; `synchronous_standby_names` is
  **empty**, so primary writes don't block when standbys disconnect.
- `pgpool`: Bitnami connection pooler in front of the HA pair.
- `analytics-worker`: background read loop on a replica (non-user-facing).
- `load-generator`: fixed request mix at ~10 req/s (50% `POST /orders`, 20% hot-row update, 25%
  read, 5% analytics); **auto-runs**, no control API.

Kubernetes runs the Bitnami HA Helm release + the app Deployments on minikube profile
`data-backbone`. Prometheus scrapes the postgres-exporter (`pg_up`, `pg_locks_count`,
`pg_replication_lag_seconds`, …) + the API's RED + pool metrics; Loki/kubectl collect logs.

## Technology Selection

Bitnami PostgreSQL-HA over a single Postgres (a real primary/standby/pgpool topology, so failover/
replica-loss/replication faults are genuine). asyncpg + a bounded pool of 20 (makes `PI03` a real,
caller-side saturation fault and surfaces the asyncpg DNS-staleness behavior). A per-pod headless
DSN with no auto-failover and empty synchronous standbys — intentional, realistic configurations
that yield the marquee DSN-stale and L2-replica-loss findings. A recorded **30-minute steady-state
baseline** (`apps/postgres/baseline_30min.json`) anchors every severity threshold.

The dataset targets vision-language models (rendered plot + text → structured JSON), with a
Single-Shot flat-text view for text-only baselines.

Key implementation pieces (in `/home/yw2399/Data_Backbone_Apps`):

- App package: `apps/postgres/` (`inventory-api/`, `analytics-worker/`, `load-generator/`,
  `app_profile.yaml`, `helm/`, `sql/`, `k8s/`, `baseline_*`, `profile_design_notes.md`)
- Profile: `apps/postgres/app_profile.yaml`
- Metric pack: `Orchestrator/orchestrator/metric_packs/postgres.yaml`
- Labeling/rendering engine: `Orchestrator/orchestrator/labeling/`, `…/training_data/`
- Run-specs: `Orchestrator/experiments/fault_catalog/apps/postgres/` (16 bindings)
- Build/QC tooling: `scripts/` (same data-platform pipeline as Kafka)

The publishable code bundle is copied to `/lp-dev/yw2399/VISTA_Publish/code/Postgres`.

## Orchestrator Integration

Same as the Kafka app: the profile drives signal extraction / packet dispatch / panel rendering;
self-contained bindings + the 12-step lifecycle produce each artifact; Chaos Mesh + SQL/engine
actions execute faults; `fault_mapping.py` pins each fault's (localization, packet). The shared
engine is byte-identical to the other frozen data-platform apps. Postgres adds three data-platform
Stage-2 packets — `db_lock_packet`, `connection_pool_packet`, `replication_packet` — because the
generic infra packets cannot represent lock-wait timing, pool-vs-max saturation, or WAL/slot
evidence.

## Internal Faults (`PI*`)

SQL/engine-level failure modes that Chaos Mesh cannot express, grounded in PostgreSQL internals:

- `PI01_lock_contention_hot_row` — concurrent `SELECT FOR UPDATE` on one row → lock-wait latency,
  `still waiting for ShareLock` (L2/L3).
- `PI02_statement_timeout` — slow statements canceled by `statement_timeout` (L1, low-severity
  diagnostic).
- `PI03_connection_pool_flood` — caller exhausts the bounded pool → pool-exhausted, **DB healthy**
  (L3; caller-side, with explicit evidence *against* DB-engine failure).
- `PI04_vacuum_starvation` — autovacuum disabled under UPDATE churn → dead-tuple bloat, gradual
  slowdown (gray failure).
- `PI05_long_running_transaction` — idle-in-transaction holding locks/snapshots (L3).
- `PI06_deadlock_induction` — cyclic lock ordering → `deadlock detected`, one txn aborted (L1
  low-severity diagnostic).
- `PI07_replication_slot_drop` — drop a slot the standby depends on → `could not receive data from
  WAL stream` (L3).

**Lessons:** the caller-side faults forced a localization-vocabulary expansion (`connection_pool`,
plus an explicit "evidence against DB-engine failure" requirement) so a pool fault isn't blamed on
Postgres. `PI04` needed a sustained UPDATE workload + extended window to bite (bloat is a *slow*
fault). Many `PI*` are log-only at the metric level (locks, deadlocks, WAL errors), which is what
drove the **log-aware severity floor**.

## External Faults (`PE*`)

Standard Chaos Mesh chaos, labeled against the system's mechanics: `PE01` primary pod kill (the
marquee DSN-stale-after-failover case, sustained 500s + read-only-tx), `PE02/PE03` replica/multi-
replica kill (L2 — no synchronous standbys, writes unaffected), `PE04/PE05` app→primary network
latency/loss (L3, clean signature), `PE06` primary DNS failure (asyncpg `gaierror`),
`PE07` inventory-api CPU stress (caller-side), `PE08` primary memory pressure (log-only — FATAL/
ERROR while metrics stay flat), `PE10` primary kill + standby partition (compound).

**Unsuccessful/excluded trials (honest):** `PE09` disk-I/O is **deferred and absent** — Chaos Mesh
IoChaos could not cleanly inject fsync delay through the Bitnami chart's read-only mounts; a stuck
IoChaos finalizer also hung a sweep (now guarded with per-run timeouts + a finalizer force-clear).
`PE06`/`PE07` were `needs_rerun` on first attempts (DNS absorbed by keepalive; CPU stress didn't
saturate at 10 req/s) and entered the dataset after escalation.

## Dataset Preparation

Identical pipeline to the Kafka app. The 5 canonical Stage-1 signals are sourced from Postgres
metrics (`inventory_api_request_duration_seconds_p95{/orders}`, `inventory_api_requests_total{5xx}`,
`{2xx,/orders}` rate, `inventory_api_pool_in_use/pool_max`, `pg_up` MIN-aggregated). Thresholds
trace to the 30-min baseline (`/orders` p95 ≈ 8.8 ms, pool ≈ 0.18, `pg_up` = 1). A log-aware
severity floor (5 FATAL / 20 ERROR) is essential here — it is what makes `PE08`/`PI06`/`PI07`
log-only faults score correctly. Augmentation re-runs each fault ×5 (strict repeats), per-run
mechanical log summaries regenerated from each run's real logs.

## Interesting Findings

- **DSN-stale-after-failover** (`PE01`/`PE10`): the per-pod headless DSN doesn't auto-failover, so a
  primary kill becomes a *caller-side handling* failure (hundreds of 500s + `read-only transaction`)
  rather than a silent reconnect — a real, page-at-3am incident captured as a clean labeled example.
- **Caller-vs-callee attribution** is the signature challenge: `PI03` (pool exhausted, DB healthy),
  `PE01` (DSN-stale), `PE07` (caller CPU) all put the symptom at the API while the root cause is
  elsewhere. The Stage-2 packets carry explicit *evidence against* DB-engine failure.
- **Empty synchronous standbys** make replica loss a *degraded read path* (L2), not a write outage
  — an earned, not assumed, severity.
- **Gray / log-only failures**: `PE08` (memory pressure, FATAL logs, flat metrics) and `PI04`
  (vacuum bloat, slow degradation) are exactly the cases coarse metric-threshold AIOps misses; the
  log-aware floor surfaces them.
- **Postgres logs are unusually explicit** (`deadlock detected`, `canceling statement due to
  statement timeout`, WAL-stream errors). These are real operator-visible evidence (kept), but they
  make some families near-trivial from logs alone — flagged for the paper as a **log-label shortcut**
  worth an ablation, distinct from the injection-control statements (scrubbed).
- The v1.5.5 review confirmed **16/16 fault families PASS** on the agent review; the only target
  fixes were removing the banned `confidence_reasons` (pe06, pi04) and the templated "test scenario /
  contrastive baseline" injection-framing on `pe07` (regenerated to production-equivalent actions).

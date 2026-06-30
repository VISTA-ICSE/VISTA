# Boutique Implementation README: Online Boutique gRPC-Mesh RCA Application (W1-Boutique)

## Project Introduction

Boutique is the web/e-commerce-microservice application for the VISTA multimodal root-cause-analysis dataset effort (paper-facing ID `W1-Boutique`). It was chosen because the dominant production failure mode for modern services is not a single crashed process but a **mesh of gRPC microservices** where the symptom surfaces far from the cause: a slow payment dependency shows up as frontend latency; a missing idempotency key shows up as a customer double-charge with *no* degradation in any performance metric at all.

The application is the Google Cloud `microservices-demo` ("Online Boutique") — 11 gRPC services behind a web frontend, deployed on Kubernetes with a Linkerd service mesh. A shopper browses the catalog, adds to cart, checks out, pays, and is shipped a confirmation. This is a compact but expressive testbed for the failures software engineers actually debug in microservice systems: dependency-edge latency/errors/timeouts, retry amplification, resource pressure, pod availability loss, and — most distinctively — **business-correctness violations** (duplicate charges, orphaned orders, unfinalized carts, price drift) that are invisible to RED metrics and only detectable in the application's own transactional records.

The topic is valuable for reviewers and dataset users because it puts a hard, realistic problem in front of the model: the same `checkout→payment` edge can fail as a network problem, a dependency-error problem, a saturation problem, or a silent correctness problem, and the only way to tell them apart is to *read the evidence*, not the packet name.

For dataset details, see `/lp-dev/yw2399/VISTA_Land/Boutique/data_readme.md` and the live dataset root `/lp-dev/yw2399/Data_Backend/Dataset/Boutique`.

## Runtime Architecture

The application topology:

```text
loadgenerator -> frontend -> checkout -> { payment, shipping, email, currency, productcatalog, cart -> redis-cart }
                             frontend -> { productcatalog, recommendation, ad, currency }
```

Component roles (boutique profile):

- `frontend`: web entrypoint and request-fan-out; the canonical RED surface.
- `checkout` (`checkoutservice`): order orchestrator — calls payment, shipping, email, currency, cart, productcatalog. The locus of the transaction-integrity faults.
- `payment` (`paymentservice`): the focal backend dependency (`checkout→payment` edge).
- `productcatalog`, `cart`(`→redis-cart`), `recommendation`, `currency`, `shipping`, `email`, `ad`: backend services.
- `payment-injector`: an Envoy proxy injected in front of payment, role `control_plane` — drives edge faults and is **excluded from every model-facing surface** (its identity is redacted to the logical callee `payment`).
- `loadgenerator`: synthetic shopper traffic.

Kubernetes runs one Deployment+Service per logical service on a minikube-in-docker cluster (profile `data-backbone`). Linkerd sidecars emit per-edge RED telemetry; Prometheus scrapes metrics and Loki/kubectl collect logs.

## Technology Selection

The app is the upstream Online Boutique (Go/C#/Python/Node gRPC services) with two services **patched** to carry deterministic, runtime-armable fault toggles:

- `services/checkoutservice` (Go, built as `patched`): Shape-B toggles for `double_charge`, `orphaned_order`, `price_drift`, `cart_skip`, plus `pool_saturation` / `cpu_hotpath` substrate. Each toggle is an atomic flag armed via an admin `POST /faults/enable`; the order path emits **symptom-only** structured business events (`biz_event: charge/ship/cart_empty/order_done`, `duplicate_charge_detected`, `charge_amount_mismatch`, …) and never logs the fault name or mechanism.
- `services/productcatalogservice` (Go, `patched`): a silent request-correlated `memory_leak` toggle (RSS grows with the service's own request volume, not freed on disarm).

The mesh edge faults (`BI01–BI05`) are reproduced by an **Envoy `payment-injector`** sidecar driven by `apps/boutique/fault-injector/apply_mode.sh` (a timed configmap-swap + rollout, executed as an `envoy_mode` orchestrator step). External resource/availability faults (`BE*`) use Chaos Mesh (`StressChaos`, `PodChaos`, `NetworkChaos`). This split was deliberate: Chaos Mesh cannot express application-semantic correctness faults, so those live inside the patched binaries; conversely, the Envoy/Chaos path gives realistic edge/resource evidence without touching app code.

Key implementation pieces (in `/home/yw2399/Data_Backbone_Apps`):

- App package: `apps/boutique/` (services, `app_profile.yaml`, `fault-injector/`, `load-generator/`, `k8s/`)
- Profile: `apps/boutique/app_profile.yaml`
- Metric pack: `Orchestrator/orchestrator/metric_packs/boutique.yaml`
- Labeling engine: `Orchestrator/orchestrator/labeling/` (`classifier_v3`, `structural_labeler`, `stage2_label`, `rephrase_narrative`, `renarrate_v2`, `evidence_builder`, `api_helpers`, `labeling_prompts`)
- Rendering: `Orchestrator/orchestrator/training_data/` (`stage1_input_builder`, `plot_renderer`, `sample_writer_v2`, `log_uid_filter`)
- Run-specs: `Orchestrator/experiments/fault_catalog/apps/boutique/` (27 specs)
- Build/QC tooling: `scripts/build_boutique_dataset.py`, `boutique_{singlepass,augment,addpass}_driver.py`, `boutique_ti_target_regen.py`, `qc_boutique_vlm.py`, `build_primary_dp.py`

The publishable code bundle is copied to `/lp-dev/yw2399/VISTA_Publish/code/Boutique`.

## Orchestrator Integration

Boutique reuses the shared Orchestrator without forking it. It plugs into existing conventions through: the app profile (`canonical_stage1_signals`, `stage2_packet_types`, `fault_to_packet_mapping`, `components`/roles); fault bindings + run-spec YAML; Prometheus/Linkerd-compatible metrics; Chaos Mesh + an `envoy_mode` executor + the in-process app-fault API for fault execution; the Stage-1/Stage-2 dataset registry; deterministic labeler/structural-labeler registration; and the shared leakage/path checks.

Every Boutique-specific enhancement to the **shared** labeling/rendering code is gated behind an opt-in, default-off profile flag (e.g. `transaction_integrity_floor`, `baseline_intent_floor`, `secondary_narrative_llm`, `component_log_aliases`, `log_identity_redactions`, per-packet `shape_layouts`). Frozen apps (Kafka/Postgres/Redis) carry none of these flags, so the shared code is byte-identical for them — this was an explicit, repeatedly-verified constraint.

## Internal Faults

Application-level correctness/runtime faults, armed through the patched binaries' admin fault API and executed under Orchestrator lifecycle control:

- `II10_double_charge`: the charge path settles two distinct transactions for one order (no idempotency). Emits `duplicate_charge_detected`.
- `II11_orphaned_order`: payment settles but the order is never shipped and never refunded (no saga compensation). Emits `orphaned_order_detected`.
- `II12_cart_not_emptied`: the cart is not finalized after a completed order. Emits `cart_not_finalized`.
- `II13_price_drift`: the charged amount diverges from the quoted amount. Emits `charge_amount_mismatch`.
- `BI09_productcatalog_memory_leak`: a silent runtime leak whose RSS slope tracks the service's own request volume; not freed on disarm (a real leak plateaus in recovery).
- `BI07_productcatalog_cpu_fanin`: fan-in CPU pressure on productcatalog.

The four `II*` faults are the **novel contribution**: they are RED-invisible. Canonical latency/error/throughput/availability stay at baseline; the only evidence is the inconsistency in the application's own business-event records. A deterministic violation detector reads those records and, on a confirmed violation, floors the issue level to L3 — a real, user-affecting correctness incident that no performance metric would catch.

## External Faults

Reproduced edge faults via the Envoy `payment-injector` (the `checkout→payment` edge):

- `BI01_payment_induced_latency`: edge p95 rises; callee inbound flat; no app errors.
- `BI02_payment_induced_errors`: gRPC error fraction rises (UNAVAILABLE/500s) on the edge.
- `BI03_payment_dependency_timeout`: deadline-exceeded; severe throughput collapse, zero errors.
- `BI04_payment_slow_dependency`: sustained high edge latency, partial throughput.
- `BI05_payment_retry_amplification`: caller retries amplify edge RPS above the caller rate while errors are masked (canonical RED stays normal; the rps-amplification panel is the discriminator).

Chaos Mesh resource/availability faults:

- `BE01_payment_cpu_stress` (StressChaos, payment 200m limit), `BE02_productcatalog_mem_stress`, `BE03_payment_pod_kill` (held), `BE04_productcatalog_pod_kill` (one-shot, absorbed → L0 contrastive), `BE04B_productcatalog_pod_failure` (held), `BE05_payment_net_delay` (NetworkChaos).

`B00_healthy_baseline` is the no-fault control (L0 `healthy_baseline`).

## Dataset Preparation Methods

The pipeline produces two VLM stages plus a raw single-shot view (see the [data README](./data_readme.md) for the on-disk layout).

**Stage 1 (broad triage):** five canonical RED panels (frontend-mesh latency/error/throughput/saturation/availability badness), the canonical numerical+exact-value tables, an auxiliary log/event severity summary, a dynamic topology snapshot, and a JSON-only target (issue level, localization, suspected components/edges, recommended Stage-2 packet).

**Stage 2 (focused RCA):** a packet-specific, shape-aware plot (e.g. transaction-integrity charge panels; service_runtime memory-working-set; edge p95/rps for the dependency packets), a scoped numerical metric summary, inlined component-log evidence, and a v1.5.5 target (root_cause_type/scope/target, resource/service root cause, evidence_for, evidence_against_alternatives, recommended_debugging_actions, recommended_recovery_actions).

Canonical signals are deliberately **frontend-RED-only** (the user-facing mesh edge). This is what makes transaction-integrity faults RED-invisible and forces a record-driven severity floor.

The combined dataset is assembled by `scripts/build_boutique_dataset.py` (phased: inventory → schema_fix → raw → singleshot → vlm), which merges the single-pass + augmentation runs into RAW/Single_Shot/VLM views, applies the v1.5.5 schema correction in-place, and purges the single-shot raw evidence of injection-mechanism leaks while keeping organic symptoms.

## Interesting Findings

- **RED-invisible correctness is the hard, valuable class.** Transaction-integrity faults produce *zero* canonical-signal movement; severity and root cause rest entirely on the application's own transactional records. This required a record-driven L3 floor and a dedicated Stage-2 "target-coherence" gate (G6) that forces the target to cite the confirmed record violation, state RED as normal, and never conclude "nothing happened."
- **Narrative generation must be bound to the primary evidence, not the canonical signals.** Generating Stage-2 prose from flat RED produced two failure modes: degenerate targets that *denied* the fault ("no logs, normal operation, integrity ruled out") and confabulated targets that attributed sub-threshold RED noise to the fault. The fix binds the prompt to the confirmed violation record and re-gates coherence — the Stage-2 sibling of a Stage-1 hallucination fix.
- **Same-edge separability is the concrete leakage test.** The `checkout→payment` edge carries `network_edge`, `service_dependency`, and `service_saturation` packets. At the canonical-panel level `BI01`(latency) and `BI02`(errors) look nearly identical; they are separable *only* by reading the logs (gRPC error codes/500s for dependency, none for latency) and the rps-amplification plot (saturation). The dataset is designed so a model cannot shortcut on the packet name.
- **Absorbed faults are honest L0 contrastives.** A one-shot productcatalog pod-kill (`BE04`) reschedules with no user-facing dip → `no_impact_observed` L0; the held variant (`BE04B`) bites `kubernetes_availability` L3. The gate quarantines an *unexpected* absorption (e.g. `BE01` cpu-stress that fails to register) rather than mislabeling it.
- **The severity band is effectively binary against a low frontend-RED threshold.** Varied-intensity augmentation produced honest L2 (milder latency/retry/leak) and L0 (absorbed/baseline) but **no L1 and no L4**: boutique faults either cross the low canonical threshold (L2/L3) or are fully absorbed (L0). This is a real property of the frontend-RED signal model, reported as-is rather than tuned.
- **Control-plane identity must be excluded by construction.** The Envoy `payment-injector` and Linkerd apparatus appear nowhere on the four VLM surfaces (redacted to logical `payment`), and are dropped/line-filtered from the single-shot raw view — while organic pod-lifecycle (`Killing`/`Started`) and domain symptoms (`duplicate_charge_detected`, `UNAVAILABLE`) are preserved.
- **Schema drift is per-datapoint, not per-family.** The v1.5.5 correction (drop `confidence_reasons`, add the two action fields) had to be classified and applied per datapoint; the contract-correct path is the leak-safe `rephrase_stage2` "rephrase-known-facts" transform, not a full rebuild (which re-emits the old schema for low-severity cases).

## Limitations of Realism

Single small minikube mesh; low single-digit RPS; short windows (baseline 60s + fault 60–120s + recovery 60s); synthetic shopper traffic; injected rather than naturally-occurring faults; no L4 hard outage and no L1 marginal band represented; the transaction-integrity faults are implemented as deterministic in-process toggles rather than emergent bugs (their *evidence* is real telemetry, but their *trigger* is scripted). Two bindings (`BI10` pool_saturation, `BI11` cpu_hotpath) are deferred for lack of biting substrate at the test load.

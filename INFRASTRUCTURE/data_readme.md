# INFRASTRUCTURE Data And Specification README

This README explains the data-like assets owned by the shared VISTA infrastructure. Unlike `AI_1`, `AI_2`, or other application datasets, `INFRASTRUCTURE` is not a model-training dataset. It is the catalog, schema, campaign, profile, and artifact-shape layer that produces and validates application datasets.

## Locations

- Source repo: `/home/yw2399/Demo_Apps/Orchestrator`
- Published code snapshot: `/lp-dev/yw2399/VISTA_Publish/code/INFRASTRUCTURE`
- Implementation README: `/lp-dev/yw2399/VISTA_Land/INFRASTRUCTURE/README.data`
- Handoff: `/home/yw2399/Demo_Apps/docs/Handoffs/INFRASTRUCTURE.md`

Application datasets are stored separately, for example:

- `/lp-dev/yw2399/Dataset/AI_1`
- `/home/yw2399/Demo_Apps/Dataset/ai_1`

## Specification Inventory

The infrastructure snapshot includes these specification families:

- Fault catalog YAML files: `158`
- Campaign YAML files: `22`
- App profile YAML files: `5`
- Chaos Mesh reusable templates: `21`
- Compound reusable templates: `6`
- GPU helper reusable templates: `8`

App-specific fault bindings in the snapshot:

- `rag`: `51`
- `support_ticket_triage`: `22`
- `llm_summarization_serving_app3`: `19`
- `distributed_elastic_training_app4`: `14`
- `tool_using_ai_operations_assistant_app5`: `13`
- `examples`: `1`

## Fault Catalog Layout

Fault specifications live under `experiments/fault_catalog`.

Important subdirectories:

- `schema/`: YAML schema for fault profiles and experiment templates.
- `templates/chaos_mesh/`: reusable single-fault and ramp templates backed by Chaos Mesh.
- `templates/compound/`: reusable compound fault templates.
- `templates/gpu_helper/`: reusable GPU helper templates.
- `apps/<app_id>/`: app-specific bindings that map reusable ideas onto app targets.
- `index.yaml`: catalog index.
- `README.md`: catalog description and guidance.

The catalog is intentionally declarative so reviewers can inspect exactly what was supposed to happen before reading any collected artifact.

## Campaign Layout

Campaign files live under `experiments/campaigns`.

A campaign defines a sequence of experiments for an app. It can include smoke checks, external fault batches, internal app faults, targeted recollections, gap-filling runs, or reruns after verification failures.

Campaigns are sequential by design. Clean data and clean cleanup boundaries matter more than parallel speed.

## App Profiles

App profiles live in both `experiments/app_profiles` and `orchestrator/app_profiles`.

They describe:

- components,
- dependency edges,
- app stages,
- metric aliases,
- log sources,
- canonical signals,
- metric semantics,
- Stage 2 packet mappings,
- parser patterns,
- expected evidence sources.

The app profile is the bridge between raw infrastructure evidence and deterministic model-facing datapoints.

## Generated Artifact Shape

When the infrastructure runs an experiment, a raw artifact directory usually contains:

- `config/`: rendered or resolved experiment configuration.
- `faults/`: fault execution status, app fault status, or helper status.
- `rendered/`: rendered Kubernetes or helper manifests when applicable.
- `metrics/`: Prometheus-style metric CSVs.
- `logs/`: app, namespace, pod, and collector logs.
- `events/`: Kubernetes event snapshots.
- `state/`: pod, deployment, service, endpoint, and target state snapshots.
- `network_probes/`: service-path probe CSV/JSONL/summary files when configured.
- `workload/`: workload request traces or status.
- `verification_report.json` and `.md`: evidence that the intended signal appeared or did not.
- `cleanup_report.json` and `.md`: cleanup status and residual resource checks.
- `artifact_verification_report.json` and `.md`: file-shape and completeness checks.

Exact files vary by app and fault type. The invariant is that evidence and cleanup status are stored beside the run.

## Evidence Classes

The infrastructure collects multiple evidence classes:

- Metrics: time-series resource, service, app, RAG, training, GPU, and availability signals.
- Logs: raw app and pod logs from Kubernetes or Loki-style collection.
- Events: Kubernetes readiness, killing, restart, scheduling, warning, and failure events.
- Workload traces: user-facing request attempts, successes, failures, latency, and status.
- Network probes: direct service-path latency samples, often inside the cluster.
- State snapshots: deployments, pods, services, endpoints, selectors, and descriptions.
- Cleanup and verification: reports explaining whether the run is usable.

These data are raw or lightly serialized. App-specific dataset builders convert them into VLM-facing records.

## Model-Facing Data Is App-Specific

The infrastructure does not define one universal model prompt by itself. Instead, app-specific dataset builders choose which evidence becomes model-facing.

Examples:

- AI_1 RAG uses deterministic Stage 1 and Stage 2 VLM datapoints with canonical badness plots and packet-specific plots.
- Support-ticket triage and LLM summarization apps use app-specific stage builders and packet mappings.
- Distributed training uses training-specific labels, metrics, and stage views.
- Tool-using assistant data uses tool-call, routing, freshness, and evidence-context packet logic.

The infrastructure supplies the raw ingredients and reusable contracts.

## Leakage And Provenance Rules

For model-facing datasets, direct-answer leakage must be removed or excluded:

- Chaos Mesh controller/operator logs can reveal the injected object.
- Rendered fault manifests can reveal the answer.
- Template names, `fault_id`, `root_cause`, and private target metadata should not be model-facing.
- Absolute local paths should not appear in VLM prompts.

Legitimate symptoms are preserved:

- timeouts,
- HTTP errors,
- readiness failures,
- pod lifecycle events,
- retrieval/generation/schema symptoms,
- raw metrics and probes,
- app logs that describe observed behavior rather than the experimental injection plan.

## Validation Data

The infrastructure emits validation outputs at several levels:

- Fault library validation.
- App profile validation.
- Campaign preflight reports.
- Per-experiment sanity verification.
- Cleanup reports.
- Artifact shape verification.
- Dataset verification.
- Profile-pattern audits.
- Value-semantics audits.
- Automated datapoint review cards.
- Final quality and coverage reviews.

These reports are as important as the successful datapoints. They explain exclusions, recollections, weak effects, unsupported faults, and framework diagnostics.

## How To Cite Or Reuse This Layer

For paper writing, cite the infrastructure as the experimental control and reproducibility layer. It provides:

- executable fault definitions,
- app bindings,
- collection contracts,
- verification reports,
- cleanup guarantees,
- deterministic datapoint builders,
- audit reports for semantic correctness.

For model training, use the app dataset roots. For reproducing or extending experiments, use the published infrastructure code snapshot plus the app-specific dataset/source manifests.


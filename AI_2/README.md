# AI_2 Implementation README: GPU-Backed Support-Ticket Triage RCA Application

## Project Introduction

AI_2 is the support-ticket triage application for the SE-Paper multimodal root-cause analysis dataset effort. It was chosen because modern AI systems are rarely a single model call: they are pipelines of preprocessing, classification, model reasoning, routing, validation, retries, queues, and infrastructure dependencies. Those layers create a useful software-engineering research problem: the component that visibly fails is often not the component that owns the root cause.

The application simulates a production support-copilot backend. A support ticket enters the gateway, is normalized, classified, summarized by a GPU-backed model stage, routed by priority/team, and validated before it becomes a useful output. This makes it a compact but expressive testbed for failures that software engineers actually debug: schema-contract drift, malformed intermediate model output, retry amplification, queue buildup, dependency latency, pod availability loss, CPU/memory pressure, and GPU resource contention.

The topic is valuable for reviewers and dataset users because it links AI-specific behavior with classic software engineering concerns. It is not just "the model was wrong"; it shows how an AI service breaks through contracts, queues, dependency edges, validators, resource pressure, and orchestration events.

For dataset details, see `/lp-dev/yw2399/VISTA_Land/AI_2/data_readme.md` and the live dataset root `/lp-dev/yw2399/Dataset/AI_2`.

## Runtime Architecture

The application topology is:

```text
api_gateway -> preprocessor -> classifier -> summarizer_reasoner -> priority_router -> validator
```

Roles:

- `api_gateway`: public request entrypoint, downstream orchestration, final response surface.
- `preprocessor`: normalizes ticket fields and prepares structured inputs.
- `classifier`: assigns ticket category and confidence.
- `summarizer_reasoner`: GPU-backed model-carrying stage that produces structured summaries/reasoning.
- `priority_router`: converts classifier and summary output into priority/team routing.
- `validator`: enforces schema and business-contract validity for final triage output.

The Kubernetes deployment keeps one Deployment and Service per logical stage. Each stage exposes health, readiness, metrics, and stage-processing endpoints. The gateway calls the downstream services in order, which preserves dependency-edge evidence.

## Technology Selection

The application is implemented as a Python/FastAPI service with deterministic stage logic, structured JSON logs, Prometheus-style metrics, and Kubernetes/Helm deployment. The GPU-backed stage uses `gpu_lightweight`: a bounded deterministic CUDA tensor workload through PyTorch on A100 hardware. This was intentionally selected instead of a production LLM because the objective is controlled RCA evidence, not application model quality or fine-tuning.

Key implementation pieces:

- App package: `ticket_pipeline/apps/support_ticket_triage/support_ticket_triage/`
- Docker images: `ticket_pipeline/apps/support_ticket_triage/Dockerfile` and `Dockerfile.gpu`
- Helm chart: `ticket_pipeline/apps/support_ticket_triage/k8s/chart/`
- App profile: `Orchestrator/experiments/app_profiles/support_ticket_triage.yaml`
- Metric pack: `Orchestrator/orchestrator/metric_packs/support_ticket_triage.yaml`
- Dataset preparers: `Orchestrator/orchestrator/dataset/support_ticket_stage1.py` and `support_ticket_stage2.py`
- Labeler/verifier: `support_ticket_labels.py` and `support_ticket_verifier.py`
- Live/internal verification: `Orchestrator/orchestrator/verification/support_ticket_internal.py`
- Augmentation and repair scripts: `Orchestrator/scripts/run_support_ticket_ai2_augmentation.py`, `merge_ai2_frozen_and_augmented_dataset.py`, and `repair_ai2_timeseries_plots.py`

The publishable code bundle is copied to `/lp-dev/yw2399/VISTA_Publish/code/AI_2`.

## Orchestrator Integration

AI_2 reuses the shared Orchestrator. It does not introduce a second campaign runner, metric collector, or dataset framework. The app plugs into existing conventions through:

- app profile routing;
- fault binding and campaign YAML;
- Prometheus-compatible metrics;
- Kubernetes/Chaos Mesh external fault execution;
- app-fault API internal fault execution;
- Stage 1/Stage 2 dataset registry path;
- deterministic labeler/verifier registration;
- shared leakage/path/raw-ticket checks.

This was an important design constraint because App #1 RAG v2 was already frozen, and AI_2 needed to coexist without mutating RAG release artifacts.

## Internal Faults

The internal faults are application-level bugs exposed through the app-fault API and executed through Orchestrator lifecycle controls:

- `classifier_invalid_category`: classifier emits a category outside the allowed enum; validator becomes the failure surface.
- `summarizer_malformed_json`: GPU-backed summarizer emits malformed structured output; validator rejects it.
- `summarizer_slow_decode`: summarizer model-stage latency increases.
- `validator_retry_storm`: validator retry behavior amplifies failures.
- `batch_queue_saturation`: summarizer queue/backlog grows and delays useful output.
- `model_backend_timeout`: model backend path times out.
- `priority_invalid_team`: priority router emits an invalid target team.
- `classifier_confidence_collapse`: classifier output remains structurally valid but confidence degrades.
- `summarizer_gpu_compute_saturation_same_gpu`: bounded in-process GPU compute pressure on the same assigned GPU.
- `summarizer_gpu_memory_pressure_same_gpu`: bounded in-process GPU memory pressure on the same assigned GPU.

The same-GPU GPU faults were a key finding. Earlier helper-based GPU attempts created GPU helper activity but did not reliably affect the summarizer app path because helper placement alone did not guarantee same physical GPU contention. Those helper artifacts were excluded as no-effect diagnostics. The accepted GPU datapoints require same-device evidence and app-path evidence.

## External Faults

External faults use the shared Orchestrator and Kubernetes/Chaos Mesh mechanisms where applicable:

- `classifier_to_summarizer_network_latency`: dependency-path latency between classifier and summarizer.
- `classifier_pod_restart`: classifier replacement/readiness disruption.
- `validator_pod_unavailable`: validator availability loss.
- `summarizer_reasoner_pod_restart`: summarizer pod restart.
- `summarizer_cpu_pressure`: CPU pressure on summarizer.
- `summarizer_memory_pressure`: memory pressure on summarizer.
- `noncritical_stage_cpu_pressure_control`: CPU pressure on a noncritical stage, retained as a negative/control resource anomaly.

The control case is deliberately L1 `RESOURCE_ANOMALY`, not main RCA service degradation. This protects the label policy from over-escalating resource signals that do not create service impact.

## Dataset Preparation Methods

The data pipeline has two VLM stages.

Stage 1 is broad triage. It combines:

- five canonical time-series panels;
- normalized signal summaries;
- mechanical diagnostic summaries;
- sanitized high-level log evidence summaries;
- a JSON-only target schema for issue level, localization, suspected components/edges, and recommended Stage 2 packet.

Stage 2 is focused RCA. It combines:

- packet-specific plot;
- packet metric summary;
- normalized events;
- sanitized packet log snippets;
- target schema for root-cause type, scope, target, evidence, alternatives, debugging actions, and recovery actions.

Canonical signals use useful-throughput semantics. Useful throughput counts successful valid support-ticket triage outputs and excludes retries, health probes, malformed-control tickets, invalid final outputs, and validator-rejected outputs.

Plots were repaired late in the project to use real sampled time-series metrics, not one-point window summaries. Stage 2 packet plots were also repaired to use packet-relevant numeric signals instead of mostly unavailable placeholders.

## Interesting Findings

Several findings shaped the final dataset:

- Root-cause owner and failure surface must be separated. For example, malformed classifier or summarizer output is often exposed by the validator.
- GPU evidence must be coupled to the app path. Detached helper activity is not enough.
- Control cases need explicit split and severity policy. `noncritical_stage_cpu_pressure_control` stays negative/control at L1.
- VLM prompts need explicit output schemas. Terse instructions were insufficiently clear.
- Logs matter differently by stage. Stage 1 needs summarized evidence; Stage 2 needs packet-relevant snippets.
- Visual existence checks are not enough. The plot must be semantic evidence, not merely a nonblank PNG.
- Training-visible prompts must avoid answer-bearing fault IDs, artifact IDs, Chaos control logs, and raw ticket bodies.

## Data Augmentation and Splits

The final package merges one frozen 17-datapoint seed release with four additional GPU-backed augmentation rounds. This yields:

- 85 total datapoints;
- 80 main RCA datapoints;
- 5 negative/control datapoints;
- 85 Stage 1 VLM records;
- 85 Stage 2 VLM records.

Each round repeats the accepted 17 incident families with fresh baseline and live evidence. The package also includes ablation variants outside the main AI_2 root:

- `/lp-dev/yw2399/Dataset/AI_2_No_Image`
- `/lp-dev/yw2399/Dataset/AI_2_No_Metric`
- `/lp-dev/yw2399/Dataset/AI_2_Patch`

## Limitations

- The GPU-backed model stage uses deterministic CUDA workload, not a production LLM.
- Faults are controlled, not naturally occurring production incidents.
- The application is a compact six-stage service, not an enterprise support stack.
- Each augmentation round repeats the same 17 family templates.
- No-effect GPU helper diagnostics are retained in reports but excluded from the trainable main RCA dataset.

These limitations are intentional and documented so the dataset can be used honestly for multimodal RCA training and evaluation.

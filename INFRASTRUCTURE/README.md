# INFRASTRUCTURE Implementation README

This document describes the shared VISTA infrastructure: the controller, fault catalog, injection mechanism, collectors, cleanup system, validation layer, and dataset-building support used across the AI application datasets.

For a data/spec inventory of the infrastructure artifacts, see `/lp-dev/yw2399/VISTA_Land/INFRASTRUCTURE/data_readme.md`.

## Big Picture

The infrastructure exists because realistic root-cause-analysis datasets cannot be produced by labels alone. The project needs controlled incidents, raw operational evidence, reproducible transformations, and defensible decisions about whether a run is clean enough for training. VISTA therefore uses an external orchestrator that treats experiment execution as a lifecycle:

1. preflight,
2. baseline or pre-fault collection,
3. fault injection,
4. fault-window collection,
5. verification,
6. cleanup,
7. recovery checking,
8. artifact verification,
9. deterministic datapoint generation.

The key research idea is that RCA training examples should be grounded in real telemetry and real failure mechanisms. The infrastructure gives each application agent a common way to create those examples while still allowing app-specific internal faults and app-specific metric semantics.

## Why This Infrastructure Matters For Software Engineering Research

Software engineering reviewers should care about this infrastructure because it addresses three common weaknesses in incident-diagnosis datasets:

- The faults are executable and auditable, not just hand-written labels.
- The evidence is operationally realistic: logs, metrics, events, probes, state snapshots, workload traces, and recovery evidence.
- The dataset-building path includes correctness checks for value units, plot semantics, leakage, target/evidence consistency, and cleanup contamination.

The framework is especially relevant for AI applications because AI failures are not only pod crashes or CPU spikes. A system can be up while retrieval is empty, context construction violates a schema contract, a model backend waits on a dependency, or GPU pressure changes batching behavior. The infrastructure supports both classic distributed-systems faults and AI-pipeline-specific internal faults.

## Technology Selection

- Kubernetes provides real deployment, pod, service, readiness, event, and resource behavior.
- Chaos Mesh provides repeatable external fault injection for pod, CPU, memory, network, DNS, and I/O categories.
- Prometheus-style metrics provide structured time-series evidence.
- kubectl and Loki-style log collectors provide raw application, namespace, and pod logs.
- Active network probes provide service-path evidence when end-to-end request latency is ambiguous.
- GPU helper jobs provide bounded GPU stress because Chaos Mesh does not expose CUDA or VRAM fault CRDs.
- Python orchestrator modules provide reproducible lifecycle control and artifact layout.
- YAML fault catalogs make fault definitions inspectable and reusable across apps.

## Controller Architecture

The controller is the `Orchestrator` repository. Its central interface is `orchestrator/cli.py`, which exposes live and offline commands for experiments, campaigns, cleanup, verification, dataset construction, and review.

Important modules:

- `orchestrator/campaign.py`: sequential campaign execution and resume state.
- `orchestrator/single_experiment.py`: one-experiment lifecycle runner.
- `orchestrator/campaign_preflight.py`: live-safety checks before campaign execution.
- `orchestrator/runtime.py`, `orchestrator/runner.py`, `orchestrator/factory.py`, and `orchestrator/spec.py`: lifecycle plumbing and specification handling.
- `orchestrator/persistence.py` and `orchestrator/artifacts.py`: artifact directory and state persistence.
- `orchestrator/reporting/workflow_report.py`: human-readable workflow reporting.

The controller is intentionally external to the target apps. It does not need to be part of the application pods. That makes it easier to run, stop, inspect, and clean up experiments without changing app code for every external fault.

## Fault Catalog And External Bugs

The external fault catalog lives under `experiments/fault_catalog`.

Core reusable Chaos Mesh templates include:

- Single pod kill.
- Repeated pod restart.
- CPU saturation.
- Memory pressure.
- Dependency network latency.
- Packet loss.
- DNS failure.
- Disk I/O latency.
- CPU ramp.
- Memory leak ramp.
- Network latency ramp.

Compound templates combine multiple ingredients, for example CPU saturation plus replica loss, memory leak plus late restart, disk latency plus network latency, and packet loss plus pod flap.

GPU helper templates include:

- GPU compute saturation.
- GPU memory leak ramp.
- GPU memory fragmentation pattern.
- GPU throttling proxy.

The catalog separates reusable templates from app bindings. A template describes the fault mechanism; an app binding chooses targets, selectors, durations, intensity, evidence expectations, and labels.

## App-Level Internal Faults

External faults are not enough for AI-app RCA. The infrastructure also supports app-level internal faults through application adapters or app fault APIs. Examples across the application suite include:

- RAG empty retrieval.
- RAG retriever timeout.
- Poor top-k retrieval quality.
- Context too long after retrieval.
- Retrieval-to-context schema mismatch.
- Vector-store latency spike.
- Stale or corrupted index behavior.
- Generator waiting on retriever.
- Support-ticket invalid category or malformed summarizer output.
- LLM summarization slow tokenization, bad gateway behavior, and context/prefill overload.
- Distributed training checkpoint failures, CUDA OOM, rank worker crash, NCCL rendezvous failure, dataloader stall, and NaN/overflow.
- Tool-using assistant stale data, tool response timeout, bad routing, tool-loop budget exhaustion, and malformed tool response.

These internal faults are important because they model semantic or pipeline-level AI failures that classic infrastructure chaos does not cover.

## Injection Lifecycle

The injection pipeline is:

- Load a fault template and app binding from YAML.
- Validate the schema and app target abstraction.
- Render concrete resources or payloads.
- Start collectors for metrics, logs, probes, state, and workload evidence.
- Run a pre-fault or known-good gate when the campaign requires it.
- Apply the fault through Chaos Mesh, the GPU helper, or an app fault endpoint.
- Monitor for expected duration and collect evidence.
- Run sanity verification to check whether the intended symptom actually appeared.
- Remove the fault and run cleanup.
- Verify recovery and check for leftover resources.
- Persist artifact verification and campaign summary files.

The framework treats incomplete cleanup as a dataset hazard. If a Chaos Mesh resource or helper job survives, the next experiment can be contaminated, so cleanup status is part of whether a run is usable.

## Collectors

The infrastructure collects several evidence classes:

- Prometheus metric CSVs for resource, service, app, RAG, training, GPU, and readiness signals.
- Kubernetes pod logs and namespace logs.
- Loki logs when configured.
- Kubernetes events, pod descriptions, deployment state, endpoints, and service state.
- Workload traces for user-facing request success/failure.
- Network probe CSV/JSONL records for service-path latency.
- Fault status snapshots and app fault state where supported.
- Cleanup and verification reports.

Collectors are intentionally raw-first. Later dataset preparers may normalize or summarize evidence, but the raw artifacts remain available for reproducibility.

## Verification And Cleanup

Verification asks whether the run showed the intended behavior. It can use metrics, logs, probes, workload status, event patterns, or app fault state. Verification prevents a successful YAML apply from being mistaken for a useful datapoint.

Cleanup handles:

- Chaos Mesh resources.
- GPU helper jobs and pods.
- App fault toggles.
- Collector jobs.
- Workload recovery.
- IOChaos finalizer edge cases.
- Residual resource checks.

Cleanup reports are explicit. A run can be clean, clean with warnings, recovered after restart, skipped unsupported, or require manual intervention.

## Dataset Preparation Support

The infrastructure includes deterministic dataset builders under `orchestrator/dataset`.

Key capabilities include:

- App profile loading and validation.
- Baseline collection and stable baseline summaries.
- Metric semantics and value-unit validation.
- Stage 1 / Stage 2 datapoint preparation.
- Log normalization and severity handling.
- VLM record generation.
- Profile-pattern coverage audits.
- Value-semantics audits.
- Automated datapoint review cards.
- App-specific dataset final review tools.

The dataset layer is offline. It does not inject faults or call LLM APIs when preparing deterministic datapoints.

## Interesting Findings

Several infrastructure lessons became first-class engineering rules:

- Network fault proof must be path-specific. End-to-end latency alone can hide a dependency-path delay, so service probes are needed.
- IOChaos is not portable across every local Kubernetes runtime and volume path. Capability checks are required before treating storage faults as available.
- GPU faults should be bounded helper jobs with explicit cleanup, not host-level manual GPU changes.
- Pre-fault gates matter. A fault run with baseline user-facing failures can be useful as a diagnostic artifact but should not become a clean training datapoint.
- Metric unit semantics must be explicit. Raw counts, cumulative counters, booleans, bytes, percentages, rates, and normalized badness values are not interchangeable.
- Logs require careful severity semantics. Message text can indicate an important compatibility event, but parsed log level must remain the source of severity truth.
- Direct-answer leakage must be purged from model-facing datasets. Chaos Mesh controller logs and rendered fault manifests can reveal the answer too directly.

## Published Code Snapshot

The infrastructure code snapshot is at `/lp-dev/yw2399/VISTA_Publish/code/INFRASTRUCTURE`.

The snapshot contains:

- `orchestrator_repo/orchestrator`
- `orchestrator_repo/experiments`
- `orchestrator_repo/chaos-mesh`
- `orchestrator_repo/scripts`
- `orchestrator_repo/tests`
- `orchestrator_repo/gpu_helper`
- `orchestrator_repo/examples`
- root files such as `README.md`, `Makefile`, and `pyproject.toml`

The snapshot excludes generated artifacts, app datasets, pycache, and temporary outputs.

## Reproduction Commands

Common offline commands:

```bash
cd /lp-dev/yw2399/VISTA_Publish/code/INFRASTRUCTURE/orchestrator_repo
PYTHONPATH=. python3 -m orchestrator.cli validate-fault-library
PYTHONPATH=. python3 -m orchestrator.cli validate-rag-v2-app-profile --app-profile orchestrator/app_profiles/rag-demo-v2.yaml
PYTHONPATH=. pytest -q tests/test_fault_library.py tests/test_chaos_mesh_catalog_renderer.py tests/test_cleanup.py
```

Example preflight and campaign commands, to be used only when live execution is intended and the Kubernetes context is correct:

```bash
PYTHONPATH=. python3 -m orchestrator.cli preflight-campaign \
  --campaign experiments/campaigns/rag_smoke_campaign.yaml \
  --output-dir artifacts/campaign_preflights

PYTHONPATH=. python3 -m orchestrator.cli run-experiment-campaign \
  --campaign experiments/campaigns/rag_smoke_campaign.yaml \
  --mode live \
  --output-dir artifacts/campaigns
```

Live commands mutate Kubernetes. They should not be run during documentation review.

## Known Limitations

- Chaos Mesh behavior depends on cluster CNI, CRDs, permissions, runtime, and namespace layout.
- IOChaos is deliberately treated as optional unless a current canary proves support for the target path.
- GPU helper evidence depends on GPU nodes, NVIDIA device plugin, and DCGM visibility.
- App-specific internal faults require app support; the infrastructure can orchestrate them but cannot invent app fault endpoints.
- The source repo may contain unrelated dirty work from current application development. The published snapshot is a curated copy and should be used for infrastructure review.


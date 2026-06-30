# AI_5 Implementation README

AI_5 is the VISTA tool-using AI operations agent. It is a GPU-backed, agentic service used to generate root-cause-analysis data for multimodal models. The application receives an incident-diagnosis request, asks a vLLM-served Qwen model for structured actions, dispatches actual HTTP tool calls, validates tool observations, enforces runtime budgets, and returns a structured final diagnosis.

For the dataset itself, see `/lp-dev/yw2399/VISTA_Land/AI_5/data_readme.md`.

## Big Picture

AI_5 was built to represent a modern software-engineering pattern: LLMs embedded into operational workflows rather than used as isolated chat systems. The application behaves like an AI operations assistant that can consult service status, metrics, logs, deployment history, runbooks, and incident history.

This is useful for VISTA because failures in agentic software cross boundaries that ordinary microservice datasets often miss:

- model action generation and structured-output validation;
- agent control-loop budgets and repeated action behavior;
- HTTP tool latency, availability, parse, and schema contracts;
- stale or freshness-invalid tool data;
- context packing and evidence truncation;
- Kubernetes pod readiness and service-to-service network edges;
- GPU model-serving infrastructure.

The application therefore gives paper reviewers and dataset users examples where "the service is down" is not enough. The model must distinguish whether the faulty locus is a component, edge, tool output contract, data source, agent policy, context window, or orchestration event.

## Runtime Architecture

The normal topology is:

```text
workload_driver -> agent_api -> vllm_engine
agent_api -> service_status_tool
agent_api -> metrics_query_tool
agent_api -> log_search_tool
agent_api -> deployment_history_tool
agent_api -> runbook_lookup_tool
agent_api -> incident_db_tool
```

The app side is deployed as Kubernetes services with Helm. The model side is a host-Docker vLLM service reached by the app through `VLLM_BASE_URL`.

Main components:

- `agent_api`: user-facing diagnosis API and action-loop controller.
- `vllm_engine`: external OpenAI-compatible model backend.
- `service_status_tool`: service health and freshness evidence.
- `metrics_query_tool`: metrics evidence.
- `log_search_tool`: log evidence.
- `deployment_history_tool`: deployment/change and pod-availability evidence.
- `runbook_lookup_tool`: runbook dependency used by network-edge records.
- `incident_db_tool`: historical incident evidence and diagnostic target for rejected OOM qualification.

## Model Runtime

The selected runtime is throughput-first. The dataset is the deliverable; model answer quality is not a gate.

```text
model: Qwen/Qwen3-4B-Instruct-2507
revision: cdbee75f17c01a7cc42f958dc650907174af0554
runtime image: vllm/vllm-openai@sha256:05a31dc4185b042e91f4d2183689ac8a87bd845713d5c3f987563c5899878271
GPU count: 1 A100
tensor parallel size: 1
dtype: bfloat16
quantization: none
CPU offload: 0
vLLM swap: 0
max model length: 8192
per-action output budget: <= 256 tokens
```

The model emits JSON actions under structured decoding, and every live action is validated against the canonical App 5 action schema before dispatch or finalization. Runtime outcomes are tracked separately from model final statuses.

## Action Protocol

Canonical action fields:

```text
action_type
tool_name
arguments
status
diagnosis_code
evidence_item_ids
public_explanation
```

Action types:

```text
call_tool
final_answer
```

Model final statuses:

```text
diagnosed
insufficient_evidence
```

The runtime includes one bounded repair completion for invalid model output, plus deterministic deadlines, retries, and tool/step budgets.

## Implementation Details

The source implementation is in:

```text
/home/yw2399/Demo_Apps/tool_ops_assistant/apps/tool_using_ai_operations_assistant
```

The publish snapshot is in:

```text
/lp-dev/yw2399/VISTA_Publish/code/AI_5
```

Important code areas:

- `src/tool_using_ai_operations_assistant/`: FastAPI app, agent loop, model client, tool handlers, metrics, constants, and fault logic.
- `schemas/`: JSON schemas for model actions, tool queries/responses, and fault controls.
- `fixtures/`: public workload and tool-visible fixture roots.
- `k8s/`: Helm chart and Kubernetes manifests.
- `scripts/run_milestone*.py`: live qualification, source gates, campaign execution, release closure, expansion, and validation scripts.
- `scripts/export_ai5_dataset.py`: final AI_5 export builder for RAW, Single_Shot, and VLM views.
- `scripts/repair_ai5_vlm_text.py`: deterministic VLM prompt-text quality repair.
- `scripts/repair_ai5_vlm_task_sections.py`: JSON-only task-section repair.
- `tests/`: runtime and source-gate regression tests.
- `preflight/`: model runtime candidate and preflight evidence.

## Internal Faults

Accepted internal families:

- `tool_response_latency_timeout`: target tool is too slow for the agent/tool deadline.
- `tool_http_5xx_response`: target tool returns server errors.
- `malformed_tool_json`: tool returns invalid JSON.
- `tool_schema_contract_violation`: tool returns parseable JSON that violates schema.
- `stale_tool_observation`: tool returns well-formed but stale data.
- `repeated_action_budget_exhaustion`: agent repeats actions until the runtime budget is exhausted.
- `evidence_context_truncation`: context packing omits needed evidence.

These are intentionally agent/tool failures rather than generic Kubernetes failures. They were selected because they are plausible in real LLM-agent systems and because their source evidence can be collected without revealing the target label.

## External Faults

Accepted external families:

- `dependency_path_latency`: network/path degradation on `agent_api->runbook_lookup_tool`.
- `unexpected_pod_restart_readiness_gap`: Kubernetes readiness/restart gap affecting `deployment_history_tool`.

Attempted but rejected:

- OOM/StressChaos qualification against `incident_db_tool`. Three attempts were run, but `0` datapoints were accepted because the source gate did not prove the full OOM causal chain.

This rejection is an important finding. The project did not accept data just because a chaos mechanism was applied; it required independent evidence of activation, local effect, user-visible impact, recovery, and target support.

## Dataset Construction Findings

The final dataset contains 81 datapoints and 162 VLM stage records. Each accepted family has nine independent executions: primary plus variations `varA` through `varH`.

Interesting engineering findings:

- Smaller model infrastructure was better for this dataset than a larger model. Qwen3-4B was sufficient to drive real structured actions while keeping the campaign practical on one A100.
- Tool/action telemetry is itself a rich software-engineering signal. Some root causes are invisible in ordinary CPU/memory metrics but clear in action validation, tool dispatch, or context-packing records.
- VLM prompt text needed human/agent review after mechanical export. The final text repair removed irrelevant redaction notes, added deterministic log/event summaries, added detailed Stage 2 rows, and made JSON-only task contracts explicit.
- Packet and folder names can leak operational categories. Training/evaluation should not expose paths, `EXP_ID`, or target files as prompt content.
- Selected/gold Stage 2 packets are materialized; all non-selected candidate packets are not present in the export.

## Published Code Snapshot

The code has been copied to:

```text
/lp-dev/yw2399/VISTA_Publish/code/AI_5
```

The snapshot contains the application implementation, schemas, fixtures, scripts, tests, Kubernetes files, selected Orchestrator references, and paper stats artifacts. It intentionally does not contain the large dataset payload. Use `/lp-dev/yw2399/Dataset/AI_5` for data.

## Validation Pointers

Primary validation artifacts:

```text
/lp-dev/yw2399/Dataset/AI_5/export_validation.json
/lp-dev/yw2399/Dataset/AI_5/dataset_manifest.json
/lp-dev/yw2399/Dataset/AI_5/purge_summary.json
/lp-dev/yw2399/Dataset/AI_5/vlm_repair_validation.json
/lp-dev/yw2399/Dataset/AI_5/vlm_task_section_repair_validation.json
/lp-dev/yw2399/Dataset/AI_5/SHA256SUMS
```

Paper-facing summaries:

```text
/home/yw2399/Demo_Apps/docs/Handoffs/AI_5.md
/home/yw2399/Demo_Apps/docs/Paper/Q_Sec_3/AI_5/AI_5_report.md
/home/yw2399/Demo_Apps/docs/Paper/Q_Sec_3/AI_5/AI_5_stats.json
```

## Known Limitations

- No healthy or insufficient-evidence datapoints are included in the final trainable dataset.
- OOM qualification was rejected and must not be counted as an accepted family.
- Packet names and folder names may act as shortcuts if exposed to models.
- The exported VLM corpus has one selected/gold Stage 2 packet per datapoint.
- The topology is compact: one agent, six tools, Kubernetes services, and one external vLLM backend.

## Recommended Usage

Use `/lp-dev/yw2399/Dataset/AI_5/VLM` for multimodal Stage 1/Stage 2 training and evaluation. Use `/lp-dev/yw2399/Dataset/AI_5/Single_Shot` for raw-evidence single-shot baselines. Use `/lp-dev/yw2399/Dataset/AI_5/RAW` only for provenance, rerendering, or raw-evidence experiments with explicit leakage controls.

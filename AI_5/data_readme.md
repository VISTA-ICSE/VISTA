# AI_5 Dataset README

AI_5 is the VISTA dataset for the Tool-Using AI Operations Agent. It contains raw experiment artifacts, sanitized single-shot raw-evidence inputs, VLM-facing two-stage records, aggregate VLM JSONL manifests, prompt repair reports, leakage/purge summaries, and checksums.

## Dataset Location

- Canonical dataset root: `/lp-dev/yw2399/Dataset/AI_5`
- Repository symlink: `/home/yw2399/Demo_Apps/Dataset/ai_5`
- Implementation notes: `/lp-dev/yw2399/VISTA_Land/AI_5/README.data`
- Published code: `/lp-dev/yw2399/VISTA_Publish/code/AI_5`
- Handoff: `/home/yw2399/Demo_Apps/docs/Handoffs/AI_5.md`

## Dataset Status

The dataset is final for paper characterization and model-training use. It was exported from the accepted App 5 v2 release and then repaired at the VLM text/task-section layer. The final export validation passed.

Important checks:

- `export_validation.json`: passed
- `vlm_repair_validation.json`: passed
- `vlm_task_section_repair_validation.json`: passed
- `SHA256SUMS`: regenerated after repairs and verified by validation
- Single_Shot leakage scan: passed
- VLM plots and targets: source-equivalent after text repairs

## Top-Level Layout

```text
/lp-dev/yw2399/Dataset/AI_5/
  RAW/
  Single_Shot/
  VLM/
  VLM_sft_qwen/
  dataset_manifest.json
  export_validation.json
  purge_summary.json
  raw_copy_summary.json
  vlm_quality_repair_report.json
  vlm_quality_repair_report.md
  vlm_repair_validation.json
  vlm_task_section_repair_report.json
  vlm_task_section_repair_report.md
  vlm_task_section_repair_validation.json
  SHA256SUMS
```

Each datapoint uses:

```text
EXP_ID = {datapoint_id}__{source_incident}__{role}
```

Examples:

```text
app5_m9_dp_0001__latency_timeout__primary
app5_m9_dp_0081__pod_restart_readiness__varH
```

Do not expose `EXP_ID` to a model unless the experiment explicitly studies shortcut leakage. The source incident slug can reveal the family.

## Dataset Counts

- Incident families: `9`
- Independent datapoints: `81`
- RAW records: `81`
- Single_Shot records: `81`
- VLM datapoint directories: `81`
- Stage 1 VLM records: `81`
- Stage 2 VLM records: `81`
- Total VLM stage records: `162`
- Healthy/no-fault records in final dataset: `0`
- Diagnostic/insufficient-evidence records in final dataset: `0`

Issue-level distribution:

```text
issue_level 2: 18
issue_level 3: 54
issue_level 4: 9
```

## Incident Families

| Source incident | Root-cause type | Target | Stage 2 packet | Executions |
|---|---|---|---|---:|
| `latency_timeout` | `tool_response_latency_timeout` | `metrics_query_tool` | `pipeline_stage_latency_packet` | 9 |
| `http_5xx` | `tool_http_5xx_response` | `deployment_history_tool` | `tool_http_availability_packet` | 9 |
| `malformed_json` | `malformed_tool_json` | `log_search_tool` | `malformed_intermediate_output_packet` | 9 |
| `schema_invalid_json` | `tool_schema_contract_violation` | `log_search_tool` | `schema_validation_packet` | 9 |
| `stale_data` | `stale_tool_observation` | `service_status_tool` | `tool_data_freshness_packet` | 9 |
| `loop_budget_exhaustion` | `repeated_action_budget_exhaustion` | `agent_api` | `agent_orchestration_packet` | 9 |
| `evidence_truncation` | `evidence_context_truncation` | `agent_api` | `agent_context_truncation_packet` | 9 |
| `agent_api_to_runbook_network_latency` | `dependency_path_latency` | `agent_api->runbook_lookup_tool` | `network_edge_packet` | 9 |
| `pod_restart_readiness` | `unexpected_pod_restart_readiness_gap` | `deployment_history_tool` | `pod_availability_packet` | 9 |

## RAW View

Path:

```text
/lp-dev/yw2399/Dataset/AI_5/RAW/<EXP_ID>/
```

Purpose: provenance, rerendering, audit, and future format generation.

Typical contents:

- copied source attempt root under `source_root/`;
- release datapoint metadata under `datapoint_release_view/`;
- workload request/response records;
- agent events, model calls, tool calls, final answers;
- tool service events and HTTP access logs;
- Kubernetes events and snapshots;
- source overlays and query records;
- artifact validation and source-reference files;
- raw manifests and checksums.

RAW may contain target labels, fault-control records, chaos/control evidence, final answers, and other audit-only files. Do not use RAW directly as model-facing training input unless you are intentionally running a raw-evidence experiment with a leakage purge.

## Single_Shot View

Path:

```text
/lp-dev/yw2399/Dataset/AI_5/Single_Shot/<EXP_ID>/
```

Purpose: raw-log/raw-metric single-shot LLM tests.

Layout:

```text
all/input.txt
all/output.txt
stage1/input.txt
stage1/plot.png
stage1/output.txt
stage2/input.txt
stage2/plot.png
stage2/output.txt
purge_manifest.json
```

Single_Shot inputs concatenate sanitized raw logs/events first, then serialized metric content. The purge excludes direct answer leakage such as fault-control state, chaos controller records, target files, final answers, evaluator results, private fault IDs, and explicit injection strings.

Ordinary production-like symptoms are retained when they do not reveal the answer directly. Examples include timeouts, HTTP 5xx, DNS-like failures, readiness gaps, stale data symptoms, schema errors, retry/budget exhaustion, and slow responses.

Use `purge_summary.json` and per-datapoint `purge_manifest.json` to inspect inclusion/exclusion decisions.

## VLM View

Path:

```text
/lp-dev/yw2399/Dataset/AI_5/VLM/<EXP_ID>/
```

Each datapoint has:

```text
stage1/input_text.md
stage1/plot.png
stage1/target.json
stage2/input_text.md
stage2/plot.png
stage2/target.json
dataset_manifest.json
```

Aggregate VLM manifests:

```text
/lp-dev/yw2399/Dataset/AI_5/VLM/stage1_records.jsonl
/lp-dev/yw2399/Dataset/AI_5/VLM/stage2_records.jsonl
/lp-dev/yw2399/Dataset/AI_5/VLM/combined_records.jsonl
```

Stage 1 is broad triage. It contains:

- topology;
- workload summary;
- five-panel mechanical summary;
- component/edge evidence;
- resource/workload evidence;
- deterministic event/log summaries;
- missing-signal notes;
- explicit JSON-only task schema.

Stage 1 target fields:

```text
issue_level
localization
suspected_components
suspected_edges
recommended_stage2_focus
rationale_brief
```

Stage 2 is selected packet RCA. It contains:

- selected packet focus;
- packet plot representation;
- workload impact;
- evidence for selected focus;
- evidence against alternatives;
- deterministic event/log summaries;
- relevant detailed packet rows;
- embedded normalized evidence;
- debugging and recovery evidence;
- explicit JSON-only task schema.

Stage 2 target fields:

```text
issue_level
root_cause_type
root_cause_scope
root_cause_target
resource_root_cause
service_incident_root_cause
evidence_for
evidence_against_alternatives
recommended_debugging_actions
recommended_recovery_actions
```

## VLM Text Repairs

The initial VLM export was mechanically valid but was later reviewed and repaired for better model-facing quality.

Repairs applied:

- removed irrelevant redaction-note content;
- rewrote contract headers;
- added deterministic Stage 1 and Stage 2 event/log summaries;
- added relevant detailed Stage 2 packet rows;
- removed over-direct candidate causal scope sections;
- added explicit JSON-only task schemas.

Repair counts:

- prompt files repaired: `162`
- Stage 1 records repaired: `81`
- Stage 2 records repaired: `81`
- plots changed: `0`
- targets changed: `0`

Validation files:

```text
vlm_repair_validation.json
vlm_quality_repair_report.json
vlm_quality_repair_report.md
vlm_task_section_repair_validation.json
vlm_task_section_repair_report.json
vlm_task_section_repair_report.md
```

## Leakage Guidance

Safe model-facing inputs:

- `VLM/<EXP_ID>/stage1/input_text.md`
- `VLM/<EXP_ID>/stage1/plot.png`
- `VLM/<EXP_ID>/stage2/input_text.md`
- `VLM/<EXP_ID>/stage2/plot.png`
- `Single_Shot/<EXP_ID>/*/input.txt` for raw-evidence single-shot experiments

Do not expose as prompts:

- `target.json`;
- `output.txt` when it is the answer side;
- `EXP_ID` or parent directory names;
- RAW fault-control files;
- chaos/controller records;
- evaluator results;
- release audit reports;
- final model answers from live runs;
- private fault IDs or source incident slugs.

Packet names can reveal operational category. For shortcut-resistant evaluation, replace model-facing `selected_packet` strings with neutral IDs such as `packet_A` while preserving the packet evidence text.

## Checksums And Validation

From the dataset root:

```bash
cd /lp-dev/yw2399/Dataset/AI_5
sha256sum -c SHA256SUMS
```

The final task-section repair validation reported:

```text
checked: 243173
errors: 0
passed: true
```

The export validation reported:

```text
datapoint_count: 81
expected_full_release_datapoints: 81
expected_full_release_vlm_records: 162
view_directory_counts: RAW=81, Single_Shot=81, VLM=81
required_files_non_empty: true
png_valid: true
single_shot_leakage_scan_passed: true
single_shot_targets_match_source: true
vlm_targets_and_plots_match_source: true
vlm_text_repaired: true
vlm_task_section_repaired: true
passed: true
```

## Recommended Use

For standard multimodal VLM training/evaluation:

```text
/lp-dev/yw2399/Dataset/AI_5/VLM
```

For single-shot raw evidence experiments:

```text
/lp-dev/yw2399/Dataset/AI_5/Single_Shot
```

For audit, rerendering, or future dataset formats:

```text
/lp-dev/yw2399/Dataset/AI_5/RAW
```

For paper statistics:

```text
/home/yw2399/Demo_Apps/docs/Paper/Q_Sec_3/AI_5/AI_5_report.md
/home/yw2399/Demo_Apps/docs/Paper/Q_Sec_3/AI_5/AI_5_stats.json
```

## Known Limitations

- No final healthy/control/insufficient-evidence records are included.
- The exported VLM corpus has one selected/gold Stage 2 packet per datapoint.
- OOM was attempted but rejected; do not treat it as an accepted family.
- Folder names and packet names can provide shortcuts if exposed.
- The topology is representative but compact: one agent, six tools, one external vLLM backend, and one Kubernetes namespace.

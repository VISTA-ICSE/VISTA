# AI_2 Flat-Test Deterministic Scoring

This project publishes the deterministic scorer used to evaluate VISTA model-facing predictions for the AI_2 support-ticket-triage flat-test single-shot evaluation.

For the dataset inventory, file layout, scoring outputs, and measured metrics, see:

- `/lp-dev/yw2399/VISTA_Land/AI_2_flat_test_scoring/data_readme.md`

## Purpose

The VISTA datasets use structured two-stage RCA targets. Human and frontier-model predictions can be syntactically valid JSON while still being wrong in operationally important ways. This scorer turns those predictions into reproducible paper metrics without asking another LLM to judge correctness.

The scorer supports two input styles:

1. Qwen/VLM SFT JSONL predictions under `test/**/predictions.jsonl`.
2. Flat single-shot folders such as `/lp-dev/yw2399/Dataset/AI_2/Single_Shot/flat_test`, where each datapoint has `stage1/` and `stage2/` gold/output files plus provider response files.

The current project is focused on the second mode.

## Implementation Details

Main entrypoint:

```text
score_vista_predictions.py
```

Core implementation choices:

- Stage 1 scoring focuses on `localization`, `recommended_stage2_focus`, `suspected_components`, and `suspected_edges`.
- Stage 2 scoring focuses on `root_cause_type`, `root_cause_scope`, `root_cause_target`, `resource_root_cause`, and `service_incident_root_cause`.
- `issue_level` is ignored because severity and RCA correctness are separate concepts.
- Predictions are parsed from raw text by extracting a balanced JSON object and stripping simple Markdown fences when needed.
- Canonicalization normalizes case, whitespace, punctuation, underscore/hyphen variants, smart quotes, and declared aliases.
- No substring matching is used. Semantic credit requires exact canonical equality after normalization/aliasing.
- Raw-source and review-report paths are blocklisted as scoring inputs.
- Dry-run mode parses and canonicalizes predictions but is intended for alias review rather than final scoring.

Alias files:

- `aliases/global_aliases.yaml`: cross-app aliases for common components, packets, root-cause types, resources, and non-diagnosable outcomes.
- `aliases/ai2_aliases.yaml`: AI_2 support-ticket-triage aliases for classifier, summarizer/reasoner, priority router, validator, malformed-output packets, GPU/CPU/memory resources, retry storm, and model-backend outcomes.

The code also includes `tools/rerun_gpt_output_limit_8196.py`, a dataset-specific helper used to retry GPT flat-test rows that previously exhausted maximum output tokens. It updates only successful retries and leaves failed artifacts untouched.

## Run Commands

Full AI_2 flat-test scoring:

```bash
cd /home/yw2399/Demo_Apps
python scoring/score_vista_predictions.py \
  --flat-dir /lp-dev/yw2399/Dataset/AI_2/Single_Shot/flat_test \
  --flat-stages stage1,stage2 \
  --alias-dir scoring/aliases \
  --out-dir /lp-dev/yw2399/Dataset/AI_2/Single_Shot/flat_test/scoring/output \
  --allow-train
```

Dry-run alias review:

```bash
cd /home/yw2399/Demo_Apps
python scoring/score_vista_predictions.py \
  --flat-dir /lp-dev/yw2399/Dataset/AI_2/Single_Shot/flat_test \
  --flat-stages stage1,stage2 \
  --alias-dir scoring/aliases \
  --out-dir /lp-dev/yw2399/Dataset/AI_2/Single_Shot/flat_test/scoring/dry_run \
  --allow-train \
  --dry-run
```

Published-code run equivalent:

```bash
cd /lp-dev/yw2399/VISTA_Publish/code/AI_2_flat_test_scoring
python score_vista_predictions.py \
  --flat-dir /lp-dev/yw2399/Dataset/AI_2/Single_Shot/flat_test \
  --flat-stages stage1,stage2 \
  --alias-dir aliases \
  --out-dir /lp-dev/yw2399/Dataset/AI_2/Single_Shot/flat_test/scoring/output \
  --allow-train
```

## Interesting Findings

The AI_2 flat-test predictions are syntactically clean after retrying output-token failures: the latest scored run has 52/52 JSON parse success and 52/52 schema validity.

Stage 1 is much easier for frontier models than Stage 2. Overall Stage 1 routing accuracy is 18/26, while Stage 2 operational RCA correctness is 7/26. The gap is informative: models often identify the broad broken area but struggle to produce the exact canonical root-cause type, scope, target, resource, and service incident mechanism.

Claude Opus performed better than GPT on this small AI_2 flat-test subset:

- Claude Opus Stage 1 routing: 11/13.
- GPT Stage 1 routing: 7/13.
- Claude Opus Stage 2 operational RCA: 5/13.
- GPT Stage 2 operational RCA: 2/13.

Strict full JSON match is intentionally reported only as a diagnostic. It is too brittle for paper-facing RCA correctness because a prediction can be operationally right while differing in rationale/action wording.

## Lessons For Future Scoring

- Keep aliases conservative. `unmapped_values.csv` is a feature, not a nuisance; it shows where model language diverges from canonical targets.
- Never let raw-source paths or review reports into scoring input discovery.
- Treat train/val/test split handling carefully. The flat-test dataset is not split like SFT JSONL, so this scorer uses `--allow-train` for these single-shot folders.
- Keep Stage 1 and Stage 2 metrics separate; combining them hides where models fail.
- If one-stage flat RCA is evaluated later, score it as `flat_one_stage` rather than mixing it into oracle Stage 2.

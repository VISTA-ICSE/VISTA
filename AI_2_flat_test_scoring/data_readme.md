# Data README: AI_2 Flat-Test Scoring Dataset

## Dataset Location

Primary dataset:

```text
/lp-dev/yw2399/Dataset/AI_2/Single_Shot/flat_test/
```

Scoring outputs:

```text
/lp-dev/yw2399/Dataset/AI_2/Single_Shot/flat_test/scoring/output/
```

Dry-run alias-review outputs:

```text
/lp-dev/yw2399/Dataset/AI_2/Single_Shot/flat_test/scoring/dry_run/
```

## Dataset Shape

The current flat-test subset contains 13 datapoints:

```text
ai2_dp_0001
ai2_dp_0016
ai2_dp_0030
ai2_dp_0040
ai2_dp_0042
ai2_dp_0051
ai2_dp_0054
ai2_dp_0058
ai2_dp_0068
ai2_dp_0071
ai2_dp_0072
ai2_dp_0081
ai2_dp_0082
```

Each datapoint has:

```text
ai2_dp_xxxx/
  all/
    input.txt
    output.txt
    gpt_err.txt or response artifact when present
    opus_err.txt or response artifact when present
  stage1/
    input.txt
    vlm_input.txt
    plot.png
    output.txt
    gpt_repsonse.txt
    opus_response.txt
  stage2/
    input.txt
    vlm_input.txt
    plot.png
    output.txt
    gpt_repsonse.txt
    opus_response.txt
  merge_metadata.json
  single_shot_manifest.json
```

The filename `gpt_repsonse.txt` is misspelled in the dataset and is supported as-is by the scorer.

## What Is Scored

The current scoring run includes only:

- `stage1`
- `stage2`

The `all/` folders are present but were not included in the latest output. They can be added later with `--flat-stages stage1,stage2,all` when one-stage response files are available.

For each datapoint and stage, the scorer evaluates two provider outputs:

- `gpt-5.5-pro`
- `claude-opus-4-8`

That gives:

- 13 Stage 1 GPT records
- 13 Stage 1 Claude/Opus records
- 13 Stage 2 GPT records
- 13 Stage 2 Claude/Opus records
- 52 total scored records

## Gold Targets

Gold targets are read from the model-facing `output.txt` files:

- Stage 1 gold: `stage1/output.txt`
- Stage 2 gold: `stage2/output.txt`

Stage 1 core fields:

- `localization`
- `recommended_stage2_focus`
- `suspected_components`
- `suspected_edges`

Stage 2 core fields:

- `root_cause_type`
- `root_cause_scope`
- `root_cause_target`
- `resource_root_cause`
- `service_incident_root_cause`

The scorer ignores `issue_level` by design.

## Scoring Outputs

The latest output directory contains:

```text
ambiguous_values.csv
error_decomposition.csv
json_parse_failures.csv
per_datapoint_scores.csv
per_datapoint_scores.jsonl
scoring_summary.md
summary_by_app.csv
summary_by_family.csv
summary_by_model_and_mode.csv
unmapped_values.csv
```

Important output roles:

- `scoring_summary.md`: human-readable summary of the run.
- `per_datapoint_scores.csv`: one row per scored prediction with field-level and aggregate correctness flags.
- `per_datapoint_scores.jsonl`: machine-readable equivalent with nested details.
- `summary_by_model_and_mode.csv`: paper-friendly model/mode metrics.
- `summary_by_family.csv`: family-level correctness.
- `unmapped_values.csv`: canonicalization misses and model values needing alias review.
- `ambiguous_values.csv`: alias conflicts; currently empty.
- `json_parse_failures.csv`: parse failures; currently empty.
- `error_decomposition.csv`: populated only when paired Stage 1 and E2E Stage 2 predictions are available; currently empty.

## Latest Results

Overall:

- Records: 52
- Completed records: 52
- Completion rate: 1.0
- JSON parse success: 52/52
- Schema valid: 52/52
- Malformed JSON: 0
- Stage 1 records: 26
- Stage 1 routing correct: 18/26
- Stage 1 routing accuracy: 0.692308
- Stage 1 localization accuracy: 0.846154
- Stage 1 suspect hit rate: 0.923077
- Stage 2 records: 26
- Stage 2 operational RCA correct: 7/26
- Stage 2 operational RCA accuracy: 0.269231
- Stage 2 strict accuracy: 0.0

By model and mode:

| Model | Mode | Records | Primary score |
|---|---|---:|---:|
| `claude-opus-4-8` | `stage1_triage` | 13 | routing accuracy 0.846154 |
| `gpt-5.5-pro` | `stage1_triage` | 13 | routing accuracy 0.538462 |
| `claude-opus-4-8` | `stage2_oracle` | 13 | operational RCA accuracy 0.384615 |
| `gpt-5.5-pro` | `stage2_oracle` | 13 | operational RCA accuracy 0.153846 |

## Known Data Notes

- The dataset is a flat-test frontier-model evaluation set, not the full AI_2 training dataset.
- The scorer runs with `--allow-train` because these flat-test folders are not named by SFT train/val/test JSONL conventions.
- Provider responses may contain Markdown fences; the scorer strips simple outer fences before JSON parsing.
- The `all/` folders include error artifacts for both providers in the current snapshot, so they are not part of the latest scored records.
- `unmapped_values.csv` is expected to be non-empty because alias review is intentionally conservative.

## Reproducibility Command

```bash
cd /home/yw2399/Demo_Apps
python scoring/score_vista_predictions.py \
  --flat-dir /lp-dev/yw2399/Dataset/AI_2/Single_Shot/flat_test \
  --flat-stages stage1,stage2 \
  --alias-dir scoring/aliases \
  --out-dir /lp-dev/yw2399/Dataset/AI_2/Single_Shot/flat_test/scoring/output \
  --allow-train
```

# Exercise 02 — Fairlearn + AIF360 Evaluation Authoring — Reference Solution

## 1. Solution overview

This reference walks a worked `mod-106-ex02-fairness-eval/` deliverable for the module's running system **TalentFlow** (defined in exercise 01). The exercise asks the engineer to author a repeatable fairness-evaluation harness that produces (a) a per-group metric table using Fairlearn, (b) a parallel view using AIF360, (c) a signed JSON evidence artifact that feeds the release gate, and (d) a short README explaining how to re-run it offline.

The deliverable is a small directory with three tracked files:

- `evaluate.py` — the eval entry-point. Reads a CSV of predictions + ground-truth + protected attributes, computes the metrics, writes the JSON evidence file.
- `README.md` — how to run, what the inputs are, what the exit codes mean.
- `expected-schema.json` — JSON Schema describing the input CSV and the output evidence JSON. This is what the CI check validates.

Design decisions:

- **Fairlearn and AIF360 are run in the same harness, not either/or.** Fairlearn's `MetricFrame` is the engineering-ergonomic layer; AIF360's `BinaryLabelDatasetMetric` and `ClassificationMetric` are the independent second measurement — if the two libraries disagree by more than a numerical-tolerance band on the same metric, the eval **fails loud** and the release is held. Two libraries, one metric, is the audit-trail discipline.
- **Inputs come from a frozen CSV.** The CSV is produced by the inference service's shadow log, pinned to a dated snapshot. The eval does not touch the model. This separation (eval reads predictions, not the model) means the eval is deterministic from the input hash.
- **Output is a JSON evidence artifact signed by the CI runner.** `evidence/fairness/<release-tag>/report.json`. The release gate (mod-109) reads the file; it is not re-computed in the gate.
- **Thresholds are not in the eval.** The eval reports; the register decides. The release-gate reader applies thresholds from `risk-register/2025/TalentFlow.yaml#fairness-envelope`.
- **Fallbacks for missing protected-attribute values.** Rows with missing `sex` or `race_ethnicity` are counted in a separate `coverage_report` block — never silently dropped. If coverage < 95% per protected attribute, the eval fails and surfaces the coverage block as the primary error.

## 2. Implementation — worked answer

### 2.1 Directory layout

```
mod-106-ex02-fairness-eval/
├── README.md
├── expected-schema.json
└── evaluate.py
```

### 2.2 `evaluate.py`

```python
"""TalentFlow fairness evaluation harness.

Reads frozen (predictions, ground_truth, protected_attrs) CSV; writes JSON
evidence artifact with per-group metrics from Fairlearn and AIF360. Fails
loud if the two libraries disagree beyond tolerance, or if protected-attribute
coverage drops below the register-set floor.

Thresholds live in risk-register/2025/TalentFlow.yaml#fairness-envelope.
This script reports; the release gate decides.
"""

from __future__ import annotations

import argparse
import hashlib
import json
import sys
from dataclasses import dataclass
from pathlib import Path

import numpy as np
import pandas as pd

from fairlearn.metrics import (
    MetricFrame,
    demographic_parity_difference,
    demographic_parity_ratio,
    equalized_odds_difference,
    false_positive_rate,
    false_negative_rate,
    selection_rate,
    true_positive_rate,
)
from sklearn.metrics import accuracy_score, precision_score, recall_score

from aif360.datasets import BinaryLabelDataset
from aif360.metrics import BinaryLabelDatasetMetric, ClassificationMetric


COVERAGE_FLOOR = 0.95           # minimum per-attribute non-null coverage
LIB_DISAGREEMENT_TOLERANCE = 1e-6  # Fairlearn vs AIF360 on shared metrics


@dataclass(frozen=True)
class EvalInputs:
    csv_path: Path
    protected_attrs: tuple[str, ...]
    label_col: str = "y_true"
    pred_col: str = "y_pred"
    score_col: str | None = "y_score"


def _sha256(path: Path) -> str:
    h = hashlib.sha256()
    h.update(path.read_bytes())
    return h.hexdigest()


def _coverage(df: pd.DataFrame, attrs: tuple[str, ...]) -> dict[str, float]:
    return {a: float(df[a].notna().mean()) for a in attrs}


def _fairlearn_block(df: pd.DataFrame, attr: str, inputs: EvalInputs) -> dict:
    y_true = df[inputs.label_col].to_numpy()
    y_pred = df[inputs.pred_col].to_numpy()
    sensitive = df[attr].to_numpy()

    mf = MetricFrame(
        metrics={
            "accuracy": accuracy_score,
            "precision": precision_score,
            "recall": recall_score,
            "selection_rate": selection_rate,
            "tpr": true_positive_rate,
            "fpr": false_positive_rate,
            "fnr": false_negative_rate,
        },
        y_true=y_true,
        y_pred=y_pred,
        sensitive_features=sensitive,
    )
    return {
        "per_group": mf.by_group.to_dict(orient="index"),
        "overall": mf.overall.to_dict(),
        "difference": mf.difference().to_dict(),
        "ratio": mf.ratio().to_dict(),
        "demographic_parity_difference": float(
            demographic_parity_difference(y_true, y_pred, sensitive_features=sensitive)
        ),
        "demographic_parity_ratio": float(
            demographic_parity_ratio(y_true, y_pred, sensitive_features=sensitive)
        ),
        "equalized_odds_difference": float(
            equalized_odds_difference(y_true, y_pred, sensitive_features=sensitive)
        ),
    }


def _aif360_block(df: pd.DataFrame, attr: str, inputs: EvalInputs) -> dict:
    # AIF360 expects a BinaryLabelDataset with a single label column and
    # the protected attribute defined. We encode the attribute as binary
    # (privileged = most-selected group at the aggregate level) so the
    # library's privileged/unprivileged metrics are well-defined.
    agg = df.groupby(attr)[inputs.pred_col].mean().sort_values(ascending=False)
    privileged_value = agg.index[0]
    binary_attr = (df[attr] == privileged_value).astype(int)

    frame = df[[inputs.label_col, inputs.pred_col]].copy()
    frame[attr] = binary_attr

    bld_truth = BinaryLabelDataset(
        df=frame.rename(columns={inputs.label_col: "label"}).assign(
            label=frame[inputs.label_col].astype(int)
        )[["label", attr]],
        label_names=["label"],
        protected_attribute_names=[attr],
    )
    bld_pred = bld_truth.copy()
    bld_pred.labels = frame[inputs.pred_col].to_numpy().reshape(-1, 1).astype(float)

    bld_metric = BinaryLabelDatasetMetric(
        bld_truth,
        privileged_groups=[{attr: 1}],
        unprivileged_groups=[{attr: 0}],
    )
    clf_metric = ClassificationMetric(
        bld_truth,
        bld_pred,
        privileged_groups=[{attr: 1}],
        unprivileged_groups=[{attr: 0}],
    )
    return {
        "disparate_impact": float(bld_metric.disparate_impact()),
        "statistical_parity_difference": float(
            bld_metric.statistical_parity_difference()
        ),
        "equal_opportunity_difference": float(clf_metric.equal_opportunity_difference()),
        "average_odds_difference": float(clf_metric.average_odds_difference()),
        "theil_index": float(clf_metric.theil_index()),
        "consistency": float(clf_metric.consistency(n_neighbors=5)[0]),
        "privileged_value": str(privileged_value),
    }


def _cross_check(fl: dict, aif: dict) -> list[str]:
    """Fail-loud when Fairlearn and AIF360 disagree on the shared metric."""
    disagreements: list[str] = []
    fl_dpd = fl["demographic_parity_difference"]
    aif_spd = aif["statistical_parity_difference"]
    # Fairlearn reports max-min (always >= 0); AIF360 reports priv - unpriv
    # (signed). Compare absolute values when the privileged binarization
    # reduces the Fairlearn multi-group problem to two groups.
    if abs(abs(aif_spd) - fl_dpd) > LIB_DISAGREEMENT_TOLERANCE:
        disagreements.append(
            f"Fairlearn DPD={fl_dpd:.6f} vs. AIF360 |SPD|={abs(aif_spd):.6f}"
        )
    return disagreements


def run(inputs: EvalInputs, out_path: Path) -> int:
    df = pd.read_csv(inputs.csv_path)
    for col in (inputs.label_col, inputs.pred_col, *inputs.protected_attrs):
        if col not in df.columns:
            print(f"missing column: {col}", file=sys.stderr)
            return 2

    coverage = _coverage(df, inputs.protected_attrs)
    coverage_fail = [a for a, v in coverage.items() if v < COVERAGE_FLOOR]

    report: dict = {
        "version": "mod-106-ex02/1",
        "input_sha256": _sha256(inputs.csv_path),
        "n_rows": int(len(df)),
        "protected_attrs": list(inputs.protected_attrs),
        "coverage_report": {
            "per_attr_non_null": coverage,
            "floor": COVERAGE_FLOOR,
            "failing_attrs": coverage_fail,
        },
        "per_attr": {},
        "library_disagreements": [],
    }

    if coverage_fail:
        report["status"] = "fail_coverage"
        out_path.write_text(json.dumps(report, indent=2, sort_keys=True))
        print(f"coverage below floor for: {coverage_fail}", file=sys.stderr)
        return 3

    for attr in inputs.protected_attrs:
        fl = _fairlearn_block(df, attr, inputs)
        aif = _aif360_block(df, attr, inputs)
        disagreements = _cross_check(fl, aif)
        report["per_attr"][attr] = {"fairlearn": fl, "aif360": aif}
        report["library_disagreements"].extend(
            [f"{attr}: {d}" for d in disagreements]
        )

    report["status"] = (
        "fail_library_disagreement"
        if report["library_disagreements"]
        else "ok"
    )

    out_path.write_text(json.dumps(report, indent=2, sort_keys=True, default=str))
    if report["status"] != "ok":
        print(report["library_disagreements"], file=sys.stderr)
        return 4
    return 0


def _parse_args() -> argparse.Namespace:
    p = argparse.ArgumentParser(description=__doc__)
    p.add_argument("--input", type=Path, required=True)
    p.add_argument("--out", type=Path, required=True)
    p.add_argument(
        "--protected",
        type=str,
        action="append",
        required=True,
        help="Protected-attribute column; may be passed multiple times.",
    )
    return p.parse_args()


def main() -> int:
    ns = _parse_args()
    inputs = EvalInputs(
        csv_path=ns.input,
        protected_attrs=tuple(ns.protected),
    )
    ns.out.parent.mkdir(parents=True, exist_ok=True)
    return run(inputs, ns.out)


if __name__ == "__main__":
    raise SystemExit(main())
```

### 2.3 `expected-schema.json` (abbreviated — illustrative keys)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "mod-106-ex02 TalentFlow fairness evidence",
  "type": "object",
  "required": ["version", "input_sha256", "protected_attrs", "coverage_report", "per_attr", "status"],
  "properties": {
    "version": {"const": "mod-106-ex02/1"},
    "input_sha256": {"type": "string", "pattern": "^[0-9a-f]{64}$"},
    "n_rows": {"type": "integer", "minimum": 1},
    "protected_attrs": {"type": "array", "items": {"type": "string"}, "minItems": 1},
    "coverage_report": {
      "type": "object",
      "required": ["per_attr_non_null", "floor", "failing_attrs"]
    },
    "per_attr": {
      "type": "object",
      "additionalProperties": {
        "type": "object",
        "required": ["fairlearn", "aif360"],
        "properties": {
          "fairlearn": {
            "type": "object",
            "required": [
              "per_group", "overall", "difference", "ratio",
              "demographic_parity_difference", "demographic_parity_ratio",
              "equalized_odds_difference"
            ]
          },
          "aif360": {
            "type": "object",
            "required": [
              "disparate_impact", "statistical_parity_difference",
              "equal_opportunity_difference", "average_odds_difference",
              "theil_index", "consistency", "privileged_value"
            ]
          }
        }
      }
    },
    "library_disagreements": {"type": "array", "items": {"type": "string"}},
    "status": {"enum": ["ok", "fail_coverage", "fail_library_disagreement"]}
  }
}
```

### 2.4 `README.md`

```markdown
# TalentFlow fairness evaluation

## Inputs

A CSV with at least these columns:

- `y_true` (0/1) — ground truth "qualified / hired within 90 days"
- `y_pred` (0/1) — model's binary "advance" decision at the production threshold
- `y_score` (float, optional) — calibration-view score
- one column per protected attribute (`sex`, `race_ethnicity`, `age_band`)

The CSV is produced by the shadow-logging pipeline at a dated snapshot; the SHA-256 of the file is pinned in the release manifest.

## Run

    python evaluate.py \
      --input ./shadow/2026-11-01.csv \
      --protected sex \
      --protected race_ethnicity \
      --protected age_band \
      --out  evidence/fairness/<release-tag>/report.json

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | OK — report written, status = "ok" |
| 2 | Input schema error — missing required column |
| 3 | Coverage below floor — one or more protected attributes < 95% populated |
| 4 | Library disagreement — Fairlearn and AIF360 disagree beyond tolerance |

The release gate (mod-109) treats any non-zero exit code as a blocking finding.

## Thresholds

Thresholds are not in this harness. They live in
`risk-register/2025/TalentFlow.yaml#fairness-envelope` and are applied by
the release-gate reader against `report.json`.
```

## 3. Validation steps

A reviewer should be able to verify the submission in under 30 minutes.

1. **Static check.** `python -m py_compile evaluate.py` returns 0. `jsonschema expected-schema.json` validates as a draft-2020-12 schema.
2. **Imports resolve.** `fairlearn`, `aif360`, `pandas`, `scikit-learn` installed; versions pinned in `requirements.txt` (not shown — environment is an adjacent concern).
3. **Smoke test on a seeded CSV.** Feed a CSV where both groups have identical selection rates and ground-truth labels. Expect: `demographic_parity_difference ≈ 0.0`, `disparate_impact ≈ 1.0`, `status: "ok"`.
4. **Negative smoke — coverage.** Feed a CSV with 50% missing `sex` values. Expect: exit code 3, `failing_attrs: ["sex"]`.
5. **Negative smoke — disagreement.** Feed a CSV that exercises the AIF360 binarization choice (three groups, not two). Expect: Fairlearn multi-group DPD and AIF360 binarized SPD will differ in sign structure; the solution's cross-check uses `abs(SPD)` vs. DPD precisely for that reason. If a scenario exists where the two cannot be reconciled, exit code 4 is the correct outcome.
6. **Determinism.** Same input CSV → byte-identical output JSON (after sorting keys). Confirm with `sha256sum report.json` across two runs.
7. **Register separation.** Grep `evaluate.py` for any numeric threshold on the output metric; the only numeric literals permitted are `COVERAGE_FLOOR` and `LIB_DISAGREEMENT_TOLERANCE` (both of which are structural, not fairness decisions).

## 4. Rubric / review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Harness entry-point present | `evaluate.py` runs from CLI with `--input`, `--out`, `--protected` | Missing CLI or args |
| R2 | Both libraries invoked | Fairlearn `MetricFrame` and AIF360 `BinaryLabelDatasetMetric` / `ClassificationMetric` both appear in output | Only one library |
| R3 | Cross-library check | Harness compares a shared metric and sets `status: "fail_library_disagreement"` on discrepancy | No cross-check |
| R4 | Coverage block present | `coverage_report` with per-attr non-null rate and `failing_attrs` | Missing column → silent drop |
| R5 | Output is JSON evidence file, not stdout | File at `--out` path, machine-readable | Stdout-only, or human-only report |
| R6 | Thresholds not in harness | Only structural constants (coverage, library-tolerance) | Fairness thresholds embedded |
| R7 | Input is CSV + input hash | SHA-256 recorded in output; CSV path logged | Input hash missing |
| R8 | JSON schema present | `expected-schema.json` validates one real output | Schema missing |
| R9 | Per-group metrics by protected attr | `per_attr` keys = each protected attribute; `per_group` dict per attr | Aggregated only |
| R10 | Readme present with run + exit codes | Run command + exit code table | Missing or incomplete |
| R11 | Deterministic re-run | Same input ⇒ same output bytes | Nondeterministic (ordering, timestamps written) |
| R12 | Fails loud, not quietly | Non-zero exit code on coverage / disagreement | Writes report and exits 0 anyway |

**Scoring bands.** *Pass* — R1–R12 all green. *Pass with revisions* — R11 or R8 yellow; fix and resubmit. *Fail* — R3 or R12 missing (the whole point of two libraries is cross-validation).

## 5. Common mistakes

- **Running only Fairlearn or only AIF360.** The exercise title is both. If a reviewer sees a single library, the measurement is unverified.
- **Silently dropping rows with missing protected-attribute values.** The dropping changes the denominator and hides the problem. The harness must surface coverage and fail below the floor.
- **Setting thresholds in the harness.** `if demographic_parity_difference > 0.1: return 1` — forbidden; the register sets the number.
- **Treating "multi-group" as "two groups twice."** Fairlearn supports multi-group natively; AIF360 is binarised. The cross-check must either reduce multi-group to the binarised comparison, or report both views explicitly — not fake agreement.
- **Writing a human-only HTML report.** The release gate reads JSON. A PDF or HTML artifact is fine *in addition*, but is not the evidence file.
- **No input hash.** Without the SHA-256, the evidence is not reproducible. The release-gate audit needs to know what CSV was scored.
- **Catching all exceptions and logging "warning".** The harness is a gate, not a notebook; exceptions must propagate or map to the documented exit codes.
- **Mislabelling `y_true`.** If the "qualified" label is upstream-biased (per exercise 01, Scenario A), the harness still reports — but the surrounding risk-register entry and model-card must disclose the label's limitations. The disclosure is not in `evaluate.py`; it is in the model card.

## 6. References

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** MEASURE 2.11 on fairness evaluation and documentation; MEASURE 1.1 on metrics selection; MEASURE 2.3 on performance measured for conditions similar to deployment. `https://www.nist.gov/itl/ai-risk-management-framework`
- **NIST AI RMF Playbook.** `airc.nist.gov/AI_RMF_Knowledge_Base/Playbook` — actions under MEASURE 2.11.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — referenced for dataset-integrity and label-quality threats that materialise as fairness-evaluation failure modes.
- **MITRE ATLAS.** `https://atlas.mitre.org/` — referenced for label-manipulation / data-poisoning techniques that an adversarial input-CSV could exercise against this harness.
- **Fairlearn documentation.** `https://fairlearn.org` — `MetricFrame`, `demographic_parity_difference`, `demographic_parity_ratio`, `equalized_odds_difference`, `selection_rate`, `true_positive_rate`, `false_positive_rate`, `false_negative_rate`.
- **AIF360 documentation.** `https://aif360.readthedocs.io` — `BinaryLabelDataset`, `BinaryLabelDatasetMetric` (`disparate_impact`, `statistical_parity_difference`), `ClassificationMetric` (`equal_opportunity_difference`, `average_odds_difference`, `theil_index`, `consistency`).
- **EEOC Uniform Guidelines on Employee Selection Procedures.** 29 CFR 1607 — the "four-fifths rule" that appears as AIF360's `disparate_impact ≥ 0.8`.
- Exercise prompt — `lessons/mod-106-fairness-bias-and-explainability-engineering/exercises/exercise-02-fairlearn-and-aif360-eval-authoring.md`.

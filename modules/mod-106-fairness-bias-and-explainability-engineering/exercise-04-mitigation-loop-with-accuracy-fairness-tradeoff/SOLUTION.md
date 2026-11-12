# Exercise 04 — Mitigation Loop and Accuracy / Fairness Trade-off — Reference Solution

## 1. Solution overview

This reference walks a worked `mod-106-ex04-mitigation-loop/` deliverable for the module's running system **TalentFlow** (exercise 01). Exercises 02 and 03 measured; exercise 04 **mitigates**, and explicitly documents the accuracy-vs-fairness trade-off instead of hiding it.

The deliverable is a loop with four stages, run once per model candidate, and a decision record that is the human-facing artifact for the AI risk council.

1. **Baseline.** Score the un-mitigated model on the evaluation set; record the fairness evidence JSON from exercise 02 and the group-conditional accuracy.
2. **Mitigate.** Try three mitigations — one from each of the AIF360 pre-/in-/post-processing categories — plus one Fairlearn reduction (Exponentiated Gradient). Record each candidate's fairness and accuracy in the same evidence schema.
3. **Plot the Pareto frontier.** Candidate points live in (accuracy, fairness-gap) space; the Pareto frontier is the set of candidates where no other candidate dominates on both axes.
4. **Decide.** Fill out a short decision record (`DECISION.md`) that names the chosen candidate, states the trade accepted, cites the register entry that gates on that trade, and records the dissenting-opinion section (clinical-safety-officer equivalent: employment counsel).

Design decisions:

- **All four mitigation families are exercised.** Fairlearn's reduction (Exponentiated Gradient) sits alongside AIF360's **Reweighing** (pre), **Adversarial Debiasing** (in), and **Reject Option Classification** (post). The loop treats these as four candidates on a shared plot, not as a ranked list.
- **The baseline is one of the candidates.** The loop never pretends the baseline is uninteresting. If the baseline is Pareto-dominant, the decision is to *not* apply a mitigation and to document why.
- **Register decides; this exercise decides.** The mitigation choice is proposed by the engineer and ratified by the AI risk council. The risk-register envelope is updated by the same council action — the decision is traceable to meeting minutes.
- **Fairness gap uses the same metric picked in exercise 01** for the same scenario (e.g., Demographic Parity Difference for Scenario A). Switching metrics between measurement (ex-02) and mitigation (ex-04) produces apparent improvements that are artefacts of the metric change.
- **Group-conditional accuracy is reported alongside the aggregate.** A mitigation that lifts aggregate accuracy by hurting the historically disadvantaged group's accuracy is a documented failure mode and fails the rubric.

## 2. Implementation — worked answer

### 2.1 Directory layout

```
mod-106-ex04-mitigation-loop/
├── README.md
├── DECISION.md
├── mitigate.py
└── plot_pareto.py
```

### 2.2 `mitigate.py` (illustrative; key moves only)

```python
"""TalentFlow mitigation loop.

For each candidate — baseline + three AIF360 mitigations + Fairlearn
Exponentiated Gradient — produce a row in the evidence frame with
(accuracy, demographic_parity_difference, equalized_odds_difference,
group_conditional_accuracy_by_sex, group_conditional_accuracy_by_race).
Thresholds live in the register; this script reports.
"""

from __future__ import annotations

import argparse
import json
from pathlib import Path
from dataclasses import dataclass

import numpy as np
import pandas as pd
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score

from fairlearn.reductions import ExponentiatedGradient, DemographicParity
from fairlearn.metrics import (
    demographic_parity_difference,
    equalized_odds_difference,
    MetricFrame,
)

from aif360.algorithms.preprocessing import Reweighing
from aif360.algorithms.inprocessing import AdversarialDebiasing
from aif360.algorithms.postprocessing import RejectOptionClassification
from aif360.datasets import BinaryLabelDataset


PROTECTED = "SEX"


@dataclass
class CandidateResult:
    name: str
    accuracy: float
    dpd: float
    eod: float
    accuracy_by_group: dict[str, float]


def _group_accuracy(y_true, y_pred, s):
    mf = MetricFrame(metrics=accuracy_score, y_true=y_true, y_pred=y_pred,
                     sensitive_features=s)
    return {str(k): float(v) for k, v in mf.by_group.items()}


def _report(name, y_true, y_pred, s) -> CandidateResult:
    return CandidateResult(
        name=name,
        accuracy=float(accuracy_score(y_true, y_pred)),
        dpd=float(demographic_parity_difference(y_true, y_pred, sensitive_features=s)),
        eod=float(equalized_odds_difference(y_true, y_pred, sensitive_features=s)),
        accuracy_by_group=_group_accuracy(y_true, y_pred, s),
    )


def _to_bld(X: pd.DataFrame, y: np.ndarray, s_col: str) -> BinaryLabelDataset:
    frame = X.copy()
    frame["label"] = y.astype(int)
    return BinaryLabelDataset(
        df=frame,
        label_names=["label"],
        protected_attribute_names=[s_col],
    )


def run(train_csv: Path, test_csv: Path, out_json: Path) -> int:
    train = pd.read_csv(train_csv)
    test = pd.read_csv(test_csv)
    y_train = train["y_true"].to_numpy()
    y_test = test["y_true"].to_numpy()
    s_train = train[PROTECTED]
    s_test = test[PROTECTED]
    features = [c for c in train.columns if c not in ("y_true", "y_pred", "y_score")]

    results: list[CandidateResult] = []

    # (0) Baseline.
    base = LogisticRegression(max_iter=1000).fit(train[features], y_train)
    results.append(_report("baseline", y_test, base.predict(test[features]), s_test))

    # (1) AIF360 Reweighing (pre).
    bld_train = _to_bld(train[features + [PROTECTED]], y_train, PROTECTED)
    rw = Reweighing(
        unprivileged_groups=[{PROTECTED: 0}],
        privileged_groups=[{PROTECTED: 1}],
    )
    bld_train_rw = rw.fit_transform(bld_train)
    weights = bld_train_rw.instance_weights
    m_rw = LogisticRegression(max_iter=1000).fit(
        train[features], y_train, sample_weight=weights
    )
    results.append(_report("reweighing_pre", y_test, m_rw.predict(test[features]), s_test))

    # (2) AIF360 Adversarial Debiasing (in).
    #     Requires tensorflow session; omitted here for brevity.
    #     Shape: AdversarialDebiasing(..., privileged_groups=..., unprivileged_groups=...)
    #     See references.
    #     results.append(_report("adversarial_in", ...))

    # (3) AIF360 Reject Option Classification (post).
    bld_test = _to_bld(test[features + [PROTECTED]], y_test, PROTECTED)
    bld_pred = bld_test.copy()
    bld_pred.scores = base.predict_proba(test[features])[:, 1].reshape(-1, 1)
    bld_pred.labels = base.predict(test[features]).reshape(-1, 1).astype(float)
    roc = RejectOptionClassification(
        unprivileged_groups=[{PROTECTED: 0}],
        privileged_groups=[{PROTECTED: 1}],
        metric_name="Statistical parity difference",
        metric_ub=0.05,
        metric_lb=-0.05,
    )
    roc.fit(bld_test, bld_pred)
    bld_post = roc.predict(bld_pred)
    results.append(_report("reject_option_post", y_test,
                           bld_post.labels.ravel().astype(int), s_test))

    # (4) Fairlearn ExponentiatedGradient reduction under DemographicParity.
    eg = ExponentiatedGradient(
        estimator=LogisticRegression(max_iter=1000),
        constraints=DemographicParity(),
    )
    eg.fit(train[features], y_train, sensitive_features=s_train)
    results.append(_report("exp_grad_dp", y_test,
                           eg.predict(test[features]), s_test))

    out = {
        "version": "mod-106-ex04/1",
        "protected": PROTECTED,
        "candidates": [vars(r) for r in results],
    }
    out_json.parent.mkdir(parents=True, exist_ok=True)
    out_json.write_text(json.dumps(out, indent=2, sort_keys=True, default=str))
    return 0


def main() -> int:
    p = argparse.ArgumentParser(description=__doc__)
    p.add_argument("--train", type=Path, required=True)
    p.add_argument("--test", type=Path, required=True)
    p.add_argument("--out", type=Path, required=True)
    ns = p.parse_args()
    return run(ns.train, ns.test, ns.out)


if __name__ == "__main__":
    raise SystemExit(main())
```

### 2.3 `plot_pareto.py` (illustrative)

```python
"""Compute the Pareto frontier of candidates in (accuracy, |DPD|) space.

A candidate is Pareto-dominated if there exists another candidate with
greater-or-equal accuracy AND lower-or-equal |DPD|, with strict inequality
on at least one axis. The remaining candidates form the frontier.
"""

from __future__ import annotations

import json
from pathlib import Path


def pareto_frontier(points: list[dict]) -> list[dict]:
    frontier: list[dict] = []
    for p in points:
        dominated = False
        for q in points:
            if p["name"] == q["name"]:
                continue
            better_or_eq = (q["accuracy"] >= p["accuracy"]
                            and abs(q["dpd"]) <= abs(p["dpd"]))
            strictly_better = (q["accuracy"] > p["accuracy"]
                               or abs(q["dpd"]) < abs(p["dpd"]))
            if better_or_eq and strictly_better:
                dominated = True
                break
        if not dominated:
            frontier.append(p)
    return frontier


def main() -> None:
    import argparse
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("--evidence", type=Path, required=True)
    parser.add_argument("--out", type=Path, required=True)
    ns = parser.parse_args()
    report = json.loads(ns.evidence.read_text())
    front = pareto_frontier(report["candidates"])
    ns.out.write_text(json.dumps(
        {"candidates": report["candidates"], "pareto_frontier": front},
        indent=2, sort_keys=True,
    ))


if __name__ == "__main__":
    main()
```

### 2.4 `DECISION.md` (illustrative values marked as *example*)

```markdown
# Mitigation decision — TalentFlow release candidate 2026-11

## 1. Scenario and metric
Exercise 01 Scenario A: external hiring with contested base rates. Primary
metric: Demographic Parity Difference. Register envelope:
`risk-register/2025/TalentFlow.yaml#fairness-envelope`:
- DPD ≤ 0.08 (release-gate hard)
- Impact ratio ≥ 0.8 (LL144 report, exercise 06)
- No group's conditional accuracy more than 3 points below the aggregate

## 2. Candidates (example values — replace with signed eval run)

| Candidate | Accuracy | DPD | Group-conditional accuracy (sex) | On frontier? |
|---|---|---|---|---|
| baseline | 0.812 | 0.17 | {M: 0.84, F: 0.78} | yes |
| reweighing_pre | 0.805 | 0.09 | {M: 0.81, F: 0.80} | yes |
| reject_option_post | 0.783 | 0.04 | {M: 0.79, F: 0.78} | yes |
| exp_grad_dp | 0.796 | 0.05 | {M: 0.80, F: 0.79} | yes |

All four sit on the Pareto frontier (none strictly dominates another).

## 3. Chosen candidate
`exp_grad_dp` (Fairlearn Exponentiated Gradient under DemographicParity).

## 4. Trade accepted
The chosen candidate accepts a 1.6-point drop in aggregate accuracy (0.812 →
0.796) to reduce DPD from 0.17 to 0.05 and to equalise group-conditional
accuracy (84/78 → 80/79). Reweighing was close behind; it was rejected
because its DPD (0.09) sits above the register's 0.08 hard cap.

## 5. Register update
Register entry `fairness-envelope` is unchanged (thresholds are external
inputs, not outputs). Register entry `active-mitigation` is set to
`fairlearn.exp_grad.dp.v1` with model-card pointer and evidence-run hash.

## 6. Dissenting opinion
Employment counsel noted the DPD of 0.05 is still positive (men selected at
a higher rate than women). The dissent is logged. Counter: the register's
hard cap is 0.08, not 0.00; the dissent is a candidate for the next-quarter
envelope-tightening proposal, not a block on this release.

## 7. Sign-off
- AI risk engineer (L25) — author
- AI eval engineer (L30) — methodology sign-off
- AI risk council chair — release sign-off
- Employment counsel — dissenting opinion acknowledged
```

## 3. Validation steps

1. **Run and inspect.** `python mitigate.py --train train.csv --test test.csv --out evidence/mitigation/report.json`; confirm a candidates list of 4+ entries.
2. **Pareto correctness.** Seed with hand-authored candidates on a 2D grid (e.g., `[(0.9, 0.2), (0.85, 0.05), (0.8, 0.02), (0.87, 0.1)]`); expected frontier is the three non-dominated points. Confirm `plot_pareto.py` returns exactly those three.
3. **Determinism.** Fix random seeds in Fairlearn's `ExponentiatedGradient` and AIF360's debiasing; same inputs → same evidence JSON.
4. **Group-conditional accuracy on every candidate.** Grep the output JSON for `accuracy_by_group`; the key must be populated for every candidate including the baseline.
5. **Register linkage.** Check that `DECISION.md` cites a specific `fairness-envelope` entry and a specific `active-mitigation` entry.
6. **Dissent section exists.** `DECISION.md` has a section 6 even if the dissent is "none"; `none` is acceptable but must be explicit.
7. **Metric consistency with ex-02.** The `dpd` value for the baseline candidate in this exercise must match the `demographic_parity_difference` from the ex-02 run on the same frozen CSV.

## 4. Rubric / review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Baseline treated as a candidate | Baseline on the Pareto plot | Baseline omitted / privileged |
| R2 | ≥3 mitigation families exercised | Pre + in + post from AIF360, or at minimum two AIF360 families plus one Fairlearn reduction | Only one family tried |
| R3 | Pareto frontier computed | `plot_pareto.py` or equivalent yields a frontier set | Candidates ranked by a single-scalar score |
| R4 | Same fairness metric as exercise 02 for this scenario | DPD used throughout if Scenario A was picked | Metric switched mid-stream |
| R5 | Group-conditional accuracy per candidate | `accuracy_by_group` for every candidate | Aggregate accuracy only |
| R6 | Decision record names chosen candidate + trade | `DECISION.md` sections 3–4 populated | Chosen candidate asserted without the trade |
| R7 | Register linkage | `DECISION.md` cites `fairness-envelope` and `active-mitigation` | Decision lands without a register pointer |
| R8 | Dissent section exists | Section 6 present (may be "none") | Omitted |
| R9 | Thresholds not in code | Register pointer cited in both code and decision | Numeric thresholds in `.py` |
| R10 | Fallback when all candidates are below envelope | Decision names "none meets envelope — do not release" path | Silently picks "best-of-bad" |
| R11 | Candidate evidence JSONs schema-stable | Same shape as exercise 02 schema | Shape diverges, breaks release gate |
| R12 | Reproducibility | Seeds fixed; input hashes recorded | Nondeterministic outputs |

**Scoring bands.** *Pass* — all green. *Pass with revisions* — R5 or R12 yellow; fix and resubmit. *Fail* — R1 missing (no baseline), R3 missing (ranked by scalar), or R7 missing (no register linkage).

## 5. Common mistakes

- **Picking a mitigation by "lowest DPD."** DPD → 0 is often bought by a large accuracy drop on the minority group too. The Pareto view and the group-conditional accuracy column are the guards.
- **Collapsing the trade into a single "fairness-accuracy score."** No such scalar is endorsed by NIST AI RMF, EEOC guidance, LL144, or canonical publications. Keep the two axes separate.
- **Omitting the baseline from the plot.** A mitigation that is worse than the baseline on both axes still has to show up or the engineer is hiding it.
- **Switching the fairness metric between measurement and mitigation.** Exercise 01 chose the metric for the scenario; exercise 04 uses it.
- **Using Reweighing alone.** Reweighing targets the training distribution but does not alter the decision threshold. If the model is thresholded post-hoc for operational reasons, pair with a post-processing step or use Reject Option Classification directly.
- **Adversarial debiasing with no seed + no convergence check.** The in-processing family is sensitive; seed and training-loss stability must be recorded.
- **Treating Fairlearn's Exponentiated Gradient as "the Fairlearn answer."** Fairlearn also exposes `GridSearch` and `ThresholdOptimizer`; the exercise is explicitly about **comparing** mitigations, not selecting one blind.
- **No "do not release" path.** If no candidate meets the envelope, the decision must be to not release — not to pick best-of-bad silently.
- **Decision without a dissent section.** Even "none" is a recorded choice.

## 6. References

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** MEASURE 2.11 (fairness); MANAGE 1.3 (risk response — mitigate / transfer / avoid / accept, explicitly). `https://www.nist.gov/itl/ai-risk-management-framework`
- **NIST AI RMF Playbook.** `airc.nist.gov/AI_RMF_Knowledge_Base/Playbook` — MANAGE 1.3 actions on documenting the response.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant for Adversarial Debiasing: the adversary in that algorithm is benign (an auxiliary head), but the broader threat model for in-processing mitigations is called out in ML03/ML04.
- **MITRE ATLAS.** `https://atlas.mitre.org/` — relevant for Reweighing's reliance on label quality; see label-manipulation techniques.
- **Fairlearn documentation.** `https://fairlearn.org` — `ExponentiatedGradient`, `GridSearch`, `ThresholdOptimizer`, `DemographicParity`, `EqualizedOdds`.
- **AIF360 documentation.** `https://aif360.readthedocs.io` — `Reweighing`, `AdversarialDebiasing`, `RejectOptionClassification`, `DisparateImpactRemover`.
- **Agarwal, Beygelzimer, Dudík, Langford, Wallach (2018).** *A Reductions Approach to Fair Classification.* ICML. — the algorithm behind Fairlearn's `ExponentiatedGradient`.
- **Kamiran, Calders (2012).** *Data preprocessing techniques for classification without discrimination.* Knowledge and Information Systems. — AIF360's `Reweighing`.
- **Zhang, Lemoine, Mitchell (2018).** *Mitigating unwanted biases with adversarial learning.* AIES. — AIF360's `AdversarialDebiasing`.
- **Kamiran, Karim, Zhang (2012).** *Decision theory for discrimination-aware classification.* ICDM. — AIF360's `RejectOptionClassification`.
- Exercise prompt — `lessons/mod-106-fairness-bias-and-explainability-engineering/exercises/exercise-04-mitigation-loop-with-accuracy-fairness-tradeoff.md`.

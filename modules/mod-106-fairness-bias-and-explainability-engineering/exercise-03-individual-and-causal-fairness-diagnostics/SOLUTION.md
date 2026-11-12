# Exercise 03 — Individual and Causal Fairness Diagnostics — Reference Solution

## 1. Solution overview

This reference walks a worked `mod-106-ex03-individual-causal/` deliverable for the module's running system **TalentFlow** (exercise 01). Where exercise 02 measured **group-level** fairness, exercise 03 asks the engineer to add two orthogonal lenses:

- **Individual fairness** — similar individuals should get similar outputs (Dwork, Hardt, Pitassi, Reingold, Zemel 2012). Operationalised as a *consistency score* and a *Lipschitz-style upper-bound audit* on the model's score function.
- **Causal / counterfactual fairness** — would the model's prediction change if we had counterfactually set the protected attribute to a different value, holding the non-descendant features fixed on the causal graph? (Kusner, Loftus, Russell, Silva 2017.) Operationalised as a counterfactual-flip test under a declared causal graph, with sensitivity analysis.

These two lenses *cannot* replace group metrics — they answer different questions. They land as:

- `individual_fairness.py` — consistency score + Lipschitz upper-bound diagnostic; emits a JSON block that merges into `evidence/fairness/<release-tag>/report.json`.
- `counterfactual_fairness.py` — graph-aware counterfactual-flip diagnostic; emits a parallel JSON block.
- `causal-graph.yaml` — the declared causal graph (DAG) used by `counterfactual_fairness.py`. Reviewed by the clinical-safety / employment-counsel equivalent (here: **employment counsel + the hiring-science subject-matter expert**); treated as a first-class artifact under configuration control.
- `README.md` — how to run, what the diagnostics mean, what the caveats are.

Design decisions:

- **The causal graph is a declared artifact, not an implicit assumption.** Per Kusner et al. (2017) §5, counterfactual fairness is a property *relative to a causal model*. The DAG is checked into the repo in `causal-graph.yaml`, carries authorship and reviewers, and is versioned; any change to the DAG re-triggers the diagnostic.
- **Individual fairness uses a documented similarity metric.** The exercise picks a Gower-distance-based similarity on the non-protected feature space (handles mixed numeric + categorical TalentFlow features). AIF360's `ClassificationMetric.consistency` implements the k-NN consistency variant; the solution uses it plus a custom Lipschitz upper-bound audit that reports the **maximum score difference** between any two individuals within similarity ε across the eval set.
- **Counterfactual fairness is reported as a *sensitivity band*, not a scalar.** Kusner et al. distinguish Level-1 (no descendants of the protected attribute used), Level-2 (descendants used), and Level-3 (full counterfactual simulation with the structural equations). The solution reports which level the audit achieves and under which assumptions.
- **Unverifiable causal assumptions are tagged.** If an edge in the DAG cannot be defended from documented hiring-domain prior work, the YAML row carries a `confidence: low` tag and the diagnostic surfaces a sensitivity arm that drops the edge.
- **Both diagnostics defer thresholds to the register.** The harness reports; `risk-register/2025/TalentFlow.yaml#fairness-envelope` sets (a) the maximum tolerated inconsistency on near-duplicate pairs, (b) the counterfactual-flip-rate ceiling.

## 2. Implementation — worked answer

### 2.1 Directory layout

```
mod-106-ex03-individual-causal/
├── README.md
├── causal-graph.yaml
├── individual_fairness.py
└── counterfactual_fairness.py
```

### 2.2 `causal-graph.yaml`

```yaml
# TalentFlow structural causal model (declared).
#
# Nodes are observed features (uppercase) and latent influences (lowercase).
# Edges are directed; "confidence" reflects how well-supported the edge is by
# documented hiring-science prior work. "low" edges are the sensitivity arm.
#
# Reviewed quarterly by: employment counsel, hiring-science SME.
# Version: 2026-11-01.

version: 2026-11-01
reviewed_by:
  - employment-counsel
  - hiring-science-sme
nodes:
  - name: SEX
    type: protected
  - name: RACE_ETHNICITY
    type: protected
  - name: AGE_BAND
    type: protected
  - name: EDUCATION
    type: observed
  - name: YEARS_EXPERIENCE
    type: observed
  - name: SKILLS_LIST
    type: observed
  - name: RESUME_STYLE_EMBEDDING
    type: observed
  - name: SCORE
    type: outcome
edges:
  # Non-controversial: experience depends on age-band mechanically.
  - from: AGE_BAND
    to: YEARS_EXPERIENCE
    confidence: high
  # Education-access patterns differ across groups per US Census data on
  # bachelor's-degree attainment; documented as prior, not invented.
  - from: RACE_ETHNICITY
    to: EDUCATION
    confidence: medium
  - from: SEX
    to: EDUCATION
    confidence: medium
  # Resume style is known to vary across demographic groups (stylometric
  # studies of application text); edge carried but flagged.
  - from: SEX
    to: RESUME_STYLE_EMBEDDING
    confidence: low
  - from: RACE_ETHNICITY
    to: RESUME_STYLE_EMBEDDING
    confidence: low
  # The model consumes non-protected features only.
  - from: EDUCATION
    to: SCORE
  - from: YEARS_EXPERIENCE
    to: SCORE
  - from: SKILLS_LIST
    to: SCORE
  - from: RESUME_STYLE_EMBEDDING
    to: SCORE
# Interpretation:
#   - Level-1 counterfactual fairness (model uses no descendants of the
#     protected attribute) is FALSE: EDUCATION, YEARS_EXPERIENCE, and
#     RESUME_STYLE_EMBEDDING are all descendants.
#   - Level-2 holds iff the model is counterfactually invariant conditional
#     on the descendants being set to their factual values.
#   - Level-3 (full structural simulation) requires the structural equations,
#     which we do NOT have. The audit therefore operates at Level-2 with a
#     sensitivity arm that drops the "low-confidence" edges and re-runs.
```

### 2.3 `individual_fairness.py`

```python
"""TalentFlow individual-fairness diagnostic.

Produces two signals:

1. AIF360 consistency score (k-NN output agreement within the non-protected
   feature space); 1.0 is perfect consistency.
2. A Lipschitz-style upper-bound audit: for every pair of candidates within
   normalized Gower distance eps=0.05 on the non-protected features, report
   the maximum absolute score difference. If this exceeds the register-set
   budget, the diagnostic fails.
"""

from __future__ import annotations

import argparse
import itertools
import json
from pathlib import Path

import numpy as np
import pandas as pd
from aif360.datasets import BinaryLabelDataset
from aif360.metrics import ClassificationMetric
from sklearn.preprocessing import MinMaxScaler


PROTECTED = ("SEX", "RACE_ETHNICITY", "AGE_BAND")
NEAR_DUPLICATE_EPS = 0.05  # Gower-distance band for "similar"


def _gower(df: pd.DataFrame) -> np.ndarray:
    """Normalized Gower distance over mixed numeric + categorical columns."""
    n = len(df)
    num_cols = df.select_dtypes(include="number").columns
    cat_cols = df.select_dtypes(exclude="number").columns

    num = MinMaxScaler().fit_transform(df[num_cols].to_numpy())
    cat = df[cat_cols].to_numpy()

    d = np.zeros((n, n), dtype=np.float32)
    for i, j in itertools.combinations(range(n), 2):
        num_dist = np.abs(num[i] - num[j]).mean() if num.size else 0.0
        cat_dist = (cat[i] != cat[j]).mean() if cat.size else 0.0
        weights = (len(num_cols), len(cat_cols))
        total = sum(weights) or 1
        d[i, j] = d[j, i] = (num_dist * weights[0] + cat_dist * weights[1]) / total
    return d


def run(input_csv: Path, out_json: Path) -> int:
    df = pd.read_csv(input_csv)
    non_protected = [c for c in df.columns if c not in PROTECTED and c not in ("y_true", "y_pred", "y_score")]
    scores = df["y_score"].to_numpy()
    y_pred = df["y_pred"].astype(int).to_numpy()

    dist = _gower(df[non_protected])

    # Lipschitz upper-bound audit: max |score_i - score_j| for pairs within eps.
    n = len(df)
    pairs = [(i, j) for i, j in itertools.combinations(range(n), 2) if dist[i, j] <= NEAR_DUPLICATE_EPS]
    if pairs:
        max_delta = max(abs(scores[i] - scores[j]) for i, j in pairs)
        pct_disagreeing_predictions = (
            sum(y_pred[i] != y_pred[j] for i, j in pairs) / len(pairs)
        )
    else:
        max_delta = 0.0
        pct_disagreeing_predictions = 0.0

    # AIF360 consistency (k-NN output agreement).
    bld = BinaryLabelDataset(
        df=df.assign(label=df["y_true"].astype(int))[["label", *PROTECTED]],
        label_names=["label"],
        protected_attribute_names=list(PROTECTED),
    )
    bld_pred = bld.copy()
    bld_pred.labels = y_pred.reshape(-1, 1).astype(float)
    consistency = float(
        ClassificationMetric(
            bld,
            bld_pred,
            privileged_groups=[{p: 1} for p in PROTECTED],
            unprivileged_groups=[{p: 0} for p in PROTECTED],
        ).consistency(n_neighbors=5)[0]
    )

    report = {
        "version": "mod-106-ex03-individual/1",
        "n_rows": int(n),
        "near_duplicate_eps": NEAR_DUPLICATE_EPS,
        "n_near_duplicate_pairs": len(pairs),
        "max_score_delta_within_eps": float(max_delta),
        "pct_disagreeing_predictions_within_eps": float(pct_disagreeing_predictions),
        "aif360_consistency_k5": consistency,
    }
    out_json.parent.mkdir(parents=True, exist_ok=True)
    out_json.write_text(json.dumps(report, indent=2, sort_keys=True))
    return 0


def main() -> int:
    p = argparse.ArgumentParser(description=__doc__)
    p.add_argument("--input", type=Path, required=True)
    p.add_argument("--out", type=Path, required=True)
    ns = p.parse_args()
    return run(ns.input, ns.out)


if __name__ == "__main__":
    raise SystemExit(main())
```

### 2.4 `counterfactual_fairness.py` (illustrative)

```python
"""TalentFlow counterfactual-fairness diagnostic.

For each candidate in the eval CSV, construct a counterfactual twin where
the protected attribute is flipped and non-descendants on the DAG are held
fixed. For descendants flagged `confidence: low` in causal-graph.yaml, run
two arms: (a) hold them at factual values (Level-2), (b) resample them under
their observed distribution for the counterfactual group (sensitivity arm).

The diagnostic requires access to the scoring function (the model), NOT just
cached predictions, because it has to re-score counterfactual inputs.

Report: counterfactual flip rate (fraction of candidates whose binary
"advance" decision changes under the counterfactual), per protected
attribute and per arm.
"""

from __future__ import annotations

import argparse
import json
from pathlib import Path

import pandas as pd
import yaml
from joblib import load as joblib_load  # or model-provider-specific loader


def _load_graph(graph_path: Path) -> dict:
    return yaml.safe_load(graph_path.read_text())


def _descendants(graph: dict, node: str) -> set[str]:
    children = {e["to"] for e in graph["edges"] if e["from"] == node}
    seen = set(children)
    frontier = list(children)
    while frontier:
        current = frontier.pop()
        next_children = {e["to"] for e in graph["edges"] if e["from"] == current}
        new = next_children - seen
        seen |= new
        frontier.extend(new)
    return seen


def _flip_rate(df_factual, df_counterfactual, model, threshold: float) -> float:
    s_f = model.predict_proba(df_factual)[:, 1]
    s_c = model.predict_proba(df_counterfactual)[:, 1]
    changed = ((s_f >= threshold) != (s_c >= threshold)).mean()
    return float(changed)


def run(
    input_csv: Path, graph_path: Path, model_path: Path, out_json: Path,
    threshold: float = 0.5,
) -> int:
    df = pd.read_csv(input_csv)
    graph = _load_graph(graph_path)
    model = joblib_load(model_path)

    report: dict = {
        "version": "mod-106-ex03-counterfactual/1",
        "graph_version": graph["version"],
        "n_rows": int(len(df)),
        "threshold": threshold,
        "per_protected": {},
        "level": "Level-2 (descendants held at factual values); sensitivity arm drops low-confidence edges",
    }

    protected_nodes = [n["name"] for n in graph["nodes"] if n["type"] == "protected"]

    for p in protected_nodes:
        other_values = sorted(set(df[p].unique()))
        descs = _descendants(graph, p)
        low_conf_descs = {
            e["to"] for e in graph["edges"]
            if e["from"] == p and e.get("confidence") == "low"
        }

        attr_report = {"groups": {}, "sensitivity_arm": {}}
        for v in other_values:
            counterfactual = df.copy()
            counterfactual[p] = v
            # Level-2: descendants are held at factual values (already in df).
            attr_report["groups"][str(v)] = _flip_rate(
                df_factual=df, df_counterfactual=counterfactual,
                model=model, threshold=threshold,
            )
            # Sensitivity arm: resample low-confidence descendants under the
            # empirical distribution conditioned on P=v.
            if low_conf_descs:
                resampled = counterfactual.copy()
                donor_pool = df[df[p] == v]
                for d_col in low_conf_descs:
                    resampled[d_col] = donor_pool[d_col].sample(
                        n=len(resampled), replace=True, random_state=0,
                    ).to_numpy()
                attr_report["sensitivity_arm"][str(v)] = _flip_rate(
                    df_factual=df, df_counterfactual=resampled,
                    model=model, threshold=threshold,
                )
        report["per_protected"][p] = attr_report

    out_json.parent.mkdir(parents=True, exist_ok=True)
    out_json.write_text(json.dumps(report, indent=2, sort_keys=True))
    return 0


def main() -> int:
    p = argparse.ArgumentParser(description=__doc__)
    p.add_argument("--input", type=Path, required=True)
    p.add_argument("--graph", type=Path, required=True)
    p.add_argument("--model", type=Path, required=True)
    p.add_argument("--out", type=Path, required=True)
    p.add_argument("--threshold", type=float, default=0.5)
    ns = p.parse_args()
    return run(ns.input, ns.graph, ns.model, ns.out, threshold=ns.threshold)


if __name__ == "__main__":
    raise SystemExit(main())
```

### 2.5 README summary

```markdown
# TalentFlow individual + causal fairness diagnostics

## When each diagnostic fires
- **Individual.** Every release candidate; against the shadow CSV from the
  release window.
- **Counterfactual.** Every release candidate; and whenever `causal-graph.yaml`
  changes, re-run on the current production candidate and log a drift entry.

## Caveats
- Individual fairness depends on the similarity metric. Gower distance treats
  categorical columns as all-or-nothing matches; two candidates with identical
  numeric features but different `SKILLS_LIST` entries are not "near duplicates."
- Counterfactual fairness depends on the declared DAG. A DAG that omits a true
  edge under-estimates the flip rate; a DAG that invents an edge over-estimates
  it. The `confidence: low` tags and the sensitivity arm are the mitigation.
- Level-3 (full structural simulation) is NOT claimed.
```

## 3. Validation steps

1. **Smoke test — identical twin.** Construct two candidates with identical non-protected features and different protected attributes. The individual-fairness diagnostic must find them as near duplicates and report any score delta as the max delta. The counterfactual diagnostic must flip the protected attribute, re-score, and either report `flip_rate=0` (good) or a non-zero rate that the risk register treats as a finding.
2. **Determinism.** Same inputs (CSV, graph, model, seed) → byte-identical JSON outputs. Confirm via `sha256sum`.
3. **Graph round-trip.** `_descendants(graph, 'SEX')` returns `{EDUCATION, RESUME_STYLE_EMBEDDING, SCORE}` (plus `YEARS_EXPERIENCE` only if the "SEX→YEARS_EXPERIENCE" edge is added, which this graph does not).
4. **Sensitivity arm.** Remove the `confidence: low` tags on the SEX→RESUME_STYLE_EMBEDDING edge and re-run. The sensitivity arm must become empty for SEX, and the two arms converge.
5. **Register linkage.** Verify `risk-register/2025/TalentFlow.yaml#fairness-envelope` has entries for `max_counterfactual_flip_rate` and `max_score_delta_within_eps`. If missing, create them before merge.
6. **Reviewer-of-record.** The DAG must list `employment-counsel` and `hiring-science-sme` under `reviewed_by`. The mod-103 risk-register update records the reviewer sign-off.

## 4. Rubric / review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Both lenses present | Individual *and* counterfactual diagnostics implemented | Only one |
| R2 | Causal graph declared as artifact | `causal-graph.yaml` committed; version + reviewer list present | Graph is implicit in code |
| R3 | DAG has confidence tags on contested edges | Low/medium/high on each edge | All edges unlabelled |
| R4 | Level is stated | Output JSON names the Kusner level (1/2/3) | Level is not claimed |
| R5 | Sensitivity arm on low-confidence edges | Counterfactual harness runs a second arm dropping or resampling low-confidence edges | Single arm only |
| R6 | Individual fairness uses documented similarity metric | Gower / normalized L1 / cited metric; justified for mixed feature types | Ad hoc similarity |
| R7 | Lipschitz-style upper-bound audit present | Max-score-delta within ε reported | Consistency score only |
| R8 | Thresholds in register, not code | Register pointer cited | Numeric thresholds in `.py` |
| R9 | Counterfactual requires live model, not cached preds | Scoring function is called with counterfactual rows | Cached preds re-used |
| R10 | DAG review sign-off | At least employment-counsel + domain SME named | No review path |
| R11 | Caveats documented | README lists metric + DAG limits | Diagnostics presented as definitive |
| R12 | Output is machine-readable JSON | Evidence JSON per diagnostic | Human-only output |

**Scoring bands.** *Pass* — all green. *Pass with revisions* — R3 or R5 yellow; fix and resubmit. *Fail* — R2 missing, R9 violated (counterfactual run on cached predictions silently passes), or R4 missing (level not claimed).

## 5. Common mistakes

- **Treating group metrics and individual metrics as substitutes.** They answer different questions. Exercise 02's group metrics can be green while exercise 03's near-duplicate audit is red — that is the fairness-through-awareness motivation, not a contradiction.
- **Computing counterfactual fairness without a declared DAG.** Without a DAG, there is no counterfactual to compute — only a swap of the protected attribute, which does not respect causal structure. Kusner et al. §5 makes the DAG explicit; the solution does the same.
- **Claiming Level-1 counterfactual fairness in a model that uses descendants of the protected attribute.** Any model that uses EDUCATION, YEARS_EXPERIENCE, or RESUME_STYLE_EMBEDDING in a world where these descend from SEX / RACE_ETHNICITY is **not** Level-1 counterfactually fair. Claiming Level-1 is a hard failure.
- **Omitting the sensitivity arm on low-confidence edges.** Counterfactual fairness is a function of the DAG; untested DAG edges mean untested conclusions.
- **Using Euclidean distance on raw mixed features for the similarity metric.** Categorical columns dominate or vanish depending on encoding. Gower (or a documented equivalent) is appropriate.
- **Running counterfactual fairness on cached predictions.** The whole point is to re-score the counterfactual input — cached preds encode the factual world.
- **Setting the counterfactual threshold inline.** The threshold is a register decision.
- **Presenting the diagnostic as definitive.** It is diagnostic; the DAG and the similarity metric are both open to challenge. The README names those caveats.

## 6. References

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** MEASURE 2.11 (fairness and bias evaluation; the Playbook action list explicitly includes individual-fairness considerations); MAP 1.1 (context, including causal reasoning about the deployed setting). `https://www.nist.gov/itl/ai-risk-management-framework`
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant because the similarity-metric choice is part of the eval-set threat surface.
- **MITRE ATLAS.** `https://atlas.mitre.org/` — relevant for adversarial-perturbation techniques that produce "near-duplicate" pairs with artificially amplified score deltas.
- **Dwork, Hardt, Pitassi, Reingold, Zemel (2012).** *Fairness Through Awareness.* ITCS. — the Lipschitz / individual-fairness formulation.
- **Kusner, Loftus, Russell, Silva (2017).** *Counterfactual Fairness.* NeurIPS. — Levels 1/2/3 and the DAG-dependence argument.
- **AIF360 documentation.** `https://aif360.readthedocs.io` — `ClassificationMetric.consistency` is the k-NN individual-fairness operationalisation used in the solution.
- **Fairlearn documentation.** `https://fairlearn.org` — group metrics consumed by exercise 02 are referenced from here for cross-reading.
- Exercise prompt — `lessons/mod-106-fairness-bias-and-explainability-engineering/exercises/exercise-03-individual-and-causal-fairness-diagnostics.md`.

## 7. Library-version note

The AIF360 citation above targets release line 0.6.x, where `consistency(n_neighbors=5)` is the k-NN individual-fairness operationalisation exposed on `BinaryLabelDatasetMetric` (inherited by `ClassificationMetric`). Pin AIF360 in `requirements.txt` and re-check the method signature on the installed version before wiring the harness; the metric name and default neighbour count are the two API surfaces that have shifted across minor releases, so the register entry should name the AIF360 version alongside the k value.

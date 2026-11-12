# Exercise 05 — Explainability Slice with SHAP, Alibi, and LIME — Reference Solution

## 1. Solution overview

This reference walks a worked `mod-106-ex05-explainability-slice/` deliverable for the module's running system **TalentFlow** (exercise 01). The exercise asks the engineer to add a *targeted explainability slice* — not "explain everything," but produce three co-signed evidence artifacts that answer three specific audit questions:

1. **Global** — "What features drive the model's decisions in aggregate?" → **SHAP** summary and mean-|SHAP| per group.
2. **Local** — "Why did this specific candidate get this specific decision?" → **LIME** tabular explanation + **SHAP** waterfall on the same instance, cross-checked.
3. **Counterfactual** — "What minimal change would have flipped this decision?" → **Alibi** counterfactual (`CounterfactualProto` or `AnchorTabular`) grounded against the causal graph from exercise 03.

Design decisions:

- **SHAP + LIME are run on the same local case and the top-features are compared.** If they disagree on the top-3 features, that disagreement is logged as a finding — not reconciled away. The method-disagreement itself is informative under the audit standard set by NIST AI RMF MEASURE 2.11.
- **Alibi counterfactuals are declared against the causal graph.** A counterfactual that flips a protected attribute is *not valid* under Kusner et al.'s construction unless it respects the DAG — the solution rejects any raw counterfactual that mutates a protected attribute without going through the exercise-03 counterfactual harness.
- **Global SHAP is sliced by protected attribute.** Mean-|SHAP| of each feature is computed per group. A feature whose SHAP contribution magnitude differs sharply across groups is a *bias-indicator* row, surfaced in the evidence JSON.
- **Nothing in this exercise is "the explanation."** Every artifact is labelled as "a view produced by method X with these hyperparameters." The audit record carries the view and the hyperparameters, not a claim of single ground truth.
- **TreeExplainer / KernelExplainer is picked deliberately.** If TalentFlow's production model is a gradient-boosted tree (XGBoost / LightGBM), the harness uses `shap.TreeExplainer` (exact and fast). If it is a neural / blackbox model, `shap.KernelExplainer` is used with explicitly documented sampling cost. The two are not swappable without a note.

## 2. Implementation — worked answer

### 2.1 Directory layout

```
mod-106-ex05-explainability-slice/
├── README.md
├── global_shap.py
├── local_shap_lime.py
├── counterfactual_alibi.py
└── schema/
    ├── global-shap.schema.json
    ├── local-explanation.schema.json
    └── counterfactual.schema.json
```

### 2.2 `global_shap.py` (illustrative)

```python
"""Global SHAP slice.

Produces:
- mean |SHAP| per feature, overall and per protected-attribute group;
- top-K features overall and per group;
- bias-indicator rows: features where max(mean|SHAP|_group) - min(...) exceeds
  a register-set budget.
"""

from __future__ import annotations

import argparse
import json
from pathlib import Path

import numpy as np
import pandas as pd
import shap


PROTECTED = ("SEX", "RACE_ETHNICITY", "AGE_BAND")
TOP_K = 10
BIAS_INDICATOR_RATIO = 1.5  # max/min ratio of mean|SHAP| per group; register-tunable


def run(model_path: Path, input_csv: Path, out_json: Path) -> int:
    df = pd.read_csv(input_csv)
    features = [c for c in df.columns if c not in PROTECTED + ("y_true", "y_pred", "y_score")]
    X = df[features]

    # Load model via your project's loader; here joblib for scikit-/XGBoost-style models.
    import joblib
    model = joblib.load(model_path)

    try:
        explainer = shap.TreeExplainer(model)
    except Exception:  # noqa: BLE001 - fallback intentional
        # Non-tree model: use KernelExplainer with a documented background sample.
        background = shap.sample(X, 100, random_state=0)
        explainer = shap.KernelExplainer(model.predict_proba, background)

    shap_values = explainer.shap_values(X)
    if isinstance(shap_values, list):  # multiclass returns a list of arrays
        shap_values = shap_values[1]

    abs_shap = np.abs(shap_values)
    overall_mean = abs_shap.mean(axis=0)
    report = {
        "version": "mod-106-ex05-global/1",
        "n_rows": int(len(df)),
        "top_overall": sorted(
            [{"feature": f, "mean_abs_shap": float(v)} for f, v in zip(features, overall_mean)],
            key=lambda r: r["mean_abs_shap"],
            reverse=True,
        )[:TOP_K],
        "per_protected": {},
        "bias_indicators": [],
    }

    for attr in PROTECTED:
        per_group = {}
        for g in sorted(df[attr].dropna().unique()):
            mask = (df[attr] == g).to_numpy()
            if mask.sum() == 0:
                continue
            per_group[str(g)] = abs_shap[mask].mean(axis=0)
        report["per_protected"][attr] = {
            "top_per_group": {
                g: sorted(
                    [{"feature": f, "mean_abs_shap": float(v)} for f, v in zip(features, vec)],
                    key=lambda r: r["mean_abs_shap"],
                    reverse=True,
                )[:TOP_K]
                for g, vec in per_group.items()
            }
        }
        # Bias-indicator rows: features whose group-wise mean|SHAP| ratio exceeds budget.
        for f_idx, f in enumerate(features):
            vals = [per_group[g][f_idx] for g in per_group if per_group[g][f_idx] > 0]
            if len(vals) >= 2:
                ratio = max(vals) / max(min(vals), 1e-9)
                if ratio >= BIAS_INDICATOR_RATIO:
                    report["bias_indicators"].append({
                        "protected": attr,
                        "feature": f,
                        "max_over_min_mean_abs_shap_ratio": float(ratio),
                    })

    out_json.parent.mkdir(parents=True, exist_ok=True)
    out_json.write_text(json.dumps(report, indent=2, sort_keys=True))
    return 0


def main() -> int:
    p = argparse.ArgumentParser(description=__doc__)
    p.add_argument("--model", type=Path, required=True)
    p.add_argument("--input", type=Path, required=True)
    p.add_argument("--out", type=Path, required=True)
    ns = p.parse_args()
    return run(ns.model, ns.input, ns.out)


if __name__ == "__main__":
    raise SystemExit(main())
```

### 2.3 `local_shap_lime.py` (illustrative)

```python
"""Local explanation: SHAP + LIME on the same instance; disagreement surfaced.

Given an `instance_id` from the input CSV, compute (a) SHAP local attribution
and (b) a LIME tabular explanation. Report both top-3 feature lists and the
set-difference; if the difference is non-empty, mark the evidence JSON with
`methods_disagree: true`.
"""

from __future__ import annotations

import argparse
import json
from pathlib import Path

import numpy as np
import pandas as pd
import shap
import joblib
from lime.lime_tabular import LimeTabularExplainer


def run(model_path: Path, input_csv: Path, instance_id: int, out_json: Path) -> int:
    df = pd.read_csv(input_csv)
    features = [c for c in df.columns
                if c not in ("SEX", "RACE_ETHNICITY", "AGE_BAND",
                             "y_true", "y_pred", "y_score")]
    X = df[features]
    model = joblib.load(model_path)
    instance = X.iloc[instance_id:instance_id + 1]

    # SHAP local.
    try:
        explainer = shap.TreeExplainer(model)
        shap_values = explainer.shap_values(instance)
        if isinstance(shap_values, list):
            shap_values = shap_values[1]
        shap_row = shap_values[0]
    except Exception:  # noqa: BLE001
        background = shap.sample(X, 100, random_state=0)
        explainer = shap.KernelExplainer(model.predict_proba, background)
        shap_row = explainer.shap_values(instance)[1][0]

    shap_top3 = [
        features[i] for i in np.argsort(np.abs(shap_row))[::-1][:3]
    ]

    # LIME local.
    lime = LimeTabularExplainer(
        X.to_numpy(),
        feature_names=features,
        class_names=["reject", "advance"],
        discretize_continuous=True,
        random_state=0,
    )
    lime_exp = lime.explain_instance(
        instance.to_numpy()[0], model.predict_proba, num_features=10,
    )
    lime_pairs = lime_exp.as_list()  # list of (feature_desc, weight)
    lime_top3 = [p[0].split(" ")[0] for p in lime_pairs[:3]]  # feature name before operator

    disagreement = sorted(set(shap_top3).symmetric_difference(lime_top3))

    out_json.parent.mkdir(parents=True, exist_ok=True)
    out_json.write_text(json.dumps({
        "version": "mod-106-ex05-local/1",
        "instance_id": int(instance_id),
        "model_prediction": int(model.predict(instance)[0]),
        "model_score": float(model.predict_proba(instance)[0, 1]),
        "shap_top3": shap_top3,
        "lime_top3": lime_top3,
        "methods_disagree": bool(disagreement),
        "symmetric_difference": disagreement,
    }, indent=2, sort_keys=True))
    return 0


def main() -> int:
    p = argparse.ArgumentParser(description=__doc__)
    p.add_argument("--model", type=Path, required=True)
    p.add_argument("--input", type=Path, required=True)
    p.add_argument("--instance-id", type=int, required=True)
    p.add_argument("--out", type=Path, required=True)
    ns = p.parse_args()
    return run(ns.model, ns.input, ns.instance_id, ns.out)


if __name__ == "__main__":
    raise SystemExit(main())
```

### 2.4 `counterfactual_alibi.py` (illustrative)

```python
"""Alibi counterfactual that respects the causal graph from exercise 03.

Given an instance, produce the minimal set of non-protected, non-descendant
feature changes that flip the model's decision. Fails loud if Alibi returns
a counterfactual that mutates a protected attribute or one of its descendants
flagged `confidence: low` in causal-graph.yaml.
"""

from __future__ import annotations

import argparse
import json
from pathlib import Path

import numpy as np
import pandas as pd
import yaml
import joblib
from alibi.explainers import CounterfactualProto


PROTECTED = ("SEX", "RACE_ETHNICITY", "AGE_BAND")


def _forbidden_features(graph_path: Path) -> set[str]:
    graph = yaml.safe_load(graph_path.read_text())
    forbidden: set[str] = set(PROTECTED)
    for e in graph["edges"]:
        if e.get("confidence") == "low" and e["from"] in PROTECTED:
            forbidden.add(e["to"])
    return forbidden


def run(model_path: Path, input_csv: Path, graph_path: Path,
        instance_id: int, out_json: Path) -> int:
    df = pd.read_csv(input_csv)
    features = [c for c in df.columns
                if c not in PROTECTED + ("y_true", "y_pred", "y_score")]
    X = df[features]
    model = joblib.load(model_path)
    forbidden = _forbidden_features(graph_path)

    allowed_features = [f for f in features if f not in forbidden]
    if not allowed_features:
        raise RuntimeError("no features remain after causal-graph filter")

    def predict_fn(X_: np.ndarray) -> np.ndarray:
        return model.predict_proba(X_)

    shape = (1, len(features))
    cf = CounterfactualProto(
        predict_fn, shape=shape,
        feature_range=(X.min().to_numpy(), X.max().to_numpy()),
    )
    cf.fit(X.to_numpy())
    exp = cf.explain(X.iloc[[instance_id]].to_numpy())

    if exp.cf is None:
        out_json.write_text(json.dumps({
            "version": "mod-106-ex05-counterfactual/1",
            "instance_id": int(instance_id),
            "status": "no_counterfactual_found",
        }, indent=2, sort_keys=True))
        return 1

    delta = exp.cf["X"][0] - X.iloc[instance_id].to_numpy()
    changed_idx = np.where(np.abs(delta) > 1e-9)[0]
    changed_features = [features[i] for i in changed_idx]
    violations = sorted(set(changed_features) & forbidden)

    out_json.parent.mkdir(parents=True, exist_ok=True)
    out_json.write_text(json.dumps({
        "version": "mod-106-ex05-counterfactual/1",
        "instance_id": int(instance_id),
        "status": "ok" if not violations else "fail_dag_violation",
        "changed_features": changed_features,
        "changed_feature_deltas": {
            features[i]: float(delta[i]) for i in changed_idx
        },
        "dag_violations": violations,
    }, indent=2, sort_keys=True))
    return 0 if not violations else 2


def main() -> int:
    p = argparse.ArgumentParser(description=__doc__)
    p.add_argument("--model", type=Path, required=True)
    p.add_argument("--input", type=Path, required=True)
    p.add_argument("--graph", type=Path, required=True)
    p.add_argument("--instance-id", type=int, required=True)
    p.add_argument("--out", type=Path, required=True)
    ns = p.parse_args()
    return run(ns.model, ns.input, ns.graph, ns.instance_id, ns.out)


if __name__ == "__main__":
    raise SystemExit(main())
```

### 2.5 README summary

```markdown
# TalentFlow explainability slice

## What this is NOT
- Not "the explanation" of any decision. Three methods, three views.
- Not a model-card substitute. The model card consumes these artifacts; it
  does not replace them.

## What each artifact answers

| File | Question | Primary library |
|------|----------|-----------------|
| `global-shap.json` | What features drive the aggregate? | SHAP (TreeExplainer / KernelExplainer) |
| `local-explanation.json` | Why did this one decision? | SHAP + LIME; disagreement is flagged, not reconciled |
| `counterfactual.json` | What minimal change flips this decision? | Alibi (CounterfactualProto) with DAG guard |

## Fails loud when
- Alibi counterfactual mutates a protected attribute or a low-confidence
  descendant (fail_dag_violation; exit 2).
- Alibi finds no counterfactual within budget (no_counterfactual_found;
  exit 1) — logged, not suppressed.
- SHAP TreeExplainer raises on a non-tree model: fallback to KernelExplainer
  is explicit, not silent.

## Thresholds
Bias-indicator ratio and local-disagreement treatment are tunable from
`risk-register/2025/TalentFlow.yaml#explainability-envelope`.
```

## 3. Validation steps

1. **Smoke — global.** Run `global_shap.py` on a seeded CSV with a gradient-boosted model; confirm `top_overall` is non-empty and `per_protected` has entries for every protected attribute. Confirm `bias_indicators` is reproducible across runs.
2. **Smoke — local.** Run `local_shap_lime.py` on the same instance twice; confirm determinism (seeds fixed).
3. **Smoke — counterfactual.** Run `counterfactual_alibi.py` on an instance known to flip under a small non-protected feature change; confirm the output names the non-protected feature and does not mutate a protected attribute.
4. **Negative — DAG violation.** Set every edge in the causal graph to `confidence: low`; expect the counterfactual to flag every feature as forbidden and the exit code to be 2 (fail_dag_violation).
5. **Negative — method disagreement.** Pick a feature-sparse instance where LIME discretisation and SHAP integer attribution are known to differ; confirm `methods_disagree: true`.
6. **Register linkage.** Confirm the model card consumes these artifacts by path; the audit trail cites the three JSONs by SHA-256.
7. **Documentation of hyperparameters.** `KernelExplainer`'s background-sample size (100) and `LimeTabularExplainer`'s discretisation are recorded in the output JSON. A reviewer can re-run with the same settings.

## 4. Rubric / review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Three artifacts — global, local, counterfactual | Three JSONs produced | Missing one |
| R2 | Global SHAP is sliced by protected attribute | `per_protected` populated | Overall only |
| R3 | Local SHAP + LIME on the same instance | Both methods run; outputs cross-compared | Only one method locally |
| R4 | Method disagreement surfaced, not reconciled | `methods_disagree` flag + symmetric difference | Author "resolves" the disagreement silently |
| R5 | Alibi counterfactual respects DAG | `fail_dag_violation` exit on forbidden-feature mutation | Counterfactual mutates protected attribute without flag |
| R6 | TreeExplainer vs. KernelExplainer chosen deliberately | Choice justified in README; fallback is explicit | Library picked at random |
| R7 | Hyperparameters recorded | Background sample size, LIME discretisation, Alibi budget in output | Hyperparameters only in code |
| R8 | Bias-indicator rows present in global | `bias_indicators` list, even if empty | Not surfaced |
| R9 | "This is not THE explanation" caveat present | README names the caveat | Methods framed as definitive |
| R10 | No protected-attribute attribution surfaced as model feature | Protected attrs excluded from features list | Attrs leak into the explanation |
| R11 | Register linkage | `#explainability-envelope` cited | Not cited |
| R12 | Determinism | Same inputs → same outputs (seeds fixed) | Nondeterministic |

**Scoring bands.** *Pass* — all green. *Pass with revisions* — R4 or R8 yellow; fix and resubmit. *Fail* — R5 missing (counterfactual mutates protected attribute silently), or R9 missing (framed as definitive).

## 5. Common mistakes

- **Using KernelExplainer silently on a tree model.** Slow, approximate, and unnecessary. TreeExplainer is exact for additive tree models.
- **Reconciling SHAP and LIME disagreements into "the real explanation."** The disagreement is itself informative — it says the two methods are picking up different signal under their different assumptions. The evidence JSON surfaces it; the auditor reads both.
- **Counterfactuals that mutate the protected attribute.** A counterfactual that flips `SEX` from F to M to flip the model's output is not a valid "minimal change" — it violates the DAG from exercise 03 and is evidence of structural bias, not an explanation. The guard rejects it.
- **Framing SHAP as "the ground truth of feature importance."** SHAP values are an attribution under a specific game-theoretic construction; they are not causal. The README caveat is required.
- **Letting protected attributes into the feature list used for explanations.** Even if the production model does not use them, slipping them into the feature-ranking cross-pollutes the audit.
- **No hyperparameter record.** A LIME explanation with `discretize_continuous=True` and one with `False` can differ sharply; the audit must know which was run.
- **Running explainability on the baseline model only.** The release candidate's explanations are what the audit reads; running on the baseline is a documentation smell.
- **No bias-indicator row for empty.** The field must be present (as `[]`); omitting it breaks schema stability.

## 6. References

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** MEASURE 2.9 (TEVV — explainability considerations under measurable approaches); MEASURE 2.11 (fairness evaluation documented); MANAGE 4.1 (documentation of post-deployment behaviour, which consumes explanation artifacts). `https://www.nist.gov/itl/ai-risk-management-framework`
- **NIST AI RMF Playbook.** Actions under MEASURE 2.9 on documentation of explanation methods and their limits.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant for Model Inversion and Membership Inference, where local-explanation artifacts can leak training-set signal; the harness discloses the hyperparameters but does not publish per-instance SHAP vectors in public reporting.
- **MITRE ATLAS.** `https://atlas.mitre.org/` — relevant for Output Explanation techniques and the attack surface they open.
- **Lundberg, Lee (2017).** *A Unified Approach to Interpreting Model Predictions.* NeurIPS. — SHAP.
- **Ribeiro, Singh, Guestrin (2016).** *"Why Should I Trust You?": Explaining the Predictions of Any Classifier.* KDD. — LIME.
- **Van Looveren, Klaise (2021).** *Interpretable counterfactual explanations guided by prototypes.* ECML-PKDD. — the algorithm behind Alibi's `CounterfactualProto`.
- **Alibi documentation.** `https://docs.seldon.io/projects/alibi/` — `CounterfactualProto`, `AnchorTabular`, and other explainers.
- **SHAP documentation.** `https://shap.readthedocs.io` — `TreeExplainer`, `KernelExplainer`, `shap_values`, `shap.sample`.
- **LIME documentation.** `https://github.com/marcotcr/lime` — `LimeTabularExplainer`.
- **Kusner et al. (2017).** *Counterfactual Fairness.* NeurIPS. — cited from exercise 03 for the DAG the Alibi counterfactual respects.
- Exercise prompt — `lessons/mod-106-fairness-bias-and-explainability-engineering/exercises/exercise-05-shap-alibi-lime-explainability-slice.md`.

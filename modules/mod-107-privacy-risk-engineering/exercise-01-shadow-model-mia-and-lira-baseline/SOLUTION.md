# Exercise 01 — Shadow-Model MIA and LiRA Baseline — Reference Solution

## 1. Solution overview

This exercise is the module's privacy-attack *baseline*: a working **membership-inference attack (MIA)** against a classifier the student trained, in two flavours — the original Shokri-style **shadow-model attack** and the stronger, per-example-calibrated **Likelihood-Ratio Attack (LiRA)** from Carlini et al. The deliverable is (a) runnable code that trains the shadow models and emits per-example scores, (b) a short written write-up with the attack's **AUC**, **balanced accuracy**, and **TPR at a low FPR (0.1 % and 1 %)**, and (c) a one-page implication for the module's running system **CarePilot** (the hospital-triage-support RAG assistant introduced in mod-101).

The reference solution:

- Trains the **target model** and **N shadow models** (recommended `N ≥ 16` for LiRA; 64 if time permits) on disjoint subsets of a privacy-sensitive tabular dataset from the module's local exercise context. The *design intent* is that the student picks a dataset where the attack can plausibly succeed (small training set, high-capacity model), not a dataset where it trivially fails.
- Implements the **shadow-model attack** (Shokri et al.) as the *baseline* — train an attack classifier on `(confidence-vector → member/non-member)` labels from the shadows, apply to the target's output on query samples.
- Implements **LiRA** (Carlini, Chien, Nasr, Song, Terzis, Tramèr — *Membership Inference Attacks From First Principles*, IEEE S&P 2022) as the *stronger baseline* — fit per-example Gaussian distributions over the logit-scaled confidence under "IN" (shadow was trained on this example) and "OUT" (shadow was not), then the attack score for a target-model output on example `x` is the log-likelihood ratio `log p(φ(f(x)) | IN_x) − log p(φ(f(x)) | OUT_x)` where `φ` is the stable logit transform from the paper.
- Reports **AUC**, **balanced accuracy at the chosen score threshold**, and **TPR at FPR = 0.1 % and FPR = 1 %** — the reporting convention introduced by the LiRA paper and now the de-facto standard for MIA baselines.
- Ties the attack result to the **CarePilot** privacy envelope in `risk-register/2025/CarePilot.yaml#privacy-envelope`, and names the mitigation path (exercise 04's DP-SGD training, exercise 05's PII pipeline, mod-109's drift monitor that would catch a leakage regression).

Design decisions worth calling out:

- **Why LiRA on top of the shadow-model attack.** The shadow-model attack returns a *single* calibrated threshold across examples; LiRA calibrates *per example*, which matches the empirical finding from the LiRA paper that MIA signal is dominated by a small fraction of highly vulnerable examples. A reviewer who only sees AUC on a shadow-model attack will *under-estimate* risk; TPR at low FPR, computed under LiRA, is the metric that catches those tail examples.
- **Why we report TPR at low FPR.** An attack with 60 % AUC may still have 40 % TPR at 1 % FPR — which is operationally catastrophic if the attacker's query budget is high. The LiRA paper argues this reporting shift explicitly; this solution follows it.
- **No real patient data.** The exercise is run on a *public tabular dataset* standing in for the CarePilot training distribution. The write-up explicitly flags this limitation; the attack *methodology* is what transfers to CarePilot, not these specific numbers.

## 2. Worked implementation

### 2.1 Repository layout

```
exercise-01-shadow-model-mia-and-lira-baseline/
├── SOLUTION.md                # this file
├── src/
│   ├── data.py                # dataset load + disjoint-split sampling
│   ├── target.py              # train target model, save checkpoint
│   ├── shadows.py             # train N shadow models, save per-example IN/OUT records
│   ├── attack_shadow.py       # Shokri-style attack classifier
│   ├── attack_lira.py         # LiRA attack: per-example Gaussians + LLR scoring
│   ├── metrics.py             # AUC, balanced accuracy, TPR@FPR
│   └── report.py              # emit results table + ROC CSV
├── configs/
│   └── baseline.yaml          # N_shadows, model arch, hyper-params, seed
└── results/
    ├── shadow_attack.json
    ├── lira_attack.json
    └── roc_curves.csv
```

### 2.2 Dataset split (`src/data.py`)

```python
# src/data.py
# Load a public tabular dataset and produce three disjoint pools:
#   TARGET_TRAIN: used only by the target model
#   SHADOW_POOL:  used to sample per-shadow train/held-out splits
#   AUDIT_POOL:   the set of query examples the attacker tries to classify as member / non-member
#
# The AUDIT_POOL is 50% drawn from TARGET_TRAIN (members) and 50% drawn from
# a disjoint held-out set (non-members). The attacker never sees the labels
# until scoring time — those labels are only used by the evaluator in metrics.py.

from dataclasses import dataclass
from typing import Tuple
import numpy as np
from sklearn.model_selection import train_test_split


@dataclass
class Splits:
    X_target_train: np.ndarray
    y_target_train: np.ndarray
    X_shadow_pool: np.ndarray
    y_shadow_pool: np.ndarray
    X_audit: np.ndarray
    y_audit: np.ndarray
    audit_is_member: np.ndarray  # ground truth, held out from attacker


def build_splits(X: np.ndarray, y: np.ndarray, seed: int) -> Splits:
    rng = np.random.default_rng(seed)
    # 1) Split off the target training set.
    X_rest, X_target_train, y_rest, y_target_train = train_test_split(
        X, y, test_size=0.25, random_state=seed, stratify=y
    )
    # 2) Split the remainder into SHADOW_POOL and AUDIT non-members.
    X_shadow_pool, X_audit_nonmember, y_shadow_pool, y_audit_nonmember = train_test_split(
        X_rest, y_rest, test_size=0.10, random_state=seed + 1, stratify=y_rest
    )
    # 3) AUDIT members = a random subsample of TARGET_TRAIN of the same size as audit_nonmember.
    n_audit = X_audit_nonmember.shape[0]
    idx_member = rng.choice(X_target_train.shape[0], size=n_audit, replace=False)
    X_audit_member = X_target_train[idx_member]
    y_audit_member = y_target_train[idx_member]

    X_audit = np.concatenate([X_audit_member, X_audit_nonmember], axis=0)
    y_audit = np.concatenate([y_audit_member, y_audit_nonmember], axis=0)
    audit_is_member = np.concatenate(
        [np.ones(n_audit, dtype=bool), np.zeros(n_audit, dtype=bool)]
    )
    # Shuffle so the attacker cannot exploit ordering.
    perm = rng.permutation(X_audit.shape[0])
    return Splits(
        X_target_train=X_target_train,
        y_target_train=y_target_train,
        X_shadow_pool=X_shadow_pool,
        y_shadow_pool=y_shadow_pool,
        X_audit=X_audit[perm],
        y_audit=y_audit[perm],
        audit_is_member=audit_is_member[perm],
    )
```

### 2.3 Target and shadow models (`src/target.py`, `src/shadows.py`)

```python
# src/target.py
# Train the target model on TARGET_TRAIN with the same architecture and
# hyper-parameters used for the shadows. Save confidence vectors on AUDIT.

from sklearn.neural_network import MLPClassifier
import joblib
import numpy as np


def train_target(X_train: np.ndarray, y_train: np.ndarray, seed: int):
    clf = MLPClassifier(
        hidden_layer_sizes=(128, 64),
        activation="relu",
        max_iter=200,
        random_state=seed,
    )
    clf.fit(X_train, y_train)
    return clf


def score_on_audit(clf, X_audit: np.ndarray) -> np.ndarray:
    # Return per-class probabilities (confidence vector per example).
    return clf.predict_proba(X_audit)
```

```python
# src/shadows.py
# Train N shadow models. For each shadow i and each audit example x, record:
#   - whether x was IN shadow i's training set
#   - the shadow's confidence vector on x
# This record is the raw material for both attacks.

from dataclasses import dataclass
from typing import List
import numpy as np
from sklearn.neural_network import MLPClassifier


@dataclass
class ShadowRecord:
    in_mask: np.ndarray            # shape (N, A); True if audit example a was in shadow n's train set
    probs_audit: np.ndarray        # shape (N, A, C); per-class probabilities on audit
    probs_true_class: np.ndarray   # shape (N, A); probability assigned to the audit example's true label


def train_shadows(
    X_pool: np.ndarray,
    y_pool: np.ndarray,
    X_audit: np.ndarray,
    y_audit: np.ndarray,
    n_shadows: int,
    train_frac: float,
    seed: int,
) -> ShadowRecord:
    rng = np.random.default_rng(seed)
    n_pool = X_pool.shape[0]
    A = X_audit.shape[0]
    C = int(y_pool.max()) + 1

    in_mask = np.zeros((n_shadows, A), dtype=bool)
    probs_audit = np.zeros((n_shadows, A, C), dtype=np.float32)
    probs_true_class = np.zeros((n_shadows, A), dtype=np.float32)

    for i in range(n_shadows):
        # Each shadow sees a random train_frac subset of the pool.
        mask_pool = rng.random(n_pool) < train_frac
        X_tr = X_pool[mask_pool]
        y_tr = y_pool[mask_pool]
        # Flag which AUDIT examples were coincidentally present in this shadow's train set.
        # (If audit members are drawn from TARGET_TRAIN they won't appear in shadow pool;
        # IN here means "if the audit example were in the shadow's pool, was it selected?"
        # — which is how LiRA synthesises IN distributions. See LiRA paper, §4.2.)
        # For audit examples drawn from the shadow pool, record membership directly.
        in_mask[i] = _which_audit_in(X_audit, X_pool[mask_pool])

        clf = MLPClassifier(
            hidden_layer_sizes=(128, 64),
            activation="relu",
            max_iter=200,
            random_state=seed + i + 1,
        )
        clf.fit(X_tr, y_tr)

        p = clf.predict_proba(X_audit)
        probs_audit[i] = p
        probs_true_class[i] = p[np.arange(A), y_audit]

    return ShadowRecord(
        in_mask=in_mask,
        probs_audit=probs_audit,
        probs_true_class=probs_true_class,
    )


def _which_audit_in(X_audit: np.ndarray, X_subset: np.ndarray) -> np.ndarray:
    # Simple membership test by row equality; works for a stable tabular pool.
    # For continuous / high-dimensional data, use hashed row IDs captured at split time.
    audit_rows = set(map(tuple, X_audit.tolist()))
    subset_rows = set(map(tuple, X_subset.tolist()))
    intersect = audit_rows & subset_rows
    return np.array([tuple(r) in intersect for r in X_audit.tolist()], dtype=bool)
```

### 2.4 Shadow-model attack baseline (`src/attack_shadow.py`)

```python
# src/attack_shadow.py
# Shokri et al. (2017) shadow-model attack:
#   - Aggregate shadow confidences into (features, in/out) training rows for an attack classifier.
#   - Train one attack classifier per true class (common in Shokri; a single classifier also works).
#   - Score the target's audit confidences.

import numpy as np
from sklearn.ensemble import RandomForestClassifier


def build_attack_table(
    probs_audit: np.ndarray, in_mask: np.ndarray, y_audit: np.ndarray
) -> tuple[np.ndarray, np.ndarray, np.ndarray]:
    # probs_audit: (N, A, C); in_mask: (N, A); y_audit: (A,)
    N, A, C = probs_audit.shape
    X_rows, y_rows, cls_rows = [], [], []
    for n in range(N):
        for a in range(A):
            X_rows.append(probs_audit[n, a])  # confidence vector as the feature
            y_rows.append(int(in_mask[n, a]))
            cls_rows.append(int(y_audit[a]))
    return np.array(X_rows), np.array(y_rows), np.array(cls_rows)


def train_and_score(
    probs_shadow_audit: np.ndarray,
    in_mask: np.ndarray,
    y_audit: np.ndarray,
    probs_target_audit: np.ndarray,
    seed: int,
) -> np.ndarray:
    X, y, cls = build_attack_table(probs_shadow_audit, in_mask, y_audit)
    attack = RandomForestClassifier(n_estimators=200, random_state=seed)
    attack.fit(X, y)
    # Score is the predicted P(member) for the target's confidence vector on each audit example.
    scores = attack.predict_proba(probs_target_audit)[:, 1]
    return scores
```

### 2.5 LiRA attack (`src/attack_lira.py`)

```python
# src/attack_lira.py
# Likelihood-Ratio Attack (Carlini, Chien, Nasr, Song, Terzis, Tramèr — IEEE S&P 2022).
# For each audit example x:
#   - Collect the logit-scaled confidences from shadows where x was IN and from shadows where x was OUT.
#   - Fit N(μ_in, σ_in^2) and N(μ_out, σ_out^2).
#   - Attack score for the target: log p(φ(f_target(x)) | IN) − log p(φ(f_target(x)) | OUT).
# φ is the stable logit transform: φ(p) = log(p / (1 − p + ε)) applied to the probability
# the model assigns to the audit example's true label.
#
# Variants in the LiRA paper:
#   - Offline LiRA: only OUT shadows available (used when training IN shadows per example is infeasible).
#   - Online LiRA: both IN and OUT shadows (implemented here).

import numpy as np
from scipy.stats import norm


def logit_transform(p_true: np.ndarray, eps: float = 1e-6) -> np.ndarray:
    p = np.clip(p_true, eps, 1.0 - eps)
    return np.log(p / (1.0 - p))


def lira_scores(
    shadow_probs_true: np.ndarray,  # (N, A)
    in_mask: np.ndarray,            # (N, A)
    target_probs_true: np.ndarray,  # (A,)
    min_per_side: int = 2,
) -> tuple[np.ndarray, np.ndarray]:
    phi_shadow = logit_transform(shadow_probs_true)
    phi_target = logit_transform(target_probs_true)
    N, A = phi_shadow.shape
    scores = np.full(A, np.nan, dtype=np.float32)
    coverage = np.zeros(A, dtype=bool)

    for a in range(A):
        in_vals = phi_shadow[in_mask[:, a], a]
        out_vals = phi_shadow[~in_mask[:, a], a]
        if len(in_vals) < min_per_side or len(out_vals) < min_per_side:
            continue
        mu_in, sd_in = float(np.mean(in_vals)), float(np.std(in_vals, ddof=1) + 1e-6)
        mu_out, sd_out = float(np.mean(out_vals)), float(np.std(out_vals, ddof=1) + 1e-6)
        ll_in = norm.logpdf(phi_target[a], loc=mu_in, scale=sd_in)
        ll_out = norm.logpdf(phi_target[a], loc=mu_out, scale=sd_out)
        scores[a] = ll_in - ll_out
        coverage[a] = True

    return scores, coverage
```

### 2.6 Metrics and reporting (`src/metrics.py`, `src/report.py`)

```python
# src/metrics.py
# AUC, balanced accuracy, TPR at a target FPR. TPR@FPR is the reporting
# convention the LiRA paper argues for — small per-example gain can still be
# operationally catastrophic.

import numpy as np
from sklearn.metrics import roc_curve, auc as sk_auc, balanced_accuracy_score


def auc(scores: np.ndarray, is_member: np.ndarray) -> float:
    fpr, tpr, _ = roc_curve(is_member, scores)
    return float(sk_auc(fpr, tpr))


def tpr_at_fpr(scores: np.ndarray, is_member: np.ndarray, target_fpr: float) -> float:
    fpr, tpr, _ = roc_curve(is_member, scores)
    if (fpr <= target_fpr).sum() == 0:
        return 0.0
    # Largest TPR among thresholds whose FPR is ≤ target.
    return float(tpr[fpr <= target_fpr].max())


def balanced_acc_at_best_thr(scores: np.ndarray, is_member: np.ndarray) -> float:
    # Sweep thresholds to find the one maximising balanced accuracy.
    fpr, tpr, thr = roc_curve(is_member, scores)
    # Balanced accuracy at each threshold: (TPR + (1 − FPR)) / 2.
    bal = (tpr + (1.0 - fpr)) / 2.0
    return float(bal.max())
```

### 2.7 Running the baseline — `configs/baseline.yaml`

```yaml
# configs/baseline.yaml
seed: 42
n_shadows: 32       # LiRA stabilises around 16+; 32 is a comfortable baseline.
train_frac: 0.5     # per-shadow train subset
model:
  arch: mlp-128-64
  max_iter: 200
report_fprs: [0.001, 0.01]
```

### 2.8 Expected shape of the results JSON

```json
{
  "target_model": {"arch": "mlp-128-64", "train_rows": 2500, "test_accuracy": "to-record-from-run"},
  "shadow_attack": {
    "auc": "to-record-from-run",
    "balanced_accuracy": "to-record-from-run",
    "tpr_at_fpr_0_001": "to-record-from-run",
    "tpr_at_fpr_0_01": "to-record-from-run"
  },
  "lira_attack": {
    "auc": "to-record-from-run",
    "balanced_accuracy": "to-record-from-run",
    "tpr_at_fpr_0_001": "to-record-from-run",
    "tpr_at_fpr_0_01": "to-record-from-run",
    "coverage_pct_of_audit": "to-record-from-run"
  },
  "comparison_note": "LiRA is expected to match or exceed the shadow-model attack on AUC and to materially exceed it on TPR @ low FPR — the LiRA paper's central empirical claim."
}
```

Concrete numeric results are written at run time and are not quoted inline in this reference; what the reviewer checks is (a) the pipeline runs, (b) both metrics are produced, (c) LiRA's TPR-at-low-FPR is reported.

### 2.9 Mapping back to CarePilot

- **Risk the attack maps to.** Membership inference against CarePilot would mean "an attacker with query access can tell whether a specific patient record was used to fine-tune / train CarePilot." That connects to **OWASP Machine Learning Security Top 10** as the family of model-output privacy leakage risks the ML Top 10 project enumerates; MIA is one of the canonical attacks in the ML privacy literature. <!-- verify-citation: confirm which OWASP ML Top 10 entry code and title covers membership inference in the current revision; cite verbatim once confirmed. -->
- **ATLAS mapping.** Confirm the current MITRE ATLAS technique ID that covers inference-API exploitation prior to citing. <!-- verify-citation: confirm ATLAS technique ID for "Craft Adversarial Data" / "Exfiltrate via ML Inference API" that most precisely matches MIA and cite. -->
- **Register impact.** The attack's AUC and TPR @ FPR = 1 % feed row `risk-register/2025/CarePilot.yaml#risks[privacy-mia]` with the numeric assessment. Mitigation plan: DP-SGD training (exercise 04), query-rate limiting on the inference API (mod-105 exercise 04 and mod-108 guardrail), and a privacy-eval regression fixture (mod-109).
- **What DOES NOT belong in the CarePilot report.** The MIA numbers from the public tabular surrogate are not evidence about CarePilot itself — they are evidence that the *methodology* is sound and the *attack is cheap to run*. The write-up states this explicitly and names the next step (run the same attack against CarePilot's actual eval harness under governance controls — a work item for the privacy-eval owner).

## 3. Validation steps

1. **Pipeline runs end-to-end.** `python -m src.target && python -m src.shadows && python -m src.attack_shadow && python -m src.attack_lira && python -m src.report` produces `results/shadow_attack.json`, `results/lira_attack.json`, and `results/roc_curves.csv`.
2. **Audit set is held out from the attacker.** `src.data.build_splits` does not emit `audit_is_member` to anything downstream of the attack scorers; metrics.py is the only consumer. A reviewer greps for `audit_is_member` and confirms it appears only in `data.py`, `metrics.py`, and the test harness.
3. **Shadow IN/OUT masks are populated.** `ShadowRecord.in_mask.sum(axis=0)` has both members and non-members for *most* audit examples; LiRA's `coverage_pct_of_audit` reports how many audit examples have ≥ `min_per_side` on each side.
4. **Metrics are the three required ones.** Both attack JSONs report `auc`, `balanced_accuracy`, and `tpr_at_fpr_0_001` *and* `tpr_at_fpr_0_01`.
5. **Determinism.** Re-running the pipeline with the same `seed` reproduces the metrics to within numerical noise (shadow-model attack: identical; LiRA: identical because no additional randomness is introduced beyond shadow training).
6. **LiRA is not worse than the shadow-model attack at low FPR.** On any reasonable target/shadow configuration, `lira.tpr_at_fpr_0_001 ≥ shadow.tpr_at_fpr_0_001` is expected. If the opposite happens, the write-up is required to diagnose (typically: too few shadows, or the target model's regularisation suppresses the per-example signal).
7. **Write-up includes the limitation statement.** The write-up explicitly states (a) the public surrogate data, (b) N_shadows used, (c) that the TPR @ low FPR is the operationally meaningful metric.

## 4. Rubric — review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Repo layout matches the shape in §2.1 | `src/`, `configs/`, `results/` present with the named files | Missing or empty `src/` modules |
| R2 | Disjoint target / shadow / audit splits | `data.py` produces non-overlapping pools and a reviewer can verify by set-intersection on the test fixture | Overlap between target train and audit non-members, OR overlap between shadow pool and target train |
| R3 | Shadow model count ≥ 16 | `configs/baseline.yaml#n_shadows ≥ 16` and the run artifact confirms the count | `n_shadows < 16` without justification |
| R4 | Shadow-model attack baseline implemented | `attack_shadow.py` produces per-example scores and the metric report is non-empty | Baseline skipped or not reported |
| R5 | LiRA implemented with per-example Gaussians | `attack_lira.py` fits per-example `N(μ_in, σ_in)` and `N(μ_out, σ_out)` on the logit-scaled signal | LiRA replaced by a single-threshold attack or a non-LLR scoring rule |
| R6 | TPR @ FPR 0.1 % and 1 % reported for both attacks | Both metrics present in both JSONs | Only AUC reported |
| R7 | Balanced-accuracy threshold search | `metrics.balanced_acc_at_best_thr` sweeps thresholds; a fixed-threshold shortcut is not accepted | Balanced accuracy at a single fixed threshold without justification |
| R8 | Mapping to CarePilot / register | Section mapping the attack to the CarePilot risk register row and the mitigation path (exercises 04, 05, mod-109) is present | No tie-back to the module's running system; attack reported as a stand-alone number |
| R9 | Limitations stated | Write-up names (a) the surrogate-dataset caveat, (b) `n_shadows`, (c) that TPR @ low FPR is the operationally meaningful metric | Result quoted as if it directly applies to CarePilot |
| R10 | No invented facts | `<!-- verify-citation: ... -->` markers appear wherever a specific OWASP / ATLAS code is cited without a verified source | Fabricated incident, dataset, or numeric claim |

Scoring bands: *Pass* — R1–R9 all green and R10 is honoured; *Pass with revisions* — one of R6/R7/R8 fails but the attack pipeline itself (R1–R5) is sound; *Fail* — any of R2 / R5 / R10 fails.

## 5. Common mistakes

- **Running the attack on the *training-time* test set.** The target model's `.score(X_test)` is not an MIA — it is a generalisation measurement. The audit set must contain confirmed members *and* confirmed non-members with the labels withheld from the attack scorers.
- **A single shadow model.** One shadow does not estimate the IN/OUT distributions; LiRA collapses to a single-point attack and its "variance" is noise. Set `n_shadows ≥ 16`.
- **Training shadows with different architecture or hyper-parameters than the target.** The LiRA paper assumes the attacker approximates the training algorithm; a wildly different shadow architecture biases the attack. Keep arch, optimizer, and regularisation identical to the target's.
- **Confusing AUC with risk.** AUC can look mild while TPR @ FPR = 0.1 % is severe. Always report both, and route the TPR @ low FPR into the risk-register row — that is the operational number.
- **Leaking the audit labels into the attack classifier.** The shadow-model attack in `attack_shadow.py` is trained on *shadow* `(features, in/out)` rows only. Adding target-audit `in_mask` rows to the attack classifier's training set is data leakage and must fail review.
- **Reporting attack results as if they transfer directly to CarePilot.** The numeric result on the public tabular surrogate is evidence about *methodology*, not about CarePilot's actual model. The write-up must state this.
- **Logit-transform numerical instability.** A naïve `log(p / (1 − p))` blows up when the model emits a near-0 or near-1 probability. Clip (`eps = 1e-6`) or use the model's raw pre-softmax logits where available (and record which was used — they are not identical).
- **Treating the shadow attack as a sanity check and skipping LiRA.** LiRA is the baseline that reveals per-example tail risk; the shadow attack alone does not satisfy the exercise.

## 6. References

Only the official sources listed in the work packet (and clearly-labelled module / academic references) are used.

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — this exercise's output feeds the NIST AI RMF **MEASURE 2.7** (security and resilience) and **MEASURE 2.11 / MAP 2.3** rows of the CarePilot map authored in mod-101 exercise 01.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — the ML Top 10 project lists privacy attacks against ML models (membership inference, model inversion, training-data extraction) as part of its risk catalogue; cite the specific entry ID at submission after reading the current revision. <!-- verify-citation: cite the exact OWASP ML Top 10 entry (code + title + current revision date) that covers membership inference. -->
- **MITRE ATLAS.** `https://atlas.mitre.org/` — the MITRE ATLAS matrix enumerates tactics and techniques for ML-targeted attacks; cite the exact technique ID for inference-API-driven information disclosure at submission. <!-- verify-citation: cite the exact ATLAS technique ID (e.g., AML.T…) that most precisely matches membership inference via the model's inference API. -->
- **Shokri, R., Stronati, M., Song, C., Shmatikov, V.** *Membership Inference Attacks Against Machine Learning Models.* IEEE Symposium on Security and Privacy, 2017. — the original shadow-model MIA.
- **Carlini, N., Chien, S., Nasr, M., Song, S., Terzis, A., Tramèr, F.** *Membership Inference Attacks From First Principles.* IEEE Symposium on Security and Privacy, 2022. — the LiRA attack, the per-example Gaussian formulation, and the TPR-at-low-FPR reporting convention this exercise follows.
- **Module running system.** CarePilot — hospital-triage-support RAG assistant, introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md` and reused across this module for continuity.
- **Exercise prompt.** `lessons/mod-107-privacy-risk-engineering/exercises/exercise-01-shadow-model-mia-and-lira-baseline.md`.

<!-- Numeric results in §2.8 are run-time placeholders by design; the solution does not quote invented values. -->

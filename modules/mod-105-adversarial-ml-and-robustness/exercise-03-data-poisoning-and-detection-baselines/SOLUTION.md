# Exercise 03 — Data Poisoning Attacks and Detection–Defence Baselines — Reference Solution

## 1. Solution overview

This reference walks a worked `mod-105-ex03-poisoning-panel/` deliverable for **CarePilot**. The target is the mod-106 auxiliary triage classifier (PredAI, tabular) — the only CarePilot component that is retrained on hospital-curated labels and therefore in scope for training-time attacks (see mod-105 ex-01 cross-tag rows P1, P2, P3). The RAG corpus poisoning path (ex-01 row E4) is cross-referenced to mod-104 and is out of scope for this training-time panel.

The deliverable has three pieces:

1. A **poisoning panel** (`panel/poison.py`) that produces three attack variants against the auxiliary classifier: (a) label-flip availability poisoning, (b) clean-label targeted poisoning, and (c) BadNets-style backdoor insertion with a feature-space trigger. Each variant has a configurable poison fraction and produces a `{clean, poisoned}_train.parquet` pair plus an evaluation split for the backdoor trigger.
2. A **detection–defence panel** (`panel/defend.py`) that runs three complementary baselines: (i) spectral-signature defence (Tran et al., 2018; implemented in ART as `SpectralSignatureDefense`), (ii) activation-clustering defence (Chen et al., 2018; ART `ActivationDefence`), and (iii) a trigger-reconstruction-style scan inspired by Neural Cleanse (Wang et al., 2019) for the backdoor case. Each defence outputs a per-sample suspicion score; thresholds come from config, not from code.
3. A **REPORT.md** template the panel populates with (a) attack effectiveness (clean accuracy delta, backdoor success rate, targeted-sample flip rate), (b) defence effectiveness (detection AUC against known-poisoned samples, precision/recall at the chosen threshold), and (c) a residual-risk roll-up for ex-05.

Design decisions:

- **Scope limited to retrained components.** Poisoning the frozen vendor LLM is not in CarePilot's ownership; the panel runs against the mod-106 auxiliary head. Row P4 (supply-chain) is scope-ruled to mod-110. The scoping is cited inline so a reviewer does not have to re-derive it.
- **Published attacks only.** Label-flip and BadNets-style backdoor insertion are canonical; both are available in ART's `art.attacks.poisoning` module. Clean-label poisoning uses ART's `FeatureCollisionAttack` or an equivalent (Poison Frogs! style from Shafahi et al., 2018), again via the ART implementation.
- **Published defences only.** Spectral signatures and activation clustering are the two training-time poisoning defences shipped in ART; they correspond to the two canonical papers (Tran 2018, Chen 2018). Neural Cleanse (Wang 2019) is referenced as the standard backdoor-trigger-reconstruction defence and implemented using ART's defence API where available, otherwise as a cited reproduction.
- **Thresholds live in config.** Suspicion thresholds, poison-fraction budgets, and detection-AUC targets are config-level and traceable to the CarePilot risk register. The panel never hard-codes a "model is poisoned" cutoff.
- **Backdoor vs targeted vs availability are kept separate.** They have different threat models, different success metrics, and different defences. Collapsing them hides the fact that spectral-signature/activation-clustering defences are backdoor- and targeted-poisoning defences; label-flip availability poisoning needs robust-training defences (e.g., trimmed loss) that are *not* exercised by this panel.
- **"Detection" is measured as a classifier over clean vs poisoned.** Detection AUC is only meaningful when the panel knows which samples are poisoned (which the attacker side of the panel does). In production, the labels are unknown — so the report is explicit that detection-AUC is a *development-time* measurement, not a runtime signal.

## 2. Worked implementation — `mod-105-ex03-poisoning-panel/`

### 2.1 Directory layout

```
mod-105-ex03-poisoning-panel/
├── README.md
├── config.yaml              # attack + defence pins, budgets, thresholds, seed
├── requirements.txt         # torch, art, numpy, pandas, scikit-learn, pyarrow
├── panel/
│   ├── __init__.py
│   ├── io.py                # data load/save; splits; seed
│   ├── train.py             # train the auxiliary triage classifier
│   ├── poison.py            # attack side: label-flip, clean-label, BadNets backdoor
│   ├── defend.py            # defence side: spectral, activation clustering, trigger-reconstruction
│   ├── metrics.py           # clean acc, backdoor ASR, detection AUC, P/R @ threshold
│   └── report.py            # write REPORT.md
├── run_panel.py             # entry point
├── artifacts/{run_id}/      # per-run artefacts
└── REPORT.md
```

### 2.2 Config — `config.yaml`

```yaml
# mod-105 ex-03 — poisoning + detection panel
# References: mod-105/exercise-01 cross-tag rows P1, P2, P3.
# Scope: CarePilot mod-106 auxiliary triage classifier only.
# Out of scope: base LLM (frozen; supply chain is mod-110), RAG corpus (mod-104/105 ex-01 E4).

run:
  id: carepilot-poisoning-panel-0001
  seed: 20260101
  output_dir: artifacts/${run.id}
  log_level: INFO

target:
  kind: tabular
  framework: pytorch
  arch: models/predai/triage-classifier.arch.py   # model constructor
  data:
    base_splits: data/predai/splits.yaml
    train_split: train
    test_split: test
    trigger_eval_split: trigger_eval              # built from test split; contains trigger
  classes: [routine, urgent, immediate]
  normalization:
    scaler: models/predai/scaler.joblib

# -------- Attacks --------
attacks:
  - id: label-flip-availability
    kind: label_flip
    fraction: 0.05                                 # 5% of training set
    pairs:
      - from: routine
        to: immediate                              # worst-case flip (routine → immediate degrades downstream behavior most)
      - from: urgent
        to: routine
    plausibility_under_carepilot: >
      IN if an insider has write access to the labeled-training set. Register
      row data-poisoning-availability; mitigation sits in access-control (mod-110)
      plus this panel's detection baselines.

  - id: clean-label-targeted
    kind: clean_label               # ART FeatureCollisionAttack (Shafahi et al. 2018)
    fraction: 0.02
    source_class: routine           # attacker wants a specific "routine" test input misclassified as "immediate"
    target_class: immediate
    feature_layer: penultimate       # collision computed in the penultimate activation space
    max_iter: 100
    learning_rate: 0.01
    plausibility_under_carepilot: >
      IN if an insider can submit labeled examples with correct labels but
      crafted features that collide with a target example in activation space.

  - id: badnets-backdoor
    kind: badnets                   # ART PoisoningAttackBackdoor (Gu et al. 2017)
    fraction: 0.03
    trigger:
      kind: feature_mask            # tabular analogue of the image patch trigger
      feature_indices: [12, 29]     # two features set to a specific value pattern
      values: [3.0, -3.0]           # outside the typical ±2σ range for these features
    target_class: routine           # trigger forces "routine" regardless of true class
    plausibility_under_carepilot: >
      IN if an insider can inject training rows with a trigger pattern they
      control at inference time (e.g., a specific chart annotation they can
      also submit). Register row backdoor-trojan.

# -------- Defences --------
defences:
  - id: spectral-signatures
    kind: spectral
    recipe: ART.SpectralSignatureDefense
    batch_size: 128
    eps_multiplier: 1.5             # suspicion cutoff: eps_multiplier × median singular-value projection
    applies_to: [label-flip-availability, clean-label-targeted, badnets-backdoor]

  - id: activation-clustering
    kind: activation_clustering
    recipe: ART.ActivationDefence
    n_clusters: 2
    clustering_method: KMeans
    reduce: FastICA
    nb_dims: 10
    cluster_analysis: smaller         # the smaller cluster is flagged suspicious (per Chen et al. 2018)
    applies_to: [clean-label-targeted, badnets-backdoor]

  - id: trigger-reconstruction
    kind: neural_cleanse_lite
    recipe: custom  # Neural Cleanse (Wang et al. 2019) style; per-class trigger L1-norm search
    max_iter: 100
    learning_rate: 0.01
    anomaly_index_threshold: 2.0      # MAD-based anomaly index from Wang et al. 2019
    applies_to: [badnets-backdoor]

report:
  path: REPORT.md
  envelope_source: risk-register/.../data-poisoning-envelope.yaml   # cited, not inlined
```

### 2.3 Attack side — `panel/poison.py` (excerpt)

```python
"""mod-105 ex-03 — poisoning attacks against the CarePilot auxiliary triage classifier."""
from __future__ import annotations

import logging
from typing import Any

import numpy as np

from art.attacks.poisoning import (
    FeatureCollisionAttack,
    PoisoningAttackBackdoor,
)
from art.attacks.poisoning.perturbations import insert_image  # tabular equivalent below
from art.estimators.classification import PyTorchClassifier


log = logging.getLogger(__name__)


def apply_label_flip(
    x: np.ndarray, y: np.ndarray, pairs: list[dict[str, str]], fraction: float, classes: list[str], rng: np.random.Generator
) -> tuple[np.ndarray, np.ndarray, np.ndarray]:
    """Flip labels of a random `fraction` of samples from pair.from to pair.to.
    Returns (x_poisoned, y_poisoned, is_poisoned_mask)."""
    class_to_idx = {c: i for i, c in enumerate(classes)}
    y_poisoned = y.copy()
    is_poisoned = np.zeros(len(y), dtype=bool)
    for pair in pairs:
        from_idx = class_to_idx[pair["from"]]
        to_idx = class_to_idx[pair["to"]]
        candidate = np.where(y == from_idx)[0]
        n_flip = int(fraction * len(candidate))
        chosen = rng.choice(candidate, size=n_flip, replace=False)
        y_poisoned[chosen] = to_idx
        is_poisoned[chosen] = True
    return x.copy(), y_poisoned, is_poisoned


def _tabular_trigger(values: list[float], indices: list[int]):
    def _apply(x: np.ndarray) -> np.ndarray:
        patched = x.copy()
        for feat_idx, val in zip(indices, values):
            patched[..., feat_idx] = val
        return patched
    return _apply


def apply_badnets_backdoor(
    x: np.ndarray, y: np.ndarray, classes: list[str], cfg: dict[str, Any], rng: np.random.Generator
) -> tuple[np.ndarray, np.ndarray, np.ndarray]:
    class_to_idx = {c: i for i, c in enumerate(classes)}
    target_idx = class_to_idx[cfg["target_class"]]
    frac = cfg["fraction"]

    trigger_fn = _tabular_trigger(cfg["trigger"]["values"], cfg["trigger"]["feature_indices"])
    # Wrap the trigger as an ART PoisoningAttackBackdoor.
    backdoor = PoisoningAttackBackdoor(trigger_fn)

    n_total = len(x)
    n_poison = int(frac * n_total)
    idx = rng.choice(np.arange(n_total), size=n_poison, replace=False)
    x_poison_subset = x[idx]
    y_target = np.full(n_poison, target_idx, dtype=np.int64)
    x_poisoned_subset, y_poisoned_subset = backdoor.poison(x_poison_subset, y_target)

    x_full = x.copy()
    y_full = y.copy()
    x_full[idx] = x_poisoned_subset
    y_full[idx] = y_poisoned_subset
    is_poisoned = np.zeros(n_total, dtype=bool)
    is_poisoned[idx] = True
    return x_full, y_full, is_poisoned


def apply_clean_label_targeted(
    x_train: np.ndarray, y_train: np.ndarray, x_target: np.ndarray, classifier: PyTorchClassifier,
    source_class: int, target_class: int, fraction: float, max_iter: int, lr: float, rng
):
    """Poison Frogs! / FeatureCollisionAttack: craft poisons that collide with x_target
    in the penultimate feature space but keep their 'clean' source_class label."""
    idx_pool = np.where(y_train == source_class)[0]
    n_poison = max(1, int(fraction * len(idx_pool)))
    chosen = rng.choice(idx_pool, size=n_poison, replace=False)
    x_seed = x_train[chosen]

    attack = FeatureCollisionAttack(
        classifier=classifier,
        target=x_target,
        feature_layer="penultimate",
        learning_rate=lr,
        max_iter=max_iter,
    )
    x_poison, y_poison = attack.poison(x_seed)

    x_full = x_train.copy()
    x_full[chosen] = x_poison
    y_full = y_train.copy()  # labels unchanged — "clean label"
    is_poisoned = np.zeros(len(x_train), dtype=bool)
    is_poisoned[chosen] = True
    return x_full, y_full, is_poisoned
```

### 2.4 Defence side — `panel/defend.py` (excerpt)

```python
"""mod-105 ex-03 — detection–defence baselines."""
from __future__ import annotations

import logging
from typing import Any

import numpy as np

from art.defences.detector.poison import ActivationDefence, SpectralSignatureDefense


log = logging.getLogger(__name__)


def spectral_signature_scan(
    classifier, x: np.ndarray, y: np.ndarray, is_poisoned_gt: np.ndarray, cfg: dict[str, Any]
) -> dict[str, Any]:
    """Tran, Carlini, Madry 2018 — spectral signatures. Flags samples whose
    projection onto the top singular direction of per-class activations is
    large (an outlier). ART ships this as SpectralSignatureDefense."""
    defence = SpectralSignatureDefense(
        classifier=classifier,
        x_train=x,
        y_train=y,
        batch_size=cfg["batch_size"],
        eps_multiplier=cfg["eps_multiplier"],
    )
    report, is_clean_pred = defence.detect_poison()
    is_flagged = ~np.asarray(is_clean_pred).astype(bool)
    metrics = _detection_metrics(is_flagged, is_poisoned_gt)
    return {"defence": "spectral-signatures", "metrics": metrics, "report": _summarize_report(report)}


def activation_clustering_scan(
    classifier, x: np.ndarray, y: np.ndarray, is_poisoned_gt: np.ndarray, cfg: dict[str, Any]
) -> dict[str, Any]:
    """Chen et al. 2018 — activation clustering. Per-class: cluster activations
    into 2 groups; flag the smaller cluster as suspicious. ART ships this as
    ActivationDefence."""
    defence = ActivationDefence(
        classifier=classifier,
        x_train=x,
        y_train=y,
    )
    report, is_clean_pred = defence.detect_poison(
        nb_clusters=cfg["n_clusters"],
        clustering_method=cfg["clustering_method"],
        reduce=cfg["reduce"],
        nb_dims=cfg["nb_dims"],
        cluster_analysis=cfg["cluster_analysis"],
    )
    is_flagged = ~np.asarray(is_clean_pred).astype(bool)
    metrics = _detection_metrics(is_flagged, is_poisoned_gt)
    return {"defence": "activation-clustering", "metrics": metrics, "report": _summarize_report(report)}


def trigger_reconstruction_scan(
    classifier, x: np.ndarray, y: np.ndarray, classes: list[str], cfg: dict[str, Any]
) -> dict[str, Any]:
    """Neural-Cleanse-style trigger reconstruction (Wang et al. 2019).

    For each candidate target class c, optimise a small L1-norm feature
    perturbation that, when applied to inputs of any class, causes the model
    to predict c. The class whose reconstructed trigger has anomalously low
    L1 norm (MAD-based anomaly index) is flagged as the backdoor target.
    """
    import torch
    anomaly_scores = {}
    model = classifier.model
    model.eval()
    for c, cname in enumerate(classes):
        norm = _reconstruct_trigger_norm(model, x, target_class=c, max_iter=cfg["max_iter"], lr=cfg["learning_rate"])
        anomaly_scores[cname] = float(norm)
    values = np.array(list(anomaly_scores.values()))
    mad = np.median(np.abs(values - np.median(values)))
    # Anomaly index per Wang et al. 2019: (median - value) / (1.4826 * MAD).
    anomaly_index = {k: float((np.median(values) - v) / (1.4826 * mad)) for k, v in anomaly_scores.items()}
    flagged = {k: v > cfg["anomaly_index_threshold"] for k, v in anomaly_index.items()}
    return {
        "defence": "trigger-reconstruction",
        "anomaly_index": anomaly_index,
        "flagged_targets": [k for k, v in flagged.items() if v],
    }


def _detection_metrics(is_flagged: np.ndarray, is_poisoned_gt: np.ndarray) -> dict[str, float]:
    tp = int(((is_flagged) & (is_poisoned_gt)).sum())
    fp = int(((is_flagged) & (~is_poisoned_gt)).sum())
    fn = int(((~is_flagged) & (is_poisoned_gt)).sum())
    tn = int(((~is_flagged) & (~is_poisoned_gt)).sum())
    precision = tp / max(1, tp + fp)
    recall = tp / max(1, tp + fn)
    f1 = 2 * precision * recall / max(1e-9, (precision + recall))
    return {
        "tp": tp, "fp": fp, "fn": fn, "tn": tn,
        "precision": precision, "recall": recall, "f1": f1,
    }


def _summarize_report(report: Any) -> dict[str, Any]:
    # ART defences return rich reports; here, only the per-class summary.
    try:
        return {"size_per_cluster": report.get("size_per_cluster"), "silhouette": report.get("silhouette_score")}
    except AttributeError:
        return {}
```

### 2.5 Entry point — `run_panel.py`

```python
"""Wire the attacks and defences together.

Per attack variant:
  1. Produce poisoned train set (and trigger_eval set, for backdoor).
  2. Train an auxiliary classifier on the poisoned set.
  3. Evaluate clean accuracy on the clean test split and — for backdoor —
     the attack success rate on the trigger_eval split.
  4. Run the applicable defences; produce detection metrics vs the known
     ground-truth poisoning mask.
  5. Write per-attack JSON into artifacts/.
"""
import argparse, json, logging
from pathlib import Path

import yaml

from panel import defend, io, metrics, poison, report, train


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--config", required=True)
    args = parser.parse_args()
    cfg = yaml.safe_load(Path(args.config).read_text())

    logging.basicConfig(level=cfg["run"]["log_level"])
    log = logging.getLogger("panel")
    rng = io.rng_from_seed(cfg["run"]["seed"])

    out = Path(cfg["run"]["output_dir"])
    out.mkdir(parents=True, exist_ok=True)

    target = io.load_target(cfg["target"])
    baseline_model = train.train(target, name="baseline", seed=cfg["run"]["seed"])
    baseline_acc = metrics.accuracy(baseline_model, target.x_test, target.y_test)

    all_results = {"baseline_accuracy": baseline_acc, "attacks": {}}

    for attack_cfg in cfg["attacks"]:
        aid = attack_cfg["id"]
        log.info("attack: %s", aid)
        x_poisoned, y_poisoned, is_poisoned = poison.produce(target, attack_cfg, rng, baseline_model)

        poisoned_model = train.train_on(target, x_poisoned, y_poisoned, name=aid, seed=cfg["run"]["seed"])
        clean_acc = metrics.accuracy(poisoned_model, target.x_test, target.y_test)
        extra = {}
        if attack_cfg["kind"] == "badnets":
            extra["backdoor_asr"] = metrics.backdoor_asr(poisoned_model, target, attack_cfg)
        if attack_cfg["kind"] == "clean_label":
            extra["targeted_flip_rate"] = metrics.targeted_flip(poisoned_model, target, attack_cfg)

        applicable = [d for d in cfg["defences"] if aid in d["applies_to"]]
        defences_out = {}
        for d in applicable:
            if d["kind"] == "spectral":
                defences_out[d["id"]] = defend.spectral_signature_scan(poisoned_model, x_poisoned, y_poisoned, is_poisoned, d)
            elif d["kind"] == "activation_clustering":
                defences_out[d["id"]] = defend.activation_clustering_scan(poisoned_model, x_poisoned, y_poisoned, is_poisoned, d)
            elif d["kind"] == "neural_cleanse_lite":
                defences_out[d["id"]] = defend.trigger_reconstruction_scan(poisoned_model, x_poisoned, y_poisoned, cfg["target"]["classes"], d)

        all_results["attacks"][aid] = {
            "attack_cfg": attack_cfg,
            "clean_accuracy_after_poisoning": clean_acc,
            "clean_accuracy_delta_vs_baseline": clean_acc - baseline_acc,
            "extra": extra,
            "defences": defences_out,
        }
        (out / f"{aid}.json").write_text(json.dumps(all_results["attacks"][aid], indent=2, default=float))

    (out / "aggregated.json").write_text(json.dumps(all_results, indent=2, default=float))
    report.write(cfg, all_results, Path(cfg["report"]["path"]))


if __name__ == "__main__":
    main()
```

### 2.6 Report template — `REPORT.md` (excerpt)

```markdown
# CarePilot Data-Poisoning Panel — Run {run.id}

**Threat-model source:** `mod-105-ex01-cross-tag.md` rows P1, P2, P3.
**Scope:** mod-106 auxiliary triage classifier only. Base LLM is frozen (vendor, scope-ruled to mod-110). RAG corpus poisoning is cross-referenced to mod-104 / mod-105 ex-01 E4 and is out of scope for this training-time panel.

## 1. Baseline

Clean accuracy on the held-out test split: **{baseline_accuracy:.3f}**.

## 2. Attack effectiveness

| Attack | Fraction | Clean-acc after | Δ vs baseline | Backdoor ASR | Targeted flip rate |
|---|---|---|---|---|---|
| label-flip-availability | {label_flip.fraction} | {label_flip.clean_acc:.3f} | {label_flip.delta:+.3f} | n/a | n/a |
| clean-label-targeted | {clean_label.fraction} | {clean_label.clean_acc:.3f} | {clean_label.delta:+.3f} | n/a | {clean_label.targeted_flip_rate:.3f} |
| badnets-backdoor | {badnets.fraction} | {badnets.clean_acc:.3f} | {badnets.delta:+.3f} | {badnets.backdoor_asr:.3f} | n/a |

**Interpretation.** The availability attack moves clean accuracy; it does not stay stealthy. The clean-label attack keeps clean accuracy ≈ baseline but flips the specific targeted sample with the configured rate. BadNets keeps clean accuracy ≈ baseline on clean inputs but produces high ASR on triggered inputs — the signature of a trojan.

## 3. Defence effectiveness

Detection metrics are computed with the ground-truth poisoning mask known to the panel (a development-time measurement — not available at runtime).

| Attack | Defence | Precision | Recall | F1 |
|---|---|---|---|---|
| label-flip-availability | spectral-signatures | {…} | {…} | {…} |
| clean-label-targeted | spectral-signatures | {…} | {…} | {…} |
| clean-label-targeted | activation-clustering | {…} | {…} | {…} |
| badnets-backdoor | spectral-signatures | {…} | {…} | {…} |
| badnets-backdoor | activation-clustering | {…} | {…} | {…} |
| badnets-backdoor | trigger-reconstruction (flagged class) | — | — | {flagged_class_or_none} |

**Caveat.** Precision/recall are only interpretable relative to the attack strength. A defence that catches 90% of a 1%-poisoned training set is more useful than one that catches 90% of a 20%-poisoned set (which would already be detectable by the clean-accuracy drop alone).

## 4. Residual-risk roll-up (ex-05 input)

For every attack row where (a) the defence's recall is below the register-envelope floor, or (b) clean-accuracy delta is positive (attack stealthy) *and* no defence reaches the floor, open a residual-risk register row for ex-05. Envelope floors are cited from `risk-register/.../data-poisoning-envelope.yaml`.

| Register row | Envelope floor | Observed | Residual? |
|---|---|---|---|
| data-poisoning-availability (detection recall) | `to-confirm-at-register-review` | {label_flip.spectral.recall:.3f} | {…} |
| data-poisoning-targeted (detection recall) | `to-confirm-at-register-review` | max(spectral, activation) | {…} |
| backdoor-trojan (trigger-reconstruction fires on target class) | per-run | {badnets.trigger_reconstruction.flagged} | {…} |

<!-- submission-todo: envelope floor numbers come from CarePilot register quarterly review. -->
```

## 3. Validation steps

A reviewer should be able to confirm the submission in under 45 minutes on a laptop:

1. **Repro sanity.** `pip install -r requirements.txt` installs ART, PyTorch, scikit-learn. `python run_panel.py --config config.yaml` completes end-to-end on the test data size specified in config (keep it modest — poisoning + retraining + defence is the expensive loop).
2. **Determinism.** Rerun with the same seed yields the same `aggregated.json` modulo floating-point tolerance. The panel seeds numpy, torch, and the attack RNGs.
3. **Scope ruling present.** README and REPORT both explicitly scope-rule the base LLM out of this panel and point to mod-110 for supply-chain and mod-104 for RAG-corpus poisoning.
4. **Attacks are canonical ART attacks.** `label_flip` is a hand-implemented primitive; `clean_label` uses `FeatureCollisionAttack`; `badnets` uses `PoisoningAttackBackdoor`. No invented attack semantics.
5. **Defences are canonical ART defences + a cited Neural-Cleanse-style scan.** `SpectralSignatureDefense`, `ActivationDefence`, and a trigger-reconstruction defence with a MAD-based anomaly index (Wang et al. 2019).
6. **Attack vs defence pairing is explicit.** The config `applies_to` field pairs each defence to the attack variants it is designed to catch. Backdoor-specific defences are not applied to label-flip, which they are not designed to detect.
7. **Metrics are correct.** Clean accuracy, backdoor ASR, targeted flip rate, and detection P/R are all computed with the right denominators. The report labels detection metrics as development-time.
8. **Register pointers present.** Every table row in the report that reports a measurement also cites the register envelope it is being compared against.

## 4. Rubric / review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Scope ruling explicit | README/REPORT scope-rule base LLM + RAG corpus out; cross-ref to mod-110/mod-104 | Scope ambiguous or missing |
| R2 | Three attack variants present | label-flip, clean-label, badnets backdoor | Fewer than three or renamed variants |
| R3 | Attacks are canonical | Use ART `FeatureCollisionAttack` + `PoisoningAttackBackdoor`; label-flip documented | Hand-rolled attacks without citation |
| R4 | Three defence baselines present | spectral, activation clustering, trigger reconstruction | Fewer than three or non-canonical choices |
| R5 | Threshold/budget in config | Poison fraction, suspicion threshold, MAD anomaly threshold all in config | Any hard-coded in Python |
| R6 | Clean-accuracy + attack-specific metrics all reported | Table has clean-acc, Δ, backdoor ASR, targeted flip rate | Only one metric reported per attack |
| R7 | Defence metrics as dev-time | Report labels precision/recall as computed vs known mask; not framed as runtime | Framed as runtime detection |
| R8 | Register linkage | Every report row cites the register-envelope row it compares against | No linkage |
| R9 | Reproducible | Deterministic seed; pinned deps; CPU-only default | Non-deterministic |
| R10 | No invented numbers | No fabricated ASR/precision/recall/envelope floors | Invented numbers |

**Scoring bands.**

- *Pass* — R1–R10 green.
- *Pass with revisions* — one of R5/R7/R8 fails; revise and resubmit.
- *Fail* — R1, R3, R4, or R10 fails.

## 5. Common mistakes

- **Collapsing availability, targeted, and backdoor into one attack.** They are three different threat models with three different success metrics. Separate them.
- **Applying backdoor-specific defences to availability attacks.** Spectral signatures and activation clustering are designed to catch targeted/backdoor poisoning. Running them against a 20% label-flip attack produces misleading metrics.
- **Reporting detection AUC as a runtime signal.** Detection metrics in the panel use a *known* poisoning mask. At runtime, that mask does not exist. The panel's measurement is development-time; the report must say so.
- **Not scoping-ruling the base LLM.** The CarePilot base LLM is vendor-supplied and frozen. Poisoning it is a supply-chain concern (mod-110), not a training-time panel concern. Mixing scopes dilutes the finding.
- **Backdoor trigger that is detectable by a one-column rule.** Setting a feature to a value that is already an outlier means the backdoor is caught by a feature-range check, not by the defence. The trigger should be *inside* the feature's plausible range, or clearly justified if outside.
- **Clean-label attack with a huge poison fraction.** The whole point of clean-label is low fraction, high stealth. A 50% clean-label attack is not clean-label — it is a targeted-poisoning attack that gave up on stealth.
- **Running `SpectralSignatureDefense` on a backdoor attack with fewer than ~10 clean-correct samples per class.** Spectral signatures need enough per-class activations to compute a meaningful singular direction. If the eval sample is too small, the defence returns noise.
- **Trigger-reconstruction without the anomaly index.** Just reconstructing per-class triggers and looking at norms visually is not a defence. Wang et al. 2019's contribution is the MAD-based anomaly index that calibrates "small enough to be suspicious".

## 6. References

- **NIST AI 100-2 E2023 — Adversarial Machine Learning: A Taxonomy and Terminology of Attacks and Mitigations.** `https://csrc.nist.gov/publications/detail/nistir/ai-100-2` — training-time integrity attacks (data poisoning, backdoor/trojan, supply chain).
- **MITRE ATLAS.** `https://atlas.mitre.org/` — AML.T0020 Poison Training Data, AML.T0018 Backdoor ML Model, AML.T0031 Erode ML Model Integrity, AML.T0010 ML Supply Chain Compromise.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — ML02 Data Poisoning Attack, ML10 Model Poisoning, ML08 Model Skewing, ML06 AI Supply Chain Attacks.
- **Adversarial Robustness Toolbox (ART).** `https://adversarial-robustness-toolbox.readthedocs.io/` — `art.attacks.poisoning.FeatureCollisionAttack`, `art.attacks.poisoning.PoisoningAttackBackdoor`, `art.defences.detector.poison.SpectralSignatureDefense`, `art.defences.detector.poison.ActivationDefence`.
- **Canonical poisoning / defence papers (cited only by name here; the panel uses ART's canonical implementations):**
  - Tran, B.; Li, J.; Madry, A. "Spectral Signatures in Backdoor Attacks" (2018) — basis for `SpectralSignatureDefense`.
  - Chen, B. et al. "Detecting Backdoor Attacks on Deep Neural Networks by Activation Clustering" (2018) — basis for `ActivationDefence`.
  - Gu, T.; Dolan-Gavitt, B.; Garg, S. "BadNets: Identifying Vulnerabilities in the Machine Learning Model Supply Chain" (2017) — basis for `PoisoningAttackBackdoor`.
  - Shafahi, A. et al. "Poison Frogs! Targeted Clean-Label Poisoning Attacks on Neural Networks" (2018) — basis for `FeatureCollisionAttack`.
  - Wang, B. et al. "Neural Cleanse: Identifying and Mitigating Backdoor Attacks in Neural Networks" (2019) — basis for the trigger-reconstruction defence and the MAD anomaly index.
- **mod-105 ex-01 solution** — rows P1, P2, P3 (training-time attack classes against the CarePilot retrained head).
- **Exercise prompt** — `lessons/mod-105-adversarial-ml-and-robustness/exercises/exercise-03-data-poisoning-and-detection-baselines.md`.

<!-- submission-todo: exact ART attack/defence signatures in the pinned version (ART APIs have evolved — confirm `FeatureCollisionAttack`, `PoisoningAttackBackdoor`, `SpectralSignatureDefense`, `ActivationDefence` constructor/parameters against the pinned release); CarePilot register-envelope floors for data-poisoning-availability, data-poisoning-targeted, and backdoor-trojan. -->

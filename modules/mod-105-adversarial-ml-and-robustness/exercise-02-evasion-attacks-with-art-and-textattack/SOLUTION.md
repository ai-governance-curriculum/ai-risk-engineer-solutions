# Exercise 02 — Evasion Attack Panel with ART and TextAttack — Reference Solution

## 1. Solution overview

This reference walks a worked `mod-105-ex02-evasion-panel/` deliverable for **CarePilot** — a panel that runs a reproducible set of evasion attacks against (a) the mod-106 auxiliary triage classifier (PredAI, tabular + optional image side-channel) and (b) an NLP sentiment-style classifier that sits on an optional triage-text intake path. The panel is a *test rig*, not a defence; defences land in ex-04 (shared API-surface defences) and in the mod-108 guardrail bench.

The deliverable has three pieces:

1. A small Python package (`panel/`) that wires ART's attack APIs against a PyTorch classifier and TextAttack's attack recipes against a HuggingFace text classifier, with a single `run_panel.py` entry point.
2. A `config.yaml` that lists the attacks to run (FGSM, PGD, HopSkipJump, Square for ART; TextFooler, PWWS, DeepWordBug, BAE for TextAttack), per-attack budgets (ε, step count, query cap), a deterministic seed, and the output directory layout.
3. A `REPORT.md` template that the panel populates with the attack-success-rate (ASR), mean perturbation size, and query count per attack, plus a deltas table versus the clean baseline. The report feeds ex-05.

Design decisions:

- **Pin everything.** Model files, dataset splits, library versions (ART, TextAttack, numpy, torch, transformers), and a seed. Evasion results are notorious for drifting between library revisions; without pinning the panel is not comparable across runs.
- **Attack budgets live in config, not code.** The threat-model axis (ε, query cap, perturbation type) is editable without a code change, and the chosen values trace back to the mod-105 ex-01 cross-tag row and the mod-103 register entry for `adv-evasion-predai` / `adv-evasion-nlp`.
- **Report "success under the stated threat model", not "model is broken".** ASR of 97% at ε = 0.3 on normalized tabular features is not the same statement as "the model fails on realistic inputs". The report template forces the author to quote the budget alongside the number.
- **Use published attacks from the two toolkits only.** No hand-rolled attack. The IBM Adversarial Robustness Toolbox and TextAttack both publish canonical implementations of the attacks we want; using those keeps the result reproducible and auditable. VeriSwarm reference kernels can be cited as practitioner implementations if they add engineering structure the toolkits do not, but the attack semantics must come from ART or TextAttack.
- **Non-trivial threat-model notes.** Each attack config carries an `adversary_knowledge` field (white-box / black-box / grey-box) and a `plausibility_under_carepilot` note that ties back to the ex-01 cross-tag — e.g., gradient-based FGSM is only plausible if the adversary has model internals, which for CarePilot's internal classifier they would not; that is why black-box transfer and query-based attacks (HopSkipJump, Square) are the realistic rows.

The implementation is structured so a reviewer can `pip install -r requirements.txt && python run_panel.py --config config.yaml` and reproduce the numbers. Where a dependency cannot be pinned exactly without access to the live PyPI index, the `requirements.txt` is marked with `<!-- submission-todo: ... -->` pins.

## 2. Worked implementation — `mod-105-ex02-evasion-panel/`

### 2.1 Directory layout

```
mod-105-ex02-evasion-panel/
├── README.md                # how to run; threat-model summary; cites ex-01 cross-tag
├── config.yaml              # pinned attack list, budgets, seed, model/data refs
├── requirements.txt         # pinned deps (torch, art, textattack, transformers, datasets, numpy, pandas)
├── panel/
│   ├── __init__.py
│   ├── io.py                # load model + data; write artifacts
│   ├── predai.py            # ART-side attacks (tabular + optional image)
│   ├── nlp.py               # TextAttack-side attacks
│   ├── metrics.py           # ASR, mean perturbation, query count, L∞/L2/word-edit distance
│   └── report.py            # write REPORT.md from a run's artifacts
├── run_panel.py             # entry point
├── artifacts/               # output per run: {run_id}/*.json, *.csv, adversarial-examples/*.pt
└── REPORT.md                # populated at run time
```

### 2.2 Config — `config.yaml`

```yaml
# mod-105 ex-02 — evasion panel configuration
# References: mod-105/exercise-01 cross-tag rows E1, E2, E5.
# Threat model and budgets are set at the risk-register level; this config cites
# the register row and copies the budget inline for reproducibility.

run:
  id: carepilot-evasion-panel-0001
  seed: 20260101           # deterministic
  output_dir: artifacts/${run.id}
  log_level: INFO

targets:
  predai:
    kind: tabular               # CarePilot mod-106 auxiliary triage classifier
    framework: pytorch
    weights: models/predai/triage-classifier.pt
    data:
      splits: data/predai/splits.yaml
      eval_split: test            # held-out, never seen during training
      sample_size: 1000           # fixed to keep ASR comparable across runs
    preprocessing:
      scaler: models/predai/scaler.joblib
    classes: [routine, urgent, immediate]
  nlp:
    kind: text_classifier
    framework: huggingface
    model_id: carepilot/triage-intake-classifier  # locally finetuned, not the base LLM
    data:
      jsonl: data/nlp/triage-intake-eval.jsonl
      sample_size: 500
    classes: [low-acuity, high-acuity]

attacks:

  # ART — PredAI
  - id: art-fgsm
    toolkit: art
    recipe: FastGradientMethod
    target: predai
    threat_model:
      adversary_knowledge: white_box
      perturbation_norm: linf
      epsilon: 0.1              # on standardised features (mean 0, sd 1)
      targeted: false
    plausibility_under_carepilot: >
      Low plausibility as a realistic attack (adversary needs model internals).
      Kept as a *reference ceiling* for ASR under full adversary knowledge.

  - id: art-pgd-linf
    toolkit: art
    recipe: ProjectedGradientDescent
    target: predai
    threat_model:
      adversary_knowledge: white_box
      perturbation_norm: linf
      epsilon: 0.1
      step_size: 0.01
      num_iterations: 40
      random_start: true
      targeted: false
    plausibility_under_carepilot: >
      Low plausibility as a realistic attack; same reasoning as FGSM.
      Reports the stronger white-box ceiling.

  - id: art-hop-skip-jump
    toolkit: art
    recipe: HopSkipJump
    target: predai
    threat_model:
      adversary_knowledge: black_box          # decision-based (labels only)
      perturbation_norm: l2
      max_iter: 50
      max_eval: 1000
      init_eval: 100
      targeted: false
    plausibility_under_carepilot: >
      Realistic worst-case for an adversary with internal-network access to the
      triage endpoint. Primary row for the register entry adv-evasion-predai.

  - id: art-square
    toolkit: art
    recipe: SquareAttack
    target: predai
    threat_model:
      adversary_knowledge: black_box          # score-based (probabilities)
      perturbation_norm: linf
      epsilon: 0.1
      max_iter: 5000
      p_init: 0.05
    plausibility_under_carepilot: >
      Realistic if the endpoint returns probabilities. CarePilot should emit
      calibrated top-1 + decision only (ex-04 defence row); this attack row
      drives that defence decision.

  # TextAttack — NLP
  - id: ta-textfooler
    toolkit: textattack
    recipe: TextFoolerJin2019
    target: nlp
    threat_model:
      adversary_knowledge: black_box          # score-based (probabilities)
      constraint_profile: default              # TextAttack default: USE embedding sim, PoS, stop-word
      query_cap: 500
    plausibility_under_carepilot: >
      Realistic if the NLP classifier is on the critical path for a triage-text
      intake form. OUT by default (CarePilot's primary surface is the RAG LLM),
      but IN whenever the intake classifier is enabled.

  - id: ta-pwws
    toolkit: textattack
    recipe: PWWSRen2019
    target: nlp
    threat_model:
      adversary_knowledge: black_box
      query_cap: 500

  - id: ta-deepwordbug
    toolkit: textattack
    recipe: DeepWordBugGao2018
    target: nlp
    threat_model:
      adversary_knowledge: black_box
      query_cap: 500

  - id: ta-bae
    toolkit: textattack
    recipe: BAEGarg2019
    target: nlp
    threat_model:
      adversary_knowledge: black_box
      query_cap: 500

report:
  path: REPORT.md
  compare_to: baselines/carepilot-clean-eval.json   # same sample, no attack
```

### 2.3 Entry point — `run_panel.py`

```python
"""mod-105 ex-02 — run the evasion panel.

Usage:
    python run_panel.py --config config.yaml

Reads attack list from config.yaml, runs each attack through the appropriate
toolkit (ART for PredAI, TextAttack for NLP), writes per-attack artefacts into
artifacts/{run.id}/, and populates REPORT.md.
"""
import argparse
import json
import logging
from pathlib import Path

import yaml

from panel import io, metrics, nlp, predai, report


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--config", required=True)
    args = parser.parse_args()
    cfg = yaml.safe_load(Path(args.config).read_text())

    logging.basicConfig(level=cfg["run"]["log_level"])
    log = logging.getLogger("panel")
    io.set_global_seed(cfg["run"]["seed"])

    out_dir = Path(cfg["run"]["output_dir"])
    out_dir.mkdir(parents=True, exist_ok=True)

    predai_target = io.load_predai(cfg["targets"]["predai"]) if any(
        a["target"] == "predai" for a in cfg["attacks"]
    ) else None
    nlp_target = io.load_nlp(cfg["targets"]["nlp"]) if any(
        a["target"] == "nlp" for a in cfg["attacks"]
    ) else None

    results = {}
    for attack_cfg in cfg["attacks"]:
        aid = attack_cfg["id"]
        log.info("running %s", aid)
        if attack_cfg["toolkit"] == "art":
            results[aid] = predai.run_attack(predai_target, attack_cfg, out_dir)
        elif attack_cfg["toolkit"] == "textattack":
            results[aid] = nlp.run_attack(nlp_target, attack_cfg, out_dir)
        else:
            raise ValueError(f"unknown toolkit: {attack_cfg['toolkit']}")
        (out_dir / f"{aid}.json").write_text(json.dumps(results[aid], indent=2))

    aggregated = metrics.aggregate(results)
    (out_dir / "aggregated.json").write_text(json.dumps(aggregated, indent=2))
    report.write(
        template_path=Path("REPORT.md"),
        config=cfg,
        results=results,
        aggregated=aggregated,
        baseline_path=Path(cfg["report"]["compare_to"]),
        out_path=Path(cfg["report"]["path"]),
    )
    log.info("done: %s", cfg["report"]["path"])


if __name__ == "__main__":
    main()
```

### 2.4 ART side — `panel/predai.py` (excerpt)

```python
"""ART-side evasion attacks against the CarePilot PredAI classifier."""
from __future__ import annotations

from dataclasses import dataclass
from typing import Any

import numpy as np
import torch
from art.attacks.evasion import (
    FastGradientMethod,
    ProjectedGradientDescent,
    HopSkipJump,
    SquareAttack,
)
from art.estimators.classification import PyTorchClassifier


@dataclass
class PredaiTarget:
    model: torch.nn.Module
    x_eval: np.ndarray
    y_eval: np.ndarray
    n_classes: int
    input_shape: tuple[int, ...]
    feature_min: float = -5.0
    feature_max: float = 5.0


def _wrap(target: PredaiTarget) -> PyTorchClassifier:
    return PyTorchClassifier(
        model=target.model,
        loss=torch.nn.CrossEntropyLoss(),
        input_shape=target.input_shape,
        nb_classes=target.n_classes,
        clip_values=(target.feature_min, target.feature_max),
        device_type="cpu",  # deterministic; GPU reproducibility is a separate thread
    )


def run_attack(target: PredaiTarget, cfg: dict[str, Any], out_dir) -> dict[str, Any]:
    classifier = _wrap(target)
    recipe = cfg["recipe"]
    tm = cfg["threat_model"]

    if recipe == "FastGradientMethod":
        attack = FastGradientMethod(classifier, eps=tm["epsilon"], norm=np.inf, targeted=tm["targeted"])
    elif recipe == "ProjectedGradientDescent":
        attack = ProjectedGradientDescent(
            classifier,
            eps=tm["epsilon"],
            eps_step=tm["step_size"],
            max_iter=tm["num_iterations"],
            norm=np.inf,
            random_eps=tm["random_start"],
            targeted=tm["targeted"],
        )
    elif recipe == "HopSkipJump":
        attack = HopSkipJump(
            classifier,
            norm=2,
            max_iter=tm["max_iter"],
            max_eval=tm["max_eval"],
            init_eval=tm["init_eval"],
            targeted=tm["targeted"],
        )
    elif recipe == "SquareAttack":
        attack = SquareAttack(
            classifier,
            norm=np.inf,
            eps=tm["epsilon"],
            max_iter=tm["max_iter"],
            p_init=tm["p_init"],
        )
    else:
        raise ValueError(f"unknown ART recipe: {recipe}")

    x_adv = attack.generate(x=target.x_eval)
    y_clean = classifier.predict(target.x_eval).argmax(axis=1)
    y_adv = classifier.predict(x_adv).argmax(axis=1)

    # Only count a success when (a) the clean prediction was correct and
    # (b) the adversarial prediction is wrong — a common-sense ASR.
    clean_correct = y_clean == target.y_eval
    successful = clean_correct & (y_adv != target.y_eval)
    asr = float(successful.sum()) / max(1, int(clean_correct.sum()))

    perturbation = x_adv - target.x_eval
    linf = float(np.max(np.abs(perturbation), axis=tuple(range(1, perturbation.ndim))).mean())
    l2 = float(np.linalg.norm(perturbation.reshape(perturbation.shape[0], -1), axis=1).mean())

    return {
        "attack_id": cfg["id"],
        "toolkit": "art",
        "recipe": recipe,
        "threat_model": tm,
        "sample_size": int(target.x_eval.shape[0]),
        "clean_accuracy": float(clean_correct.mean()),
        "asr": asr,
        "mean_linf": linf,
        "mean_l2": l2,
    }
```

### 2.5 TextAttack side — `panel/nlp.py` (excerpt)

```python
"""TextAttack-side evasion against the CarePilot NLP intake classifier."""
from __future__ import annotations
from typing import Any

import textattack
from textattack.attack_recipes import (
    TextFoolerJin2019,
    PWWSRen2019,
    DeepWordBugGao2018,
    BAEGarg2019,
)
from textattack.models.wrappers import HuggingFaceModelWrapper


_RECIPES = {
    "TextFoolerJin2019": TextFoolerJin2019,
    "PWWSRen2019": PWWSRen2019,
    "DeepWordBugGao2018": DeepWordBugGao2018,
    "BAEGarg2019": BAEGarg2019,
}


def run_attack(target, cfg: dict[str, Any], out_dir) -> dict[str, Any]:
    wrapper = HuggingFaceModelWrapper(target.model, target.tokenizer)
    recipe_cls = _RECIPES[cfg["recipe"]]
    attack = recipe_cls.build(wrapper)

    # Per-attack query cap: TextAttack uses a GoalFunction with a `query_budget`.
    if "query_cap" in cfg["threat_model"]:
        attack.goal_function.query_budget = cfg["threat_model"]["query_cap"]

    dataset = textattack.datasets.Dataset(target.examples)
    args = textattack.AttackArgs(
        num_examples=len(target.examples),
        disable_stdout=True,
        shuffle=False,
    )
    attacker = textattack.Attacker(attack, dataset, args)
    results = list(attacker.attack_dataset())

    successes = sum(
        1 for r in results if isinstance(r, textattack.attack_results.SuccessfulAttackResult)
    )
    skipped = sum(
        1 for r in results if isinstance(r, textattack.attack_results.SkippedAttackResult)
    )
    asr = successes / max(1, len(results) - skipped)

    perturbed_pct = [
        r.perturbed_result.attacked_text.words_diff_num(r.original_result.attacked_text)
        / max(1, len(r.original_result.attacked_text.words))
        for r in results
        if isinstance(r, textattack.attack_results.SuccessfulAttackResult)
    ]
    mean_word_edits = float(sum(perturbed_pct) / max(1, len(perturbed_pct)))

    return {
        "attack_id": cfg["id"],
        "toolkit": "textattack",
        "recipe": cfg["recipe"],
        "threat_model": cfg["threat_model"],
        "sample_size": len(results),
        "asr": asr,
        "mean_word_edit_fraction": mean_word_edits,
    }
```

### 2.6 Report template — `REPORT.md`

```markdown
# CarePilot Evasion Panel — Run {run.id}

**Date:** {generated_at}

**Threat-model source:** `mod-105-ex01-cross-tag.md` rows E1, E2, E5, plus CarePilot register entries `adv-evasion-predai` and `adv-evasion-nlp`.

**Toolkits and versions:**

- Adversarial Robustness Toolbox {art_version}
- TextAttack {textattack_version}
- PyTorch {torch_version}
- Transformers {transformers_version}

## 1. PredAI classifier — summary

Clean baseline accuracy on the held-out eval sample: **{predai_clean_accuracy:.3f}** (n = {predai_sample_size}).

| Attack | Adversary knowledge | Budget | ASR | Mean L∞ | Mean L2 |
|---|---|---|---|---|---|
| art-fgsm | white-box | ε = {art_fgsm.epsilon:.2f} | {art_fgsm.asr:.3f} | {art_fgsm.mean_linf:.3f} | {art_fgsm.mean_l2:.3f} |
| art-pgd-linf | white-box | ε = {art_pgd.epsilon:.2f}, iter = {art_pgd.num_iterations} | {art_pgd.asr:.3f} | {art_pgd.mean_linf:.3f} | {art_pgd.mean_l2:.3f} |
| art-hop-skip-jump | black-box (decision) | max_eval = {art_hsj.max_eval} | {art_hsj.asr:.3f} | — | {art_hsj.mean_l2:.3f} |
| art-square | black-box (score) | ε = {art_sq.epsilon:.2f}, iter = {art_sq.max_iter} | {art_sq.asr:.3f} | {art_sq.mean_linf:.3f} | — |

**Interpretation.** White-box ASR is the ceiling — this is what an adversary with model internals could achieve under this budget. Black-box ASR (HopSkipJump, Square) is the realistic row for the CarePilot threat model. Compare both against the register envelope in `risk-register/.../adv-evasion-predai#envelope`.

## 2. NLP classifier — summary

Clean baseline accuracy: **{nlp_clean_accuracy:.3f}** (n = {nlp_sample_size}).

| Attack | Adversary knowledge | Budget | ASR | Mean word-edit fraction |
|---|---|---|---|---|
| ta-textfooler | black-box (score) | query_cap = {ta_tf.query_cap} | {ta_tf.asr:.3f} | {ta_tf.mean_word_edit_fraction:.3f} |
| ta-pwws | black-box (score) | query_cap = {ta_pw.query_cap} | {ta_pw.asr:.3f} | {ta_pw.mean_word_edit_fraction:.3f} |
| ta-deepwordbug | black-box (score) | query_cap = {ta_dw.query_cap} | {ta_dw.asr:.3f} | {ta_dw.mean_word_edit_fraction:.3f} |
| ta-bae | black-box (score) | query_cap = {ta_bae.query_cap} | {ta_bae.asr:.3f} | {ta_bae.mean_word_edit_fraction:.3f} |

**Interpretation.** NLP attacks typically succeed with few word edits under the default constraints; the "mean word-edit fraction" column tells a reviewer how perceptible the perturbation is. If a human reviewer cannot distinguish the adversarial text from the clean text, the risk is higher.

## 3. Register deltas

For each attack row above whose ASR exceeds the register-envelope floor in `risk-register/.../adv-evasion-predai#envelope` or `adv-evasion-nlp#envelope`, open an OPEN item in the register and file against mod-105 ex-05.

| Register row | Envelope floor | Observed ASR | Delta | Action |
|---|---|---|---|---|
| adv-evasion-predai (black-box) | `to-confirm-at-register-review` | {art_hsj.asr:.3f} / {art_sq.asr:.3f} | +/-… | — |
| adv-evasion-nlp (black-box) | `to-confirm-at-register-review` | max ASR across TA recipes | +/-… | — |

<!-- submission-todo: per-row envelope floors are set at the CarePilot risk-register quarterly review; this report cites the envelope but does not set it. -->
```

## 3. Validation steps

A reviewer should be able to confirm the submission in under 30 minutes with the panel checked out.

1. **Repro sanity.** `pip install -r requirements.txt` succeeds against the pinned versions. `python run_panel.py --config config.yaml` completes end-to-end on the eval sample size (sample sizes are intentionally small — 1000 tabular, 500 text — for deterministic repro in minutes, not days).
2. **Determinism.** Running the panel twice with the same seed produces byte-identical `aggregated.json`. (Floating-point nondeterminism on GPU is documented; CPU-only is the reproducible default.)
3. **Attack coverage.** Config includes at least one white-box and one black-box ART attack and ≥3 TextAttack recipes. Each attack config carries a `threat_model` block *and* a `plausibility_under_carepilot` note.
4. **Threat-model cross-ref.** Each attack config cites the ex-01 cross-tag row it is exercising (E1, E2, or E5) and the register entry it rolls up into.
5. **Metrics defined.** ASR is defined over clean-correct samples only (adversarial misclassification on an already-wrong sample is not counted). Perturbation sizes are reported in L∞ *and* L2 for ART attacks where the norm applies, and word-edit fraction for TextAttack.
6. **Report populated.** `REPORT.md` is populated from `aggregated.json` and the ex-01 cross-tag row IDs; placeholder `to-confirm-at-register-review` envelopes are allowed (they block auto-merge and force the submitter to pull the register value).
7. **No invented attack semantics.** Every attack name in config maps to a canonical class in `art.attacks.evasion` or a canonical recipe in `textattack.attack_recipes`. If a row names a hand-rolled attack, reject.
8. **Dependency pinning.** `requirements.txt` has explicit version pins. A `submission-todo` marker is allowed for packages whose exact pinned version has not been verified at submission time.

## 4. Rubric / review checklist

Scoring is pass/fail per criterion; the submission is accepted if all criteria pass.

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Panel runs end-to-end | `python run_panel.py --config config.yaml` completes without error and produces `aggregated.json` + `REPORT.md` | Panel fails or requires manual intervention mid-run |
| R2 | Config pins everything | Seed, library versions, model/data refs all pinned | Any unpinned attack or data source |
| R3 | ART attacks are canonical | Each recipe maps to a class in `art.attacks.evasion` | Hand-rolled or renamed attack |
| R4 | TextAttack attacks are canonical | Each recipe maps to a class in `textattack.attack_recipes` | Hand-rolled or renamed recipe |
| R5 | Threat model per attack | `adversary_knowledge`, budget, and norm/constraint profile populated | Missing threat-model fields |
| R6 | ASR definition | ASR computed over clean-correct samples only | Counts already-wrong samples as successes |
| R7 | Report calls out white- vs black-box | Report separates white-box ceiling from black-box realistic row | Mixes the two without distinction |
| R8 | Register cross-ref present | Attack configs cite ex-01 cross-tag row and register entry | No linkage to upstream artifacts |
| R9 | No invented numbers in README/report | Only computed metrics are quoted; envelope floors are either cited or marked `submission-todo` | Any fabricated ASR, envelope, or benchmark number |
| R10 | Plausibility notes present | Each attack has a `plausibility_under_carepilot` string | Attack config with no CarePilot context |

**Scoring bands.**

- *Pass* — R1–R10 green.
- *Pass with revisions* — one of R2/R5/R10 fails but the panel runs and produces a report; revise and resubmit.
- *Fail* — R3/R4 (hand-rolled attacks) or R9 (invented numbers) fails.

## 5. Common mistakes

- **Reporting one ASR number without the budget.** ASR = 0.97 is meaningless without the ε, the norm, the step count (if iterative), and the adversary-knowledge assumption. Always pair.
- **Measuring ASR over the whole eval set, including already-misclassified samples.** This inflates ASR. Use clean-correct samples only.
- **Using FGSM as the only attack and calling the model "robust" because ASR is low.** FGSM is a single-step gradient method; it is the weakest common attack. Run PGD or AutoAttack-style iterated attacks for a defensible ceiling and use HopSkipJump/Square for a black-box realistic row.
- **Running TextAttack with its default constraint profile and not reporting that.** TextAttack recipes come with a set of semantic/syntactic constraints (USE embedding similarity, PoS match, stop-word policy) that materially change ASR. Always log the constraint profile.
- **Attacking the base vendor LLM with an evasion-against-classifier attack.** ART/TextAttack attacks target classifiers with a differentiable loss or a score output. They are not the right tool for prompt injection against a generative model — mod-104 owns that suite.
- **Treating sponge / latency attacks as evasion.** Sponge attacks are availability attacks (AML.T0029 Denial of ML Service); they belong in ex-04's shared API-surface defence row, not here.
- **No clean baseline.** The report compares attack results to a *clean baseline* run on the same sample; without it, deltas are not computable.
- **Flaky seeds / non-deterministic GPU.** If the panel uses GPU without `torch.use_deterministic_algorithms(True)` or without pinning cuDNN, reruns will not match. The reference config uses CPU for this reason.
- **Confusing "model is broken" with "the stated threat model succeeds here".** The submission is a measurement, not a verdict. The verdict is written in ex-05 against register envelopes.

## 6. References

- **NIST AI 100-2 E2023 — Adversarial Machine Learning: A Taxonomy and Terminology of Attacks and Mitigations.** `https://csrc.nist.gov/publications/detail/nistir/ai-100-2` — attack-class axes (attacker knowledge, timing, goal).
- **MITRE ATLAS.** `https://atlas.mitre.org/` — AML.T0040 (API inference), AML.T0043 (Craft Adversarial Data).
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — ML01 Input Manipulation Attack.
- **Adversarial Robustness Toolbox (ART).** Hosted by the Linux Foundation AI & Data; project home `https://adversarial-robustness-toolbox.readthedocs.io/` and source `https://github.com/Trusted-AI/adversarial-robustness-toolbox`. Attack classes referenced: `FastGradientMethod`, `ProjectedGradientDescent`, `HopSkipJump`, `SquareAttack`.
- **TextAttack.** UVA NLP group; project home `https://textattack.readthedocs.io/` and source `https://github.com/QData/TextAttack`. Attack recipes referenced: `TextFoolerJin2019`, `PWWSRen2019`, `DeepWordBugGao2018`, `BAEGarg2019`.
- **mod-105 ex-01 solution** — `modules/mod-105-adversarial-ml-and-robustness/exercise-01-nist-ai-100-2-to-atlas-to-owasp-cross-tag/SOLUTION.md` — cross-tag rows E1, E2, E5.
- **mod-101 ex-01 solution** — MEASURE 2.7 for CarePilot — the sub-category this panel files evidence under.
- **Exercise prompt** — `lessons/mod-105-adversarial-ml-and-robustness/exercises/exercise-02-evasion-attacks-with-art-and-textattack.md`.

<!-- submission-todo: exact ART and TextAttack version pins; CarePilot register-envelope floor values for adv-evasion-predai and adv-evasion-nlp. -->

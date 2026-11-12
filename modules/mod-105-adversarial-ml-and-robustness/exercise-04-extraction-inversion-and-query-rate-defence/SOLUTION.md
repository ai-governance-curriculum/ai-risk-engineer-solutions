# Exercise 04 — Extraction, Membership Inference, and the Shared API-Surface Defence Set — Reference Solution

## 1. Solution overview

This reference walks a worked `mod-105-ex04-extraction-mia-panel/` deliverable for **CarePilot**. The panel exercises three attack classes that share a common API-surface — model extraction, membership inference (MIA), and attribute inversion — and the shared defence set that pushes back on all three (query-rate limit, output restriction / confidence masking, differential-privacy training for the retrained head, and a PRADA-style extraction-detector). The attacks target CarePilot's *retrained* head (the mod-106 auxiliary triage classifier) and its *prompt scaffolding + retrieval corpus* (the parts CarePilot owns), per the mod-105 ex-01 cross-tag rows X1, X2, X3, X5 and A1.

The deliverable has two tracks:

1. **Attack panel** — a Python driver that runs: (a) a KnockoffNets-style extraction attack using ART's `KnockoffNets` (Orekondy et al., 2019) against the triage classifier; (b) a confidence-based membership-inference attack using ART's `MembershipInferenceBlackBox` with the loss-based or rule-based attack from Shokri et al. (2017) / Yeom et al. (2018); (c) an attribute-inversion probe using ART's `AttributeInferenceBlackBox`; (d) a system-prompt-leakage canary set against the RAG assistant (cross-referenced to mod-104's garak suite and documented here).
2. **Defence panel** — a Flask/FastAPI-style serving shim that layers: (i) query-rate limiting (per-user + per-source-IP, with sliding-window counters), (ii) output restriction (top-1 label and binned confidence only — no probability vector), (iii) a PRADA-style detector (Juuti et al., 2019) that flags clients whose query distribution looks extraction-like, and (iv) a hook for DP-SGD-trained variants of the triage classifier (training config references Opacus / TF-Privacy, with `(ε, δ)` targets set in the CarePilot register, not here).

Design decisions:

- **Shared defence set, separate attack panels.** The three attacks exploit the same API-surface — output probabilities, query freedom, training-set membership signals. The defences are therefore *shared*: rate-limit, output restriction, extraction-detector, and DP-SGD. The panel structure reflects this.
- **Scope is the retrained head + the scaffolding.** The base vendor LLM's extraction/MIA/inversion exposure is a vendor concern. CarePilot can still test leakage of its *own* system prompt and retrieval corpus via the mod-104 red-team suite; those attack rows are listed here but cross-referenced to mod-104.
- **Published attack implementations only.** Each attack maps to an ART class: `KnockoffNets`, `MembershipInferenceBlackBox`, `AttributeInferenceBlackBox`. These implement the canonical papers (Orekondy 2019, Shokri 2017, Yeom 2018, Fredrikson 2015) and preserve the semantics.
- **Published defences with measurable knobs.** Rate-limit has a sliding-window counter, PRADA-style detector has a KS-test threshold, DP-SGD has `(ε, δ)`. "Harden the API" without a knob is not a control.
- **Defence efficacy is a before/after measurement.** The report runs each attack *against the defended endpoint* and *against the undefended endpoint* and reports the delta. The absolute number is not the signal; the delta is.
- **DP-SGD training is referenced, not run from scratch.** A full DP training run is a separate cycle; the panel ships a config entry and a `train_dp_sgd.py` stub (using Opacus for PyTorch) and reports whichever `(ε, δ)` the retrained head was actually trained under. The report explicitly marks `submission-todo` where the register has not yet set `(ε, δ)` for CarePilot.

## 2. Worked implementation — `mod-105-ex04-extraction-mia-panel/`

### 2.1 Directory layout

```
mod-105-ex04-extraction-mia-panel/
├── README.md
├── config.yaml                  # attack list, defence stack, budgets
├── requirements.txt             # torch, art, opacus (optional), fastapi, uvicorn
├── panel/
│   ├── __init__.py
│   ├── io.py
│   ├── attacks.py               # extraction, MIA, attribute inversion
│   ├── defences/
│   │   ├── __init__.py
│   │   ├── rate_limit.py        # sliding-window per-user + per-IP
│   │   ├── output_restriction.py  # top-1 + binned confidence
│   │   ├── prada_detector.py    # query-distribution KS-test
│   │   └── dp_sgd.py            # training config + assertion that ε is met
│   ├── serving/
│   │   ├── app.py               # defended endpoint (FastAPI)
│   │   └── undefended_app.py    # baseline — plain endpoint for A/B
│   ├── metrics.py
│   └── report.py
├── run_panel.py
├── artifacts/{run_id}/
└── REPORT.md
```

### 2.2 Config — `config.yaml`

```yaml
# mod-105 ex-04 — extraction + MIA + inversion panel
# References: mod-105/exercise-01 cross-tag rows X1, X2, X3, X5, A1.
# Scope: CarePilot mod-106 auxiliary triage classifier (retrained head).
# System-prompt leakage (X5) is cross-referenced to mod-104 and is listed
# here only to record the attack surface and the rate-limit defence.

run:
  id: carepilot-extraction-panel-0001
  seed: 20260101
  output_dir: artifacts/${run.id}
  log_level: INFO

target:
  kind: tabular
  weights: models/predai/triage-classifier.pt
  weights_dp: models/predai/triage-classifier-dp.pt   # optional; if present, panel runs the DP variant too
  classes: [routine, urgent, immediate]
  data:
    train_split: data/predai/train.parquet
    test_split: data/predai/test.parquet
    member_split: data/predai/members-shadow.parquet   # explicitly-labelled member set for MIA ground truth
    nonmember_split: data/predai/nonmembers.parquet

serving:
  defended_url: http://localhost:8080
  undefended_url: http://localhost:8081

# -------- Attacks --------
attacks:
  - id: knockoff-extraction
    recipe: ART.KnockoffNets
    threat_model:
      adversary_knowledge: black_box
      query_budget: 5000
      sampling_strategy: adaptive        # KnockoffNets' adaptive sampling (reinforcement-learning-like reward)
      substitute_arch: mlp-small          # smaller surrogate the attacker trains
      substitute_epochs: 20
    success_metric: substitute_agreement_on_held_out  # fraction of held-out eval where substitute and victim agree

  - id: mia-black-box
    recipe: ART.MembershipInferenceBlackBox
    threat_model:
      adversary_knowledge: black_box
      attack_model_type: rf              # Shokri et al. / Yeom et al. style using an attack classifier
      input_type: loss                   # alternatives: 'prediction' (probability vector)
      shadow_models: 1                   # for simplicity, 1 shadow; multi-shadow is stronger but longer
    success_metric: attack_auc_on_member_vs_nonmember

  - id: attribute-inversion
    recipe: ART.AttributeInferenceBlackBox
    threat_model:
      adversary_knowledge: black_box
      attack_feature: age_bucket         # the sensitive attribute to try to reconstruct
      prior: uniform
    success_metric: attribute_accuracy_vs_prior_delta

  - id: system-prompt-canary
    recipe: cross-referenced          # not implemented here; mod-104 red-team suite owns this
    cross_ref: modules/mod-104-llm-red-teaming-and-attack-engineering/.../exercise-01-red-team-program-charter-for-one-target/
    threat_model:
      adversary_knowledge: black_box
      interface: RAG assistant chat endpoint
    success_metric: canary_leak_rate_from_mod-104

# -------- Defences --------
defences:
  rate_limit:
    window_seconds: 60
    requests_per_user_per_window: 30
    requests_per_ip_per_window: 60
    burst_grace: 5                      # first N requests in a window pass without a short-circuit
  output_restriction:
    emit: top1_binned                   # options: top1_binned | top1_only | full_probs (not used)
    confidence_bins: 5                  # discretise [0, 1] into 5 bins
  prada_detector:
    window_queries: 200
    ks_test_threshold: 0.3              # KS statistic between recent client queries and a reference distribution
    reference: baselines/prada-reference-distribution.npy
  dp_sgd:
    enabled: true
    epsilon_target: null                # must be set in CarePilot register before training
    delta_target: null
    noise_multiplier: null
    max_grad_norm: null
    training_script: train_dp_sgd.py

report:
  path: REPORT.md
  envelope_source: risk-register/.../extraction-mia-envelope.yaml
```

### 2.3 Attack side — `panel/attacks.py` (excerpt)

```python
"""mod-105 ex-04 — extraction + MIA + attribute inversion against the retrained head."""
from __future__ import annotations

import numpy as np
from art.attacks.extraction import KnockoffNets
from art.attacks.inference.membership_inference import MembershipInferenceBlackBox
from art.attacks.inference.attribute_inference import AttributeInferenceBlackBox
from art.estimators.classification import PyTorchClassifier


def run_extraction(victim: PyTorchClassifier, x_attack: np.ndarray, y_attack_dummy, substitute_cls, cfg: dict) -> dict:
    """Train a surrogate ('substitute') model from queries to the victim.
    Agreement between substitute and victim on held-out inputs is the fidelity metric."""
    tm = cfg["threat_model"]
    substitute = substitute_cls()
    attack = KnockoffNets(
        classifier=victim,
        batch_size_fit=64,
        batch_size_query=64,
        nb_epochs=tm["substitute_epochs"],
        nb_stolen=tm["query_budget"],
        sampling_strategy=tm["sampling_strategy"],
    )
    stolen = attack.extract(x_attack, thieved_classifier=substitute)
    return {"stolen_classifier": stolen}


def run_mia(victim: PyTorchClassifier, x_member, y_member, x_nonmember, y_nonmember, cfg: dict) -> dict:
    tm = cfg["threat_model"]
    attack = MembershipInferenceBlackBox(
        classifier=victim,
        input_type=tm["input_type"],
        attack_model_type=tm["attack_model_type"],
    )
    attack.fit(x_member, y_member, x_nonmember, y_nonmember)
    pred_member = attack.infer(x_member, y_member)
    pred_nonmember = attack.infer(x_nonmember, y_nonmember)
    y_true = np.concatenate([np.ones(len(pred_member)), np.zeros(len(pred_nonmember))])
    y_pred = np.concatenate([pred_member, pred_nonmember])
    auc = _auc(y_true, y_pred)
    return {"attack_auc": float(auc)}


def run_attribute_inference(victim, x_train_known, y_train_known, attack_feature_idx, x_target, cfg) -> dict:
    attack = AttributeInferenceBlackBox(classifier=victim, attack_feature=attack_feature_idx)
    attack.fit(x_train_known)
    inferred = attack.infer(x_target, y=y_target_labels(x_target, victim))
    true = x_target[:, attack_feature_idx]
    attribute_accuracy = float((inferred == true).mean())
    prior_accuracy = _uniform_prior_accuracy(true)
    return {
        "attribute_accuracy": attribute_accuracy,
        "prior_accuracy": prior_accuracy,
        "delta_vs_prior": attribute_accuracy - prior_accuracy,
    }
```

### 2.4 Defence side — `panel/defences/*`

```python
# rate_limit.py — sliding window per-user and per-IP
"""Simple sliding-window rate limiter. Not a production-grade scheme; it exists
to express the policy knob (window seconds × per-user cap × per-IP cap) and to
produce measurable denial events for the report.
"""
from collections import deque
from dataclasses import dataclass, field
from time import monotonic


@dataclass
class SlidingWindow:
    window_seconds: float
    cap: int
    history: dict = field(default_factory=dict)

    def allow(self, key: str) -> bool:
        now = monotonic()
        dq = self.history.setdefault(key, deque())
        while dq and (now - dq[0]) > self.window_seconds:
            dq.popleft()
        if len(dq) >= self.cap:
            return False
        dq.append(now)
        return True


# output_restriction.py — emit top-1 label plus a binned confidence
"""Output restriction. The panel uses discrete confidence bins so the attacker
cannot infer training-set membership from fine-grained probability deltas
(this is the mechanism of the Yeom 'confidence-based' MIA)."""
import numpy as np


def restrict_output(probs: np.ndarray, bins: int) -> dict:
    top1 = int(np.argmax(probs))
    top1_prob = float(probs[top1])
    bin_idx = int(min(bins - 1, int(top1_prob * bins)))
    return {"label_id": top1, "confidence_bin": bin_idx, "bins": bins}


# prada_detector.py — KS-test on client query distribution
"""PRADA-style detector (Juuti et al. 2019): over a window of queries from a
given client, compute the empirical distribution of pairwise distances
between queries; if the KS test vs a 'benign reference' distribution is
above threshold, flag the client."""
import numpy as np
from scipy.stats import ks_2samp


class PradaDetector:
    def __init__(self, window_queries: int, ks_threshold: float, reference_distances: np.ndarray):
        self.window = window_queries
        self.threshold = ks_threshold
        self.reference = reference_distances
        self.recent: dict[str, list[np.ndarray]] = {}

    def observe(self, client_id: str, query_vec: np.ndarray) -> bool:
        buf = self.recent.setdefault(client_id, [])
        buf.append(query_vec)
        if len(buf) > self.window:
            buf.pop(0)
        if len(buf) < self.window:
            return False
        dists = _pairwise_distances(np.stack(buf))
        stat, _ = ks_2samp(dists, self.reference)
        return bool(stat > self.threshold)


def _pairwise_distances(x: np.ndarray) -> np.ndarray:
    diffs = x[:, None, :] - x[None, :, :]
    d = np.linalg.norm(diffs, axis=-1)
    iu = np.triu_indices_from(d, k=1)
    return d[iu]


# dp_sgd.py — assertion wrapper for the DP-trained variant
"""If a DP-trained variant exists (weights_dp in config), the panel loads it,
reads the (ε, δ) stored in the training metadata, and asserts it meets the
register's target. Panel does not re-run DP training."""
import json
from pathlib import Path


def load_dp_metadata(dp_weights_path: Path) -> dict:
    meta_path = dp_weights_path.with_suffix(".meta.json")
    return json.loads(meta_path.read_text())


def assert_dp_envelope(meta: dict, epsilon_target: float, delta_target: float) -> dict:
    ok = (meta["epsilon"] <= epsilon_target) and (meta["delta"] <= delta_target)
    return {"epsilon_trained": meta["epsilon"], "delta_trained": meta["delta"], "envelope_met": bool(ok)}
```

### 2.5 Serving shim — `panel/serving/app.py` (defended endpoint)

```python
"""Defended endpoint. Composes rate-limit → PRADA detector → classifier → output restriction."""
from fastapi import FastAPI, HTTPException, Request
import numpy as np
import torch

from panel.defences.rate_limit import SlidingWindow
from panel.defences.output_restriction import restrict_output
from panel.defences.prada_detector import PradaDetector
from panel.io import load_target_for_serving, load_prada_reference

app = FastAPI()

cfg = app.state.cfg = _load_cfg()
victim = load_target_for_serving(cfg)
rl_user = SlidingWindow(cfg["defences"]["rate_limit"]["window_seconds"], cfg["defences"]["rate_limit"]["requests_per_user_per_window"])
rl_ip = SlidingWindow(cfg["defences"]["rate_limit"]["window_seconds"], cfg["defences"]["rate_limit"]["requests_per_ip_per_window"])
prada = PradaDetector(
    cfg["defences"]["prada_detector"]["window_queries"],
    cfg["defences"]["prada_detector"]["ks_test_threshold"],
    load_prada_reference(cfg["defences"]["prada_detector"]["reference"]),
)


@app.post("/predict")
async def predict(request: Request):
    body = await request.json()
    user = body["user"]
    ip = request.client.host
    x = np.array(body["features"], dtype=np.float32)

    if not rl_user.allow(user):
        raise HTTPException(status_code=429, detail="rate_limited:user")
    if not rl_ip.allow(ip):
        raise HTTPException(status_code=429, detail="rate_limited:ip")

    if prada.observe(user, x):
        raise HTTPException(status_code=403, detail="prada:flagged")

    with torch.no_grad():
        probs = torch.softmax(victim(torch.from_numpy(x).unsqueeze(0)), dim=-1)[0].numpy()

    return restrict_output(probs, bins=cfg["defences"]["output_restriction"]["confidence_bins"])
```

### 2.6 Report template — `REPORT.md` (excerpt)

```markdown
# CarePilot Extraction / MIA / Inversion Panel — Run {run.id}

**Threat-model source:** `mod-105-ex01-cross-tag.md` rows X1, X2, X3, X5 (cross-ref mod-104), A1.
**Scope:** mod-106 auxiliary triage classifier + CarePilot-owned scaffolding. Base LLM vendor concern: cross-referenced only.

## 1. Attack results — undefended endpoint

| Attack | Metric | Undefended |
|---|---|---|
| knockoff-extraction | substitute agreement on held-out (fidelity) | {kn.undef.agreement:.3f} |
| mia-black-box | member-vs-nonmember attack AUC | {mia.undef.auc:.3f} |
| attribute-inversion | Δ vs prior accuracy | {ai.undef.delta:+.3f} |
| system-prompt-canary | leak rate (from mod-104 run) | {sp.rate:.3f} |

## 2. Attack results — defended endpoint

| Attack | Metric | Defended | Δ vs undefended |
|---|---|---|---|
| knockoff-extraction | substitute agreement on held-out | {kn.def.agreement:.3f} | {kn.delta:+.3f} |
| mia-black-box | attack AUC | {mia.def.auc:.3f} | {mia.delta:+.3f} |
| attribute-inversion | Δ vs prior accuracy | {ai.def.delta:+.3f} | {ai.delta:+.3f} |

## 3. Defence activations

| Defence | Fired | Comment |
|---|---|---|
| rate-limit (per-user) | {rl_user.fire_rate:.3f} of knockoff-extraction queries blocked | Window = {cfg.rate_limit.window_seconds}s, cap = {cfg.rate_limit.requests_per_user_per_window} |
| rate-limit (per-IP) | {rl_ip.fire_rate:.3f} | Cap = {cfg.rate_limit.requests_per_ip_per_window} |
| PRADA detector | {prada.fire_rate:.3f} | KS threshold = {cfg.prada.ks_test_threshold} |
| output-restriction | always on | Emits top-1 label + {cfg.output_restriction.confidence_bins}-way binned confidence |
| DP-SGD training | {dp.enabled} | Trained ε = {dp.epsilon_trained}, δ = {dp.delta_trained}, envelope-met = {dp.envelope_met} |

## 4. Register-envelope comparison

| Register row | Envelope floor | Observed (defended) | Residual? |
|---|---|---|---|
| extraction-scaffolding (substitute agreement) | `to-confirm-at-register-review` | {kn.def.agreement:.3f} | {…} |
| mia-local-head (attack AUC) | `to-confirm-at-register-review` | {mia.def.auc:.3f} | {…} |
| inversion-local-head (Δ vs prior) | `to-confirm-at-register-review` | {ai.def.delta:+.3f} | {…} |
| system-prompt-leakage (canary rate) | cross-ref mod-104 | {sp.rate:.3f} | {…} |

<!-- submission-todo: CarePilot register envelopes for extraction/MIA/inversion; DP-SGD (ε, δ) target must be set in the register before this report binds a verdict. -->
```

## 3. Validation steps

A reviewer should be able to confirm the submission in under 45 minutes.

1. **Repro sanity.** `pip install -r requirements.txt`; `python run_panel.py --config config.yaml --mode both` starts both serving shims (defended on :8080, undefended on :8081) and runs all four attack entries end-to-end. The system-prompt-canary entry is a cross-reference stub — it reads a mod-104 artifact and quotes the rate.
2. **Determinism.** Same seed, same `aggregated.json`. KnockoffNets' RL-style adaptive sampling uses a seeded RNG.
3. **Scope ruling present.** Base-LLM extraction, base-LLM MIA, and training-data regurgitation are scope-ruled out and cross-referenced to mod-104 / mod-110.
4. **Attacks are canonical ART attacks.** `KnockoffNets`, `MembershipInferenceBlackBox`, `AttributeInferenceBlackBox`.
5. **Defences are composed, not just listed.** The defended endpoint is a single serving app that chains rate-limit → PRADA detector → classifier → output restriction. The undefended endpoint exposes the classifier raw.
6. **Before/after reported.** Each attack is run against both endpoints; the report shows the delta.
7. **DP-SGD handled explicitly.** If the retrained head has a DP-trained variant, its `(ε, δ)` is asserted against the register envelope. If not, `dp.enabled = false` is set and the report says so; it does not silently ignore the row.
8. **No invented metric envelopes.** Every envelope floor is cited from the register or marked `submission-todo`.
9. **Register linkage.** Every row in the "residual?" column is computed as `observed > envelope_floor`; rows with `submission-todo` floors are left as `TBD` and surfaced in section 4.

## 4. Rubric / review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Scope ruling | Base LLM and vendor supply-chain cross-ref'd out | Scope mixed |
| R2 | Four attack rows present | extraction, MIA, inversion, system-prompt-canary (cross-ref) | Fewer or renamed |
| R3 | Attacks are canonical | ART `KnockoffNets`, `MembershipInferenceBlackBox`, `AttributeInferenceBlackBox` | Hand-rolled without citation |
| R4 | Shared defence set present | Rate limit, PRADA detector, output restriction, DP-SGD | Any of the four missing |
| R5 | Defences are composed, not listed | A single serving endpoint chains all four | Defences described but not wired |
| R6 | Before/after measurement | Both undefended and defended results reported | Only one |
| R7 | DP-SGD config references the register | `(ε, δ)` target is register-sourced or `submission-todo` | Hard-coded ε in config |
| R8 | PRADA uses a reference distribution | KS test vs a persisted `prada-reference-distribution.npy` | Threshold with no reference |
| R9 | Rate-limit has measurable knobs | Window × cap, per-user and per-IP | Just "add rate limit" |
| R10 | Register linkage | Report cites every register row the attack rolls into | No linkage |
| R11 | No invented envelope numbers | Envelopes cited or `submission-todo` | Fabricated numbers |

**Scoring bands.**

- *Pass* — R1–R11 green.
- *Pass with revisions* — one of R6/R7/R10 fails; revise.
- *Fail* — R1 or R3 or R5 or R11 fails.

## 5. Common mistakes

- **Treating the base LLM as the extraction target.** The base LLM is vendor-hosted; CarePilot cannot DP-train or watermark it. The panel targets the *retrained head* and the *scaffolding/corpus*.
- **MIA without ground-truth members.** Membership inference needs an explicitly-labelled member set and a disjoint non-member set to score the attack. "Infer membership and see what happens" is not measurable.
- **MIA on a model with only top-1 output.** If the model already emits top-1 only, an MIA attack that relies on probability vectors will underperform. Report the output format the attack ran against. The defended endpoint *emits* top-1 + binned confidence — that is the defence, not a bug.
- **PRADA detector with no reference distribution.** PRADA (Juuti et al. 2019) distinguishes benign-client query distributions from extraction-style distributions. Without a persisted reference distribution, the KS test is meaningless.
- **DP-SGD with `(ε, δ)` set inline in panel code.** Privacy budget is a policy decision, not a code decision. It lives in the CarePilot register.
- **Reporting substitute-model *accuracy* instead of *agreement*.** The success metric for extraction is agreement with the victim on held-out inputs (fidelity), not standalone accuracy. A substitute that merely reaches high accuracy on its own eval is not evidence of successful extraction.
- **One-attack-wins framing.** The point of the exercise is the *shared API-surface defence set*. If the report says "we blocked extraction, so we're done", the MIA and inversion rows got missed. Report all three.
- **Rate-limit that short-circuits to 429 without logging.** Rate-limit denials are the loudest evidence of an in-progress attack; they must be logged and feed the mod-109 monitor plan.

## 6. References

- **NIST AI 100-2 E2023 — Adversarial Machine Learning: A Taxonomy and Terminology of Attacks and Mitigations.** `https://csrc.nist.gov/publications/detail/nistir/ai-100-2` — privacy attack family (extraction, MIA, inversion).
- **MITRE ATLAS.** `https://atlas.mitre.org/` — AML.T0024 Discover ML Model Family; AML.T0048 Steal ML Model; AML.T0052 Infer Training Data; AML.T0053 Infer Training Data Membership; AML.T0056 Extract LLM System Prompt; AML.T0029 Denial of ML Service; AML.T0040 ML Model Inference API Access.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — ML03 Model Inversion; ML04 Membership Inference; ML05 Model Theft.
- **Adversarial Robustness Toolbox (ART).** `https://adversarial-robustness-toolbox.readthedocs.io/` — `art.attacks.extraction.KnockoffNets`; `art.attacks.inference.membership_inference.MembershipInferenceBlackBox`; `art.attacks.inference.attribute_inference.AttributeInferenceBlackBox`.
- **Opacus** (PyTorch DP-SGD library). `https://opacus.ai/` — DP-SGD training for the retrained head's DP variant (referenced; not implemented in this panel).
- **Canonical papers (used only by name):** Shokri et al. 2017 (MIA); Yeom et al. 2018 (loss-based MIA); Fredrikson et al. 2015 (attribute inversion); Orekondy et al. 2019 (KnockoffNets extraction); Juuti et al. 2019 (PRADA detector); Abadi et al. 2016 (DP-SGD).
- **mod-105 ex-01 solution** — rows X1, X2, X3, X5, A1.
- **mod-104 red-team suite** — system-prompt leakage (X5 cross-reference).
- **Exercise prompt** — `lessons/mod-105-adversarial-ml-and-robustness/exercises/exercise-04-extraction-inversion-and-query-rate-defence.md`.

<!-- submission-todo: exact ART constructor signatures in the pinned release; Opacus `(ε, δ)` target from the CarePilot register; PRADA reference-distribution snapshot path from the baseline-traffic dataset. -->

# Exercise 02 — Attribute Inference and Model Inversion Drill — Reference Solution

## 1. Solution overview

Exercise 01 established the *membership* question (did an example appear in training?). Exercise 02 is the twin privacy drill: the *content* question. It splits into two attacks, both run in a sandbox against a model the student trained:

- **Attribute-inference attack (AIA).** The attacker has the model's prediction on a record *plus* a subset of the record's attributes, and tries to recover one or more *missing* sensitive attributes (e.g., the sensitive attribute that was deliberately held out). This is the classic Fredrikson-style *inference about the input* attack applied to tabular ML.
- **Model-inversion attack (MI).** The attacker, given a label (and in some variants, access to intermediate activations), tries to reconstruct a *representative input* for that label — the attack that produced the well-known "reconstruct a training-class face from a facial-recognition model" result in the literature.

The deliverable is (a) runnable code for both attacks, (b) a short write-up comparing the AIA's per-attribute inference accuracy to a **naïve majority-class baseline** and a **marginal-distribution baseline**, (c) a reconstruction-quality readout for model inversion (SSIM / pixelwise-MSE against the class centroid, with an explicit acknowledgement that a confident reconstruction is *not* "the person" — it is the class representation the model has absorbed), and (d) a mapping into the **CarePilot** risk register and the corresponding **OWASP Machine Learning Security Top 10** / **MITRE ATLAS** entries.

Design decisions worth calling out:

- **Baselines are mandatory.** An AIA that beats random is not interesting; an AIA that beats the marginal baseline by a statistically meaningful margin *is*. The exercise explicitly requires the delta-against-baseline readout.
- **MI reports class-representation similarity, not identification confidence.** The "reconstruct a specific individual" framing is the one the literature is careful to push back on; the artifact describes what MI *does* return (the class-representative signal) and what it does *not* return (ground-truth identification).
- **Both attacks use the same running system.** The tabular AIA runs against a model from a public tabular dataset standing in for CarePilot's structured prediction flow; the MI drill runs against a small-image classifier standing in for an imaging-assisted triage flow. Both are surrogates and are flagged as such in the write-up.
- **The attack suite is version-pinned.** Attack code is committed with a config hash; the mod-104 red-team suite reuses the same manifest shape (pinning + config hash → evidence manifest).

## 2. Worked implementation

### 2.1 Repository layout

```
exercise-02-attribute-inference-and-model-inversion-drill/
├── SOLUTION.md                 # this file
├── src/
│   ├── data_tabular.py         # AIA dataset + holdout-attribute split
│   ├── data_image.py           # MI small-image classifier dataset
│   ├── target_tabular.py       # train the AIA target model
│   ├── target_image.py         # train the MI target model
│   ├── attack_aia.py           # black-box AIA + two baselines
│   ├── attack_inversion.py     # gradient-based model inversion
│   ├── metrics_aia.py          # per-attribute accuracy, delta vs baselines
│   ├── metrics_inversion.py    # SSIM / MSE against class centroid
│   └── report.py               # JSON + markdown report
├── configs/
│   ├── aia.yaml
│   └── inversion.yaml
└── results/
    ├── aia_report.json
    ├── inversion_report.json
    └── inversion_samples/<class_id>.png
```

### 2.2 Attribute inference (`src/attack_aia.py`, `src/metrics_aia.py`)

Threat model for the black-box variant:

- The attacker has **oracle access** to the target model's probability output on any input.
- The attacker knows the record's **non-sensitive** attributes.
- The attacker tries to recover one **sensitive** attribute (categorical in the reference solution; continuous is a straightforward extension).

```python
# src/attack_aia.py
# Black-box AIA: for each candidate value v of the sensitive attribute,
# construct the record (non_sensitive, sensitive = v), query the target model,
# score how well the predicted label (or confidence vector) matches the record's
# known ground-truth label, and pick the argmax.

import numpy as np


def aia_score_candidate(
    target,
    non_sensitive: np.ndarray,  # shape (N, D_ns)
    candidate_value: int,        # discrete sensitive attribute candidate
    known_labels: np.ndarray,    # shape (N,), the record's ground-truth label (known to attacker here)
    sens_index: int,
    num_sens_values: int,
) -> np.ndarray:
    N = non_sensitive.shape[0]
    X_q = np.zeros((N, non_sensitive.shape[1] + num_sens_values), dtype=np.float32)
    X_q[:, :non_sensitive.shape[1]] = non_sensitive
    # One-hot the candidate value for the sensitive attribute.
    X_q[:, non_sensitive.shape[1] + candidate_value] = 1.0
    probs = target.predict_proba(X_q)
    # Score = P(true label | candidate record); higher means the record is more "plausible"
    # under the model, which is the attacker's signal.
    return probs[np.arange(N), known_labels]


def run_aia(
    target,
    X_nonsens: np.ndarray,
    y_known: np.ndarray,
    sens_true: np.ndarray,
    sens_index: int,
    num_sens_values: int,
) -> np.ndarray:
    # For each record, pick the sensitive-attribute value that maximises the model's confidence
    # in the record's known label.
    scores = np.stack(
        [
            aia_score_candidate(target, X_nonsens, v, y_known, sens_index, num_sens_values)
            for v in range(num_sens_values)
        ],
        axis=1,
    )
    pred = scores.argmax(axis=1)
    return pred
```

```python
# src/metrics_aia.py
# Report:
#   - attack accuracy       = (pred == sens_true).mean()
#   - majority-class baseline = fraction of records where the most frequent value of the
#                               sensitive attribute equals sens_true
#   - marginal-distribution baseline = sample from the marginal distribution of the sensitive
#                                       attribute conditional on nothing about the record; report
#                                       expected accuracy over many samples.

import numpy as np


def majority_baseline(sens_true: np.ndarray, train_sens: np.ndarray) -> float:
    major = int(np.bincount(train_sens).argmax())
    return float((sens_true == major).mean())


def marginal_baseline(sens_true: np.ndarray, train_sens: np.ndarray, n_samples: int = 10_000, seed: int = 0) -> float:
    rng = np.random.default_rng(seed)
    p = np.bincount(train_sens) / train_sens.shape[0]
    samples = rng.choice(len(p), size=(n_samples, sens_true.shape[0]), p=p)
    acc = (samples == sens_true[None, :]).mean(axis=0)  # avg per-sample accuracy for each record
    return float(acc.mean())


def report_aia(pred: np.ndarray, sens_true: np.ndarray, train_sens: np.ndarray) -> dict:
    attack_acc = float((pred == sens_true).mean())
    maj = majority_baseline(sens_true, train_sens)
    marg = marginal_baseline(sens_true, train_sens)
    return {
        "attack_accuracy": attack_acc,
        "majority_class_baseline": maj,
        "marginal_distribution_baseline": marg,
        "delta_vs_majority": attack_acc - maj,
        "delta_vs_marginal": attack_acc - marg,
    }
```

### 2.3 Model inversion (`src/attack_inversion.py`, `src/metrics_inversion.py`)

Threat model:

- White-box: the attacker has model weights and can back-propagate (the strongest MI setting).
- The target is a small-image classifier (e.g., a CNN on a public small-image dataset) standing in for an imaging-assisted triage classifier. The training set does not contain anything derived from real patient images; this is explicitly flagged in the write-up.

The attack optimises an input `x` (initialised from noise) to maximise the model's log-probability of a chosen target class, with a smoothness prior and class-probability floor:

```python
# src/attack_inversion.py
# White-box gradient-based model inversion with a smoothness prior.
# Returns the reconstructed input for each target class, along with the model's
# final class probability for the reconstruction.

import torch
import torch.nn.functional as F


def invert_class(
    model: torch.nn.Module,
    target_class: int,
    input_shape: tuple,
    steps: int = 500,
    lr: float = 0.1,
    smooth_lambda: float = 1e-3,
    device: str = "cpu",
    seed: int = 42,
) -> tuple[torch.Tensor, float]:
    g = torch.Generator(device=device).manual_seed(seed + target_class)
    x = torch.rand(input_shape, generator=g, device=device, requires_grad=True)
    opt = torch.optim.Adam([x], lr=lr)
    target = torch.tensor([target_class], device=device)
    for step in range(steps):
        opt.zero_grad()
        logits = model(x.unsqueeze(0))
        ce = F.cross_entropy(logits, target)
        # Total-variation smoothness prior; keeps reconstructions from drifting into high-frequency noise.
        tv = torch.mean(torch.abs(x[..., 1:, :] - x[..., :-1, :])) + torch.mean(
            torch.abs(x[..., :, 1:] - x[..., :, :-1])
        )
        loss = ce + smooth_lambda * tv
        loss.backward()
        opt.step()
        with torch.no_grad():
            x.clamp_(0.0, 1.0)
    with torch.no_grad():
        prob = F.softmax(model(x.unsqueeze(0)), dim=-1)[0, target_class].item()
    return x.detach(), prob
```

```python
# src/metrics_inversion.py
# Report per-class reconstruction quality against the class centroid of the public dataset.
# A high SSIM to the class centroid is the "class-representation signal"; it is NOT
# evidence that the reconstruction matches any specific training record.

import torch
import numpy as np
from skimage.metrics import structural_similarity as ssim


def class_centroid(images: np.ndarray, labels: np.ndarray, target_class: int) -> np.ndarray:
    mask = labels == target_class
    return images[mask].mean(axis=0)


def report_inversion(
    reconstructions: dict[int, np.ndarray],
    centroids: dict[int, np.ndarray],
    reconstruction_probs: dict[int, float],
) -> dict:
    out = {}
    for cls, rec in reconstructions.items():
        cen = centroids[cls]
        s = float(ssim(rec, cen, data_range=1.0, channel_axis=0 if rec.ndim == 3 else None))
        mse = float(np.mean((rec - cen) ** 2))
        out[str(cls)] = {
            "ssim_to_centroid": s,
            "mse_to_centroid": mse,
            "target_class_probability": float(reconstruction_probs[cls]),
        }
    return out
```

### 2.4 Config and run manifest

```yaml
# configs/aia.yaml
seed: 7
sens_attribute: "sensitive"   # column name of the attribute held out from the attacker
num_sens_values: 2            # binary for the reference run; the attack generalises
holdout_fraction: 0.30         # size of the AIA test slice (records whose sensitive value we try to recover)
```

```yaml
# configs/inversion.yaml
seed: 42
steps: 500
lr: 0.1
smooth_lambda: 1e-3
target_classes: [0, 1, 2, 3, 4]
input_shape: [1, 28, 28]       # 1x28x28 for a greyscale small-image classifier
```

The run manifest captures `(dataset_hash, target_model_commit, attack_commit, config_hash)` and is written into `results/*.json`. The manifest is what links the attack's output to the risk register.

### 2.5 Expected reporting shape

```json
{
  "aia": {
    "sensitive_attribute": "to-record-from-config",
    "attack_accuracy": "to-record-from-run",
    "majority_class_baseline": "to-record-from-run",
    "marginal_distribution_baseline": "to-record-from-run",
    "delta_vs_marginal": "to-record-from-run",
    "num_records_scored": "to-record-from-run",
    "notes": "AIA is interesting only when delta_vs_marginal is materially positive and stable across seeds; report both."
  },
  "inversion": {
    "per_class": {
      "0": {"ssim_to_centroid": "to-record-from-run", "mse_to_centroid": "to-record-from-run",
             "target_class_probability": "to-record-from-run"}
    },
    "disclaimer": "SSIM to the public class centroid is the class-representation signal; reconstructions are NOT identifications of any training example."
  }
}
```

### 2.6 Mapping back to CarePilot

- **AIA.** For CarePilot, the plausible concern is "an attacker with query access can reconstruct patient attributes the model infers but was not explicitly asked to return." The register row `risk-register/2025/CarePilot.yaml#risks[privacy-aia]` carries the attack's `delta_vs_marginal` as its measured number and the per-attribute candidate list from the harm model (mod-102). Mitigation: minimise the surface area of attributes the model accepts; limit the probability-vector return in the inference API (clip or discretise); enforce rate-limits.
- **Model inversion.** CarePilot does not deploy a classifier whose *input* is a patient's full photograph. The MI drill is therefore a *transferable methodology*, not a direct CarePilot finding; the write-up states this explicitly. The methodology is relevant to any downstream imaging-assisted triage flow added later (e.g., a chest-X-ray pre-reader). The register's `risks[privacy-mi]` row carries this note.
- **OWASP ML Top 10 mapping.** <!-- verify-citation: cite the exact OWASP Machine Learning Security Top 10 entry (code + title + revision date) for Model Inversion Attack; the ML Top 10 project carries a dedicated entry for this attack class. -->
- **MITRE ATLAS mapping.** <!-- verify-citation: cite the ATLAS technique ID that most precisely covers model inversion / attribute inference via the inference API. -->

## 3. Validation steps

1. **Pipeline runs end-to-end.** `python -m src.target_tabular && python -m src.attack_aia && python -m src.target_image && python -m src.attack_inversion && python -m src.report` produces `results/aia_report.json`, `results/inversion_report.json`, and the per-class reconstructions.
2. **AIA uses a held-out attribute.** `data_tabular.py` partitions records such that the sensitive attribute of the records in the attack-test slice was *never* exposed to the attack code outside `metrics_aia.py`.
3. **Both baselines are reported.** `aia_report.json` contains `majority_class_baseline`, `marginal_distribution_baseline`, and both deltas.
4. **MI reconstructions are per-class, not per-record.** `inversion_report.json` has one entry per *class* in `target_classes`, not one per training record. A reviewer checks that the code does not attempt per-record inversion (which would be the "specific training example" claim the exercise is careful not to make).
5. **MI disclaimer is present.** `inversion_report.json["inversion"]["disclaimer"]` is non-empty and the write-up includes the "class-representation, not identification" note.
6. **Run manifest.** `(dataset_hash, target_model_commit, attack_commit, config_hash)` fields are present in both JSONs; the risk-register row points at the manifest.
7. **Determinism.** Re-running with the same seed reproduces AIA accuracy to within numerical noise; MI reconstructions are visually consistent (SSIM ≥ 0.95 run-to-run with the same seed).

## 4. Rubric — review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Both attacks implemented | `attack_aia.py` and `attack_inversion.py` present and runnable | One attack skipped or stubbed |
| R2 | AIA reports delta vs *both* baselines | `delta_vs_majority` and `delta_vs_marginal` present and ≥0 is NOT required (negative is a legitimate finding — "the attack fails") | Only attack accuracy reported |
| R3 | Sensitive attribute is genuinely held out | No code outside `metrics_aia.py` dereferences `sens_true` for the attack-test slice | Attack code leaks the ground-truth value |
| R4 | MI reconstructions are per-class | `inversion_report.json["per_class"]` is keyed by class, not record ID | Per-record reconstructions and claims of "training-example recovery" |
| R5 | MI disclaimer | Disclaimer text appears in the report and in the write-up | No disclaimer; write-up frames reconstructions as identifications |
| R6 | Smoothness prior used in MI | `attack_inversion.py` includes a TV / smoothness regulariser; without it the gradients produce high-frequency noise and SSIM is meaningless | Pure cross-entropy maximisation with no prior |
| R7 | Baselines code is auditable | `majority_baseline` and `marginal_baseline` are < 10 lines each and a reviewer can read them | Hidden in an opaque helper; unverifiable |
| R8 | CarePilot tie-back | Both attacks are mapped to the CarePilot register row and the OWASP ML Top 10 + ATLAS mapping is cited (or marked `<!-- verify-citation: … -->`) | Attack numbers reported with no mapping |
| R9 | Run manifest | `(dataset_hash, target_model_commit, attack_commit, config_hash)` present in both JSONs | Result files have no provenance |
| R10 | No invented facts | `<!-- verify-citation: ... -->` markers appear wherever OWASP / ATLAS codes cannot be verified | Fabricated citations or metric numbers |

Scoring bands: *Pass* — R1–R9 all green and R10 is honoured; *Pass with revisions* — R6 or R7 fails but the methodology is otherwise correct; *Fail* — any of R3 / R4 / R5 / R10 fails.

## 5. Common mistakes

- **Dropping the majority / marginal baselines.** Attack accuracy without a baseline is uninformative. A 70 % accuracy on a 70:30 attribute is 0 % useful information.
- **Leaking the sensitive attribute into the attack features.** AIA fails review if the attack has access to the ground-truth sensitive value at inference time, for the same record whose value it is predicting.
- **Claiming MI reconstructs a specific training example.** The reconstructed image is the class representation; it is not an identification of any specific person. This framing error is the most common one in the privacy-attack literature and must be caught in review.
- **Running MI with no smoothness prior.** The optimiser will happily drive pixel values to extremes that maximise class confidence while looking like noise. The SSIM-to-centroid readout becomes random. Always include a smoothness regulariser.
- **Testing MI only on a class the model is confident about.** Pick classes across the confidence spectrum so the report is representative.
- **Reporting AIA accuracy as evidence about CarePilot.** The AIA numbers are produced on a public tabular surrogate. The relevant output for CarePilot is the *methodology* and the risk-register row; the surrogate numbers belong to the methodology, not to CarePilot's actual risk posture.
- **Skipping the run manifest.** Without `(config_hash, commit, dataset_hash)` the result is unreproducible; the risk register entry points at an empty slot.

## 6. References

Only the official sources listed in the work packet (and clearly-labelled module / academic references) are used.

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — this exercise's output feeds the CarePilot **MEASURE 2.7** row (security and resilience) and the **MEASURE 2.11 / MAP 2.3** rows via the privacy eval set.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — the ML Top 10 project enumerates model inversion and inference-API data exfiltration as model-privacy risks; cite the exact entry at submission. <!-- verify-citation: cite the exact OWASP ML Top 10 entry codes and revision date for (a) Model Inversion Attack and (b) the attribute-inference risk family. -->
- **MITRE ATLAS.** `https://atlas.mitre.org/` — the ATLAS matrix lists techniques for information disclosure via the ML inference API; cite exact technique IDs at submission. <!-- verify-citation: cite ATLAS technique IDs for model inversion and attribute inference. -->
- **Fredrikson, M., Jha, S., Ristenpart, T.** *Model Inversion Attacks that Exploit Confidence Information and Basic Countermeasures.* ACM CCS, 2015. — the canonical reference for both attribute inference and model inversion against ML classifiers.
- **Module running system.** CarePilot — hospital-triage-support RAG assistant, introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`.
- **Exercise prompt.** `lessons/mod-107-privacy-risk-engineering/exercises/exercise-02-attribute-inference-and-model-inversion-drill.md`.

<!-- Numeric results in §2.5 are run-time placeholders by design; the solution does not quote invented values. -->

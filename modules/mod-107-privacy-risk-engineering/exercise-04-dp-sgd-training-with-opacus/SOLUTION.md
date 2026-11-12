# Exercise 04 — Differentially Private Training with Opacus — Reference Solution

## 1. Solution overview

This exercise is the mitigation counterpart to exercise 01 and exercise 03. A privacy-attack result is only actionable if there is a *dial* the engineer can turn to reduce it; **differentially private stochastic gradient descent (DP-SGD)** is the canonical dial, and [**Opacus**](https://opacus.ai/) is the PyTorch library that provides a validated, drop-in implementation. The deliverable is (a) a working DP-SGD training script on a small reference model, (b) a sweep across privacy budgets `ε ∈ {1, 3, 8, ∞}` at a fixed `δ`, (c) a side-by-side report of utility (test accuracy) vs. privacy (`ε`) vs. privacy-attack signal (the LiRA score from exercise 01 re-run on each checkpoint), and (d) the mapping of the sweep result into CarePilot's `privacy-envelope` in the risk register.

The reference solution:

- Trains a small classifier three ways: **non-private (`ε = ∞`)**, **DP-SGD at `ε = 8`**, **DP-SGD at `ε = 3`**, **DP-SGD at `ε = 1`**, all at `δ = 1e−5` (or `δ ≤ 1 / n_train` per the standard rule of thumb — the config states which).
- Uses Opacus's `PrivacyEngine` to attach the DP-SGD machinery (per-sample-gradient clipping, Gaussian noise injection, Rényi-DP accountant for the privacy budget). The accountant provides the realised `(ε, δ)` at the end of training — the solution reports the *realised* `ε`, not the configured one.
- Runs the LiRA attack from exercise 01 against every checkpoint, so the sweep is visible as a *triple*: `(ε, test_accuracy, lira_tpr_at_fpr_0_001)`. This triple is what the risk register needs.
- Explicitly flags the **compatibility caveats** — not every `nn.Module` is Opacus-compatible out of the box. `BatchNorm` layers must be replaced with `GroupNorm`-equivalents; models with per-sample statistical dependence need `ModuleValidator.fix` or manual replacement. The reference solution uses a model that is Opacus-compatible by design, and the common fix for `BatchNorm` is documented.
- Lets the write-up draw the only conclusion a reviewer should accept: **there is no free DP-SGD `ε`.** Smaller `ε` yields stronger privacy at a monotone cost to utility; the slope depends on the model, the dataset, and the clipping/noise multiplier. The sweep's *shape* is the conclusion, not any single row.

## 2. Worked implementation

### 2.1 Repository layout

```
exercise-04-dp-sgd-training-with-opacus/
├── SOLUTION.md                 # this file
├── src/
│   ├── data.py                 # dataset load, batching with `BatchMemoryManager` for virtual batches
│   ├── model.py                # small CNN or MLP; Opacus-compatible (GroupNorm not BatchNorm)
│   ├── train_dp.py             # DP-SGD training loop driven by Opacus PrivacyEngine
│   ├── train_nondp.py          # non-private baseline (ε = ∞)
│   ├── run_lira_on_checkpoint.py # re-use attack code from exercise 01 against each saved checkpoint
│   └── report.py               # sweep table + CSV
├── configs/
│   ├── eps_inf.yaml
│   ├── eps_8.yaml
│   ├── eps_3.yaml
│   └── eps_1.yaml
└── results/
    ├── sweep.json
    └── sweep.csv
```

### 2.2 Model (`src/model.py`)

Opacus validates models with `ModuleValidator`. `nn.BatchNorm*` layers fail validation because the running statistics violate per-sample independence; standard replacement is `nn.GroupNorm` (per the Opacus library documentation at `https://opacus.ai`).

```python
# src/model.py
import torch.nn as nn


class SmallCNN(nn.Module):
    def __init__(self, num_classes: int = 10):
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(1, 32, 3, padding=1),
            nn.GroupNorm(num_groups=8, num_channels=32),   # NOT BatchNorm — Opacus-compatible
            nn.ReLU(inplace=True),
            nn.MaxPool2d(2),
            nn.Conv2d(32, 64, 3, padding=1),
            nn.GroupNorm(num_groups=8, num_channels=64),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(2),
        )
        self.classifier = nn.Sequential(
            nn.Flatten(),
            nn.Linear(64 * 7 * 7, 128),
            nn.ReLU(inplace=True),
            nn.Linear(128, num_classes),
        )

    def forward(self, x):
        return self.classifier(self.features(x))
```

### 2.3 DP-SGD training loop (`src/train_dp.py`)

```python
# src/train_dp.py
# Standard DP-SGD training loop using Opacus's PrivacyEngine.
# Key ingredients (all documented at https://opacus.ai):
#   - PrivacyEngine.make_private_with_epsilon(...) attaches the engine, wires per-sample
#     gradient clipping at `max_grad_norm`, injects Gaussian noise with the sampled
#     multiplier, and installs the RDP accountant.
#   - BatchMemoryManager batches virtual micro-batches into a target `max_physical_batch_size`
#     to avoid OOM at large logical batch sizes (which DP-SGD benefits from).
#   - ModuleValidator.fix / ModuleValidator.is_valid should be called before training.

from pathlib import Path
import torch
import torch.nn as nn
import torch.optim as optim
from opacus import PrivacyEngine
from opacus.utils.batch_memory_manager import BatchMemoryManager
from opacus.validators import ModuleValidator

from .model import SmallCNN


def train_dp(
    train_loader,
    test_loader,
    *,
    target_epsilon: float,
    target_delta: float,
    epochs: int,
    max_grad_norm: float,
    max_physical_batch_size: int,
    device: str,
    seed: int,
    ckpt_path: Path,
) -> dict:
    torch.manual_seed(seed)
    model = SmallCNN().to(device)
    assert ModuleValidator.is_valid(model), ModuleValidator.validate(model, strict=False)

    optimizer = optim.SGD(model.parameters(), lr=0.1, momentum=0.9)
    criterion = nn.CrossEntropyLoss()

    privacy_engine = PrivacyEngine()
    model, optimizer, train_loader = privacy_engine.make_private_with_epsilon(
        module=model,
        optimizer=optimizer,
        data_loader=train_loader,
        epochs=epochs,
        target_epsilon=target_epsilon,
        target_delta=target_delta,
        max_grad_norm=max_grad_norm,
    )

    for epoch in range(epochs):
        model.train()
        with BatchMemoryManager(
            data_loader=train_loader,
            max_physical_batch_size=max_physical_batch_size,
            optimizer=optimizer,
        ) as memory_safe_loader:
            for x, y in memory_safe_loader:
                x, y = x.to(device), y.to(device)
                optimizer.zero_grad()
                logits = model(x)
                loss = criterion(logits, y)
                loss.backward()
                optimizer.step()

    realised_epsilon = privacy_engine.get_epsilon(delta=target_delta)
    test_acc = _evaluate(model, test_loader, device)
    torch.save({"state_dict": model.state_dict(), "epsilon": realised_epsilon}, ckpt_path)
    return {"realised_epsilon": float(realised_epsilon), "test_accuracy": float(test_acc)}


def _evaluate(model, loader, device) -> float:
    model.eval()
    n, correct = 0, 0
    with torch.no_grad():
        for x, y in loader:
            x, y = x.to(device), y.to(device)
            pred = model(x).argmax(dim=-1)
            correct += (pred == y).sum().item()
            n += y.numel()
    return correct / max(1, n)
```

### 2.4 Non-private baseline (`src/train_nondp.py`)

```python
# src/train_nondp.py
# Same model, same dataset, same optimiser — no PrivacyEngine attached. This is the
# ε = ∞ row. Its test accuracy sets the utility ceiling; its LiRA score sets the
# privacy-attack floor (worst case).

from pathlib import Path
import torch
import torch.nn as nn
import torch.optim as optim
from .model import SmallCNN


def train_nondp(train_loader, test_loader, *, epochs: int, device: str, seed: int, ckpt_path: Path) -> dict:
    torch.manual_seed(seed)
    model = SmallCNN().to(device)
    opt = optim.SGD(model.parameters(), lr=0.1, momentum=0.9)
    crit = nn.CrossEntropyLoss()
    for _ in range(epochs):
        model.train()
        for x, y in train_loader:
            x, y = x.to(device), y.to(device)
            opt.zero_grad()
            crit(model(x), y).backward()
            opt.step()
    model.eval()
    test_acc, n = 0, 0
    with torch.no_grad():
        for x, y in test_loader:
            x, y = x.to(device), y.to(device)
            pred = model(x).argmax(dim=-1)
            test_acc += (pred == y).sum().item()
            n += y.numel()
    torch.save({"state_dict": model.state_dict(), "epsilon": float("inf")}, ckpt_path)
    return {"realised_epsilon": float("inf"), "test_accuracy": test_acc / max(1, n)}
```

### 2.5 Re-running LiRA on each checkpoint (`src/run_lira_on_checkpoint.py`)

The attack code is identical to exercise 01; the only difference is which checkpoint plays the role of the "target model." For the Opacus-trained checkpoints, the shadow models must be trained with the *same* DP-SGD configuration as the target — otherwise the IN/OUT distributions in LiRA are biased against the target and the attack result is uninterpretable.

```python
# src/run_lira_on_checkpoint.py
# Thin wrapper that loads a checkpoint, runs the exercise-01 shadow-and-LiRA pipeline
# with the same (dataset, splits, N_shadows, shadow architecture, shadow training recipe)
# as the target, and returns the LiRA TPR @ FPR = 0.1 %.

from pathlib import Path


def run_lira(checkpoint_path: Path, shadows_dir: Path) -> dict:
    # Delegates to exercise-01 code (imported as a package in the sibling exercise).
    # Returns {"auc": ..., "tpr_at_fpr_0_001": ..., "tpr_at_fpr_0_01": ...}.
    ...
```

### 2.6 Config snippets

```yaml
# configs/eps_8.yaml
target_epsilon: 8.0
target_delta: 1.0e-5
epochs: 10
max_grad_norm: 1.0
batch_size: 512
max_physical_batch_size: 128
seed: 42
```

```yaml
# configs/eps_1.yaml
target_epsilon: 1.0
target_delta: 1.0e-5
epochs: 10
max_grad_norm: 1.0
batch_size: 512
max_physical_batch_size: 128
seed: 42
```

### 2.7 Expected sweep report

```json
{
  "fixed_delta": 1.0e-5,
  "model": "SmallCNN",
  "dataset": "to-record-from-run",
  "sweep": [
    {"config": "eps_inf",  "configured_epsilon": "inf", "realised_epsilon": "inf", "test_accuracy": "to-record-from-run",  "lira_tpr_at_fpr_0_001": "to-record-from-run"},
    {"config": "eps_8",    "configured_epsilon": 8.0,   "realised_epsilon": "to-record-from-run", "test_accuracy": "to-record-from-run",  "lira_tpr_at_fpr_0_001": "to-record-from-run"},
    {"config": "eps_3",    "configured_epsilon": 3.0,   "realised_epsilon": "to-record-from-run", "test_accuracy": "to-record-from-run",  "lira_tpr_at_fpr_0_001": "to-record-from-run"},
    {"config": "eps_1",    "configured_epsilon": 1.0,   "realised_epsilon": "to-record-from-run", "test_accuracy": "to-record-from-run",  "lira_tpr_at_fpr_0_001": "to-record-from-run"}
  ],
  "expected_pattern": [
    "test_accuracy is monotone (approximately) non-decreasing in epsilon",
    "lira_tpr_at_fpr_0_001 is monotone (approximately) non-decreasing in epsilon",
    "if either pattern is violated, diagnose: clipping too aggressive, batch size too small, or shadow training recipe does not match target"
  ]
}
```

Concrete numeric accuracy / attack values are written at run time; this reference does not quote invented numbers.

### 2.8 Mapping back to CarePilot

- **How this feeds the privacy envelope.** CarePilot's `risk-register/2025/CarePilot.yaml#privacy-envelope` sets (a) a target `ε` for any fine-tune of CarePilot's LLM / classifier components, (b) a maximum acceptable LiRA `tpr_at_fpr_0_001`, and (c) the test-accuracy floor the clinical-safety officer has approved. The three sweep triples are the input to the envelope's calibration.
- **Note on LLM fine-tuning.** DP-SGD at the *LLM-fine-tune* scale is costly in both compute and utility; the reference solution's small-model sweep is a *methodology demonstration*. For LLM fine-tuning the engineering trade-off typically favours **PII removal from the fine-tune corpus (exercise 05)** plus **output guardrails (mod-108)** rather than DP-SGD for the full fine-tune — the write-up states this and notes DP-SGD is the right fit for the structured-prediction components (classifiers in the pipeline) where it is cheaper. <!-- verify-citation: cite a specific, verifiable DP-SGD benchmark result on a transformer fine-tune (e.g., a paper or an Opacus tutorial result) rather than relying on summary claims. -->
- **OWASP ML Top 10 mapping.** <!-- verify-citation: cite the exact OWASP ML Top 10 entry for the mitigation class (privacy-preserving training) in the current revision. -->
- **MITRE ATLAS mitigation.** The ATLAS matrix includes mitigation entries; cite the mitigation ID that most precisely covers "train with differential privacy". <!-- verify-citation: cite the exact ATLAS mitigation ID for DP training. -->

## 3. Validation steps

1. **Realised `ε` reported, not configured.** The sweep JSON contains `realised_epsilon` from `PrivacyEngine.get_epsilon(delta=...)`. Configured and realised match within accountant rounding; if they diverge, diagnose (usually: fewer epochs than planned).
2. **Model passes `ModuleValidator.is_valid`.** The DP training script calls the validator and the script exits before training if the model fails. Review `src/train_dp.py` to confirm.
3. **Shadow training matches target training.** For each `ε` config, the shadows in the LiRA run use the *same* `ε`, `max_grad_norm`, and `epochs`. If the shadow recipe is "vanilla SGD" while the target is "DP-SGD at ε=1", the LiRA result is meaningless; the reviewer rejects.
4. **ε = ∞ baseline present.** A sweep that only reports DP rows cannot show the utility vs. privacy slope. The non-private row is required.
5. **Sweep direction sanity check.** `test_accuracy` is non-increasing as `ε` decreases (monotonicity holds *on average*; one row off by noise is tolerated; a reversed trend is a reviewer finding).
6. **LiRA re-uses exercise 01's code.** The LiRA code is imported from exercise 01's module, not re-implemented in this exercise. A reviewer checks the import path.
7. **Config hashes, dataset hashes, and checkpoint commits recorded.** Every row of `sweep.json` has the provenance triple.
8. **GPU / CPU reproducibility note.** DP-SGD with Opacus is approximately deterministic with a fixed seed on CPU; on GPU, cuDNN non-determinism may introduce run-to-run variation. The write-up states which hardware was used.

## 4. Rubric — review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | DP-SGD training uses Opacus's `PrivacyEngine` | `make_private_with_epsilon` is called; the accountant is read with `get_epsilon` at the end | Hand-rolled DP-SGD; no accountant |
| R2 | Realised ε reported | `realised_epsilon` present per row from the accountant, not the configured value | Only configured ε reported |
| R3 | Four rows in the sweep: ∞, 8, 3, 1 | All four present at the fixed δ | Missing the non-private baseline, or sweep collapsed to one row |
| R4 | LiRA rerun per checkpoint | For each row, `lira_tpr_at_fpr_0_001` present and the shadow recipe matches the target | LiRA skipped; or reused shadows from a different training recipe |
| R5 | Model is Opacus-compatible | No BatchNorm; `ModuleValidator.is_valid` returns True | Raw BatchNorm in the model; validator skipped |
| R6 | Monotonicity check honoured | Write-up acknowledges the expected monotonic shape and diagnoses any deviation | Deviation present but unacknowledged |
| R7 | CarePilot tie-back | Sweep feeds `privacy-envelope`; LLM-fine-tune vs classifier-fine-tune trade-off discussed; mitigation cross-links to exercises 05, mod-108 named | Numbers reported standalone |
| R8 | Hardware + reproducibility note | Hardware named; GPU-non-determinism caveat stated if GPU was used | Silent on hardware |
| R9 | Provenance recorded | `(dataset_hash, model_commit, config_hash)` per row | Missing provenance |
| R10 | No invented facts | `<!-- verify-citation: … -->` on any benchmark number not personally run; OWASP / ATLAS citations gated by the same marker | Fabricated benchmark result |

Scoring bands: *Pass* — R1–R9 all green and R10 is honoured; *Pass with revisions* — R6 or R8 fails; *Fail* — any of R1 / R2 / R3 / R4 / R10 fails.

## 5. Common mistakes

- **Reporting configured ε instead of realised ε.** The configured value is the *target*; the accountant returns the *achieved*. Always report the achieved value.
- **Forgetting `ModuleValidator`.** A model with `BatchNorm` will silently run through Opacus but the per-sample-gradient semantics are broken; the training is not actually DP. Fail fast on `is_valid == False`.
- **Training shadows without DP but attacking a DP target.** This inflates the LiRA signal and overstates the attack's success against the DP target. Shadows must mirror the target's training recipe.
- **Skipping the `ε = ∞` row.** Without the baseline, the sweep cannot be read as a trade-off — only as an isolated set of (ε, accuracy) points.
- **Treating the sweep's absolute accuracy numbers as portable.** Accuracy at `ε = 1` on this small CNN says nothing about accuracy at `ε = 1` on CarePilot's LLM. The methodology transfers; the numbers do not.
- **Choosing δ > 1 / n_train.** The standard rule is `δ ≪ 1 / n_train`; δ = 1e-5 or smaller for the usual training sizes. A larger δ means the privacy guarantee can fail more often than the training set is large, which is operationally the wrong setting.
- **Running on GPU and not stating it.** Non-determinism in GPU kernels makes the sweep less reproducible than on CPU. State the hardware.
- **Claiming DP-SGD is the mitigation for LLM memorisation at scale without naming the cost.** DP fine-tuning of LLMs is costly; the responsible report acknowledges that for LLMs, PII-level mitigations (exercise 05) and output guardrails (mod-108) are usually the practical first line.

## 6. References

Only the official sources listed in the work packet (and clearly-labelled project documentation / academic references) are used.

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — the sweep feeds **MANAGE 1.3** (response planning for prioritised privacy risks) and **MEASURE 2.11 / MEASURE 2.7** of CarePilot's RMF map.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — cite the exact mitigation / risk entry at submission. <!-- verify-citation: cite the exact OWASP ML Top 10 entry (code + title + revision date) that pairs with DP-SGD as a mitigation. -->
- **MITRE ATLAS.** `https://atlas.mitre.org/` — the ATLAS matrix enumerates both techniques and mitigations; cite the mitigation ID that covers DP training. <!-- verify-citation: cite exact ATLAS mitigation ID for DP training. -->
- **Opacus (PyTorch library for differentially private training).** Project documentation at `https://opacus.ai`. API references used in this solution: `PrivacyEngine.make_private_with_epsilon`, `PrivacyEngine.get_epsilon`, `BatchMemoryManager`, `ModuleValidator`. The library implements the DP-SGD algorithm of Abadi et al. (2016) with the Rényi-DP accountant.
- **Abadi, M., Chu, A., Goodfellow, I., McMahan, H. B., Mironov, I., Talwar, K., Zhang, L.** *Deep Learning with Differential Privacy.* ACM CCS, 2016. — the DP-SGD algorithm Opacus implements.
- **Module running system.** CarePilot — hospital-triage-support RAG assistant, introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`.
- **Exercise prompt.** `lessons/mod-107-privacy-risk-engineering/exercises/exercise-04-dp-sgd-training-with-opacus.md`.

<!-- Numeric accuracy / TPR values in §2.7 are run-time placeholders by design; the solution does not quote invented values. -->

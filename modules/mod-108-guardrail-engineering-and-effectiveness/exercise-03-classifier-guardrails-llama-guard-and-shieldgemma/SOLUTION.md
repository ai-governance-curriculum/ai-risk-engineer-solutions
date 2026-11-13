# Exercise 03 — Classifier Guardrails: Llama Guard + ShieldGemma (CarePilot) — Reference Solution

## 1. Solution overview

This exercise stacks the **classifier layer** on top of the rule / flow layer shipped in exercise 02. The stubs in `nemo/actions.py` (`llama_guard_action`, `shieldgemma_action`) get their real bodies here. Thresholds get tuned on a labelled calibration slice. Composition is documented so a reviewer can see that the output slot uses **two different models with different training data**, not two variants of the same model at two thresholds.

Four decisions drive the shape:

- **Llama Guard at slot 1 and slot 4; ShieldGemma at slot 2 and slot 4.** Follows chapter 03 §"Worked composition — customer-support RAG assistant" for CarePilot's RAG-with-tool-calls shape. Slot 4 is the defence-in-depth slot: Llama Guard (safety taxonomy) *and* ShieldGemma (per-policy probabilities) must both return `safe`; either `unsafe` rejects. Different training data, different mechanisms — chapter 03's composition test passes.
- **Threshold discipline, not vendor defaults.** Llama Guard's categories are tuned by *taxonomy scope per slot* (which categories are blocking vs. warning); ShieldGemma's thresholds are tuned per policy on a labelled slice. Shipping the vendor default is the chapter 03 anti-pattern most commonly bitten.
- **Pin everything.** Every classifier call carries the model checkpoint, the taxonomy / policy version, and the threshold in the trace. Exercise 05's measurement contract requires reproducibility; the pins land here.
- **Fail closed on classifier error.** A classifier that returns HTTP 500, times out, or returns an unparseable response produces a `reject` verdict at input and retrieval slots and a refusal at the output slot. A fault-injection test proves this.

Calibration is on a labelled slice (200–500 items) that overlaps the mod-104 hosted adversarial-and-legitimate set. The slice is **not** the final measurement set — that stays in exercise 05 — this is scaffolding so exercise 05's measurement is not chasing an obviously bad threshold.

Deliverables live under `risk/mod-108/carepilot/classifiers/` and are indexed from `mod-108-ex03-classifiers.md`.

## 2. Worked implementation

### 2.1 Repository layout

```
risk/mod-108/carepilot/classifiers/
├── llama_guard.py                  # client + action body swapped into exercise-02 stubs
├── shieldgemma.py                  # client + action body
├── tuning/
│   ├── slice.md                    # calibration slice: source, labelling, counts
│   ├── slice.jsonl                 # labelled items (or a fingerprint if real data)
│   └── thresholds.md               # sweep results, chosen thresholds per classifier per slot
├── composition.md                  # slot-by-slot ordered stack + disagreement handling
├── latency.md                      # p50 / p95 per classifier per slot (middleware-measured)
├── cost.md                         # mean cost per call + per-day roll-up
└── failure-modes.md                # four chapter-03 anti-pattern probes
```

### 2.2 Llama Guard action (`llama_guard.py`)

Pin the model checkpoint, the taxonomy version, and the runtime. CarePilot is a hospital-triage system under HIPAA — self-hosted on GPU inside the clinical network boundary is the only acceptable deployment shape; hosted vendor egress on user content is disqualified for this use case (and would also disqualify OpenAI Moderation; see chapter 03 §"OpenAI Moderation").

```python
# classifiers/llama_guard.py
# Llama Guard — self-hosted on GPU inside the CarePilot network boundary.
# Model card: https://huggingface.co/meta-llama/<pin> — read before shipping.
# <!-- verify-citation: pin the exact released version (Llama Guard 3 or 4) and
#      the taxonomy revision the model card names at submission time. -->

from __future__ import annotations

import time
from dataclasses import dataclass
from typing import Any, Iterable

import httpx


LLAMA_GUARD_CHECKPOINT = "<meta-llama/Llama-Guard-3-8B-pin>"      # pin at install
LLAMA_GUARD_TAXONOMY = "<llama-guard-taxonomy-v3-pin>"            # pin alongside
LLAMA_GUARD_RUNTIME = "<vllm-pin or tgi-pin or transformers-pin>"
LLAMA_GUARD_ENDPOINT = "http://llama-guard.internal:8080/classify"


# Slot-specific taxonomy scope. Categories flagged "block" below produce a
# reject verdict; categories flagged "warn" are logged but pass. The scope is
# tuned on the calibration slice (see tuning/thresholds.md); do not use the
# vendor default across all categories.
#
# Chapter 03 §"Meta Llama Guard": the taxonomy codes (S1..S14 in Llama Guard 3)
# are the vendor's safety categories. Which codes are block/warn is the
# CarePilot risk-council decision, not Meta's.
#
# <!-- verify-citation: pin the exact category codes and their current titles
#      from the Llama Guard 3 model card at submission time. -->

TAXONOMY_SCOPE_INPUT: dict[str, str] = {
    "S1": "block", "S2": "block", "S3": "block", "S4": "block",
    "S5": "warn",  "S6": "block", "S7": "block", "S8": "block",
    "S9": "block", "S10": "block", "S11": "block", "S12": "warn",
    "S13": "warn", "S14": "block",
}
TAXONOMY_SCOPE_OUTPUT: dict[str, str] = {
    # Narrower at output: block only on high-confidence categories; the per-
    # policy ShieldGemma score covers the rest of the defence-in-depth.
    "S1": "block", "S2": "block", "S3": "block",
    "S4": "block", "S9": "block", "S10": "block",
    "S11": "block", "S14": "block",
}


@dataclass(frozen=True)
class LlamaGuardVerdict:
    verdict: str              # "safe" | "reject"
    categories: tuple[str, ...]
    checkpoint: str
    taxonomy: str
    latency_ms: float
    raw: dict[str, Any]


def _call(text: str, timeout_s: float = 2.0) -> dict[str, Any]:
    t0 = time.perf_counter()
    try:
        r = httpx.post(LLAMA_GUARD_ENDPOINT, json={"text": text}, timeout=timeout_s)
        r.raise_for_status()
        return {"json": r.json(), "latency_ms": (time.perf_counter() - t0) * 1000}
    except Exception as exc:
        # Fail closed: the caller treats this as a reject verdict.
        return {"json": None, "latency_ms": (time.perf_counter() - t0) * 1000, "error": str(exc)}


def classify(text: str, scope: dict[str, str]) -> LlamaGuardVerdict:
    out = _call(text)
    if out["json"] is None:
        return LlamaGuardVerdict(
            verdict="reject", categories=("error",), checkpoint=LLAMA_GUARD_CHECKPOINT,
            taxonomy=LLAMA_GUARD_TAXONOMY, latency_ms=out["latency_ms"],
            raw={"error": out.get("error", "unknown")},
        )
    j = out["json"]
    # Llama Guard returns a list of flagged categories (or [] for safe). The
    # exact JSON shape depends on the version — pin the response contract and
    # cite the model card.
    flagged = tuple(j.get("categories", []))
    blocking = tuple(c for c in flagged if scope.get(c) == "block")
    return LlamaGuardVerdict(
        verdict="reject" if blocking else "safe",
        categories=flagged,
        checkpoint=LLAMA_GUARD_CHECKPOINT,
        taxonomy=LLAMA_GUARD_TAXONOMY,
        latency_ms=out["latency_ms"],
        raw=j,
    )


def llama_guard_action_input(text: str) -> dict[str, Any]:
    v = classify(text, TAXONOMY_SCOPE_INPUT)
    # Trace emission happens in the NeMo GR action wrapper so request_id is
    # in scope; see the actions.py file in the rails package.
    return {
        "verdict": v.verdict, "categories": list(v.categories),
        "checkpoint": v.checkpoint, "taxonomy": v.taxonomy,
        "latency_ms": v.latency_ms,
    }


def llama_guard_action_output(text: str) -> dict[str, Any]:
    v = classify(text, TAXONOMY_SCOPE_OUTPUT)
    return {
        "verdict": v.verdict, "categories": list(v.categories),
        "checkpoint": v.checkpoint, "taxonomy": v.taxonomy,
        "latency_ms": v.latency_ms,
    }
```

### 2.3 ShieldGemma action (`shieldgemma.py`)

Size choice: start at 2B or 9B. ShieldGemma at 27B is stronger but has a much larger latency and cost tail; step up only if the calibration slice forces it.

```python
# classifiers/shieldgemma.py
# ShieldGemma — self-hosted on GPU. Per-policy probability API.
# Model card: https://huggingface.co/google/shieldgemma-<size>-pin
# Technical report: https://arxiv.org/abs/2407.21772 (read before shipping)
# <!-- verify-citation: pin exact release version and the policy set at submission. -->

from __future__ import annotations

import time
from dataclasses import dataclass
from typing import Any

import httpx


SHIELDGEMMA_CHECKPOINT = "<google/shieldgemma-2b-pin>"     # pin
SHIELDGEMMA_POLICY_SET = "<policy-set-v1-pin>"             # pin
SHIELDGEMMA_RUNTIME = "<vllm-pin or tgi-pin or transformers-pin>"
SHIELDGEMMA_ENDPOINT = "http://shieldgemma.internal:8080/score"


# Thresholds tuned on the calibration slice. Per policy per slot.
# See tuning/thresholds.md for the sweep that produced these.
POLICY_THRESHOLDS: dict[str, dict[str, float]] = {
    "retrieval": {
        "injection_likely": 0.55,
        "unsafe":           0.60,
        "sensitive_info":   0.45,
    },
    "output": {
        "dangerous_content": 0.50,
        "harassment":        0.55,
        "hate_speech":       0.55,
        "sexually_explicit": 0.50,
    },
}


@dataclass(frozen=True)
class ShieldGemmaVerdict:
    verdict: str
    scores: dict[str, float]
    failed_policies: tuple[str, ...]
    checkpoint: str
    policy_set: str
    latency_ms: float


def _call(text: str, policies: list[str], timeout_s: float = 2.0) -> dict[str, Any]:
    t0 = time.perf_counter()
    try:
        r = httpx.post(SHIELDGEMMA_ENDPOINT, json={"text": text, "policies": policies}, timeout=timeout_s)
        r.raise_for_status()
        return {"json": r.json(), "latency_ms": (time.perf_counter() - t0) * 1000}
    except Exception as exc:
        return {"json": None, "latency_ms": (time.perf_counter() - t0) * 1000, "error": str(exc)}


def classify(text: str, slot: str) -> ShieldGemmaVerdict:
    thresholds = POLICY_THRESHOLDS[slot]
    out = _call(text, list(thresholds))
    if out["json"] is None:
        return ShieldGemmaVerdict(
            verdict="reject", scores={}, failed_policies=("error",),
            checkpoint=SHIELDGEMMA_CHECKPOINT, policy_set=SHIELDGEMMA_POLICY_SET,
            latency_ms=out["latency_ms"],
        )
    scores = {p: float(out["json"]["scores"].get(p, 0.0)) for p in thresholds}
    failed = tuple(p for p, s in scores.items() if s >= thresholds[p])
    return ShieldGemmaVerdict(
        verdict="reject" if failed else "safe",
        scores=scores, failed_policies=failed,
        checkpoint=SHIELDGEMMA_CHECKPOINT, policy_set=SHIELDGEMMA_POLICY_SET,
        latency_ms=out["latency_ms"],
    )
```

### 2.4 Wire-in to the NeMo GR actions (`rails/nemo/actions.py` diff)

The exercise-02 stubs are replaced:

```python
# actions.py — diff from exercise-02 stubs to exercise-03 real calls

from classifiers.llama_guard import (
    llama_guard_action_input, llama_guard_action_output,
    LLAMA_GUARD_CHECKPOINT, LLAMA_GUARD_TAXONOMY,
)
from classifiers.shieldgemma import classify as shieldgemma_classify

def llama_guard_action(text: str) -> dict:
    t0 = time.perf_counter()
    v = llama_guard_action_input(text)
    emit_trace(
        slot="input", rail="carepilot input filter", action="llama_guard_action",
        verdict=v["verdict"], latency_ms=(time.perf_counter() - t0) * 1000,
        config_version=CONFIG_VERSION,
        reason=",".join(v["categories"]) if v["categories"] else "no_category",
        extra={"checkpoint": v["checkpoint"], "taxonomy": v["taxonomy"],
               "categories": v["categories"]},
    )
    return v

def shieldgemma_action(chunk: str) -> dict:
    t0 = time.perf_counter()
    v = shieldgemma_classify(chunk, slot="retrieval")
    emit_trace(
        slot="retrieval", rail="carepilot retrieval filter", action="shieldgemma_action",
        verdict=v.verdict, latency_ms=(time.perf_counter() - t0) * 1000,
        config_version=CONFIG_VERSION,
        reason=",".join(v.failed_policies) if v.failed_policies else "no_policy_fail",
        extra={"checkpoint": v.checkpoint, "policy_set": v.policy_set,
               "scores": v.scores},
    )
    return {"verdict": v.verdict, "failed_policies": list(v.failed_policies), "scores": v.scores}
```

Trace records now carry classifier-specific fields: `categories` (Llama Guard), `scores` and `failed_policies` (ShieldGemma), plus the pinned `checkpoint`, `taxonomy`, `policy_set`. These are the fields mod-109 reads to compute per-classifier drift.

Slot 4 wires both classifiers in parallel through an `asyncio.gather` wrapper in the Guardrails AI guard call site — both must return `safe` for the response to pass. Chapter 03 §"Composing classifier + rule + retrieval":

```python
# rails/output_guards.py (excerpt — the Guardrails AI guard delegates here)

import asyncio

async def output_dual_classifier(text: str) -> dict:
    lg_task = asyncio.to_thread(llama_guard_action_output, text)
    sg_task = asyncio.to_thread(lambda: shieldgemma_classify(text, slot="output"))
    lg, sg = await asyncio.gather(lg_task, sg_task)
    verdict = "safe" if (lg["verdict"] == "safe" and sg.verdict == "safe") else "reject"
    return {
        "verdict": verdict,
        "llama_guard": {"verdict": lg["verdict"], "categories": lg["categories"],
                        "checkpoint": lg["checkpoint"], "taxonomy": lg["taxonomy"]},
        "shieldgemma": {"verdict": sg.verdict, "failed_policies": list(sg.failed_policies),
                        "scores": sg.scores, "checkpoint": sg.checkpoint,
                        "policy_set": sg.policy_set},
    }
```

### 2.5 Calibration slice (`tuning/slice.md`) and threshold sweep (`tuning/thresholds.md`)

The slice:

- **Source.** 320 items drawn from the mod-104 hosted adversarial-and-legitimate set, in categories relevant to CarePilot: direct prompt injection, indirect prompt injection, PII exfiltration shape, self-harm, violent-content, medical-misinformation.
- **Labelling.** Items carry mod-104's category label; two labellers reviewed every item; disagreements adjudicated by a third reviewer. Protocol captured in `tuning/slice.md` with the labelling-confidence breakdown.
- **Split.** Calibration only — exercise 05 runs on the full mod-104 set. The slice shares *items* with exercise 05's measurement set; the slice numbers are **not** the measured numbers because the slice is tiny and thresholds are tuned on it.

Sweep summary table shape (populate at run time):

| Classifier | Slot | Policy / scope | Thresholds tried | Chosen | FP on legit | FN on adversarial |
|---|---|---|---|---|---|---|
| Llama Guard | input | taxonomy scope (14 cats) | three scope variants | TAXONOMY_SCOPE_INPUT | `to-record-from-run` | `to-record-from-run` |
| Llama Guard | output | taxonomy scope (narrower) | two scope variants | TAXONOMY_SCOPE_OUTPUT | `to-record-from-run` | `to-record-from-run` |
| ShieldGemma | retrieval | injection_likely | 0.3, 0.4, 0.5, 0.6, 0.7 | 0.55 | `to-record-from-run` | `to-record-from-run` |
| ShieldGemma | retrieval | unsafe | 0.3–0.7 | 0.60 | `to-record-from-run` | `to-record-from-run` |
| ShieldGemma | retrieval | sensitive_info | 0.3–0.7 | 0.45 | `to-record-from-run` | `to-record-from-run` |
| ShieldGemma | output | dangerous_content | 0.3–0.7 | 0.50 | `to-record-from-run` | `to-record-from-run` |
| ShieldGemma | output | harassment | 0.3–0.7 | 0.55 | `to-record-from-run` | `to-record-from-run` |
| ShieldGemma | output | hate_speech | 0.3–0.7 | 0.55 | `to-record-from-run` | `to-record-from-run` |
| ShieldGemma | output | sexually_explicit | 0.3–0.7 | 0.50 | `to-record-from-run` | `to-record-from-run` |

ROC-style plot per ShieldGemma policy: include if the slice supports it honestly (at least ~30 labelled items per label in the per-policy denominator). If not, state that and defer to exercise 05.

### 2.6 Composition specification (`composition.md`)

For each slot stacking a classifier on a rule:

- **Slot 1 Input.** Order: signature denylist (rule) → Llama Guard (classifier). Rule-first short-circuits before the classifier spend on obvious patterns. Classifier covers novel / paraphrased payloads the regex misses. Disagreement: there is only one classifier at slot 1; the rule and the classifier are AND-combined (rule reject OR classifier reject → reject).
- **Slot 2 Retrieval.** Order per chunk: Presidio fast analyser (exercise 04 — rule) → ShieldGemma per-policy score (classifier). Presidio covers PII; ShieldGemma covers injection-likely / unsafe / sensitive_info semantic tags. Disagreement: any Presidio hit *or* any ShieldGemma policy failure drops the chunk.
- **Slot 3 Tool policy.** Pure rule; no classifier. Chapter 02's "when rule-based fits" is explicit.
- **Slot 4 Output.** Dual classifier in parallel: Llama Guard (safety taxonomy) and ShieldGemma (per-policy). Presidio output scan (exercise 04) as a Guardrails AI validator. Disagreement: all three must return `safe` for the response to pass; any reject rejects. Trace emits one record per classifier so mod-109 can alert per classifier on drift.
- **Slot 5 Refusal shaping.** No classifier; templated content. Confirms the exercise-01 design.

Why slot 4 is defence-in-depth and not two-thresholds-on-one-model (chapter 03 §"Composing classifier + rule + retrieval"):

- Llama Guard is Meta-trained on Meta's safety taxonomy; ShieldGemma is Google-trained on a different policy set; Presidio is pattern + NER. Three mechanisms, three training-data families. An attacker's payload that evades one has to independently evade the others — the probability floor is the product, not the max.

### 2.7 Latency budget check (`latency.md`)

Measured in the middleware (not server-side); chapter 05 §"Latency accounting" is the discipline. Fields per classifier per slot:

| Classifier | Slot | Mode | p50 (ms) | p95 (ms) | Budget (ms, exercise-01) | Status |
|---|---|---|---|---|---|---|
| Llama Guard | input | serial | `to-record-from-run` | `to-record-from-run` | per exercise-01 | pass / flag |
| Llama Guard | output | parallel (with ShieldGemma) | `to-record-from-run` | `to-record-from-run` | per exercise-01 | pass / flag |
| ShieldGemma | retrieval | per-chunk serial | `to-record-from-run` | `to-record-from-run` | per exercise-01 | pass / flag |
| ShieldGemma | output | parallel (with Llama Guard) | `to-record-from-run` | `to-record-from-run` | per exercise-01 | pass / flag |

Any slot's p95 above the exercise-01 budget is flagged for a treatment plan — options: swap to a smaller ShieldGemma size, speculative-cancel the second classifier when the first returns `reject`, or renegotiate the budget. The write-up names the treatment, not just the flag.

### 2.8 Cost budget check (`cost.md`)

Three lines per chapter 05 §"Cost accounting": compute (GPU-hour amortised across throughput), operational (log ingestion, on-call fraction), third-party licence (N/A for self-hosted Llama Guard + ShieldGemma). Per-slot per-day roll-up at CarePilot's production request volume. Table shape:

| Classifier | Slot | Mean $/call | Calls/day | Daily $/slot | Monthly $ |
|---|---|---|---|---|---|
| Llama Guard | input | `to-record-from-run` | `to-record-from-run` | `to-record-from-run` | `to-record-from-run` |
| Llama Guard | output | `to-record-from-run` | `to-record-from-run` | `to-record-from-run` | `to-record-from-run` |
| ShieldGemma | retrieval | `to-record-from-run` | per-chunk ×retr-k | `to-record-from-run` | `to-record-from-run` |
| ShieldGemma | output | `to-record-from-run` | `to-record-from-run` | `to-record-from-run` | `to-record-from-run` |

Honest amortisation: the GPU is **not** "free because it is already there." If the GPU is shared with the main model, the classifier cost is the fraction of GPU-hours it consumes; if dedicated, the whole GPU-hour cost is the classifier's.

### 2.9 Failure-mode probe report (`failure-modes.md`)

Four sections, one per chapter-03 anti-pattern:

- **Vendor threshold.** Confirmed `TAXONOMY_SCOPE_INPUT` and `TAXONOMY_SCOPE_OUTPUT` are tuned (not the vendor default), and every ShieldGemma threshold is a tuned value from `tuning/thresholds.md`. If a threshold cell in the sweep table still says `default`, that is a fail.
- **Category-taxonomy mismatch.** CarePilot's "medical misinformation" category is not in Llama Guard 3's S1–S14 taxonomy. The supplementary check: a custom ShieldGemma policy prompt (stretch goal) — until then, a rule-based keyword-plus-citation-requirement validator in the Guardrails AI output guard catches the obvious cases, and the gap is filed as open work for exercise 04 / 06.
- **Same-model stack.** Slot 4 uses Llama Guard + ShieldGemma + Presidio — three training-data families. The anti-pattern (Llama Guard at threshold 0.6 plus Llama Guard at threshold 0.8 "for defence in depth") is not shipped. The report names the three mechanisms and the parallel composition.
- **Missing FP report.** Exercise 05 produces the full FP report on the mod-104 set. The calibration numbers here are a preview; the report states that explicitly and points to the exercise-05 handoff.

### 2.10 Fail-closed proof

A fault-injection fixture forces the Llama Guard endpoint to return HTTP 500; another forces ShieldGemma to time out. The assertion is that the request path produces a refusal and the trace carries `verdict="reject"` with `reason` naming the error. The pytest file lives under `classifiers/tests/test_fail_closed.py`.

## 3. Validation steps

1. **All eight artefacts exist.** `llama_guard.py`, `shieldgemma.py`, wired-in `actions.py`, `tuning/slice.md`, `tuning/thresholds.md`, `composition.md`, `latency.md`, `cost.md`, `failure-modes.md` all present.
2. **Stubs gone.** `grep '# TODO exercise-03' rails/nemo/actions.py` returns no hits; the stubs have been replaced by real classifier calls.
3. **Pins in trace.** Every classifier trace record carries `checkpoint`, `taxonomy` (Llama Guard) or `policy_set` (ShieldGemma), and the threshold / scope in force.
4. **Threshold sweep documented.** `thresholds.md` has at least one sweep per policy per slot and names the chosen threshold.
5. **Slot 4 uses two independent classifiers.** Reviewer greps for `llama_guard` and `shieldgemma` in the output guard call site and confirms both must-pass logic is in place.
6. **Parallel at slot 4.** `output_dual_classifier` uses `asyncio.gather`; latency at slot 4 is parallel-mode, not serial-mode.
7. **Latency numbers from middleware.** `latency.md` states p50 / p95 were measured on the request path, not on the classifier server. Sanity-check row compares the two to show network overhead.
8. **Cost roll-up present.** Mean $/call and per-day roll-up at production volume are populated.
9. **Fail-closed test green.** `pytest classifiers/tests/test_fail_closed.py` returns 0; the trace file shows the refusal path.
10. **No OpenAI Moderation on raw booleans.** If OpenAI Moderation is used at all (CarePilot does not, due to HIPAA), it is used on raw probabilities, not booleans. The failure-mode probe states "not used; vendor egress on PHI disqualifies."
11. **No fabricated Constitutional Classifier.** `composition.md` does not conjure a `constitutional_classifier.py`; the approach is named as a treatment plan and the Anthropic paper is cited.

## 4. Rubric — review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Llama Guard and ShieldGemma wired in | Real calls replace exercise-02 stubs; grep returns no `TODO exercise-03` | Stubs still present |
| R2 | Model checkpoints and taxonomies pinned | `checkpoint`, `taxonomy`, `policy_set` in every trace record | Unpinned |
| R3 | Threshold sweep documented | Sweep table per policy per slot; chosen threshold with FP / FN on calibration slice | Vendor default used |
| R4 | Composition is defence-in-depth | Slot 4 uses two different models (Llama Guard + ShieldGemma) plus Presidio; `composition.md` names the three mechanisms | Two-thresholds-on-one-model |
| R5 | Latency measured in middleware | `latency.md` states measurement point is the request path; parallel-mode numbers at slot 4 | Server-side numbers or missing parallel measurement |
| R6 | Cost roll-up present | Mean $/call, calls/day, monthly $ per classifier per slot | Only compute line reported or missing |
| R7 | Fail-closed proved | Fault-injection test passes; trace shows reject on error | Fail-open on classifier error |
| R8 | No Constitutional Classifier fabrication | Named as an approach if bespoke policy is a roadmap item; no `constitutional_classifier.py` conjured | Fabricated library |
| R9 | OpenAI Moderation discipline | If used, raw probabilities; if not used, reason stated (HIPAA for CarePilot) | Booleans used without calibration |
| R10 | Register handoff | Measured numbers deferred to exercise 05; calibration numbers called out as preview | Residual credit claimed on calibration slice |

Scoring bands: *Pass* — R1–R9 green, R10 honoured; *Pass with revisions* — R6 or R10 partial; *Fail* — any of R3 / R4 / R7 / R8 fails.

## 5. Common mistakes

- **Shipping with the vendor default threshold.** The default is calibrated for the vendor's risk appetite, not CarePilot's. Chapter 03 §"Anti-patterns" is explicit; the sweep table proves it was tuned.
- **Two-thresholds-on-one-model at slot 4.** "Llama Guard at threshold 0.6 plus Llama Guard at threshold 0.8" is one control. The attacker only has to defeat one model. Use two models with different training data — or one classifier plus one strong rule.
- **Running output classifiers sequentially.** When both must pass, running them in parallel cuts p95 roughly in half. Chapter 05 §"Latency accounting" is the discipline; `asyncio.gather` is enough.
- **Measuring classifier latency on the server.** Server-side timing misses network, serialisation, and dispatch overhead. The middleware stamp is what the user experiences.
- **"GPU-hour is free because the GPU is already there."** It is not free — amortise honestly. Chapter 06's build-vs-buy TCO analysis will bite in exercise 06 if this step is skipped.
- **Fabricating a Constitutional Classifier body.** Anthropic's Constitutional Classifiers paper is an approach, not a drop-in library. If the policy is bespoke and off-the-shelf taxonomies do not fit, name the approach as a treatment plan; do not conjure a product.
- **Letting Llama Guard model updates land silently.** A `pip install --upgrade` of the serving runtime, or a new model-card revision, changes the taxonomy. Pin the checkpoint in `requirements.txt` and in the trace; the mod-109 monitor detects drift when either moves.
- **Missing a taxonomy-mismatch supplementary check.** Llama Guard 3's taxonomy does not have "medical misinformation" or "clinical-advice outside-licensed-source" categories. If the harm model has such categories, name a supplementary check (ShieldGemma custom policy, rule, Presidio recogniser) or file the gap as open work.
- **Overfitting to the calibration slice.** 200–500 labelled items is scaffolding, not evidence. A threshold that minimises FN to zero on the slice almost certainly overfits; exercise 05's full measurement will overturn it.
- **Treating "classifier error rate 0%" as the operating point.** Real classifier endpoints error at some rate. The fail-closed-on-error discipline inflates FN if not surfaced; the trace carries the error rate so mod-109 and exercise 05 see it.

## 6. References

- **NIST AI Risk Management Framework 1.0.** `https://www.nist.gov/itl/ai-risk-management-framework`. The threshold-tuning discipline and the fail-closed-on-classifier-error discipline both feed MEASURE 2.7 (security and resilience) and MANAGE 2.2 (sustained controls).
- **OWASP Top 10 for Large Language Model Applications.** `https://genai.owasp.org/llm-top-10/`. The classifier layer at slot 1 addresses LLM01 (direct prompt injection); slot 2 addresses LLM01 (indirect) and the sensitive-information-disclosure entry (2025 rename of 2023's LLM06); slot 4 addresses the dual coverage at output. <!-- verify-citation: pin 2025 category codes at submission. -->
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/`. Classifier-side degradation under distribution shift is a cross-track handshake with mod-105 and maps to the ML Top 10's data-poisoning / adversarial-example entries. <!-- verify-citation: pin exact ML Top 10 entry codes. -->
- **MITRE ATLAS.** `https://atlas.mitre.org/`. The classifier compositions here target evasion techniques listed in the ATLAS matrix. <!-- verify-citation: pin tactic / technique IDs. -->
- **Meta Llama Guard model card.** Pin the exact release (Llama Guard 3 or 4) and taxonomy at submission; link to the Hugging Face model card page. <!-- verify-citation: confirm model name, release date, taxonomy revision. -->
- **Google ShieldGemma technical report.** `https://arxiv.org/abs/2407.21772`. Pin the size (2B / 9B / 27B) and the policy set at submission.
- **Anthropic Constitutional Classifiers.** Named as an *approach*, not a drop-in library. The paper is the reference; do not conjure a product. <!-- verify-citation: cite the paper's arXiv ID at submission. -->
- **Module running system.** CarePilot — hospital triage-support RAG assistant, introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`.
- **Exercise 01 — slot design** and **exercise 02 — rule / flow layer** — this exercise plugs classifier bodies into the stubs shipped there.
- **Exercise prompt.** `lessons/mod-108-guardrail-engineering-and-effectiveness/exercises/exercise-03-classifier-guardrails-llama-guard-and-shieldgemma.md`.

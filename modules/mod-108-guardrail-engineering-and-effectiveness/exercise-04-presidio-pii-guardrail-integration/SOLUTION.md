# Exercise 04 — Microsoft Presidio Integration Across Ingest, Retrieval, Output (CarePilot) — Reference Solution

## 1. Solution overview

This exercise integrates **Microsoft Presidio** into CarePilot's exercise-01 slot set at the three chapter-04 paths — **ingest**, **retrieval**, **output** — and plugs each integration into the slot the architecture designated (retrieval into slot 2, output into slot 4's Guardrails AI guard, ingest into the corpus-build pipeline upstream of the RAG store).

The mod-107 exercise-05 solution already stood up a Presidio pipeline for *privacy-risk* purposes (the three-hooks pipeline over the CarePilot corpus). Chapter 04 §"Cross-references" is explicit that this exercise *integrates* that pipeline into the **guardrail-slot architecture** — do not fork; extend. Specifically:

- Mod-107 exercise 05 produced `src/pipeline.py` with `ingest_redact`, `retrieval_redact`, `output_redact` hooks, custom recognisers (`CAREPILOT_MRN`), a decision log, and a per-entity recall / precision evaluation on a synthetic test corpus.
- This exercise wraps those hooks into guardrail-slot entry points: `presidio/ingest.py` writes anonymised records to the corpus and a detection log; `presidio/retrieval.py` is wired into the NeMo GR retrieval rail from exercise 02 (alongside ShieldGemma) and applies per-caller entitlement; `presidio/output.py` is wired into the Guardrails AI guard from exercise 02 as a per-field validator.

Four decisions drive the shape:

- **Three paths, three thresholds, three logs.** Ingest runs the full analyser (heavy NER, offline, latency not a concern). Retrieval runs a fast analyser (regex + context, no heavy NER) because it sits in the request path. Output runs the full analyser scoped per field (`answer` field gets a wide net, `citations` field gets a URL-shape check + domain allow-list only).
- **Custom recogniser calibrated against real data.** CarePilot's MRN pattern (`MRN-\d{8}`) is the required custom recogniser; a 20-positive / 20-negative labelled test set drawn from the corpus-shape (not invented) calibrates `score`.
- **Per-turn whitelist, not global.** The output path's whitelist of allowed PII is derived per-turn from (a) PII the caller supplied in their own input and (b) PII present in the authorised retrieved context. The whitelist does not persist across turns. Mod-107 exercise 05 §"Output-side whitelist logic is subtle" is the reference.
- **"PII already in the store" audit, not full remediation.** Chapter 04 §"The 'PII already in the RAG store' mitigation" describes the systemic mitigation. This exercise ships the **audit report**, the **prioritised remediation queue**, and the **interim hardened retrieval-side threshold**. Actual remediation of the whole store is a production project outside this exercise's scope.

Deliverables live under `risk/mod-108/carepilot/presidio/` and are indexed from `mod-108-ex04-presidio.md`.

## 2. Worked implementation

### 2.1 Repository layout

```
risk/mod-108/carepilot/presidio/
├── ingest.py                          # corpus-build-time anonymisation + detection log
├── retrieval.py                       # fast analyser in the retrieval rail
├── output.py                          # Guardrails AI validator; per-field policy
├── custom_recognizers/
│   ├── mrn.py                         # CAREPILOT_MRN PatternRecognizer (reuses mod-107 ex05 pattern)
│   └── mrn_tests.json                 # 20 positives + 20 negatives from corpus-shape data
├── thresholds.md                      # per-recogniser per-path threshold with reason
├── store-audit.md                     # full-scan audit + prioritised queue + interim threshold
├── logs/
│   ├── detection-log.jsonl            # ingest path
│   ├── chunk-redaction-log.jsonl      # retrieval path
│   └── response-redaction-log.jsonl   # output path
└── anti-patterns.md                   # four chapter-04 anti-pattern probes
```

### 2.2 Ingest pipeline (`presidio/ingest.py`)

Runs the full analyser including heavy NER; latency is not a concern because ingest runs offline. Writes anonymised records to the index-ready store and a `detection-log.jsonl` (counts, types, offsets; no raw PII in the log — mod-107 exercise 05 discipline carried forward). A record whose detection score on any entity exceeds the "reject" threshold is routed to `review-queue.jsonl` instead of the index.

```python
# presidio/ingest.py
# Ingest-path Presidio pipeline. Offline; the full recogniser set runs.
# Anonymises with placeholder-preserving strategy: <PERSON_1>, <EMAIL_1>, etc.
# Chapter 04 §"Ingest path" is the reference.
#
# Reuses mod-107 exercise-05 `src/pipeline.py` for the shared Presidio config.

from __future__ import annotations

import json
import time
from pathlib import Path

from presidio_anonymizer.entities import OperatorConfig

from mod107.src.pipeline import PiiPipeline, build_analyzer, Hook       # reuse mod-107 ex05
from mod107.src.recognizers import mrn_recognizer
from mod107.src.decision_log import append as append_decision_log


INGEST_THRESHOLDS = {
    "PERSON": 0.55, "EMAIL_ADDRESS": 0.40, "PHONE_NUMBER": 0.40,
    "URL": 0.45, "IP_ADDRESS": 0.50, "CREDIT_CARD": 0.50, "IBAN_CODE": 0.50,
    "CAREPILOT_MRN": 0.85,       # calibrated on mrn_tests.json
}
INGEST_REJECT_THRESHOLDS = {
    "CREDIT_CARD": 0.80,         # hard-reject records with credit cards (shouldn't be in clinical corpus)
    "IBAN_CODE":   0.80,
}


def ingest_corpus(records_path: Path, out_path: Path, review_queue_path: Path,
                  detection_log_path: Path, pipeline: PiiPipeline) -> dict:
    counts: dict[str, int] = {}
    n_total = n_indexed = n_rejected = 0
    with records_path.open() as f_in, out_path.open("w") as f_out, \
         review_queue_path.open("w") as f_rev:
        for line in f_in:
            rec = json.loads(line)
            n_total += 1
            result = pipeline.redact(rec["text"], hook="ingest")

            decisions_payload = [d.__dict__ for d in result.decisions]
            event_id = append_decision_log(
                detection_log_path, "ingest", rec["text"], decisions_payload)

            must_reject = any(
                d.applied and d.entity_type in INGEST_REJECT_THRESHOLDS
                and d.score >= INGEST_REJECT_THRESHOLDS[d.entity_type]
                for d in result.decisions
            )
            if must_reject:
                n_rejected += 1
                f_rev.write(json.dumps({"id": rec["id"], "reason": "reject_threshold",
                                        "event_id": event_id}) + "\n")
                continue

            rec["text"] = result.text
            rec["pii"] = {
                "event_id": event_id,
                "counts": {d.entity_type: 1 for d in result.decisions if d.applied},
            }
            for d in result.decisions:
                if d.applied:
                    counts[d.entity_type] = counts.get(d.entity_type, 0) + 1
            f_out.write(json.dumps(rec) + "\n")
            n_indexed += 1

    return {"n_total": n_total, "n_indexed": n_indexed, "n_rejected": n_rejected,
            "counts": counts, "ingest_thresholds": INGEST_THRESHOLDS}
```

The anonymisation strategy is placeholder-preserving (`<PERSON_1>`, `<EMAIL_1>`) because downstream retrieval still needs to anchor on entity position / count; a destructive redaction (`[REDACTED]`) breaks per-entity reasoning in the retrieval-side ShieldGemma pass.

### 2.3 Retrieval pipeline (`presidio/retrieval.py`)

Runs the **fast** configuration (regex + context + custom recogniser; **no** heavy NER) because latency is a first-class concern. Wraps CarePilot's retriever (a LangChain retriever or the NeMo GR retrieval action from exercise 02). Applies per-caller entitlement: the caller is entitled to see `{PERSON, EMAIL_ADDRESS}` for their own tickets but not `{CAREPILOT_MRN, CREDIT_CARD}` — the RBAC stub below is the hook point for CarePilot's real ACL layer.

```python
# presidio/retrieval.py
# Retrieval-path Presidio pipeline. Fast analyser only; measured p50 / p95 goes
# into the latency budget check. Applies per-caller entitlement.

from __future__ import annotations

import json
import time
from pathlib import Path
from typing import Any

from presidio_analyzer import AnalyzerEngine, RecognizerRegistry
from presidio_anonymizer import AnonymizerEngine
from presidio_anonymizer.entities import OperatorConfig

from mod107.src.recognizers import mrn_recognizer


def build_fast_analyzer() -> AnalyzerEngine:
    # Fast configuration: no spaCy NER model. Only regex-based recognizers and
    # custom patterns. CAREPILOT_MRN + PHONE_NUMBER + EMAIL_ADDRESS + IP_ADDRESS
    # + CREDIT_CARD + IBAN. PERSON NER lives on the ingest path only.
    registry = RecognizerRegistry()
    registry.load_predefined_recognizers()
    registry.add_recognizer(mrn_recognizer())
    # Drop the NER-backed recognizers for speed.
    registry.remove_recognizer("SpacyRecognizer")
    return AnalyzerEngine(registry=registry, nlp_engine=None)


RETRIEVAL_THRESHOLDS = {
    "PHONE_NUMBER": 0.50, "EMAIL_ADDRESS": 0.50, "URL": 0.50,
    "IP_ADDRESS": 0.60, "CREDIT_CARD": 0.70, "IBAN_CODE": 0.70,
    "CAREPILOT_MRN": 0.85,
}


def entitled_types(caller: str) -> set[str]:
    # RBAC stub. Production CarePilot wires this to the hospital ACL service.
    # Default: callers may see PHONE / EMAIL for the patients they own; never
    # MRN, CREDIT_CARD, IBAN_CODE.
    base = {"PHONE_NUMBER", "EMAIL_ADDRESS"}
    if caller.startswith("admin:"):
        base |= {"CAREPILOT_MRN"}
    return base


def redact_chunk(text: str, caller: str, analyzer: AnalyzerEngine, anonymizer: AnonymizerEngine,
                 log_path: Path, request_id: str) -> tuple[str, list[dict]]:
    t0 = time.perf_counter()
    results = analyzer.analyze(text=text, entities=list(RETRIEVAL_THRESHOLDS), language="en")
    allowed = entitled_types(caller)
    actionable = [r for r in results
                  if r.score >= RETRIEVAL_THRESHOLDS.get(r.entity_type, 1.0)
                  and r.entity_type not in allowed]
    operators = {e: OperatorConfig("replace", {"new_value": f"<{e}>"})
                 for e in {r.entity_type for r in actionable}}
    anon = anonymizer.anonymize(text=text, analyzer_results=actionable, operators=operators)
    latency_ms = (time.perf_counter() - t0) * 1000

    decisions = [{"entity_type": r.entity_type, "start": r.start, "end": r.end,
                  "score": r.score, "applied": r in actionable,
                  "caller_entitled": r.entity_type in allowed}
                 for r in results]
    with log_path.open("a") as f:
        f.write(json.dumps({
            "request_id": request_id, "slot": "retrieval", "action": "presidio_retrieval",
            "caller": caller, "decisions": decisions, "latency_ms": latency_ms,
        }) + "\n")
    return anon.text, decisions
```

The retrieval wrapper is the entry point NeMo GR's retrieval action calls alongside `shieldgemma_action` (exercise 03). Chunks are first passed through Presidio (PII) then ShieldGemma (semantic policy); either reject drops the chunk from the assembly.

### 2.4 Output pipeline (`presidio/output.py`) — Guardrails AI validator

CarePilot's structured response (`answer`, `citations`) means per-field policy is natural. The output validator is registered against the Guardrails AI guard from exercise 02.

```python
# presidio/output.py
# Guardrails AI validator that runs Presidio against the model's answer field.
# Whitelists entities the caller supplied in their input turn and entities
# that appear in the authorised retrieved context.

from __future__ import annotations

from typing import Any

from guardrails.validator_base import register_validator, Validator, FailResult, PassResult
from presidio_analyzer import AnalyzerEngine
from presidio_anonymizer import AnonymizerEngine
from presidio_anonymizer.entities import OperatorConfig


OUTPUT_THRESHOLDS = {
    "PERSON": 0.55, "EMAIL_ADDRESS": 0.40, "PHONE_NUMBER": 0.40,
    "URL": 0.50, "IP_ADDRESS": 0.60, "CREDIT_CARD": 0.70, "IBAN_CODE": 0.70,
    "CAREPILOT_MRN": 0.85,
}


@register_validator(name="carepilot/presidio-output", data_type=["string"])
class PresidioOutput(Validator):
    """Reject the answer string if it contains PII the caller is not entitled to see."""

    def __init__(self, analyzer: AnalyzerEngine, anonymizer: AnonymizerEngine,
                 on_fail: str | None = None):
        super().__init__(on_fail=on_fail)
        self._analyzer = analyzer
        self._anonymizer = anonymizer

    def validate(self, value, metadata):
        # metadata is per-turn: caller id, whitelisted entities derived from the
        # caller's input and the authorised retrieved context.
        whitelist: set[str] = set(metadata.get("pii_whitelist_values", set()))
        entitled_types: set[str] = set(metadata.get("entitled_types", set()))
        results = self._analyzer.analyze(
            text=value, entities=list(OUTPUT_THRESHOLDS), language="en")
        bad = [r for r in results
               if r.score >= OUTPUT_THRESHOLDS.get(r.entity_type, 1.0)
               and r.entity_type not in entitled_types
               and value[r.start:r.end] not in whitelist]
        if not bad:
            return PassResult()
        ops = {e: OperatorConfig("replace", {"new_value": f"<{e}>"})
               for e in {r.entity_type for r in bad}}
        fixed = self._anonymizer.anonymize(
            text=value, analyzer_results=bad, operators=ops).text
        return FailResult(
            error_message=f"answer contains {len(bad)} un-entitled PII spans",
            fix_value=fixed,
        )
```

Guard construction (diff from exercise 02):

```python
# rails/guardrails_ai/guard.py — exercise-04 additions

from presidio.output import PresidioOutput
from presidio.retrieval import build_fast_analyzer       # for a shared analyzer; or build_analyzer

def build_guard_v2() -> Guard:
    guard = build_guard()          # exercise 02 guard
    guard.use(PresidioOutput(analyzer=..., anonymizer=..., on_fail=OnFailAction.FIX_REASK),
              on="answer")
    return guard
```

`FIX_REASK` is the `on_fail` policy: Presidio returns a fixed value (the answer with PII replaced), and Guardrails AI re-prompts the model with the fix if the application wants to; otherwise `EXCEPTION` is used to produce a slot-5 refusal directly. The choice is declared per deployment.

### 2.5 Custom recogniser (`custom_recognizers/mrn.py`) and test set

`mrn.py` reuses the mod-107 exercise-05 pattern recogniser (do not fork). The test set is 20 positives drawn from CarePilot's corpus-shape data and 20 negatives that *look* like MRN but should not match (invoice IDs starting with `MRN`, telephone extensions, etc.). Chapter 04 §"Custom recognisers" is explicit: invented examples miss real-data false positives.

```json
// custom_recognizers/mrn_tests.json (excerpt)
[
  {"text": "Patient MRN-12345678 presents with...", "expect": true,  "source": "corpus-shape-paraphrase"},
  {"text": "See invoice MRN-INV-0001 attached",     "expect": false, "source": "corpus-shape-paraphrase"},
  {"text": "Call MRN-ext at 555-1234",               "expect": false, "source": "corpus-shape-paraphrase"}
]
```

Calibration script (`tests/test_mrn_recognizer.py`):

```python
import json, pytest
from mod107.src.recognizers import mrn_recognizer
from presidio_analyzer import AnalyzerEngine

def test_mrn_precision_recall_above_90():
    ae = AnalyzerEngine(registry=None)
    ae.registry.add_recognizer(mrn_recognizer())
    tests = json.load(open("custom_recognizers/mrn_tests.json"))
    tp = fp = tn = fn = 0
    for t in tests:
        hits = ae.analyze(text=t["text"], entities=["CAREPILOT_MRN"], language="en")
        got = bool(hits)
        if t["expect"] and got: tp += 1
        elif t["expect"] and not got: fn += 1
        elif not t["expect"] and got: fp += 1
        else: tn += 1
    precision = tp / max(tp + fp, 1)
    recall = tp / max(tp + fn, 1)
    assert precision >= 0.90
    assert recall >= 0.90
```

The threshold on `mrn_recognizer()` is calibrated — not hard-coded to `score=1.0`. The sweep lives in `thresholds.md`; the chosen `score` sits at 0.85 for the CarePilot pattern.

### 2.6 Threshold-per-path config (`thresholds.md`)

Summary table (populate numbers at run time):

| Recogniser | Ingest threshold | Retrieval threshold | Output threshold | Reason |
|---|---|---|---|---|
| PERSON | 0.55 | — (no NER at retrieval) | 0.55 | Chapter 04 §"Common recogniser tuning"; PERSON is noisy in the clinical corpus (drug brand names) |
| EMAIL_ADDRESS | 0.40 | 0.50 | 0.40 | Regex is strong; lower at ingest for recall |
| PHONE_NUMBER | 0.40 | 0.50 | 0.40 | Same |
| CREDIT_CARD | 0.50 (reject ≥0.80) | 0.70 | 0.70 | Clinical corpus should not contain cards; reject at ingest |
| IBAN_CODE | 0.50 (reject ≥0.80) | 0.70 | 0.70 | Same |
| CAREPILOT_MRN | 0.85 | 0.85 | 0.85 | Calibrated on `mrn_tests.json`; high score because the pattern is specific |

### 2.7 Store audit (`store-audit.md`)

Full-scan report, prioritised remediation queue, interim-hardening threshold. Shape:

```
Store audit — CarePilot RAG store (slice) — <run_id>
---------------------------------------------------
Total records scanned: <n>
Records with detected PII:
  PERSON:        <count> records (<pct>% of total)
  EMAIL_ADDRESS: <count>
  PHONE_NUMBER:  <count>
  CAREPILOT_MRN: <count>
  CREDIT_CARD:   <count>   # expected 0 — any hits are prioritised
Retrieval-frequency bands (if tracked):
  Top-10% most-retrieved documents with PII: <list of doc IDs or counts>
Prioritised remediation queue:
  1. <doc_id>  (reason: top-5% retrieval frequency; contains PERSON + CAREPILOT_MRN)
  2. <doc_id>  (reason: contains CREDIT_CARD — should not be in store)
  ...
Interim hardened retrieval threshold:
  CAREPILOT_MRN: raise from 0.85 → 0.60 (over-trigger, under-allow) until the
  prioritised queue is remediated. Logged in the retrieval-path trace.
Cross-reference:
  mod-107 exercise-05 risk-register row: risks[pii-retrieval]
  Mitigation pattern: chapter 04 §"The 'PII already in the RAG store' mitigation"
```

The audit is **not** a remediation — it is the artefact that proves the pattern can be executed. Full remediation of the store is a production project.

### 2.8 Trace format update

Exercise-02 trace records are extended with Presidio-specific fields:

```json
{
  "request_id": "...",
  "slot": "ingest | retrieval | output",
  "action": "presidio_ingest | presidio_retrieval | presidio_output",
  "config_version": "...",
  "entity_type": "CAREPILOT_MRN",
  "recogniser": "carepilot/mrn",
  "confidence": 0.92,
  "action_taken": "replace | hash | reject | pass",
  "caller_entitled_types": ["PHONE_NUMBER", "EMAIL_ADDRESS"]
}
```

Update to `rails/traces/README.md` adds the fields. Mod-109 reads the three log files (`detection-log.jsonl`, `chunk-redaction-log.jsonl`, `response-redaction-log.jsonl`) and alerts on per-entity-type drift; the schema decision made here is what the monitor sees.

### 2.9 Anti-pattern probes (`anti-patterns.md`)

- **Single-path Presidio.** All three paths integrated. If the ingest pipeline is legacy and cannot be modified, the store-audit + interim-hardening pattern (chapter 04) is the mitigation; the write-up names this explicitly.
- **Uniform threshold.** The threshold table in §2.6 shows per-path differentiation (EMAIL_ADDRESS at 0.40 ingest vs. 0.50 retrieval; CREDIT_CARD reject at 0.80 ingest only). The probe greps for "ingest" and "retrieval" and "output" and confirms each has its own threshold values.
- **Unfixtured tuning.** Every custom recogniser has a labelled test set (`mrn_tests.json` — 20 + 20); `tests/test_mrn_recognizer.py` asserts precision and recall above the floor claimed in §2.6. Any recogniser without a test set is a fail.
- **Pseudonymisation without key management.** The ingest path's `hash` strategy for CAREPILOT_MRN (if used) uses a keyed HMAC with the key stored in CarePilot's secret manager — the key boundary is documented and lives outside Presidio, in the security track's KMS. A deterministic salt-free hash is a chapter-04 anti-pattern; the probe confirms the key is present and rotated.

### 2.10 Register and cross-exercise handoff

- `risks[pii-output]` and `risks[pii-retrieval]` (mod-107 exercise 05) are the rows this exercise's integration binds to. Residual credit still requires exercise 05 measurement on the mod-104 set.
- The three log files are mod-109's consumer surface.
- The CAREPILOT_MRN recogniser is the first custom recogniser; a second (jurisdiction-specific — e.g., an insurance claim ID format) is a stretch goal.
- Semantic PII (the customer's employer, an inferred condition) is **not** caught by Presidio's built-in or pattern recognisers; chapter 04 §"Motivation" and mod-107 exercise 05 both flag this. The stretch "supplementary semantic classifier" is the treatment plan; file as open work if not shipped.

## 3. Validation steps

1. **All eight artefacts exist.** `ingest.py`, `retrieval.py`, `output.py`, `custom_recognizers/mrn.py` + `mrn_tests.json`, `thresholds.md`, `store-audit.md`, log files, `anti-patterns.md` all present.
2. **Three paths wired.** Ingest writes to `index/`; retrieval is called from the NeMo GR retrieval action; output is registered as a Guardrails AI validator on the `answer` field.
3. **Retrieval latency measured.** `latency.md` (reuse exercise 03's) adds a row for the Presidio retrieval scan; p50 / p95 reported; any over-budget flagged.
4. **Custom recogniser test passes.** `pytest tests/test_mrn_recognizer.py` returns 0; precision and recall above the 90% floor claimed.
5. **Threshold per path, not uniform.** Reviewer greps `thresholds.md` for each entity type and confirms at least one entity type has different ingest / retrieval / output thresholds.
6. **Per-turn whitelist works.** Fixture: a caller input containing their own `patient@example.com`; the output validator does not redact that email string but does redact a different email the model confabulates.
7. **Store audit produces a prioritised queue.** `store-audit.md` has at least three rows in the queue with reason text.
8. **Trace schema updated.** `rails/traces/README.md` names the Presidio fields; grep for `entity_type`, `recogniser`, `caller_entitled_types` in the file.
9. **Pseudonymisation key externalised.** The hash strategy references a secret-manager key, not an inline string. The anti-pattern probe confirms.
10. **No raw PII in the log files.** Reviewer greps the three log files for literal PII patterns (e.g., `@gmail.com`); no hits.

## 4. Rubric — review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Three Presidio paths integrated | Ingest, retrieval, output pipelines all exist and are wired into the actual serving path | Any path missing or stubbed |
| R2 | Output path wired as Guardrails AI validator | `PresidioOutput` registered on the `answer` field | Output path runs outside the guard |
| R3 | Retrieval latency measured in middleware | p50 / p95 reported; over-budget flagged | Missing or server-side |
| R4 | Custom recogniser calibrated | `CAREPILOT_MRN` recogniser with 20 + 20 labelled test set; `score` calibrated (not 1.0) | Hard-coded score or no test set |
| R5 | Threshold per path | Each entity type carries at least one path-specific threshold | Uniform threshold across paths |
| R6 | Per-turn whitelist | Output validator uses per-turn metadata; whitelist does not persist across turns | Global whitelist |
| R7 | Store audit produced | `store-audit.md` with prioritised queue and interim-hardening threshold | Missing or no queue |
| R8 | Trace schema updated | `traces/README.md` names Presidio fields | Not updated |
| R9 | Key management externalised | Hash strategy uses a secret-manager key, not an inline value | Deterministic salt-free hash |
| R10 | No raw PII in logs | Decision log carries types / offsets / confidence only | Raw PII values in log |

Scoring bands: *Pass* — R1–R9 green, R10 honoured; *Pass with revisions* — R3 or R7 partial; *Fail* — any of R1 / R4 / R6 / R9 / R10 fails.

## 5. Common mistakes

- **One terminal Presidio pass at output.** PII that enters the vector store at ingest will leak through the retrieval path and through the output path. Three independent paths with three independent thresholds.
- **Heavy NER at retrieval.** The spaCy large model adds 20–100 ms per chunk to retrieval; it is unacceptable in the request path. Use the fast analyser (regex + context) at retrieval; the heavy NER belongs on ingest.
- **Uniform threshold across paths.** Presidio's default confidence threshold is often 0.5. The right value at ingest (recall-favouring, 0.4 for strong regex entities) is different from the right value at retrieval (precision-favouring to avoid over-redaction).
- **Invented test-set examples for the custom recogniser.** 20 positives that are all clean `MRN-\d{8}` strings prove the regex works; they do not prove it handles the real corpus's surprising false positives. Pull from corpus-shape data.
- **Hard-coded `score=1.0`.** A recogniser that cannot be tuned cannot be calibrated. Calibrate on the test set.
- **Global PII whitelist.** Caller A supplies `patient@example.com` in their turn; the whitelist persists across turns; caller B receives caller A's email because the system "already knows it is allowed." The whitelist is per-turn.
- **Per-turn whitelist built from unauthorised retrieved context.** The whitelist is derived from (a) the caller's own input and (b) the authorised retrieved context only. Unauthorised retrieved content that the retrieval filter missed does not create a whitelist entry.
- **Writing raw PII into the decision log.** The log becomes a PII store. Record type / offset / score / strategy; not the value.
- **Deterministic salt-free hash for pseudonymisation.** `hashlib.sha256(mrn.encode()).hexdigest()` is a dictionary attack away from the original. Use a keyed HMAC with the key in a secret manager, rotated per policy.
- **Treating the store audit as remediation.** The audit is the artefact. Full remediation is a production project; the interim hardened retrieval threshold runs while remediation is scheduled.
- **Forgetting semantic PII.** Presidio's built-in + pattern recognisers do not catch "the customer's employer" or "an inferred medical condition." Name the gap; file it as open work or ship a supplementary semantic classifier.

## 6. References

- **NIST AI Risk Management Framework 1.0.** `https://www.nist.gov/itl/ai-risk-management-framework`. The three-path integration maps to MEASURE 2.7 (security and resilience, PII-containment aspect) and MANAGE 2.2 (sustained controls). The audit pattern feeds MANAGE 4.1 (post-deployment monitoring).
- **OWASP Top 10 for Large Language Model Applications.** `https://genai.owasp.org/llm-top-10/`. Sensitive-information-disclosure entry (2025 renumbering) covers the output-side PII mitigation; the retrieval-side integration addresses LLM01 (indirect prompt injection via PII-bearing poisoned chunks). <!-- verify-citation: pin 2025 category codes at submission. -->
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/`. <!-- verify-citation: pin the ML Top 10 entry covering PII leakage / data-handling in ML pipelines. -->
- **MITRE ATLAS.** `https://atlas.mitre.org/`. The output-path and retrieval-path PII containment maps to the sensitive-information-disclosure technique(s). <!-- verify-citation: pin ATLAS IDs. -->
- **Microsoft Presidio.** Project documentation at `https://microsoft.github.io/presidio/`. APIs used: `AnalyzerEngine`, `AnonymizerEngine`, `RecognizerRegistry`, `PatternRecognizer`, `OperatorConfig`. Presidio ships a spaCy-NER-backed analyser and a declarative anonymiser.
- **mod-107 exercise 05 solution** — `modules/mod-107-privacy-risk-engineering/exercise-05-presidio-pii-pipeline-and-redaction/SOLUTION.md` — the shared Presidio config, the `CAREPILOT_MRN` recogniser, and the three-hooks pipeline are reused here.
- **Module running system.** CarePilot — hospital triage-support RAG assistant, introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`.
- **Exercise 01 — slot design** and **exercise 02 — rule / flow layer** — this exercise plugs into the Guardrails AI guard from exercise 02 and the retrieval rail from exercise 02.
- **Exercise prompt.** `lessons/mod-108-guardrail-engineering-and-effectiveness/exercises/exercise-04-presidio-pii-guardrail-integration.md`.

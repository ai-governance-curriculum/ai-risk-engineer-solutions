# Exercise 05 — Presidio PII Pipeline and Redaction Across Ingest, Retrieval, and Output — Reference Solution

## 1. Solution overview

Exercises 01–04 concentrated on *model-level* privacy risk (inference-time attacks and DP-SGD as the model-side mitigation). Exercise 05 is the complementary *pipeline-level* control: PII detection, redaction, and auditing at **three hinge points** of a RAG assistant — at **ingest** (corpus being loaded into the vector store), at **retrieval** (chunks returned by retrieval before they enter the model prompt), and at **output** (model output before it reaches the user). The reference implementation is built on **Microsoft Presidio**, a library documented at `https://microsoft.github.io/presidio/` that bundles a NER-backed PII analyser (`presidio-analyzer`) with a rule-driven anonymiser (`presidio-anonymizer`).

The deliverable:

- A runnable Python package that wraps Presidio into three callable hooks (`ingest_redact`, `retrieval_redact`, `output_redact`) with shared, declaratively-configured allow / deny rules. The hooks accept text and return `(redacted_text, decision_record)`, where the decision record is the audit trail.
- A **recognizer-pack configuration** covering the entities relevant to the running system (person names, phone numbers, email addresses, URLs, IP addresses, credit-card numbers, IBANs, and medical-record-number patterns).
- A **recall/precision evaluation** of the pipeline against a labelled synthetic test corpus. Every detection the pipeline emits is scored against the gold labels; the write-up reports **entity-level recall**, **entity-level precision**, and **F1**, plus a per-entity-type breakdown.
- A **decision log schema** that captures `(hook, entity_type, source_span, strategy, outcome)` for every redaction, with the event hash included in the risk-register row.
- A mapping into the CarePilot risk register that treats ingest, retrieval, and output as three *independent* controls — each with its own failure mode (ingest: PII enters the vector store; retrieval: a PII-bearing chunk is injected into the prompt; output: the model synthesises PII in its response).

Design decisions worth calling out:

- **Three hooks, not one.** A single terminal redactor at output cannot catch PII that has already been embedded into the vector store; a single ingest redactor cannot catch PII that the model generates by combining non-PII chunks. The hooks must be independent and the eval must score each separately.
- **Allow-list in addition to deny-list.** The pipeline carries a small allow-list for strings that look like PII but are not (e.g., the fictional CarePilot system ID, specific drug names that NER mis-tags as person names). The allow-list is version-pinned and auditable.
- **Synthetic test data only.** The labelled test corpus is a synthetic set modelled on the exercise context — no real patient records. The exercise is clear that the pipeline is evaluated on synthetic data and tuned for behaviour on that data; a production evaluation must be re-run on a governance-approved real-data sample, which is out of scope for this exercise.
- **Audit log is the first-class deliverable.** A redacted string with no audit trail is unverifiable. Every redaction decision becomes a row in an append-only log keyed by the event hash.

## 2. Worked implementation

### 2.1 Repository layout

```
exercise-05-presidio-pii-pipeline-and-redaction/
├── SOLUTION.md                    # this file
├── src/
│   ├── pipeline.py                # ingest / retrieval / output hooks sharing a single Presidio config
│   ├── recognizers.py             # custom recognizers (MRN pattern, system IDs) + allow-list
│   ├── decision_log.py            # append-only audit log
│   └── eval_pipeline.py           # per-entity recall / precision on the synthetic test set
├── configs/
│   ├── presidio.yaml              # entities, confidence thresholds, redaction strategies per entity
│   └── allowlist.yaml             # declaratively-approved strings that look like PII but are not
├── test_corpus/
│   ├── gold.jsonl                 # synthetic sentences with span-labelled PII
│   └── README.md                  # provenance and construction protocol
└── results/
    ├── eval_report.json
    ├── decision_log.sample.jsonl  # redacted (!!) 20-row sample for reviewer spot-check
    └── pipeline_run_trace.json
```

### 2.2 Shared Presidio setup (`src/pipeline.py`)

```python
# src/pipeline.py
# A thin wrapper around Presidio's AnalyzerEngine + AnonymizerEngine. The same
# configured analyser is used at all three hinge points; the redaction *strategy*
# per entity type is configurable (replace, hash, mask) per hook.
#
# Library docs: https://microsoft.github.io/presidio/
# - presidio-analyzer: entity recognition over text (bundles spaCy NER + regex recognizers).
# - presidio-anonymizer: configurable anonymisation operators (replace, redact, mask, hash).

from dataclasses import dataclass, asdict
import hashlib
import time
from typing import Literal

from presidio_analyzer import AnalyzerEngine, RecognizerRegistry, RecognizerResult
from presidio_analyzer.nlp_engine import NlpEngineProvider
from presidio_anonymizer import AnonymizerEngine
from presidio_anonymizer.entities import OperatorConfig


Hook = Literal["ingest", "retrieval", "output"]


@dataclass
class RedactionDecision:
    hook: Hook
    entity_type: str
    start: int
    end: int
    score: float
    strategy: str
    applied: bool
    allowlist_hit: bool


@dataclass
class RedactionResult:
    text: str
    decisions: list[RedactionDecision]


def build_analyzer(nlp_model_name: str = "en_core_web_lg") -> AnalyzerEngine:
    provider = NlpEngineProvider(
        nlp_configuration={
            "nlp_engine_name": "spacy",
            "models": [{"lang_code": "en", "model_name": nlp_model_name}],
        }
    )
    nlp_engine = provider.create_engine()
    registry = RecognizerRegistry()
    registry.load_predefined_recognizers(nlp_engine=nlp_engine)
    # Custom recognizers are attached here (e.g., MRN pattern); see recognizers.py
    return AnalyzerEngine(registry=registry, nlp_engine=nlp_engine)


class PiiPipeline:
    def __init__(
        self,
        analyzer: AnalyzerEngine,
        anonymizer: AnonymizerEngine,
        entities: list[str],
        min_scores: dict[str, float],
        strategies: dict[Hook, dict[str, OperatorConfig]],
        allowlist: set[str],
    ):
        self.analyzer = analyzer
        self.anonymizer = anonymizer
        self.entities = entities
        self.min_scores = min_scores
        self.strategies = strategies
        self.allowlist = allowlist

    def redact(self, text: str, hook: Hook) -> RedactionResult:
        results = self.analyzer.analyze(text=text, entities=self.entities, language="en")
        kept: list[RecognizerResult] = []
        decisions: list[RedactionDecision] = []
        for r in results:
            if r.score < self.min_scores.get(r.entity_type, 0.5):
                continue
            span_text = text[r.start : r.end]
            if span_text in self.allowlist:
                decisions.append(
                    RedactionDecision(
                        hook=hook,
                        entity_type=r.entity_type,
                        start=r.start,
                        end=r.end,
                        score=r.score,
                        strategy="allowlist",
                        applied=False,
                        allowlist_hit=True,
                    )
                )
                continue
            kept.append(r)
            decisions.append(
                RedactionDecision(
                    hook=hook,
                    entity_type=r.entity_type,
                    start=r.start,
                    end=r.end,
                    score=r.score,
                    strategy=self.strategies[hook][r.entity_type].operator_name
                    if r.entity_type in self.strategies[hook]
                    else "default",
                    applied=True,
                    allowlist_hit=False,
                )
            )
        if not kept:
            return RedactionResult(text=text, decisions=decisions)
        op = {e: self.strategies[hook].get(e, OperatorConfig("replace", {"new_value": f"<{e}>"})) for e in self.entities}
        anon = self.anonymizer.anonymize(text=text, analyzer_results=kept, operators=op)
        return RedactionResult(text=anon.text, decisions=decisions)
```

### 2.3 Custom recognizer for CarePilot's MRN pattern (`src/recognizers.py`)

```python
# src/recognizers.py
# Custom pattern recognizer for the CarePilot medical-record-number convention.
# The MRN pattern used here is the fictional CarePilot convention from the running
# system (an uppercase "MRN-" prefix followed by 8 digits). Any real pipeline must
# be fed the real convention of the deploying hospital.

from presidio_analyzer import PatternRecognizer, Pattern


def mrn_recognizer() -> PatternRecognizer:
    pattern = Pattern(name="carepilot-mrn", regex=r"\bMRN-\d{8}\b", score=0.85)
    return PatternRecognizer(
        supported_entity="CAREPILOT_MRN",
        patterns=[pattern],
        context=["MRN", "medical record number", "chart"],
    )
```

### 2.4 Decision log (`src/decision_log.py`)

```python
# src/decision_log.py
# Append-only JSONL log of every redaction decision made by the pipeline. Each row
# includes the event hash (which is what the risk register cites) and the hook
# that made the decision. Tokens of the redacted span are NOT written to the log;
# only the type, score, strategy, and offsets.

import hashlib
import json
import time
from pathlib import Path


def event_hash(hook: str, text: str, decisions_payload: list[dict]) -> str:
    h = hashlib.sha256()
    h.update(hook.encode("utf-8"))
    h.update(text.encode("utf-8"))
    h.update(json.dumps(decisions_payload, sort_keys=True).encode("utf-8"))
    return h.hexdigest()


def append(log_path: Path, hook: str, text: str, decisions_payload: list[dict]) -> str:
    eid = event_hash(hook, text, decisions_payload)
    row = {
        "event_id": eid,
        "hook": hook,
        "timestamp": int(time.time() * 1000),
        "decisions": decisions_payload,
    }
    with log_path.open("a", encoding="utf-8") as f:
        f.write(json.dumps(row) + "\n")
    return eid
```

### 2.5 Evaluation on the synthetic corpus (`src/eval_pipeline.py`)

```python
# src/eval_pipeline.py
# Per-entity-type evaluation. A gold-labelled sentence is a tuple:
#   {"text": "...", "spans": [{"entity_type": "PERSON", "start": 12, "end": 20}, ...]}
#
# For each sentence, we run the pipeline at each hook, compare the recogniser
# results to the gold spans (span overlap ≥ 50% of either span, same entity type),
# and compute per-entity precision / recall / F1 and an aggregate (micro + macro).

from dataclasses import dataclass
from collections import defaultdict
import json
from pathlib import Path


@dataclass
class EvalRow:
    entity_type: str
    tp: int
    fp: int
    fn: int

    @property
    def precision(self) -> float:
        d = self.tp + self.fp
        return self.tp / d if d else 0.0

    @property
    def recall(self) -> float:
        d = self.tp + self.fn
        return self.tp / d if d else 0.0

    @property
    def f1(self) -> float:
        p, r = self.precision, self.recall
        return 2 * p * r / (p + r) if (p + r) else 0.0


def _span_overlap(a, b) -> bool:
    if a["entity_type"] != b["entity_type"]:
        return False
    inter = max(0, min(a["end"], b["end"]) - max(a["start"], b["start"]))
    a_len = max(1, a["end"] - a["start"])
    b_len = max(1, b["end"] - b["start"])
    return inter / a_len >= 0.5 or inter / b_len >= 0.5


def evaluate(pipeline, gold_path: Path, hook: str) -> dict:
    rows = defaultdict(lambda: EvalRow(entity_type="", tp=0, fp=0, fn=0))
    with gold_path.open() as f:
        for line in f:
            rec = json.loads(line)
            result = pipeline.redact(rec["text"], hook)
            pred = [
                {"entity_type": d.entity_type, "start": d.start, "end": d.end}
                for d in result.decisions
                if d.applied
            ]
            matched_gold = [False] * len(rec["spans"])
            for p in pred:
                hit = False
                for i, g in enumerate(rec["spans"]):
                    if matched_gold[i]:
                        continue
                    if _span_overlap(p, g):
                        rows[p["entity_type"]].tp += 1
                        matched_gold[i] = True
                        hit = True
                        break
                if not hit:
                    rows[p["entity_type"]].fp += 1
            for i, g in enumerate(rec["spans"]):
                if not matched_gold[i]:
                    rows[g["entity_type"]].fn += 1
    per_entity = {k: {"precision": r.precision, "recall": r.recall, "f1": r.f1, "tp": r.tp, "fp": r.fp, "fn": r.fn}
                  for k, r in rows.items()}
    micro_tp = sum(r.tp for r in rows.values())
    micro_fp = sum(r.fp for r in rows.values())
    micro_fn = sum(r.fn for r in rows.values())
    micro_p = micro_tp / (micro_tp + micro_fp) if (micro_tp + micro_fp) else 0.0
    micro_r = micro_tp / (micro_tp + micro_fn) if (micro_tp + micro_fn) else 0.0
    micro_f1 = 2 * micro_p * micro_r / (micro_p + micro_r) if (micro_p + micro_r) else 0.0
    return {
        "hook": hook,
        "per_entity": per_entity,
        "micro": {"precision": micro_p, "recall": micro_r, "f1": micro_f1},
    }
```

### 2.6 Running the eval — `configs/presidio.yaml`

```yaml
# configs/presidio.yaml
nlp:
  engine: spacy
  model: en_core_web_lg
entities:
  - PERSON
  - EMAIL_ADDRESS
  - PHONE_NUMBER
  - URL
  - IP_ADDRESS
  - CREDIT_CARD
  - IBAN_CODE
  - CAREPILOT_MRN
min_scores:
  PERSON: 0.60
  EMAIL_ADDRESS: 0.50
  PHONE_NUMBER: 0.50
  URL: 0.50
  IP_ADDRESS: 0.50
  CREDIT_CARD: 0.50
  IBAN_CODE: 0.50
  CAREPILOT_MRN: 0.85
strategies:
  ingest:    { PERSON: { operator: replace, params: { new_value: "<PERSON>" } }, CAREPILOT_MRN: { operator: hash,    params: { hash_type: sha256 } } }
  retrieval: { PERSON: { operator: replace, params: { new_value: "<PERSON>" } }, CAREPILOT_MRN: { operator: hash,    params: { hash_type: sha256 } } }
  output:    { PERSON: { operator: replace, params: { new_value: "<PERSON>" } }, CAREPILOT_MRN: { operator: replace, params: { new_value: "<MRN>" } } }
```

### 2.7 Expected report shape

```json
{
  "corpus": {
    "gold_sha256": "to-record-from-run",
    "num_sentences": "to-record-from-run",
    "num_spans_by_entity": "to-record-from-run"
  },
  "per_hook": {
    "ingest":    {"per_entity": "to-record-from-run", "micro": "to-record-from-run"},
    "retrieval": {"per_entity": "to-record-from-run", "micro": "to-record-from-run"},
    "output":    {"per_entity": "to-record-from-run", "micro": "to-record-from-run"}
  },
  "allowlist_hits_total": "to-record-from-run",
  "decision_log_path": "results/decision_log.sample.jsonl",
  "notes": "Precision and recall are computed per-entity and aggregated as micro/macro; a per-entity recall collapse (e.g., PERSON recall below the NIST AI RMF MEASURE 2.11 band for this control) is the primary failure mode to watch for."
}
```

### 2.8 Mapping back to CarePilot

- **Three risk-register rows, not one.** `risks[pii-ingest]`, `risks[pii-retrieval]`, `risks[pii-output]` are independent rows. Each carries its own measured `(precision, recall, f1)` triple and its own mitigation / escalation owner.
- **Register thresholds live elsewhere.** The map *cites* the thresholds; the register *sets* them. For PII at the output hook, the clinical-safety officer and legal set the recall floor; engineering supplies the measurement. This is the same discipline used in mod-101 exercise 01.
- **Cross-link into exercise 03.** If the LLM memorisation eval (exercise 03) shows the fine-tuned model has absorbed any of the entities in the deny list, the output hook alone is not a sufficient control — the mitigation must also propagate upstream to the ingest hook and to the fine-tune corpus curation.
- **OWASP ML Top 10 mapping.** <!-- verify-citation: cite the exact OWASP ML Top 10 entry (code + title + revision date) that covers PII leakage and data-handling controls in ML pipelines; the ML Top 10 project enumerates privacy risks in its catalogue. -->
- **OWASP LLM Top 10 mapping.** For a RAG assistant, the pipeline-level PII mitigation also maps to the OWASP Top 10 for Large Language Model Applications entries on sensitive-information disclosure and input handling. <!-- verify-citation: cite exact entries. -->
- **MITRE ATLAS mapping.** <!-- verify-citation: cite the ATLAS techniques covering (a) sensitive-information disclosure via the inference API and (b) ingestion / corpus-poisoning techniques that a PII filter would close out. -->

## 3. Validation steps

1. **Three hooks are independent.** The reviewer grep for `hook="ingest"`, `hook="retrieval"`, `hook="output"` and confirms the pipeline is called three times, in three distinct call sites.
2. **Decision log is append-only and PII-free.** `decision_log.sample.jsonl` carries `entity_type`, `start`, `end`, `score`, `strategy` but **not** the redacted span text. A reviewer greps for literal PII patterns in the log and finds none.
3. **Gold corpus is synthetic.** `test_corpus/README.md` names the construction protocol; a reviewer spot-checks 10 sentences and confirms none look like real records.
4. **Per-entity metrics present.** `results/eval_report.json` has `per_entity` with precision, recall, F1 for every configured entity type, and `micro` for aggregate.
5. **Allow-list is version-pinned.** `configs/allowlist.yaml` is referenced from the eval report with a SHA; changing the allow-list changes the hash.
6. **Custom recognizer registered.** `AnalyzerEngine.registry` contains the CAREPILOT_MRN recognizer; the reviewer can confirm by running `pipeline.analyzer.analyze("MRN-12345678", entities=["CAREPILOT_MRN"])` and observing a result.
7. **Reproducibility.** Re-running the eval on the same corpus with the same configs produces the same metrics; `pipeline_run_trace.json` logs the Presidio / spaCy versions used.
8. **Register tie-back explicit.** The write-up names the three register rows and the (owner, threshold, evidence) for each.

## 4. Rubric — review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Three hooks implemented and independently evaluated | `per_hook` has `ingest`, `retrieval`, `output` each with their own metrics | Single terminal redactor; or three hooks sharing the same scored result |
| R2 | Per-entity precision / recall / F1 reported | `per_entity` present for every configured entity type | Only aggregate reported |
| R3 | Synthetic gold corpus | `test_corpus/README.md` documents synthetic construction; `gold_sha256` present | Gold corpus contains plausibly-real PII |
| R4 | Custom recognizer for a CarePilot-specific pattern | `recognizers.py` registers at least one `PatternRecognizer` (the MRN) and the registry is attached to the engine | Presidio used out-of-the-box with no local configuration |
| R5 | Decision log is PII-free | Audit log carries offsets / types / scores but no redacted text content | Log echoes the redacted content |
| R6 | Allow-list in addition to deny-list | `configs/allowlist.yaml` is non-empty and `allowlist_hits_total` is non-zero over the eval corpus | No allow-list; every PII-shaped token is redacted without exceptions |
| R7 | Three register rows mapped | Write-up maps metrics into `risks[pii-ingest]`, `risks[pii-retrieval]`, `risks[pii-output]` each with owner and evidence | Metrics reported with no risk-register tie-back |
| R8 | Reproducibility | Presidio / spaCy versions pinned and recorded in the run trace | Silent on versions |
| R9 | Cross-links to exercises 03 and mod-108 | Write-up names the memorisation-eval and guardrail paths as complementary controls | Pipeline treated as the sole mitigation |
| R10 | No invented facts | `<!-- verify-citation: ... -->` on every OWASP / ATLAS citation that has not been personally verified | Fabricated citations or metric numbers |

Scoring bands: *Pass* — R1–R9 all green and R10 is honoured; *Pass with revisions* — R6 or R8 fails but the methodology is sound; *Fail* — any of R1 / R3 / R5 / R10 fails.

## 5. Common mistakes

- **One terminal redactor.** PII that enters the vector store at ingest will leak through the retrieval hook and through the output hook. The three hinge points are independent controls.
- **Deny-list with no allow-list.** A production hospital-triage system sees legitimate strings (system IDs, specific drug names) that NER mis-tags as PERSON. Without an allow-list, the output becomes garbled and the pipeline is disabled by users — a worse outcome than the risk it was meant to mitigate.
- **Writing redacted content back into the audit log.** The log is itself a PII store if it records what was redacted. Record *where and what type*, not *which value*.
- **Treating "PERSON recall = 0.95" as the SLA.** The SLA lives in the register, not in the pipeline. The pipeline *reports*; the register *sets*.
- **Only evaluating at ingest.** Ingest coverage can be perfect while retrieval or output still leaks (model confabulation, chunk-level mis-routing). All three hooks must be evaluated separately.
- **Treating synthetic-corpus numbers as production evidence.** The synthetic corpus is an engineering test; production evaluation needs governance-approved sampling of real traffic. The write-up states this.
- **Letting Presidio's spaCy model silently update.** A `pip install --upgrade` of the spaCy model changes the per-entity recall overnight. Pin the model version in `configs/presidio.yaml` and in `pipeline_run_trace.json`.
- **No custom recognizer.** Every production deployment needs at least one organisation-specific pattern (MRN format, internal system IDs, etc.). A pure out-of-the-box Presidio config fails review.
- **Hashing the output-hook MRN to a value that leaks.** Hashing with a deterministic salt and no truncation allows an attacker with hash-dictionary access to recover MRNs. Either use a non-deterministic token (`replace`) at the output hook, or use a keyed HMAC with the key stored in a secret manager.

## 6. References

Only the official sources listed in the work packet (and clearly-labelled project documentation) are used.

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — the three-hook eval feeds **MEASURE 2.7** (security and resilience, PII-containment aspect), **MANAGE 2.2** (sustained controls on the fixture set), and **MANAGE 4.1** (post-deployment monitoring feeds the pipeline).
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — cite the exact entry at submission. <!-- verify-citation: cite the exact ML Top 10 entry (code + title + current revision date) that covers PII leakage / data-handling controls. -->
- **OWASP Top 10 for Large Language Model Applications.** (Separate OWASP project from the ML Top 10.) Covers sensitive-information disclosure and input-handling for LLM apps. <!-- verify-citation: cite the exact entry IDs. -->
- **MITRE ATLAS.** `https://atlas.mitre.org/` — cite the technique ID(s) for sensitive-information disclosure and the corresponding mitigations. <!-- verify-citation: cite exact ATLAS IDs. -->
- **Microsoft Presidio.** Project documentation at `https://microsoft.github.io/presidio/`. APIs used in this solution: `AnalyzerEngine`, `AnonymizerEngine`, `RecognizerRegistry`, `PatternRecognizer`, `OperatorConfig`. Presidio bundles a spaCy-NER-backed analyser and a declarative anonymiser.
- **Module running system.** CarePilot — hospital-triage-support RAG assistant, introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`.
- **Exercise prompt.** `lessons/mod-107-privacy-risk-engineering/exercises/exercise-05-presidio-pii-pipeline-and-redaction.md`.

<!-- Numeric precision / recall / F1 values in §2.7 are run-time placeholders by design; the solution does not quote invented values. -->

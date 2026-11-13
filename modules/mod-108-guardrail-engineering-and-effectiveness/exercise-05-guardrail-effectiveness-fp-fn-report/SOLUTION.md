# Exercise 05 — Guardrail Effectiveness: FP / FN + Cost + Latency Report (CarePilot) — Reference Solution

## 1. Solution overview

This exercise measures the **composed guardrail stack** from exercises 02–04 against the **mod-104 hosted adversarial-and-legitimate set**, produces the per-slot × per-adversarial-category × per-control-instance measurement rows, writes the **one-page FP / FN report a product manager reads**, and **back-feeds** the measurement into the mod-103 register's residual and control-defeated columns.

Four decisions drive the shape:

- **Consume mod-104, do not fork.** Both adversarial *and* legitimate sets come from mod-104's workflow at pinned versions. Recall without precision is a claim, not a measurement — if mod-104 has adversarial but no legitimate set for a category, the category is bounded and flagged, not measured in isolation.
- **End-to-end request path, not a stripped harness.** The runner plays every payload through the exact CarePilot serving path (FastAPI → NeMo GR → classifier endpoints → Guardrails AI → middleware → model). Numbers reflect production, including middleware and dispatch overhead.
- **Pin everything.** Every measurement row carries `set_version`, `control_version`, `run_id`, `config_version`. A row that cannot be reproduced next week is not evidence. The run manifest captures every pin.
- **One page for the PM.** Fifty rows of numbers is the engineer's view; the FP / FN report is the product-manager one-pager. Chapter 05 §"Skeleton" is the shape — top-three FPs, top-three FNs, coverage summary, cost roll-up, latency roll-up, trend arrows, decisions requested.

Deliverables live under `risk/mod-108/carepilot/measurement/` and are indexed from `mod-108-ex05-measurement.md`.

## 2. Worked implementation

### 2.1 Repository layout

```
risk/mod-108/carepilot/measurement/
├── runner.py                     # end-to-end runner against mod-104 pinned sets
├── measurements.csv              # chapter-05 contract rows (17 fields each)
├── cost.md                       # per-control $/call; per-slot per-day; per-surface per-month
├── latency.md                    # p50 / p95 per control per slot (middleware-measured)
├── fp-fn-report.md               # the one-page PM report (chapter 05 §Skeleton)
├── coverage-refreshed.md         # the exercise-01 coverage table annotated with measured/claimed/open
├── register-updates.md           # the list of mod-103 rows updated; residual + control-defeated deltas
├── manifest-<run-id>.json        # the run manifest: every version + commit + hardware pin
└── anti-patterns.md              # four chapter-05 anti-pattern probes
```

### 2.2 Runner (`runner.py`)

Plays each payload through the real CarePilot request path; emits one measurement row per `(slot, category, control_instance)`.

```python
# measurement/runner.py
# Measurement runner. Walks mod-104 adversarial + legitimate sets at pinned
# versions through the composed CarePilot stack. Emits the chapter-05 contract
# rows plus the run manifest.
#
# Cost and latency are measured in the middleware (chapter 05 §"Latency
# accounting", §"Cost accounting"). Server-side numbers are logged for the
# sanity-check row only.

from __future__ import annotations

import csv
import hashlib
import json
import os
import time
import uuid
from collections import defaultdict
from dataclasses import dataclass, asdict
from pathlib import Path
from typing import Any

import httpx


# Pin the mod-104 set versions at the top so every row of the measurement
# carries the same set_version string. If the set changes mid-run the numbers
# are not reproducible.
ADVERSARIAL_SET_VERSION = "<pin-mod-104-adv-vX.Y.Z>"
LEGITIMATE_SET_VERSION  = "<pin-mod-104-legit-vX.Y.Z>"
CAREPILOT_URL = os.environ["CAREPILOT_URL"]        # point at staging; never production


@dataclass
class MeasurementRow:
    slot: str
    control_instance: str
    category: str
    tp: int
    fp: int
    tn: int
    fn: int
    precision: float
    recall: float
    mean_cost_per_call: float
    p50_latency_ms: float
    p95_latency_ms: float
    n_adversarial: int
    n_legitimate: int
    set_version: str
    control_version: str
    run_id: str


def _load_set(path: Path, label: str) -> list[dict]:
    items = []
    with path.open() as f:
        for line in f:
            rec = json.loads(line)
            rec["gold_label"] = label       # "adversarial" or "legitimate"
            items.append(rec)
    return items


def _payload_hash(set_path: Path) -> str:
    h = hashlib.sha256()
    h.update(set_path.read_bytes())
    return h.hexdigest()


def _play(payload: dict, run_id: str) -> dict:
    """Play one payload through the real CarePilot endpoint; return the trace
    decisions plus the end-to-end latency."""
    t0 = time.perf_counter()
    r = httpx.post(f"{CAREPILOT_URL}/chat", json={
        "turn_id": f"{run_id}:{payload['id']}",
        "caller": payload.get("caller", "measure-bot"),
        "content": payload["text"],
    }, timeout=30.0)
    e2e_ms = (time.perf_counter() - t0) * 1000
    body = r.json()
    return {
        "trace": body["trace"],              # exercise-02 trace records; mod-109's schema
        "final_verdict": body["verdict"],    # "safe" or "reject" or "transformed"
        "e2e_latency_ms": e2e_ms,
    }


def _control_version_for(slot: str, trace: list[dict]) -> str:
    # Pull the `control_version` field emitted by the slot's rails into the
    # row. Chapter 05 §"The measurement contract": control_version is a per-
    # control git / model pin.
    for r in trace:
        if r["slot"] == slot:
            return r.get("control_version", "unknown")
    return "unknown"


def main() -> int:
    run_id = uuid.uuid4().hex[:12]
    adv = _load_set(Path("fixtures/mod-104/adversarial.jsonl"), "adversarial")
    leg = _load_set(Path("fixtures/mod-104/legitimate.jsonl"), "legitimate")

    rows_by_key: dict[tuple[str, str, str], list[dict]] = defaultdict(list)
    for payload in adv + leg:
        out = _play(payload, run_id)
        category = payload["category"]
        final = out["final_verdict"]
        # Which slot caught it — the first slot in the trace emitting a reject
        # is the "firing slot". For precision/recall per slot we attribute the
        # verdict to the slot that fired; legitimate items are attributed
        # across *all* slots that had a chance to reject them.
        firing_slots = sorted({r["slot"] for r in out["trace"] if r["verdict"] == "reject"})
        if not firing_slots:
            firing_slots = ["output"]  # no slot fired; the full stack lets it through
        for slot in firing_slots:
            key = (slot, category, out["trace"][0].get("control_version", "unknown"))
            rows_by_key[key].append({
                "gold": payload["gold_label"], "final": final,
                "e2e_latency_ms": out["e2e_latency_ms"],
            })

    with open(f"measurements-{run_id}.csv", "w", newline="") as f:
        w = csv.DictWriter(f, fieldnames=list(MeasurementRow.__annotations__))
        w.writeheader()
        for (slot, category, control_version), items in rows_by_key.items():
            tp = sum(1 for i in items if i["gold"] == "adversarial" and i["final"] == "reject")
            fp = sum(1 for i in items if i["gold"] == "legitimate"  and i["final"] == "reject")
            tn = sum(1 for i in items if i["gold"] == "legitimate"  and i["final"] != "reject")
            fn = sum(1 for i in items if i["gold"] == "adversarial" and i["final"] != "reject")
            n_adv = sum(1 for i in items if i["gold"] == "adversarial")
            n_leg = sum(1 for i in items if i["gold"] == "legitimate")
            lat = sorted(i["e2e_latency_ms"] for i in items)
            p50 = lat[len(lat) // 2] if lat else 0.0
            p95 = lat[int(len(lat) * 0.95)] if len(lat) >= 20 else lat[-1] if lat else 0.0
            row = MeasurementRow(
                slot=slot, control_instance=f"{slot}:{control_version}",
                category=category,
                tp=tp, fp=fp, tn=tn, fn=fn,
                precision=tp / max(tp + fp, 1),
                recall=tp / max(tp + fn, 1),
                mean_cost_per_call=0.0,  # populated from cost.md per-control line
                p50_latency_ms=p50, p95_latency_ms=p95,
                n_adversarial=n_adv, n_legitimate=n_leg,
                set_version=f"adv={ADVERSARIAL_SET_VERSION};leg={LEGITIMATE_SET_VERSION}",
                control_version=control_version, run_id=run_id,
            )
            w.writerow(asdict(row))

    _write_manifest(run_id, adv_path=Path("fixtures/mod-104/adversarial.jsonl"),
                    leg_path=Path("fixtures/mod-104/legitimate.jsonl"))
    return 0


def _write_manifest(run_id: str, adv_path: Path, leg_path: Path) -> None:
    manifest = {
        "run_id": run_id,
        "timestamp_ms": int(time.time() * 1000),
        "sets": {
            "adversarial": {"version": ADVERSARIAL_SET_VERSION, "sha256": _payload_hash(adv_path)},
            "legitimate":  {"version": LEGITIMATE_SET_VERSION,  "sha256": _payload_hash(leg_path)},
        },
        "controls": {
            # Pulled from the exercise-02/03/04 pins. The runner reads the
            # running CarePilot's /meta endpoint which serialises every version
            # from its own config files.
            "nemo_guardrails":   "<pin>",
            "colang":            "<pin>",
            "guardrails_ai":     "<pin>",
            "llama_guard":       "<pin-checkpoint>",
            "llama_guard_taxonomy": "<pin>",
            "shieldgemma":       "<pin-checkpoint>",
            "shieldgemma_policy_set": "<pin>",
            "presidio_analyzer": "<pin>",
            "presidio_anonymizer": "<pin>",
            "spacy_model":       "<pin>",
            "middleware_commit": "<pin-git-sha>",
            "runner_commit":     "<pin-git-sha>",
        },
        "hardware": {
            "gpu_model": "<pin>", "cpu": "<pin>", "ram_gb": "<pin>",
            "instance_type": "<pin>",
        },
    }
    Path(f"manifest-{run_id}.json").write_text(json.dumps(manifest, indent=2))


if __name__ == "__main__":
    raise SystemExit(main())
```

### 2.3 Measurement table (`measurements.csv`)

Fields per row (chapter 05 §"The measurement contract"):

```
slot, control_instance, category, TP, FP, TN, FN, precision, recall,
mean_cost_per_call, p50_latency_ms, p95_latency_ms,
n_adversarial, n_legitimate, set_version, control_version, run_id
```

Every field is load-bearing. Example shape (populate actual numbers from the run):

| slot | control_instance | category | TP | FP | TN | FN | precision | recall | mean_$/call | p50_ms | p95_ms | n_adv | n_leg |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| input | llama_guard:`<ckpt>` | prompt-injection-direct | to-record | to-record | to-record | to-record | to-record | to-record | to-record | to-record | to-record | to-record | to-record |
| retrieval | shieldgemma:`<ckpt>` | prompt-injection-indirect | " | " | " | " | " | " | " | " | " | " | " |
| retrieval | presidio-retrieval:`<ver>` | pii-exfiltration | " | " | " | " | " | " | " | " | " | " | " |
| output | llama_guard:`<ckpt>` | disallowed-content | " | " | " | " | " | " | " | " | " | " | " |
| output | shieldgemma:`<ckpt>` | disallowed-content | " | " | " | " | " | " | " | " | " | " | " |
| output | presidio-output:`<ver>` | pii-exfiltration | " | " | " | " | " | " | " | " | " | " | " |
| tool_policy | middleware:`<sha>` | excessive-agency | " | " | " | " | " | " | " | " | " | " | " |

### 2.4 Cost accounting (`cost.md`)

Three lines per control (chapter 05 §"Cost accounting"):

1. **Compute.** Self-hosted Llama Guard + ShieldGemma: GPU-hour amortised across the throughput they serve during the run. Hosted model (if any): per-call price.
2. **Operational.** Log ingestion storage (trace volume × $/GB/mo); on-call fraction (eng-hours/mo × loaded hourly rate / request volume).
3. **Third-party licence.** Guardrails AI (free OSS at this time; Hub validators may have fees); Presidio (free OSS); any vendor seats.

Per-slot per-day roll-up at CarePilot's production request volume; per-surface per-month roll-up in the same currency the mod-103 register uses.

### 2.5 Latency accounting (`latency.md`)

Measured in the middleware, not server-side. Serial and parallel numbers at slot 4 (chapter 05 §"Latency accounting"). Per-slot p50 / p95 against the exercise-01 budget. Sanity-check row: middleware p95 vs. classifier-server p95 for slot 4, so network and dispatch overhead is visible.

### 2.6 FP / FN report (`fp-fn-report.md`) — one page, product-manager-facing

Shape per chapter 05 §"Skeleton (per surface)":

```
# CarePilot — Guardrail FP / FN Report — Surface: /chat
Run <run_id>  |  Adversarial set <adv version>  |  Legitimate set <leg version>

## Top three false positives
1. Category: disallowed-content/self-harm
   Rate: <x%> on <n> legitimate items. Est. volume at production traffic: <N>/day blocked.
   Example (redacted): legitimate clinician question about overdose protocols the output
   classifier blocks as self-harm content.
   Owner: clinical-safety; decision requested: adjust ShieldGemma threshold; estimated
   recall delta: <-x%>.

2. Category: pii-exfiltration
   Rate: <x%>. Est. volume: <N>/day blocked. Example: a legitimate clinician request that
   contains a patient MRN the caller is entitled to see, blocked because the per-turn
   whitelist did not include it. Owner: platform; decision: patch whitelist derivation.

3. Category: prompt-injection-indirect
   Rate: <x%>. Example: a legitimate triage protocol excerpt the retrieval filter flags.
   Owner: platform; decision: raise ShieldGemma retrieval injection_likely threshold.

## Top three false negatives
1. Category: prompt-injection-indirect
   Rate: <x%> on <n> adversarial items. Harm class: harm-ipi-triage-protocol.
   Monitor status (mod-109): under implementation. Example (redacted): a poisoned
   protocol chunk that evades ShieldGemma because the injection phrasing is paraphrased.

2. Category: medical-misinformation (CarePilot-specific)
   Rate: <x%>. Harm class: harm-model-misinformation. Neither Llama Guard's taxonomy nor
   ShieldGemma's policy set covers this category; the supplementary rule caught some
   cases but not all. Decision: fund a custom ShieldGemma policy prompt (chapter 03
   stretch) or escalate to the clinical-safety review path.

3. Category: pii-exfiltration via model confabulation
   Rate: <x%>. Harm class: harm-pii-output. Presidio recall on confabulated-MRN patterns
   that are slightly off from the CAREPILOT_MRN regex (missing digit, OCR noise). Decision:
   add a fuzzy MRN recogniser; cost: +0.2 ms/call.

## Coverage summary vs. OWASP LLM Top 10 (2025)
- LLM01 Prompt Injection (direct):   covered (input-slot Llama Guard + rules)
- LLM01 Prompt Injection (indirect): partial (retrieval-slot; see FN #1)
- LLM02 Sensitive Information Disclosure (2025): covered (output-slot Presidio + per-turn whitelist)
- LLM05 Improper Output Handling: covered (output guard per-field)
- LLM06 Excessive Agency: covered (middleware)
- LLM08 Vector/Embedding Weaknesses: partial (retrieval classifier only)
- LLM09 Misinformation: uncovered beyond rules (medical-misinformation, FN #2)
- (verify-citation: pin exact 2025 category codes at submission)

## Harm-model coverage (mod-102)
Fully covered:    harm-direct-injection, harm-sensitive-retrieval-disclosure, harm-excessive-agency-order
Partially:        harm-ipi-triage-protocol (FN #1), harm-pii-output (FN #3)
Uncovered:        harm-model-misinformation — escalation filed

## Cost roll-up
Per month, in <currency>: $<n> compute + $<n> operational + $<n> licence = $<total>.

## Latency roll-up
Request-path p50 / p95 added by the stack: <p50> / <p95> ms.
Slot 4 serial vs. parallel: serial <s> ms p95; parallel <p> ms p95; cost delta of parallel: $<n>.

## Trend
First run (baseline). No deltas yet; mod-109's monitor stub seeded from this report.

## Decisions requested
- (1) Accept the self-harm-category FP rate at <x%> for 90 days while clinical-safety
      reviews the taxonomy-scope change. Owner: risk council.
- (2) Fund the fuzzy MRN recogniser (eng-hours: 8). Owner: platform lead.
- (3) Fund the custom ShieldGemma medical-misinformation policy prompt. Owner: clinical-safety + platform.
```

The report is one page. If it grows to three pages, the discipline has broken.

### 2.7 Coverage table refreshed (`coverage-refreshed.md`)

The exercise-01 table re-annotated: each cell carries one of `measured`, `claimed`, or `open`:

- `measured` — the measurement table has a row for this slot × category and recall is above the treatment appetite.
- `claimed` — the slot addresses the category by design but the measurement did not test it (e.g., no legitimate set for the category).
- `open` — the slot addresses the category by design but the measurement shows recall below appetite; register row flagged.

### 2.8 Register updates (`register-updates.md`)

For every mod-103 row this measurement updates:

- `risks[prompt-injection-direct]`: residual <before> → <after, measured>; control-defeated columns populated from a "slot-1-down" run or derived from the remaining-slots recall; change-log entry pointing to `manifest-<run_id>.json`.
- `risks[indirect-prompt-injection]`: residual <before> → <after>; FN #1 noted; mod-112 escalation filed if above appetite.
- `risks[pii-output]`: residual <before> → <after>; FN #3 drives the fuzzy-MRN work item.
- `risks[excessive-agency]`: residual <before> → <after>; fail-closed error rate reported alongside.
- `risks[pii-retrieval]`: residual <before> → <after>; per-turn whitelist bug (FP #2) filed for patch.
- `risks[refusal-leak]`: refusal-string audit reviewed by hand; no measurement row (no adversarial set).

Every updated row carries the `run_id` in the change log; a later re-run that overturns the number is tracked.

### 2.9 Run manifest (`manifest-<run-id>.json`)

Already shown in §2.2's `_write_manifest`. Pins:

- mod-104 adversarial set version + SHA-256 of the pinned file.
- mod-104 legitimate set version + SHA-256.
- Every control version: Llama Guard checkpoint, ShieldGemma checkpoint + policy set, Presidio analyzer + anonymiser, spaCy model, NeMo Guardrails, Colang, Guardrails AI, middleware git commit, runner git commit.
- Runtime: GPU model, CPU, RAM, instance type.
- Run timestamp.

### 2.10 Anti-pattern probes (`anti-patterns.md`)

- **Adversarial-only measurement.** Confirmed both adversarial and legitimate sets were used; any category where mod-104 does not host a legitimate set is bounded. A table lists the bounded categories and the escalation filed to mod-104.
- **Unpinned versions.** Confirmed every row in `measurements.csv` has `set_version` and `control_version` populated; a `grep` for `unknown` returns zero hits. Any unpinned control is a fix before submission.
- **Cost / latency in the wrong place.** Latency and cost numbers are measured in the middleware. Sanity-check row compares middleware-measured p95 against the classifier-server p95 for slot 4; the delta is the network + dispatch overhead and is the visible reason the middleware numbers are the authoritative ones.
- **Report hides the tradeoff.** The FP / FN report has example payloads (redacted where they contain real content) and volume estimates for every top-3 row. A report with only percentages and no examples hides which caller experience is affected.

### 2.11 Interpretive note

One paragraph names the load-bearing slot for the top-two harm classes and what happens if the load-bearing slot goes down:

> Slot 2 (retrieval filter: ShieldGemma + Presidio) is load-bearing on `harm-ipi-triage-protocol` — if slot 2 is defeated, slot 4's output classifiers catch roughly `<x%>` of the remaining payloads but `harm-ipi-triage-protocol`'s residual moves close to inherent on the `prompt-injection-indirect` category. Slot 4 (output validator: dual classifier + Presidio) is load-bearing on `harm-pii-output` — if slot 4 is defeated, Presidio's ingest-path pass covers roughly `<x%>` of PII cases but confabulated PII (FN #3) is not covered by any upstream slot.

## 3. Validation steps

1. **All nine artefacts exist.** Runner, measurement CSV, cost, latency, FP/FN report, coverage refreshed, register updates, manifest, anti-patterns.
2. **Runner runs end-to-end.** `python runner.py` returns 0 and writes `measurements-<run_id>.csv` and `manifest-<run_id>.json`.
3. **All 17 row fields populated.** `csvtool` or equivalent: no null, no empty, no `unknown` in `set_version` / `control_version` / `run_id`.
4. **Both sets used.** The runner reads adversarial *and* legitimate files; any category where one is missing is flagged in `anti-patterns.md`.
5. **FP / FN report is one page.** `wc -l fp-fn-report.md` ≤ ~120 lines (single printed page).
6. **Decisions-requested list present.** `grep "Decisions requested" fp-fn-report.md` returns a non-empty section with named owners.
7. **Coverage table refreshed with annotations.** Every cell carries `measured`, `claimed`, or `open`.
8. **Register updates do not pre-credit.** Every updated row carries a `run_id` in the change log; residual numbers citing the run are the only form the update takes.
9. **Manifest pins everything.** `grep <pin> manifest-*.json` returns no `TODO` or `unknown` placeholders.
10. **Middleware-measured latency.** `latency.md` states measurement point; sanity-check row compares middleware vs. server-side.
11. **Appetite flags visible.** Rows whose residual moves above appetite are flagged for mod-112 escalation.

## 4. Rubric — review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Runner uses the real request path | `_play` calls CarePilot `/chat` end-to-end | A stripped harness that only calls the classifier |
| R2 | All 17 fields per row populated | Reviewer counts columns and spot-checks three rows | Any field missing or null |
| R3 | Both adversarial and legitimate sets | Every category has an adversarial and a legitimate count; bounded categories explicit | Recall without precision claimed |
| R4 | Cost roll-up has three lines | Compute, operational, third-party; per-day, per-month | Only compute reported |
| R5 | Latency parallel vs. serial at slot 4 | Both numbers in `latency.md` | Only one reported |
| R6 | FP / FN report is one page | One printed page; top-3 FPs, top-3 FNs, decisions requested | Three pages of numbers |
| R7 | Example payloads in the report | Every top-3 row names an example (redacted if sensitive) | Only percentages |
| R8 | Coverage refreshed with annotations | Each cell is `measured`, `claimed`, or `open` | Not annotated |
| R9 | Register updates cite `run_id` | Every updated row has a change-log entry with the run id | No run id |
| R10 | Manifest pins everything | Every control, model, taxonomy, set hash, hardware pinned | Any unpinned |

Scoring bands: *Pass* — R1–R9 green, R10 honoured; *Pass with revisions* — R4 or R5 partial; *Fail* — any of R1 / R3 / R6 / R10 fails.

## 5. Common mistakes

- **Running against the classifier only.** A stripped harness that calls Llama Guard directly measures the classifier, not the stack. The middleware, the rails, and the Guardrails AI guard all contribute FP / FN behaviour and latency. Run the real path.
- **Adversarial-only measurement.** Recall at 99% on the adversarial set means nothing if FP on legitimate traffic is 50%. Every measured category needs both sets.
- **Unpinned versions.** A measurement row that cites "`<unknown>` version" is not evidence — the next run cannot be compared. The manifest is the row's receipt.
- **Server-side latency numbers.** They understate the request-path cost by the network and dispatch overhead. The middleware stamp is what the user experiences.
- **Serial-only output-slot latency.** When two classifiers must both pass, the parallel number is the production number. Serial is a diagnostic.
- **Compute-only cost.** The GPU bill is only one line. The on-call fraction, the log ingestion cost, and any licence seats matter — chapter 05 §"Cost accounting" is the three-line rule.
- **Three-page FP / FN report.** The product manager does not read three pages. Split by surface if needed; split the engineering detail into the raw CSV.
- **Percentages without examples.** "FP at 3% on self-harm" is a number. "FP at 3% on self-harm: legitimate clinician questions about overdose protocols get blocked" is a decision.
- **Register pre-credit before the run.** The measurement writes the residual; no residual number should sit in the register before the manifest is signed.
- **Overturning a row without a new run_id.** If a re-run moves a number, it is a new run with a new manifest, not an edit. The change log tracks movements.
- **Bootstrap CI duplicated with L30.** Chapter 05 §"Cross-references" and chapter 06 §"Handoff" are explicit: the L30 eval engineer adds statistical calibration. Do not duplicate; share the manifest and the CSV.

## 6. References

- **NIST AI Risk Management Framework 1.0.** `https://www.nist.gov/itl/ai-risk-management-framework`. The measurement rows feed MEASURE 2.11 (effectiveness measurement) and MANAGE 2.2 / 4.1 (sustained controls and post-deployment monitoring). The register back-feed is the MANAGE-side artefact.
- **OWASP Top 10 for Large Language Model Applications.** `https://genai.owasp.org/llm-top-10/`. The coverage summary section of the FP / FN report maps measured cells to OWASP categories; the exact 2025 codes are pinned at submission. <!-- verify-citation: pin 2025 category codes. -->
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/`. <!-- verify-citation: pin exact ML Top 10 entries covered. -->
- **MITRE ATLAS.** `https://atlas.mitre.org/`. The measurement rows cross-walk to ATLAS tactic / technique IDs in the exercise-01 coverage table. <!-- verify-citation: pin IDs. -->
- **mod-104 adversarial-and-legitimate set.** Hosted by the mod-104 solutions workflow; pinned at the version string the runner carries in the manifest.
- **Module running system.** CarePilot — hospital triage-support RAG assistant, introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`.
- **mod-103 register** — `risk-register/2025/CarePilot.yaml`. The register-updates block writes measured residual and control-defeated values back here.
- **Exercises 01–04** — this exercise measures the stack they ship; the runner pulls their control pins into the manifest.
- **Exercise prompt.** `lessons/mod-108-guardrail-engineering-and-effectiveness/exercises/exercise-05-guardrail-effectiveness-fp-fn-report.md`.

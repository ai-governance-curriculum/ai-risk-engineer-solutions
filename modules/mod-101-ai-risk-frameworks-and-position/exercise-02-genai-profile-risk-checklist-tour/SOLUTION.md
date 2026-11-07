# Exercise 02 — NIST GenAI Profile 12-Risk Checklist Tour — Reference Solution

## 1. Solution overview

This reference covers two deliverables:

1. `mod-101-ex02-applicability.md` — a twelve-row applicability sheet for the AI 600-1 risks plus a five-sentence decomposition for every **in-scope** row.
2. `mod-101-ex02-backlog.md` — a backlog file for any **NEW-TOOLING-NEEDED** items and for inherited-control-evidence requests upstream.

The running system is **CarePilot**, the fictional hospital triage-support RAG assistant introduced in exercise 01. Keeping the same system across exercises 01–05 is deliberate: the engineering evidence set converges, and the AI 600-1 decompositions re-file against the exercise 01 sub-category map.

Core decisions reflected in the solution:

- The twelve risk short-forms are used verbatim from AI 600-1 (July 2024). The applicability ruling for each row is paired with a one-sentence reason — chapter 02 is explicit that "n/a without a reason" is a reject.
- Every in-scope row cites a tool. Where no good open tool exists for the measurement at CarePilot's scale, the row is marked **NEW-TOOLING-NEEDED** and filed in the backlog.
- Every in-scope row cross-references an RMF sub-category from the exercise 01 map (by its ID). If no mapping is possible, chapter 02 instructs you to go back and add a row to exercise 01 — this solution does both (notes which exercise-01 rows got added, so a reviewer can see the trace both ways).
- The backlog file carries supply-chain evidence requests for risks mitigated upstream by the base-model provider; chapter 02's "inherited control" paragraph is the template.

## 2. Implementation — worked answer

### 2.1 `mod-101-ex02-applicability.md`

```markdown
---
system: CarePilot (hospital triage-support RAG assistant, HITL)
framework: NIST AI RMF Generative AI Profile (NIST AI 600-1, July 2024)
cross-ref: mod-101-ex01-rmf-map.md
author: ai-risk-engineer (L25)
reviewers: AI risk council (chair), clinical-safety officer, ai-governance-analyst (L15)
status: draft-for-review
---

# CarePilot — NIST AI 600-1 Applicability Sheet and In-Scope Decompositions

## 1. Twelve-row applicability sheet

Short names are those used in AI 600-1 §2 (July 2024).

| # | AI 600-1 risk | Applicability | Reason |
|---|---|---|---|
| 1 | CBRN Information or Capabilities | **Out** | CarePilot is a clinical-triage assistant grounded in hospital triage protocols; the retrieval corpus is scoped, and the base model is procured from a vendor whose provider-side controls refuse CBRN uplift. The residual uplift risk is handled upstream; CarePilot adds a refusal-regression fixture to catch any slip (see backlog item INH-01). |
| 2 | Confabulation | **In** | CarePilot summarises and routes clinician-facing triage information. A hallucinated instruction can propagate into a clinician's triage decision even with HITL; HITL depends on the clinician catching the error, so measurable groundedness is non-negotiable. |
| 3 | Dangerous, Violent, or Hateful Content | **In (narrow)** | CarePilot is clinician-facing, not patient-facing, so classical hateful-content risk is lower; but violent-ideation and self-harm content do appear in triage intake (the clinician types what the patient said). The system must handle and classify these without amplifying or misclassifying. |
| 4 | Data Privacy | **In** | CarePilot operates on PHI; HIPAA applies. Both input-side PHI handling and output-side leakage (e.g., retrieving another patient's record into a response) are in scope. |
| 5 | Environmental Impacts | **Out (narrow — inherited)** | CarePilot's inference footprint is small (RAG over a scoped corpus; low tokens per decision); training footprint is carried by the base-model vendor. Disclosure is filed from vendor evidence; see backlog item INH-02 for the vendor disclosure request. |
| 6 | Harmful Bias or Homogenization | **In** | Triage classification exposure across patient demographics directly implicates clinical-fairness risk; homogenisation of recommendations across patients of different demographics can mask disparate treatment. |
| 7 | Human-AI Configuration | **In** | HITL is the central safety assumption. If the clinician over-trusts CarePilot (automation bias) or under-trusts (ignores helpful signal), the HITL assumption collapses. Oversight design is in scope and is also the Article 14 artifact (exercise 04). |
| 8 | Information Integrity | **Out** | CarePilot's outputs are clinician-facing, internal to the hospital; the system does not generate content for public distribution. Synthetic-content provenance (C2PA / watermarking) is not applicable to the output channel; the risk is handled by hospital IT content-security policy at the perimeter, not by this system. |
| 9 | Information Security | **In** | Direct prompt injection (clinician types adversarial text, deliberately or accidentally echoing patient-reported text) and indirect prompt injection (a poisoned protocol document in the retrieval corpus) are both plausible. PHI exfiltration via model is also in scope. |
| 10 | Intellectual Property | **In (narrow)** | The retrieval corpus is the hospital's own documentation and licensed clinical references; outputs must not reproduce licensed clinical-reference text verbatim beyond fair-use/quotation thresholds. Memorisation risk is low relative to open-domain systems but non-zero. |
| 11 | Obscene, Degrading, and/or Abusive Content | **Out (narrow)** | Clinician-facing, internal, SSO-authenticated use; neither the input distribution nor the output use case creates meaningful risk of obscene / NCII / CSAM production. Residual risk is handled by the base-model vendor's content policy; CarePilot adds a defensive output-safety classifier regression fixture (see INH-03). |
| 12 | Value Chain and Component Integration | **In** | CarePilot depends on a procured base LLM, a procured embedding model, and open-source retrieval components. ML-BOM, model-weight provenance, and ingest controls all apply. |

## 2. In-scope decompositions

Each in-scope row below is written as the five-sentence decomposition from chapter 02: *means / measure / gate / guard / evidence*. Each ends with the cross-reference to exercise 01.

### Risk 2 — Confabulation

*Means:* for CarePilot, any claim in the model's response must be supported by a retrieved triage-protocol passage or by the structured input the clinician provided; an unsupported claim is a hallucination that can propagate into a clinical decision.

*Measure:* groundedness and retrieval-alignment on a clinician-adjudicated eval set (methodology owned by `ai-eval-engineer` at L30; metrics consumed here). Judge-vs-human calibration is cross-walked to `model-evaluation-engineer` (L30) when uncertainty bars are tight.

*Gate:* candidate model does not ship unless groundedness and retrieval-alignment exceed the floor defined in `risk-register/2025/CarePilot.yaml#groundedness-envelope`; the floor is set by the AI risk council and reviewed quarterly.

*Guard:* citation-required guardrail — model must emit retrieved-passage citations with every clinical claim; missing citations trigger a UI "cannot verify" warning and a mandatory HITL confirmation.

*Evidence:* groundedness scorecard at `evidence/measure-2.3/<release>/shadow-eval-report.json`; citation-guardrail config at `guardrails/carepilot/citation-required.yaml`; signed by `ai-eval-engineer` (L30) methodology + AI risk council for the gate.

*Cross-ref to exercise 01:* MEASURE 2.3, MAP 2.3, MEASURE 1.1.

**Tool(s):** DeepEval or RAGAS (groundedness, retrieval-alignment); `ai-eval-engineer` chooses between them in mod-108. Cross-check methodology is a clinician-adjudicated gold set. Harm model (mod-102) sets adjudication protocol.

### Risk 3 — Dangerous, Violent, or Hateful Content

*Means:* clinicians may enter patient-reported statements that reference self-harm, suicide ideation, or violent content; the model must not amplify, must classify correctly, and must escalate per the hospital's safety-escalation SOP.

*Measure:* output-safety classifier scores (Llama Guard or ShieldGemma or OpenAI Moderation — whichever the base-model provider already ships, instrumented at CarePilot's output); clinician-reported misclassification rate on a labelled eval slice.

*Gate:* candidate does not ship unless classifier false-negative rate on the safety-escalation subset is at or below the floor in `risk-register/2025/CarePilot.yaml#safety-escalation-envelope`; the floor is set jointly with the clinical-safety officer.

*Guard:* escalate-to-safety-SOP hook — on classifier hit above threshold, the UI shows the hospital's safety-escalation SOP card instead of a clinical recommendation; the clinician cannot suppress the SOP card.

*Evidence:* classifier version pinned in `guardrails/carepilot/output-safety.yaml`; safety-escalation eval report at `evidence/measure-2.11/<release>/safety-escalation-report.json`; co-signed by clinical safety.

*Cross-ref to exercise 01:* MEASURE 2.11 (slice: safety-escalation subset), MANAGE 4.1.

**Tool(s):** Llama Guard, ShieldGemma, OpenAI Moderation — whichever the base model's provider SDK supports natively and the hospital's data-residency policy allows. HarmBench-style adversarial probing is applied via the red-team suite (see Risk 9).

### Risk 4 — Data Privacy

*Means:* CarePilot operates on PHI; the risk is (a) PHI leaving the environment improperly, (b) PHI from one patient surfacing in another patient's session, (c) PHI appearing in logs that are over-broadly accessible.

*Measure:* PII / PHI detection on all model inputs and outputs (Microsoft Presidio with HIPAA-flavoured recognisers); cross-session leakage eval (synthetic eval set injecting identifiers from patient A, verifying patient B's session cannot elicit them); membership-inference-attack baseline (MIA / LiRA) against the base model on the retrieval corpus (mod-107 methodology).

*Gate:* candidate does not ship unless PHI-redaction coverage on the eval set is at or above the floor in `risk-register/2025/CarePilot.yaml#phi-envelope`; cross-session leakage eval must be zero on the controlled eval.

*Guard:* Presidio-based PHI redaction on input, retrieval, and output; retrieval scope limited by session-level authorisation; audit log scrubs free-text PHI (mod-107 pattern).

*Evidence:* PHI-leak report at `evidence/privacy/<release>/phi-leak-report.json`; MIA baseline at `evidence/privacy/<release>/mia-baseline.json`; redaction-config version pinned.

*Cross-ref to exercise 01:* added row **MEASURE 2.10** (privacy / IP risks evaluated) to the exercise 01 map — flagged as an **errata to exercise 01** in the appendix.

**Tool(s):** Microsoft Presidio for PHI detection; ML Privacy Meter (or a lightweight LiRA implementation) for MIA baseline. Cross-session-leakage evaluator is **NEW-TOOLING-NEEDED** — see backlog item NEW-01.

### Risk 6 — Harmful Bias or Homogenization

*Means:* CarePilot's triage-classification distribution and groundedness metric must not differ materially across patient demographic slices defined in the harm model (age, sex, ethnicity grouping used by the hospital, insurance-type as a proxy-stratification control).

*Measure:* Fairlearn slice report on a clinician-adjudicated eval set; disparate-impact ratio on the triage-classification output; subgroup groundedness delta; homogenisation check via diversity-of-recommendation metric on repeat-patient cohorts.

*Gate:* candidate does not ship unless subgroup metric bands and disparate-impact floor in `risk-register/2025/CarePilot.yaml#fairness-envelope` are met; the envelope is set jointly with clinical safety and the AI risk council.

*Guard:* reweighing / post-processing (mod-106); UI warning when a clinical recommendation diverges materially from the base rate for a slice.

*Evidence:* `evidence/measure-2.11/<release>/fairness-report.json`, co-signed by clinical safety and by `ai-eval-engineer` (L30) methodology.

*Cross-ref to exercise 01:* MEASURE 2.11.

**Tool(s):** Fairlearn (primary); AI Fairness 360 (AIF360) as an alternative. Intersectional slicing beyond two attributes is **NEW-TOOLING-NEEDED** — see backlog item NEW-02.

### Risk 7 — Human-AI Configuration

*Means:* HITL is CarePilot's central safety assumption. Automation bias (clinician accepts without reading), under-reliance (clinician ignores a correct recommendation), and unclear override paths all defeat the HITL claim.

*Measure:* HITL override rate monitored in mod-109; automation-bias mitigation tested in user-research sessions (owned by product + clinical-safety; engineering supplies the technical content); clinician-training completion rate per clinician cohort.

*Gate:* HITL override rate band is defined in the risk register; the band has upper and lower bounds (always-accept and always-reject both indicate HITL breakdown). Release-gate uses the band as a monitor, not as a pre-release block, because the band is only meaningful post-deployment.

*Guard:* UI design that surfaces model confidence, retrieved citations, and "cannot verify" warnings; one-click escalation to a senior clinician; explicit "override reason" field logged with each override.

*Evidence:* human-oversight design doc at `docs/oversight/carepilot.md` (also the Article 14 artifact from exercise 04); HITL monitor spec at `monitors/carepilot/hitl-override.md`; clinician training materials cross-referenced to the same doc.

*Cross-ref to exercise 01:* MANAGE 4.1, GOVERN 3.1 (diverse-team review of the oversight design).

**Tool(s):** no dedicated OSS tool; product + clinical-safety + engineering co-author the oversight design document and the monitor spec. The HITL override-rate monitor is custom and lives in mod-109.

### Risk 9 — Information Security

*Means:* direct prompt injection (clinician types adversarial content, or echoes patient-reported adversarial text) and indirect prompt injection (a poisoned triage protocol or clinical-reference document in the retrieval corpus) are both plausible; PHI exfiltration via model is also in scope.

*Measure:* attack-success-rate on a garak + PyRIT direct-injection suite and an AgentDojo / InjecAgent-style indirect-injection suite; model-extraction baseline (NIST AI 100-2 coverage in mod-105); jailbreak monitor on production trace (mod-109). OWASP ML Top 10 risks applicable to RAG — notably *Input Manipulation Attack* and *Output Integrity Attack* — are the review frame. MITRE ATLAS tactics and techniques (e.g., *Evade ML Model*) are the cross-walk for the suite coverage.

*Gate:* attack-success-rate ≤ target in `risk-register/2025/CarePilot.yaml#security-envelope`; every confirmed finding becomes a permanent regression fixture (mod-111).

*Guard:* input validator (strip-and-flag); retrieval-side provenance filter (reject documents not signed by the hospital's clinical-governance CMS); output validator (must contain citations; must not emit system-prompt-style strings).

*Evidence:* signed red-team report at `evidence/measure-2.7/<release>/report.json`; suite commit hash + config hash pinned; monitor spec at `monitors/carepilot/jailbreak-detection.md`.

*Cross-ref to exercise 01:* MEASURE 2.7.

**Tool(s):** garak (primary direct-injection), PyRIT (campaign orchestration), AgentDojo / InjecAgent (indirect-injection; selection pending in backlog item NEW-03 — integration with the hospital retrieval stack is non-trivial), NIST AI 100-2 coverage in mod-105, MITRE ATLAS as the taxonomy cross-walk, OWASP ML Top 10 as the risk-frame.

### Risk 10 — Intellectual Property

*Means:* CarePilot's outputs must not reproduce licensed clinical-reference text verbatim beyond fair-use thresholds; memorisation of PHI from the retrieval corpus must be bounded.

*Measure:* memorisation eval against a sample of licensed reference material (longest-common-substring, n-gram-overlap) on the base model (mod-107 methodology); licence-flag guardrail on retrieved documents and on outputs.

*Gate:* memorisation eval metric below the floor in `risk-register/2025/CarePilot.yaml#memorisation-envelope`; licence-flag guardrail must be present on every output path.

*Guard:* licence-flag guardrail (reject or caveat outputs that reproduce flagged text); retrieval-side licensing filter on the ingest pipeline.

*Evidence:* memorisation report at `evidence/privacy/<release>/memorisation-report.json`; licence-flag guardrail config pinned in `guardrails/carepilot/licence-flag.yaml`.

*Cross-ref to exercise 01:* MEASURE 2.10 (privacy / IP) — same errata row added for Risk 4.

**Tool(s):** no single OSS tool covers memorisation + licence-flag end-to-end; existing substring / n-gram tooling (e.g., HELM memorization tasks, custom LCS evaluator) suffices for memorisation. Licence-flag guardrail is a thin config layer — not **NEW-TOOLING-NEEDED**.

### Risk 12 — Value Chain and Component Integration

*Means:* CarePilot depends on a procured base LLM, a procured embedding model, and open-source retrieval components. A compromise upstream (poisoned model weights, backdoored dependency, unsigned artifact ingested into the registry) propagates into CarePilot.

*Measure:* ML-BOM coverage for every model and dataset referenced; model-weight integrity check at registry ingest (ModelScan + picklescan); dependency supply-chain scan (mod-110 methodology); vendor-attestation presence on the base model.

*Gate:* candidate does not ship unless ML-BOM is current, model-weight integrity check is green, and base-model vendor attestation covers the release.

*Guard:* registry gate (ModelScan + picklescan on every ingest); signed ML-BOM published per release.

*Evidence:* ML-BOM at `bom/carepilot/<release>/mlbom.json`; ingest-scan log; vendor attestation at `vendor-attestations/<vendor>/<release>.json`.

*Cross-ref to exercise 01:* added row **GOVERN 6.1** (supply-chain mapping) to the exercise 01 map as **errata** — flagged in the appendix.

**Tool(s):** ModelScan (ProtectAI), picklescan, SPDX / CycloneDX ML-BOM shape, Sigstore-based signatures. Mod-110 teaches the full pipeline.

## 3. Appendix — errata to exercise 01

This applicability sheet surfaced three rows missing from the exercise 01 map; chapter 02 calls these out as the cross-reference step that *feeds back into* exercise 01. They are:

| Added row | Reason surfaced | Driven by |
|---|---|---|
| MEASURE 2.10 (privacy and IP risks) | Data-privacy decomposition (Risk 4) and IP decomposition (Risk 10) had no exercise-01 home | AI 600-1 Risks 4 and 10 |
| GOVERN 6.1 (supply chain) | Value-chain decomposition (Risk 12) needs an exercise-01 GOVERN row | AI 600-1 Risk 12 |

A follow-up PR to `mod-101-ex01-rmf-map.md` adds these rows with the five-sentence read.
```

### 2.2 `mod-101-ex02-backlog.md`

```markdown
---
system: CarePilot
framework: NIST AI 600-1 (July 2024)
author: ai-risk-engineer (L25)
status: open
---

# CarePilot — AI 600-1 Backlog (NEW-TOOLING-NEEDED and Inherited-Control Evidence)

## 1. NEW-TOOLING-NEEDED

| ID | Source risk | What is needed | Target | Owner |
|---|---|---|---|---|
| NEW-01 | Risk 4 — Data Privacy | A *cross-session leakage* evaluator — injects identifiers from a synthetic patient A into retrieval, verifies patient B's session cannot elicit them. OSS options are incomplete; a thin wrapper around Presidio + the retrieval API is the likely shape. | Before mod-107 submission | ai-risk-engineer (L25) with `ai-eval-engineer` (L30) consult |
| NEW-02 | Risk 6 — Harmful Bias | Intersectional-fairness tooling beyond Fairlearn's two-attribute default. The slice set defined by the clinical-safety officer requires three- and four-way intersections; existing OSS tools either under-count small slices or fail silently on empty cells. | Before mod-106 submission | ai-risk-engineer (L25) with `ai-eval-engineer` (L30) + `model-evaluation-engineer` (L30) |
| NEW-03 | Risk 9 — Information Security | An AgentDojo / InjecAgent-style indirect-prompt-injection suite integrated with CarePilot's actual retrieval stack. Off-the-shelf harnesses target generic retrieval; a shim is needed for the hospital's protocol store. | Before mod-104 submission | ai-risk-engineer (L25) |

## 2. Inherited-control evidence requests

Where a risk is ruled narrow / out *because the base-model vendor's RLHF or content policy covers it*, an evidence request goes upstream (chapter 02's "inherited control" paragraph is the template). These items are routed to mod-110 for supply-chain tracking.

| ID | Source risk | What we ask the vendor to attest | Target cadence | Fallback if denied |
|---|---|---|---|---|
| INH-01 | Risk 1 — CBRN | Current CBRN-uplift refusal eval summary; coverage of the suite the vendor ran; version of the suite; the vendor's own refusal-regression fixture. | Annual + on major model version | Maintain CarePilot's own refusal-regression fixture; flag risk in register |
| INH-02 | Risk 5 — Environmental Impacts | Training-footprint and typical-inference-footprint disclosure for the base model. | Annual | Document the gap in the model card; disclose only CarePilot inference-side footprint |
| INH-03 | Risk 11 — Obscene / Degrading / Abusive Content | Attestation that the base model refuses NCII / CSAM generation across the vendor's published eval suite. | Annual + on major model version | Enable an additional output-safety classifier; flag as "defence-in-depth, upstream relied upon" in the model card |

## 3. Supply-chain evidence requests (feeds mod-110)

In addition to the inherited-control items above, the following are standing supply-chain asks:

| ID | For | Ask |
|---|---|---|
| SC-01 | Base LLM provider | SBOM / ML-BOM of model artifacts, provenance of training-data governance claims, SOC 2 Type II report (if sector requires) |
| SC-02 | Embedding model provider | Same shape as SC-01, scoped to the embedding model |
| SC-03 | Retrieval stack (OSS) | Dependency SBOM, Sigstore signatures where available |
```

## 3. Validation steps

1. Count rows in the applicability sheet: must be **exactly twelve**. This solution has twelve. No row is skipped.
2. For each row, verify the ruling (In / Out / In narrow / Out narrow) and the reason sentence. Rows marked Out *must* have a reason; this solution has one per Out row.
3. For every In row, verify the five-sentence decomposition is present with named tooling. This solution has decompositions for Risks 2, 3, 4, 6, 7, 9, 10, 12.
4. For every In row, verify the cross-reference to at least one exercise-01 sub-category ID. Where the cross-reference revealed a missing exercise-01 row, the errata section at the bottom of the applicability sheet records what was added.
5. Verify that the backlog file carries NEW-TOOLING-NEEDED items and inherited-control evidence requests. This solution has three of each class, plus three supply-chain standing asks.
6. Scan for payloads: no raw jailbreak or adversarial prompt is included in either file — those belong in the mod-104 payload workflow.

## 4. Rubric / review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Applicability sheet has exactly 12 rows | 12 rows with AI 600-1 short-names verbatim | <12 or renamed |
| R2 | Every row has an applicability ruling | In / Out / In narrow / Out narrow (or equivalent) | Blank ruling |
| R3 | Every row has a reason ≥1 sentence | Reason paragraph present | "n/a" / blank |
| R4 | Every In row has a 5-sentence decomposition | Means / measure / gate / guard / evidence sentences | Missing sentence |
| R5 | Every In row names its tool(s) or is marked NEW-TOOLING-NEEDED | Tool name present with a 1-line rationale | Vague "we would measure" without named tool |
| R6 | Every In row cross-references ≥1 RMF sub-category from exercise 01 | Exercise-01 row ID present | No cross-ref |
| R7 | Missing exercise-01 sub-categories are filed as errata | Errata appendix present and lists additions | Exercise-01 gap noticed but not back-filed |
| R8 | NEW-TOOLING-NEEDED items are in the backlog file | Backlog exists with named NEW-* items | Items mentioned only inline |
| R9 | Inherited-control evidence requests are filed | Backlog lists vendor-ask items | Upstream risks flagged as "not our problem" without evidence request |
| R10 | No raw payloads in the submission | No jailbreak / adversarial prompt strings | Payloads pasted in |
| R11 | Documents readable by a peer under 20 minutes | Clear ruling + clear decomposition | Reviewer needs context to interpret |

**Scoring bands.**

- *Pass* — R1–R11 green.
- *Pass with revisions* — one row violates R3/R4/R5 but shape is intact.
- *Fail* — R1 (row count) or R2 (missing ruling) fails, or R10 (payloads present).

## 5. Common mistakes

- **Skipping a row as "n/a."** The sheet must be twelve rows; "n/a" without a reason sentence is rejected (chapter 02 is explicit).
- **In rows without a named tool.** "We would measure PII leakage" with no tool is not a decomposition; it's a wish. Name the tool (Presidio, Fairlearn, garak, PyRIT, ML Privacy Meter, etc.) — if there is no good option, mark **NEW-TOOLING-NEEDED** and file into the backlog.
- **Inherited controls filed as "handled upstream"** with no evidence request. Chapter 02 says the *inherited-control reasoning is the evidence*, not silence — file the vendor-ask.
- **Pasting raw adversarial prompts into the applicability sheet.** Payloads live in the mod-104 harmful-payload workflow; the applicability sheet is reviewable by anyone.
- **Not cross-referencing exercise 01.** If an in-scope AI 600-1 risk maps to no sub-category in exercise 01, that is a *gap in exercise 01* — both documents get updated.
- **Collapsing Risk 7 (Human-AI Configuration) into Risk 2 (Confabulation).** They are related (HITL is CarePilot's confabulation mitigation) but Risk 7 is an *oversight-design* decomposition and needs its own row.
- **Treating Risk 5 (Environmental Impacts) as always-out without disclosure.** Even small inference footprints need a one-sentence disclosure; chapter 02's examples are explicit.

## 6. References

- **NIST AI RMF Generative AI Profile — NIST AI 600-1, July 2024.** The twelve-risk enumeration and the §3 suggested-actions structure used in the decompositions. Published on `nvlpubs.nist.gov` and summarised on `airc.nist.gov`. (Linked from the official NIST AI Risk Management Framework page: `https://www.nist.gov/itl/ai-risk-management-framework`.)
- **NIST AI RMF 1.0 (AI 100-1, January 2023).** Four-function structure used in the cross-references to exercise 01. `https://www.nist.gov/itl/ai-risk-management-framework`
- **OWASP Machine Learning Security Top 10.** Used as the risk-frame for Risk 9 (Information Security). `https://owasp.org/www-project-machine-learning-security-top-10/`
- **MITRE ATLAS.** Used as the tactic/technique cross-walk for the red-team suite in Risk 9. `https://atlas.mitre.org/`
- **Chapter 02 of this module** — `lessons/mod-101-ai-risk-frameworks-and-position/02-nist-genai-profile-as-harm-checklist.md` — five-sentence decomposition shape, applicability-sheet shape, inherited-control template.
- **Exercise 01 reference solution** — `modules/mod-101-ai-risk-frameworks-and-position/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map/SOLUTION.md` — the exercise-01 map that this exercise cross-references and feeds errata back into.
- Exercise prompt — `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-02-genai-profile-risk-checklist-tour.md`.

<!-- Tooling identifiers (garak, PyRIT, Fairlearn, AIF360, Presidio, ML Privacy Meter, ModelScan, picklescan, HarmBench, AgentDojo, InjecAgent, HELM, Llama Guard, ShieldGemma, OpenAI Moderation, DeepEval, RAGAS, Microsoft Presidio) are named in the exercise prompt and in chapter 02; the applicability sheet uses the names those sources use. -->

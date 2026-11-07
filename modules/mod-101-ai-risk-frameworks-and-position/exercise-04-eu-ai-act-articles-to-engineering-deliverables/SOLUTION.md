# Exercise 04 — EU AI Act Articles → Engineering Deliverables — Reference Solution

## 1. Solution overview

This reference walks a worked `mod-101-ex04-eu-ai-act-map.md` for **CarePilot**, the running system used through exercises 01–05. CarePilot is treated here as a *high-risk AI system under Annex III* (clinical decision-support in a healthcare setting falls under Annex III §5 health; the regulation reads naturally against CarePilot without needing to pivot to a different example). The classification assumption is stated in the front-matter of the deliverable; chapter 04 is explicit that legal classification is counsel's call — this exercise *proceeds against the assumed classification*.

The map covers all seven articles required by the exercise: **9, 10, 14, 15, 55, 61, 72**. Each section has the five required paragraphs/tables: obligation paraphrase, in-scope ruling + reason, artifact(s) with paths and signers, traceability line to exercises 01–03, and open items.

Design decisions:

- **Article 15 gets the longest section** — chapter 04 says so explicitly, and this reference's Article 15 traceability table maps every planned accuracy declaration back to an eval-set + result + evaluation-run.
- **Article 55 is marked out of scope** for CarePilot with a one-paragraph reason. CarePilot is not a general-purpose AI model under the regulation — it is a system built on a procured GPAI. The regulation's systemic-risk obligations live with the GPAI *provider*, not with CarePilot's deployer. The row is still written per chapter 04's rule: "do not skip."
- **All artifact paths resolve to exercises 01 or 03.** No new artifacts are invented here. Where Article 10 or Article 15 would motivate a new artifact, the solution files an *addendum back to exercise 01* rather than inventing a fresh path.
- **Article numbering is cited to the consolidated OJ text** on `eur-lex.europa.eu`; the front-matter notes that where numbering ambiguity appears between early drafts and the published OJ text, the consolidated OJ numbering is authoritative.

## 2. Implementation — worked answer — `mod-101-ex04-eu-ai-act-map.md`

```markdown
---
system: CarePilot (hospital triage-support RAG assistant, HITL)
classification_assumption: "High-risk AI system under Annex III (healthcare-sector decision-support). Legal classification is counsel's determination; this document proceeds against the assumed classification."
regulation_numbering_source: "Regulation (EU) 2024/1689 — consolidated text on eur-lex.europa.eu"
regulation_link: "https://eur-lex.europa.eu/eli/reg/2024/1689/oj"
cross-ref:
  - mod-101-ex01-rmf-map.md
  - mod-101-ex02-applicability.md
  - mod-101-ex03-iso-map.md
author: ai-risk-engineer (L25)
reviewers: counsel, senior-ai-governance-architect (L50), ai-evaluation-engineer (L35)
status: draft-for-review
---

# CarePilot — EU AI Act Article → Engineering Deliverable Map

## Front-matter

- **Running system:** CarePilot — hospital triage-support RAG assistant with human-in-the-loop clinician sign-off.
- **Classification assumption:** Annex III high-risk (health-sector AI). Legal counsel owns the final classification; this document discharges obligations against the assumed classification.
- **Article numbering source:** Regulation (EU) 2024/1689, consolidated text on `eur-lex.europa.eu`. Where OJ and consolidated text differ, this document follows the consolidated text.
- **Shared-evidence rule:** every artifact path in this document resolves to an artifact in exercise 01 or exercise 03. New artifacts needed here are filed back to exercise 01 as an addendum.

## Article 9 — Risk-management system

### 1. Obligation (paraphrased, one paragraph)

Article 9 requires providers of high-risk AI systems to establish, implement, document, and maintain a risk-management system across the AI system's whole lifecycle: identify and analyse known and reasonably foreseeable risks, estimate and evaluate risks under intended use and reasonably foreseeable misuse, adopt appropriate and targeted risk-management measures, test that those measures are effective, and iterate. The system is a living process, not a one-shot document.

### 2. In scope for CarePilot? Reason.

**In scope.** CarePilot is Annex III high-risk under the assumed classification. Article 9 applies in full; the CarePilot risk register is the artifact that discharges the obligation.

### 3. Engineering artifact(s), paths, signers

- **Risk register** — `risk-register/2025/CarePilot.yaml`; signed by the AI risk council chair; owner of the schema is mod-103; updated on every release and quarterly otherwise.
- **Harm model** — `docs/harm-model/carepilot.md` (mod-102 target); feeds the risk register's risk-identification column.
- **Testing-of-measures evidence** — the eval reports under `evidence/measure-2.3/`, `evidence/measure-2.7/`, `evidence/measure-2.11/`, and `evidence/privacy/` (from exercise 01).

### 4. Traceability

Shares evidence with exercise 01 **MEASURE 1.1, MANAGE 1.3, MANAGE 2.2** rows, and with exercise 03 **clause 6.1** (actions to address risks and opportunities) and **Annex A.6.1.2** (impact assessment) / **A.6.2.4** (V&V).

### 5. Open items

- Harm model file (mod-102) is a Planned row; close date aligns with the harm-model exercise.
- Risk register residual-risk statements for several rows are Planned-close by mod-103 submission.

## Article 10 — Data and data governance

### 1. Obligation (paraphrased)

Article 10 requires that training, validation, and testing datasets meet quality criteria — relevance, sufficient representativeness, and, to the best extent possible, freedom from errors and completeness. It explicitly requires examination for bias likely to affect health, safety, or fundamental rights, and measures to detect, prevent, and mitigate such bias; it permits processing special-category data subject to safeguards where strictly necessary for bias detection and correction.

### 2. In scope for CarePilot?

**In scope.** CarePilot's retrieval corpus is a data asset under Article 10 (the regulation's data obligations cover training, validation, and testing data; for a RAG system, the retrieval corpus is in-scope data because it conditions model behaviour). Bias examination across patient demographics is also in scope.

### 3. Engineering artifact(s)

- **Dataset card** for the retrieval corpus — `data-cards/carepilot/retrieval-corpus.md` (Datasheets-for-Datasets shape); signed by the clinical-governance CMS owner.
- **Bias audit** — `evidence/measure-2.11/<release>/fairness-report.json` (from exercise 01); co-signed by `ai-eval-engineer` (L30) methodology and clinical safety.
- **Data-lineage manifest** — `bom/carepilot/<release>/datalineage.json`; ties corpus content to source documents and licensing (links to the ML-BOM from mod-110).
- **Special-category-data safeguard memo** — `docs/privacy/carepilot/special-category-safeguards.md`; addresses HIPAA + Article 10(5)-style safeguards for any protected-attribute use in bias evaluation.

### 4. Traceability

Shares evidence with exercise 01 **MEASURE 2.11** and (errata) **MEASURE 2.10**, with exercise 02 **Risks 4 and 6**, and with exercise 03 **A.6.2.8** (data for AI systems) and **23894 risk sources: Bias, Privacy**.

### 5. Open items

- Dataset card is Planned; closes with the first production release.
- Data-lineage manifest is Planned (mod-110 shape); scaffolded in exercise 03 Annex A.10.2 row.

## Article 14 — Human oversight

### 1. Obligation (paraphrased)

Article 14 requires that high-risk AI systems be designed so natural persons can effectively oversee them during use: understand capacities and limitations, monitor operation and remain aware of automation bias, correctly interpret output, override or disregard outputs, and interrupt the system via a stop button or similar procedure.

### 2. In scope for CarePilot?

**In scope.** HITL is CarePilot's central safety assumption; the oversight design directly discharges Article 14.

### 3. Engineering artifact(s)

- **Human-oversight design document** — `docs/oversight/carepilot.md`; names the operator (clinician), the signal they see (confidence, citations, "cannot verify" warnings), and the override paths they have (accept / modify / reject / escalate to senior clinician). Signed by product + clinical safety + AI risk council.
- **Intervention API** — `src/carepilot/oversight/` (planned); documents the operator-facing halt / rollback / hand-off endpoints. Signed by engineering + product.
- **Automation-bias mitigation note** — `docs/oversight/carepilot-automation-bias.md`; UI wording, training material, and user-research summary. Signed by product + clinical safety.
- **Stop-button design** — included in the oversight design doc; the operator can interrupt an in-flight recommendation and can disable CarePilot for a session with a single UI action.

### 4. Traceability

Shares evidence with exercise 01 **MANAGE 4.1, GOVERN 3.1** rows, with exercise 02 **Risk 7 (Human-AI Configuration)**, and with exercise 03 **A.8.2** (responsible AI design) and **A.6.2.5** (operation and monitoring).

### 5. Open items

- Intervention API spec is Planned — target close aligns with the first production release.
- Clinician-training materials are Planned (owned by L&D, with technical content from this role).

## Article 15 — Accuracy, robustness, and cybersecurity

Article 15 is the longest section — this is where all the evaluation engineering lands. The *traceability table* in subsection 3 is the required artifact: every sentence in the accuracy declaration must trace back to an eval-set + result + evaluation run.

### 1. Obligation (paraphrased)

Article 15 requires that high-risk AI systems be designed and developed to achieve appropriate levels of accuracy, robustness, and cybersecurity throughout their lifecycle. Accuracy metrics must be declared in the instructions for use; robustness must address errors, faults, inconsistencies, feedback loops, and drift; cybersecurity must resist attempts to alter use, outputs, or performance via data poisoning, model poisoning, evasion, prompt injection, model extraction, and confidentiality attacks.

### 2. In scope for CarePilot?

**In scope.** CarePilot is high-risk under the assumed classification; all three of accuracy, robustness, and cybersecurity apply directly.

### 3. Engineering artifact(s) and Article 15 traceability table

- **Accuracy declaration** (in the instructions for use and in the model card) — `model-cards/carepilot/<release>.md`.
- **Robustness report** — adversarial-ML suite output (mod-105) plus noise / distribution-shift results plus drift-monitor spec (mod-109).
- **Cybersecurity report** — LLM red-team suite output (mod-104); NIST AI 100-2 adversarial-ML taxonomy coverage; MITRE ATLAS mapping; OWASP ML Top 10 mapping for RAG; prompt-injection detection in production (mod-108/109).

#### Article 15 traceability table

Every sentence in the model-card *accuracy declaration* must be traceable to an eval-set, a result, and an evaluation run. The template below is the shape; the first three rows are populated with CarePilot's known evaluations from exercise 01.

| Model-card claim (shape, one sentence) | Eval set | Metric + result (shape) | Eval run file | Signer |
|---|---|---|---|---|
| "CarePilot's retrieval-aligned groundedness on the clinician-adjudicated gold set is above the risk-register floor." | `evals/carepilot/groundedness/gold-set-<version>` (clinician-adjudicated) | Groundedness score on a 95% CI basis vs. the floor in `risk-register/2025/CarePilot.yaml#groundedness-envelope` | `evidence/measure-2.3/<release>/shadow-eval-report.json` | `ai-eval-engineer` (L30) + ai-evaluation-engineer (L35) |
| "CarePilot's fairness slice bands on patient-demographic slices (age, sex, ethnicity grouping, insurance-type stratification) are within the risk-register envelope." | `evals/carepilot/fairness/slices-<version>` | Per-slice metric delta + disparate-impact ratio vs. `#fairness-envelope` | `evidence/measure-2.11/<release>/fairness-report.json` | `ai-eval-engineer` (L30) methodology + clinical safety |
| "CarePilot's attack-success-rate on the pinned red-team suite is at or below the risk-register target." | `suites/garak-pyrit-indirect/<commit>` | Attack-success-rate vs. `#security-envelope` | `evidence/measure-2.7/<release>/report.json` | AI risk council + `ai-evaluation-engineer` (L35) |
| *(further rows per additional eval sets as mod-104/105/106/107 land)* | — | — | — | — |

- **Cybersecurity taxonomy cross-walks:**
  - MITRE ATLAS tactics/techniques → red-team suite coverage in `suites/garak-pyrit-indirect/<commit>/ATLAS-coverage.md`.
  - OWASP ML Top 10 → same file; focus on *Input Manipulation Attack*, *Output Integrity Attack*, and *Model Supply-Chain Attack* classes.
  - NIST AI 100-2 adversarial-ML taxonomy → covered by mod-105 output under `evidence/mod-105/`.

### 4. Traceability

Shares evidence with exercise 01 **MEASURE 2.3, MEASURE 2.7, MEASURE 2.11**, with exercise 02 **Risks 2, 3, 6, 9**, and with exercise 03 **A.6.2.4** (V&V) and **A.6.2.5** (operation and monitoring).

### 5. Open items

- Drift-monitor spec is Planned (mod-109); accuracy-declaration rows that depend on production drift metrics are Planned-close with the first production release.
- Mod-104 and mod-105 outputs are Planned; several traceability-table rows carry placeholders.

## Article 55 — Systemic-risk GPAI obligations

### 1. Obligation (paraphrased)

Article 55 imposes additional obligations on providers of general-purpose AI models with systemic risk: model evaluation with adversarial testing, serious-incident tracking and reporting, and cybersecurity of the model and its physical infrastructure.

### 2. In scope for CarePilot?

**Out of scope.** CarePilot is not a general-purpose AI model under the regulation; it is a deployer-built system operating on a procured GPAI. Article 55 obligations live with the GPAI provider, not with CarePilot's deployer. The reasoning is kept here per chapter 04's rule — *do not skip*.

If CarePilot were ever repositioned as a GPAI (it is not currently), the systemic-risk obligations would attach; that pivot is governance's call (L50+), not engineering's.

### 3. Engineering artifact(s)

- **None owned by CarePilot.** The *inherited-control* evidence requests upstream to the GPAI provider — covered by exercise 02 backlog items INH-01 (CBRN uplift), INH-03 (obscene content), and SC-01 (SBOM / provenance / SOC 2) — are the surface this role touches.

### 4. Traceability

Shares reasoning with exercise 02 **Risks 1 (CBRN) and 11 (obscene/degrading)** inherited-control rows and with exercise 03 **A.10.2** (supplier relationships / third-party AI).

### 5. Open items

- Vendor-attestation requests INH-01, INH-02, INH-03 are open; see exercise 02 backlog.
- If the hospital pivots to self-hosting a GPAI, Article 55 would come back into scope and this row would be re-written. Trigger: L50 pivot decision.

## Article 61 — Registration (EU AI database)

Article numbering note: in the published consolidated text, obligations related to registration of high-risk systems are set out around Articles 49 and 71. Article 61 in some early drafts or the Commission's internal numbering may refer to adjacent obligations. This document follows the consolidated OJ numbering; where ambiguity arises, the `legal-register/CarePilot/2025.yaml` row is authoritative. *To-confirm-at-submission:* confirm against the published OJ text whether the deployer-side registration obligation for CarePilot lands on Article 49, Article 61, Article 71, or a combination, and update the article label here accordingly.

### 1. Obligation (paraphrased)

The registration obligation requires providers (and, in defined circumstances, deployers) of certain high-risk AI systems to register the system in the EU AI database before placing it on the market or putting it into service. The registration record carries technical fields (intended purpose, algorithms, resources, training data, foreseeable output, cybersecurity controls, oversight measures) and lifecycle metadata.

### 2. In scope for CarePilot?

**In scope (deployer-side).** CarePilot is deployed by the hospital for internal use; the hospital's status under the regulation is deployer (and possibly provider-of-modifications depending on how much the base model is adapted in-house). The technical fields are supplied from engineering evidence.

### 3. Engineering artifact(s) — technical fields supplied

The registration record itself is a governance artifact (first draft by L15 analyst, signed at L35+). Engineering supplies these fields:

| Registration field | Artifact that supplies it | Path |
|---|---|---|
| Intended purpose | Context brief | `docs/systems/carepilot/context.md` |
| Algorithms and models used | Model card | `model-cards/carepilot/<release>.md` |
| Computing / hardware resources needed | AIMS scope resources section | `docs/governance/aims/scope-carepilot.md#resources` |
| Data used for training (and retrieval corpus) | Dataset card + data-lineage manifest | `data-cards/`, `bom/carepilot/<release>/datalineage.json` |
| Foreseeable output range | Harm model + model card | `docs/harm-model/carepilot.md`, model card |
| Cybersecurity controls | Cybersecurity report | `evidence/measure-2.7/<release>/report.json` |
| Human-oversight measures | Oversight design document | `docs/oversight/carepilot.md` |
| Release-manifest tie | Release manifest | `releases/carepilot/<release>/manifest.json` |

### 4. Traceability

Shares evidence with exercise 01 **MAP 1.1, MEASURE 2.7, MEASURE 2.11, MANAGE 4.1**, with exercise 02 **Risks 2, 6, 9, 12**, and with exercise 03 clauses **4.1, 7.5, 8.2, 9.1**.

### 5. Open items

- Which registration article applies (and whether CarePilot's hospital is provider or deployer under the final OJ text) is counsel's determination; the `to-confirm-at-submission` placeholder above is the flag.
- First-release registration cadence is Planned; aligns with the first production release.

## Article 72 — Post-market monitoring

### 1. Obligation (paraphrased)

Article 72 requires providers to establish and document a post-market monitoring system that actively and systematically collects, documents, and analyses data on the system's performance throughout its lifecycle, and that feeds back into the provider's quality / risk-management process.

### 2. In scope for CarePilot?

**In scope.** Mod-109's monitoring wiring is the artifact. The post-market surveillance (PMS) report is the signed-off packet that fulfils the obligation.

### 3. Engineering artifact(s)

- **Monitor specs** — `monitors/carepilot/` (drift, quality regression, HITL override rate, near-miss queue depth, jailbreak / prompt-injection detection, safety classifier false-negative rate).
- **Post-market surveillance report** — `reports/pms/carepilot/<quarter>.md`; the same report used for internal executive review, with the summary tuned for Article 72 audience. Signed by the AI risk council.
- **Incident-escalation hooks** — ties into Article 73 serious-incident reporting path (mod-111 owns the incident-reporting data shape; the escalation hook in `incidents/carepilot/escalation-hook.md` connects mod-109 monitors to the mod-111 workflow).

### 4. Traceability

Shares evidence with exercise 01 **MANAGE 4.1, MANAGE 2.2, MEASURE 3.2**, with exercise 02 **Risk 9 monitor signal**, and with exercise 03 **clause 9.1** (monitoring, measurement, analysis, evaluation).

### 5. Open items

- PMS report cadence (quarterly) is Planned; first-quarter target aligns with the first production release.
- Serious-incident thresholds (what counts as "serious" under Article 73) are counsel + AI risk council call — the mod-111 playbook has the engineering trigger criteria; the regulator-facing definition is governance's to confirm.

## Harmonised-standards tracker (stretch)

As CEN-CENELEC JTC 21 publishes the harmonised standards supporting the AI Act, this table will backfill with reference numbers and status. Chapter 04's `to-confirm-at-submission` discipline applies — these rows carry placeholders that are resolved at submission time against the current JTC 21 output list on the CEN-CENELEC site.

| Obligation | Candidate harmonised standard family | Status |
|---|---|---|
| Article 9 (risk management) | ISO/IEC 23894 + ISO/IEC 42001 risk clauses | *to-confirm-at-submission — confirm harmonisation status* |
| Article 10 (data governance) | ISO/IEC 42001 A.6.2.8 + Datasheets-for-Datasets practice | *to-confirm-at-submission* |
| Article 14 (oversight) | Human-factors / usability family (ISO 9241, ISO/IEC 25059 controllability / intervenability) | *to-confirm-at-submission* |
| Article 15 (accuracy, robustness, cybersecurity) | ISO/IEC 25059 robustness + NIST AI 100-2 adversarial-ML + emerging JTC 21 outputs | *to-confirm-at-submission* |
| Article 72 (post-market) | ISO/IEC 42001 clause 9 + 23894 monitoring guidance | *to-confirm-at-submission* |
```

## 3. Validation steps

1. Count the article sections: expect 7 — Articles 9, 10, 14, 15, 55, 61, 72. This solution has all seven.
2. For each article, verify the five required components are present: paraphrased obligation, in-scope ruling + reason, artifact list with paths and signers, traceability line, open items.
3. For Article 15, verify the traceability table is present and shaped so every accuracy declaration row can trace back to an eval-set + result + eval-run file.
4. For every artifact cited, verify its path is one of: (a) the exercise 01 map, (b) the exercise 03 map, or (c) marked as an addendum to exercise 01. No orphan new artifacts.
5. Verify Article 55 is marked out of scope with a one-paragraph reason — not silently skipped.
6. Verify Article 61 enumerates technical fields even though the registration record itself is governance-owned.
7. Verify the front-matter cites the regulation numbering source and includes the classification assumption.
8. Verify open items have target dates or triggers.

## 4. Rubric / review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | All seven articles covered | Sections for 9, 10, 14, 15, 55, 61, 72 | Any article missing |
| R2 | Each section has all five paragraphs/tables | Obligation / in-scope / artifacts / traceability / open items | Any sub-part missing |
| R3 | Article 15 traceability table complete | One row per accuracy claim with eval-set, metric+result, run file, signer | Missing or incomplete table |
| R4 | Article 55 ruling with reason | "Out of scope" with paragraph justification | Skipped or empty |
| R5 | Article 61 technical fields enumerated | List of fields with artifact pointers | Omitted |
| R6 | All artifact paths resolve to exercises 01/03 (or filed back as addenda) | Paths traceable to prior exercises or marked as addenda | Invented orphan paths |
| R7 | Regulation numbering source cited | OJ link in front-matter | Not cited |
| R8 | Classification assumption stated | "Annex III high-risk" stated in front-matter | Missing |
| R9 | Open items dated and owned | Target close / trigger + owner per open item | Open items listed vaguely |
| R10 | No verbatim article text | Paraphrased with one paragraph per article | Verbatim article text pasted in |
| R11 | Under 15-minute reviewer path | Reviewer can walk each article and find its artifact fast | Requires re-reading for each article |

**Scoring bands.** Pass = R1–R11 green; Pass with revisions = one section missing a single sub-part; Fail = R1 (an article skipped without a stub) or R3 (Article 15 traceability missing).

## 5. Common mistakes

- **Skipping Article 55.** Writing nothing is not a ruling. Even when out of scope, Article 55 needs the paragraph and the inherited-control evidence request.
- **Inventing new artifacts.** The point of this exercise is re-filing. If a new artifact is actually needed, file it back to exercise 01 as an addendum rather than orphaning it here.
- **Article 15 traceability table missing or shallow.** Every sentence in the accuracy declaration must trace back to an eval set + metric + run file. A traceability table with vague rows ("accuracy is high") defeats the purpose.
- **Confusing provider vs. deployer.** CarePilot's hospital is a deployer; if the hospital customises the base model in-house, it may also become a provider-of-modifications under the regulation. Which obligations attach depends on the role; counsel's determination is authoritative.
- **Treating registration as "GRC fills a form."** The technical fields are the engineering contribution; enumerating them (not the full record) is the exercise.
- **Pasting article text verbatim.** Obligations should be paraphrased; the article citation is the pointer.
- **Omitting the OJ numbering source.** Different trackers use different numbering. Without an OJ pointer, the reviewer cannot verify which article is which.

## 6. References

- **Regulation (EU) 2024/1689 — European AI Act.** Consolidated text on `eur-lex.europa.eu/eli/reg/2024/1689/oj`. Authoritative numbering source for Articles 9, 10, 14, 15, 55, 61, 72.
- **European AI Office — implementation guidance.** Published guidance on GPAI and systemic-risk classification; referenced from Article 55.
- **NIST AI RMF 1.0 (AI 100-1, January 2023).** Used for exercise-01 cross-references. `https://www.nist.gov/itl/ai-risk-management-framework`
- **NIST AI 600-1 (July 2024).** Used for exercise-02 cross-references. `https://www.nist.gov/itl/ai-risk-management-framework`
- **ISO/IEC 42001:2023** and **ISO/IEC 23894:2023.** Used for exercise-03 cross-references. ISO overview pages: `https://www.iso.org/standard/81230.html` and `https://www.iso.org/standard/77304.html`.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — cybersecurity cross-walk for Article 15.
- **MITRE ATLAS.** `https://atlas.mitre.org/` — cybersecurity taxonomy cross-walk for Article 15.
- **Chapter 04 of this module** — `lessons/mod-101-ai-risk-frameworks-and-position/04-eu-ai-act-articles-to-deliverables.md` — the article-to-artifact map shape and the "do not skip Article 55" discipline.
- **Exercise 01, 02, 03 reference solutions** (same repository, prior exercise folders).
- Exercise prompt — `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-04-eu-ai-act-articles-to-engineering-deliverables.md`.

<!-- Article numbering ambiguity (e.g., registration obligation location between Article 49, Article 61, Article 71 in consolidated text) carries a `to-confirm-at-submission` placeholder — the article label is confirmed against the published OJ text at submission time. -->

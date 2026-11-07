# Exercise 03 — ISO/IEC 42001 & 23894 Clause → Artifact Map — Reference Solution

## 1. Solution overview

This reference walks a worked `mod-101-ex03-iso-map.md` for **CarePilot**, the same running system used in exercises 01, 02, 04, and 05. The deliverable has three sections — a 42001 clauses 4-10 map, a 42001 Annex A control-to-artifact map covering 12 controls (≥10 required), and a 23894 risk-source table covering 10 sources (≥8 required).

Key design decisions:

- **No new artifacts are invented in this exercise.** Every row cites an artifact that already appears in the exercise 01 or exercise 02 submissions (or is marked **GAP** / **Planned**). Chapter 03 is explicit — this exercise is re-filing existing evidence against ISO.
- **Clause and control text is paraphrased, not quoted.** The exercise prompt forbids verbatim ISO text (licensing). Each paraphrased row is marked, and the ISO edition is version-stamped at the top. Where the solution author worked from publicly available summaries rather than the paywalled standard text, the *source of the summary* is cited inline and the "clause-level identifiers only; text paraphrased" convention from the exercise prompt is observed.
- **Implementation status is explicit.** Every Annex A control is marked Implemented / Planned / Not applicable with a reason. "Planned" rows carry a target completion date.
- **23894 Annex A risk sources are mapped onto 23053 lifecycle stages (per 23894 Annex B).** Each row carries the lifecycle stage it applies at, the in-scope ruling, and the risk-register row(s) that address it.
- **The "one artifact files against several standards at once" insight (chapter 03) is operationalised.** The red-team report and the risk register file against multiple clauses/standards and the solution says so explicitly in the Annex A map.

Important **caveat on ISO access.** The ISO/IEC 42001:2023, 23894:2023, 42005:2025, and 25059:2023 standards are paywalled. This reference solution works from the *publicly available* clause / control identifiers published on `iso.org`'s standard overview pages and from published lead-implementer training curricula. No verbatim control text is reproduced here. Where paraphrased, the control objective is a one-sentence summary and is marked `(paraphrased)`.

## 2. Implementation — worked answer — `mod-101-ex03-iso-map.md`

```markdown
---
system: CarePilot (hospital triage-support RAG assistant, HITL)
iso_editions_cited:
  - ISO/IEC 42001:2023
  - ISO/IEC 23894:2023
  - ISO/IEC 42005:2025 (stretch)
  - ISO/IEC 25059:2023 (stretch)
source_note: "Standards paywalled. This map uses publicly available clause / control identifiers from the iso.org standard overview pages and published lead-implementer curricula. Control objective text is paraphrased; no verbatim standard text is reproduced."
cross-ref:
  - mod-101-ex01-rmf-map.md
  - mod-101-ex02-applicability.md
  - mod-101-ex02-backlog.md
author: ai-risk-engineer (L25)
reviewers: senior-ai-governance-architect (L50, SoA owner), ai-evaluation-engineer (L35), ai-governance-analyst (L15)
status: draft-for-review
---

# CarePilot — ISO/IEC 42001 & 23894 Clause → Artifact Map

## Section A — ISO/IEC 42001:2023 clauses 4-10 (clause-to-artifact map)

### Reading guide for the lead auditor

This section maps each 42001 clause 4-10 requirement to the CarePilot engineering artifact that satisfies it. Every artifact already appears in the exercise 01 or exercise 02 deliverable; this section is pure re-filing. Where a requirement has no candidate artifact, the row is marked **GAP** and filed into the open-items table at the end of the section. A lead auditor can walk clause by clause and ask only one question: *does the artifact in the cited path actually look like what the clause asks for?*

| Clause (42001:2023) | Clause requirement (paraphrased, ≤1 line) | Artifact | Path | Status |
|---|---|---|---|---|
| 4.1 | Determine external and internal issues relevant to the AIMS (paraphrased) | Context brief for CarePilot | `docs/systems/carepilot/context.md` (also MAP 1.1) | Implemented |
| 4.2 | Determine needs and expectations of interested parties (paraphrased) | Stakeholder list within AI risk council charter | `docs/governance/ai-risk-council/charter.md#stakeholders` | Implemented |
| 4.3 | Determine the scope of the AIMS (paraphrased) | AIMS scope statement for CarePilot | `docs/governance/aims/scope-carepilot.md` | **GAP** |
| 4.4 | Establish, implement, maintain, and continually improve the AIMS (paraphrased) | AIMS operating-procedures index | `docs/governance/aims/README.md` | **GAP** |
| 5.1 | Leadership commitment to the AIMS (paraphrased) | Head of AI governance endorsement memo | `docs/governance/aims/leadership-endorsement.md` | **GAP** |
| 5.2 | AI policy (paraphrased) | Organisation-wide AI policy (SoA-adjacent; owned by L50) | `docs/governance/aims/ai-policy.md` | Planned (owned by L50) |
| 5.3 | Roles, responsibilities, and authorities (paraphrased) | Deferral contract (exercise 05) + role matrix | `modules/.../exercise-05-role-scope-and-deferral-contract/SOLUTION.md` | Planned (exercise 05) |
| 6.1 | Actions to address risks and opportunities, incl. AI risk assessment (paraphrased) | CarePilot risk register | `risk-register/2025/CarePilot.yaml` | Implemented |
| 6.1.* | Statement of Applicability (SoA) for Annex A controls (paraphrased) | SoA for CarePilot (owned by L50) | `docs/governance/aims/soa-carepilot.md` | **GAP** (L50 scope) |
| 6.2 | AI objectives and planning to achieve them (paraphrased) | Release objectives per quarter | `docs/governance/aims/objectives/<quarter>.md` | **GAP** |
| 6.3 | Impact assessment (paraphrased, references 42005) | Skeleton AI-system impact assessment | `docs/impact-assessments/carepilot/skeleton.md` | Planned (stretch section D) |
| 7.1 | Resources (paraphrased) | Staffing / resource table in AIMS scope + budget | `docs/governance/aims/scope-carepilot.md#resources` | **GAP** |
| 7.2 | Competence (paraphrased) | Training matrix (clinician + engineering) | `docs/training/carepilot/matrix.md` | Planned |
| 7.3 | Awareness (paraphrased) | Awareness campaigns + onboarding stub | `docs/training/carepilot/awareness.md` | Planned |
| 7.4 | Communication (paraphrased) | Communication plan + risk-council cadence | `docs/governance/ai-risk-council/charter.md#communication` | Implemented |
| 7.5 | Documented information (paraphrased) | Model card + dataset card + decision log | `model-cards/carepilot/<release>.md`, `data-cards/`, risk-register | Implemented |
| 8.1 | Operational planning and control (paraphrased) | Release-gate contract (points to mod-108/109) | `docs/release/carepilot/gate-contract.md` | Planned |
| 8.2 | AI risk assessment (paraphrased) | CarePilot risk register + harm model (mod-102) | `risk-register/2025/CarePilot.yaml` | Implemented (register), Planned (harm-model deeper) |
| 8.3 | AI risk treatment (paraphrased) | Controls + residual-risk column in register | `risk-register/2025/CarePilot.yaml` | Implemented (shape), closing (evidence) |
| 8.4 | AI system impact assessment (paraphrased, references 42005) | Skeleton impact assessment | `docs/impact-assessments/carepilot/skeleton.md` | Planned (stretch) |
| 9.1 | Monitoring, measurement, analysis, evaluation (paraphrased) | Monitor specs + post-market surveillance report | `monitors/carepilot/`, `reports/pms/carepilot/<quarter>.md` (also MANAGE 4.1) | Planned (specs in; PMS report cadence TBD) |
| 9.2 | Internal audit (paraphrased) | Internal-audit plan + findings log | `docs/governance/aims/internal-audit/` | **GAP** |
| 9.3 | Management review (paraphrased) | Quarterly management review record | `docs/governance/aims/mgmt-review/<quarter>.md` | **GAP** |
| 10.1 | Continual improvement (paraphrased) | Improvement log tied to incidents + monitor signal | `docs/governance/aims/improvement-log.md` | Planned |
| 10.2 | Nonconformity and corrective action (paraphrased) | Incident post-mortems (mod-111) + fixture back-feed | `incidents/carepilot/`, `regressions/carepilot/` | Planned (incidents mod-111) |

### Open items for Section A

| Clause | Gap | Target close | Owner |
|---|---|---|---|
| 4.3, 4.4, 5.1, 6.1.* SoA, 6.2, 7.1 | AIMS-level foundations owned by L50 / head of AI governance; CarePilot is the first system filed against them | 2026 Q1 (illustrative) | senior-ai-governance-architect (L50), head-of-ai-governance (L60) |
| 9.2, 9.3 | Internal audit plan and management review cadence for AIMS not yet stood up | 2026 Q1 | ai-evaluation-engineer (L35) — audit methodology; head-of-ai-governance (L60) — MR cadence |

The clear pattern: the L25 role's contributions are **implemented** for 4.1, 4.2, 7.4, 7.5, 6.1, 8.2, 8.3 — the operational planning, documented-information, and risk-assessment clauses. The clauses marked GAP are *L50 and L60 scope*; they are visible here because this map shows the auditor-facing shape of what is and isn't yet in place.

---

## Section B — ISO/IEC 42001:2023 Annex A control-to-artifact map

### Reading guide for the lead implementer

42001's Annex A is a reference-control list (shape parallels ISO/IEC 27001 Annex A). The *Statement of Applicability* — which controls are in scope and why — is authored by L50 (senior-ai-governance-architect), not here. This section pretends the SoA marks the twelve controls below as applicable (a realistic SoA for a hospital-sector RAG assistant would include these) and shows *the engineering evidence that discharges them*. Every artifact already exists in exercise 01 / 02 or is marked **Planned** with a target.

Status legend: **Imp** = implemented · **Plan** = planned, not yet run · **N/A** = not applicable (reason stated).

| Annex A (42001) | Control objective (paraphrased, ≤1 line) | Status | Artifact | Path | Signer |
|---|---|---|---|---|---|
| A.2.2 — AI policy | Organisation-wide AI policy published and reviewed (paraphrased) | Plan | AI policy (owned by L50) | `docs/governance/aims/ai-policy.md` | head-of-ai-governance (L60) |
| A.2.3 — Alignment with other organisational policies | AI policy consistent with existing IT-security / data-governance policy (paraphrased) | Plan | AIMS-to-ISMS crosswalk memo | `docs/governance/aims/crosswalks/aims-isms.md` | L50 |
| A.3.2 — Roles and responsibilities | Roles and responsibilities assigned for AI system lifecycle (paraphrased) | Imp (via Plan) | Deferral contract (exercise 05) | `modules/.../exercise-05-role-scope-and-deferral-contract/SOLUTION.md` | AI risk council chair |
| A.5.2 — Objectives for responsible development and use | Objectives for responsible development of CarePilot (paraphrased) | Plan | Objectives file per quarter | `docs/governance/aims/objectives/<quarter>.md` | AI risk council |
| A.6.1.2 — AI system impact assessment | Impact assessment process defined and run (paraphrased; aligns with 42005) | Plan | Skeleton impact assessment | `docs/impact-assessments/carepilot/skeleton.md` | AI risk council |
| A.6.2.4 — Verification and validation | V&V processes for AI systems, incl. accuracy and robustness (paraphrased) | Imp | Evaluation evidence set (exercise 01 MEASURE rows) | `evidence/measure-*/` | ai-evaluation-engineer (L35) |
| A.6.2.5 — AI system operation and monitoring | Operational monitoring in production (paraphrased; aligns with MANAGE 4.1) | Plan | Monitor specs + PMS report | `monitors/carepilot/`, `reports/pms/carepilot/<quarter>.md` | AI risk council |
| A.6.2.6 — Technical documentation | Technical documentation maintained for the AI system (paraphrased) | Imp | Model card + dataset card + decision log | `model-cards/carepilot/<release>.md`, `data-cards/` | ai-evaluation-engineer (L35) |
| A.6.2.8 — Data for AI systems | Data quality, provenance, representativeness (paraphrased; aligns with Article 10) | Plan | Dataset cards + data-lineage manifest + bias-audit | `data-cards/`, `bom/`, `evidence/measure-2.11/` | data owner + AI risk council |
| A.7.2 — Information for interested parties of AI systems | Information for deployers / users (paraphrased) | Plan | Instructions for use + user-facing model-card section | `docs/instructions-for-use/carepilot.md` | product + AI risk council |
| A.8.2 — Responsible AI design | Responsible design considerations in system design (paraphrased) | Imp | Human-oversight design doc (also Article 14) | `docs/oversight/carepilot.md` | product + AI risk council |
| A.10.2 — Supplier relationships / third-party AI | Risk management for third-party AI components (paraphrased; aligns with AI 600-1 Risk 12) | Plan | ML-BOM + vendor attestations + supply-chain ingest gate | `bom/carepilot/<release>/mlbom.json`, `vendor-attestations/` | ai-risk-engineer (L25) + head-of-ai-governance (L60) |

### A note on the "one artifact, several standards" pattern

The *red-team report* at `evidence/measure-2.7/<release>/report.json` (from exercise 01) files against:
- 42001 Annex A **A.6.2.4** (V&V) and **A.6.2.5** (operation and monitoring) — as implementation evidence.
- 23894 clause 6 (risk treatment) — as treatment evidence.
- 42005 — impact-assessment robustness section (stretch).
- 25059 — *robustness* quality-characteristic evidence (stretch).
- NIST AI RMF MEASURE 2.7 (exercise 01 row).
- EU AI Act Article 15 cybersecurity portion (exercise 04).

The *risk register* at `risk-register/2025/CarePilot.yaml` files against:
- 42001 clause 6.1 (actions to address risks and opportunities).
- 23894 — the risk process artifact itself.
- 42005 — impact-assessment risk section.
- NIST AI RMF MEASURE 1.1, MANAGE 1.3, MANAGE 2.2.
- EU AI Act Article 9.

The point from chapter 03 is that this is normal, not a double-count: one engineering artifact answers several frameworks' questions about the *same* underlying work.

---

## Section C — ISO/IEC 23894:2023 risk-source table

### Reading guide

23894 Annex A enumerates AI-related risk sources; 23894 Annex B maps them onto 23053 lifecycle stages. The table below cites, for each 23894 risk source in scope for CarePilot: whether it is in scope (with reason), the lifecycle stage(s) it applies at, and the row(s) in the CarePilot risk register (from exercise 02 decompositions) that address it. Any risk-source name below is drawn from the 23894 Annex A public summary; none of the standard's text is reproduced verbatim.

Lifecycle-stage shorthand (23053 / 23894 Annex B): **DA** data acquisition · **DP** data preparation · **MT** model training · **DV** development · **DEP** deployment · **OPR** operation · **MON** monitoring · **DC** decommissioning.

| 23894 risk source (paraphrased, Annex A) | In scope? | Lifecycle stage(s) | Addressed by risk-register row(s) | Cross-ref |
|---|---|---|---|---|
| Bias (incl. societal, dataset, algorithmic) | **In** | DA, DP, MT, OPR, MON | `RR-CP-FAIR-01`, `RR-CP-FAIR-02` | AI 600-1 Risk 6 (ex02); MEASURE 2.11 (ex01) |
| Opacity / explainability | **In** | MT, DV, OPR | `RR-CP-EXPL-01` | AI RMF MEASURE 2.9 (not yet added; errata item to ex01); Article 14 (ex04) |
| Complexity (model + system) | **In** | DV, DEP, OPR, MON | `RR-CP-COMPLX-01` | Harm-model mod-102 input; chapter 03 complexity treatment |
| Autonomy / human-AI configuration | **In** | DV, DEP, OPR | `RR-CP-HITL-01` | AI 600-1 Risk 7 (ex02); Article 14 (ex04) |
| ML-specific issues (data poisoning, distribution shift, adversarial examples) | **In** | DA, MT, OPR, MON | `RR-CP-SEC-01`, `RR-CP-SEC-02`, `RR-CP-DRIFT-01` | AI 600-1 Risk 9 (ex02); MEASURE 2.7 (ex01); mod-105 |
| Security of AI assets (model weights, retrieval corpus, prompts) | **In** | DA, MT, DEP, OPR | `RR-CP-SEC-03`, `RR-CP-SUPCH-01` | AI 600-1 Risk 12 (ex02); mod-110 |
| Privacy | **In** | DA, DP, MT, OPR | `RR-CP-PHI-01`, `RR-CP-MIA-01` | AI 600-1 Risk 4 (ex02); mod-107 |
| Safety (physical / clinical harm pathway) | **In** | DEP, OPR, MON | `RR-CP-SAFETY-01` | Clinical-safety-officer input to harm model; Article 14 oversight chain |
| Environmental impact | **Out (narrow — inherited)** | MT (training footprint; inherited from vendor), OPR (inference footprint) | `RR-CP-ENV-01` (disclosure-only) | AI 600-1 Risk 5 (ex02); backlog item INH-02 |
| Transparency to users / deployers | **In** | DEP, OPR | `RR-CP-TRANSP-01` | Article 13 (stretch, ex04); A.7.2 (section B above) |

That's ten risk sources, above the ≥8 required.

### Residual-risk statement shape (per 23894 clause 6 treatment)

For each in-scope risk source, the risk register carries a one-line `residual_risk_after_control` statement whose shape is: *control implemented → residual risk is <bound> given <condition>; conditions for re-evaluation are <list>.* Example for *ML-specific issues*:

> Residual risk after control: adversarial-robustness gate at `evidence/measure-2.7/<release>/report.json` holds attack-success-rate below the register-defined envelope at the suite's pinned commit; residual exposure is bounded by (a) jailbreak classes outside the pinned suite's coverage and (b) failure modes introduced by future model versions. Re-evaluation triggers: suite coverage expansion in mod-104; any major model version change; a confirmed jailbreak in production.

Every row's residual-risk statement is in the register, not duplicated here — chapter 03's "don't build a second thing" rule.

### Notes on usage of paraphrased summaries

- Risk-source names in the table are paraphrased from the 23894 Annex A public summary; where a named risk source combines multiple phenomena (e.g., "ML-specific issues"), the register row IDs split them into discrete rows.
- The lifecycle-stage shorthand mirrors the 23053 lifecycle; the exact ordering and granularity in 23894 Annex B is honoured where visible in public summaries.
```

## 3. Validation steps

1. Check section A covers **every clause 4-10 requirement**. This solution has one row per numbered clause (4.1, 4.2, 4.3, 4.4, 5.1, 5.2, 5.3, 6.1, 6.1.* SoA, 6.2, 6.3, 7.1, 7.2, 7.3, 7.4, 7.5, 8.1, 8.2, 8.3, 8.4, 9.1, 9.2, 9.3, 10.1, 10.2). GAPs are explicit; L50/L60-owned clauses are labelled.
2. Check section B covers **≥10 Annex A controls**. This solution covers 12.
3. Check section C covers **≥8 23894 risk sources** with lifecycle stage and register row IDs. This solution covers 10.
4. For every artifact cited, verify its path matches the paths used in exercises 01 and 02 (or is a GAP/Planned placeholder). No new artifact invention.
5. Verify every paraphrased control text is marked `(paraphrased)`, and no verbatim standard text is reproduced.
6. Verify the ISO editions are version-stamped in the frontmatter.
7. Verify the "source of summary" caveat is present if the solution author did not access the paywalled standards directly.
8. Verify that each Annex A control has an implementation status (Imp / Plan / N/A); N/A rows need a reason (none are N/A in this solution, which is a conservative choice).

## 4. Rubric / review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Clauses 4-10 map complete | Every clause 4-10 requirement has a row | Any clause requirement missing without a GAP row |
| R2 | GAP rows filed as backlog | GAP rows have target close + owner | GAP mentioned, no owner / date |
| R3 | ≥10 Annex A controls covered | 10+ rows with implementation status | <10 |
| R4 | Every Annex A row has implementation status | Imp / Plan / N/A with reason if N/A | Blank status |
| R5 | ≥8 23894 risk sources covered | 8+ rows with lifecycle + register row IDs | <8 |
| R6 | Every 23894 row has in-scope ruling + lifecycle stage(s) | In / Out with reason; stages named | Blank ruling or stage |
| R7 | Every row cites a path | Path present in exercise 01/02 or a Planned/GAP shape | Invented path with no provenance |
| R8 | ISO editions version-stamped | 42001:2023, 23894:2023 cited at top | Not cited |
| R9 | No verbatim standard text | Paraphrased with `(paraphrased)` marker | Verbatim text copied in |
| R10 | Source-of-summary caveat present | Caveat if standards not accessed directly | Pretending to access paywalled text |
| R11 | Peer ownership named | L50 for SoA / policy; L35 for V&V signing; L15 for intake | Ownership silently absorbed into L25 |
| R12 | Residual-risk statement shape explicit | Section C notes the shape | No residual-risk discipline |

**Scoring bands.**

- *Pass* — R1–R12 green.
- *Pass with revisions* — one row violates R6 or R7 but the overall shape is right.
- *Fail* — R3 (<10 controls) or R5 (<8 risk sources), or R9 (verbatim standard text), or R10 (misrepresenting access).

## 5. Common mistakes

- **Building new artifacts.** This exercise is pure re-filing. If a row cites an artifact that does not exist in exercise 01 or 02, the correct move is to go back to exercise 01 / 02 and *add the artifact there*, then re-file against ISO here. A new artifact invented in this exercise alone is a finding.
- **Pretending to access paywalled standards.** The exercise prompt explicitly says to work from public summaries if you don't have ISO access, and to say so. Omitting the source caveat is a finding.
- **Reproducing standard text verbatim.** 42001:2023 and 23894:2023 are paywalled; verbatim reproduction is a licensing violation and will be rejected by a reviewer.
- **Collapsing clauses 4-10 into "all the structural stuff."** Clauses 4-10 carry different weights — 4.x (context) and 9.x (performance evaluation) are the most-audited. Collapsing the map hides the relative emphasis.
- **Implementing status as "yes/no."** Chapter 03 teaches Imp / Plan / N/A-with-reason. "Yes" with no evidence path is a fail; "Planned" with no target date is a fail.
- **Claiming SoA ownership.** The Statement of Applicability is a L50 artifact. A L25 engineer *contributes* to it — they do not sign it.
- **Treating 23894 risk sources as independent of the risk register.** 23894 asks for the register to be *mapped* onto its Annex A — if the register has 20 risks and the 23894 table has 20 risk sources with no overlap, either the register is wrong or the mapping is.

## 6. References

- **ISO/IEC 42001:2023 — Information technology — Artificial intelligence — Management system.** Standard overview: `https://www.iso.org/standard/81230.html`. Clauses 4-10 and Annex A names are from the standard's overview page; no verbatim clause text is reproduced here.
- **ISO/IEC 23894:2023 — Information technology — Artificial intelligence — Guidance on risk management.** Standard overview: `https://www.iso.org/standard/77304.html`. Annex A risk-source names and Annex B lifecycle-stage mapping are paraphrased from the public overview.
- **ISO/IEC 42005:2025 (stretch) — AI system impact assessment guidance.** Standard overview: `https://www.iso.org/standard/44545.html`.
- **ISO/IEC 25059:2023 (stretch) — AI quality model extension.** Standard overview: `https://www.iso.org/standard/80655.html`.
- **ISO/IEC 23053 — Framework for AI systems using ML** (referenced for lifecycle-stage vocabulary): `https://www.iso.org/standard/74438.html`.
- **NIST AI RMF 1.0 (AI 100-1, January 2023).** Used for the exercise-01 cross-reference. `https://www.nist.gov/itl/ai-risk-management-framework`
- **NIST AI 600-1 (July 2024).** Used for the exercise-02 cross-reference. `https://www.nist.gov/itl/ai-risk-management-framework`
- **Chapter 03 of this module** — `lessons/mod-101-ai-risk-frameworks-and-position/03-iso-standards-your-program-plugs-into.md` — the "re-file, don't re-implement" rule and the cross-standard artifact-reuse table.
- **Exercise 01 reference solution** — `modules/mod-101-ai-risk-frameworks-and-position/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map/SOLUTION.md`.
- **Exercise 02 reference solution** — `modules/mod-101-ai-risk-frameworks-and-position/exercise-02-genai-profile-risk-checklist-tour/SOLUTION.md`.
- Exercise prompt — `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-03-iso-42001-and-23894-clause-to-artifact-map.md`.

<!-- All paraphrased ISO clause / control / risk-source text is marked (paraphrased). The ISO edition years cited (42001:2023, 23894:2023, 42005:2025, 25059:2023) match the public iso.org catalogue entries; if a new edition supersedes any, this map updates to the new year. -->

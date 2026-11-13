# Exercise 01 — SR 11-7 Shape Applied to an ML/AI Model — Reference Solution

## 1. Solution overview

This exercise takes the US Federal Reserve / OCC supervisory letter **SR 11-7** *Guidance on Model Risk Management* (jointly issued 4 April 2011; OCC companion bulletin **OCC 2011-12**) and applies its three-part MRM shape — **(i)** robust model development and implementation, **(ii)** ongoing model use and validation, **(iii)** governance, policies, and controls — to the module's running ML/AI system, **CarePilot** (the hospital-triage-support RAG assistant introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`).

The deliverable is a single markdown document, `carepilot-mrm-sr-11-7.md`, that reads as the **model tier assessment**, the **development and implementation** section, the **validation** section (effective challenge + ongoing monitoring + outcomes analysis + benchmarking), and the **governance** section (policies, roles, model inventory). Each row cites the SR 11-7 paragraph by section number, states what SR 11-7 says, what the ML/AI translation is for CarePilot, what artifact discharges the obligation, and who owns it.

The reference solution:

- Keeps SR 11-7's three-part structure as the top-level headings and does **not** re-order or rename them.
- Explicitly tiers CarePilot (criticality rating) with a written rationale that references the two tiering drivers SR 11-7 names — materiality of model use and complexity / uncertainty.
- Applies **effective challenge** to the ML/AI case by naming (a) the independent validator seat, (b) the challenger-model baseline, and (c) the ongoing-monitoring thresholds that fire re-validation.
- Treats the **three elements of model validation** from SR 11-7 — *evaluation of conceptual soundness*, *ongoing monitoring*, and *outcomes analysis including back-testing or benchmarking* — as three separate subsections, each tied to a CarePilot engineering artifact.
- Flags, in writing, the three places SR 11-7's banking vocabulary needs *translation* for ML/AI (distributional drift, prompt / retrieval change control, human-in-the-loop decision boundary) and names the module artifact that supplies the translation (mod-109 for monitoring, mod-108 for the HITL guardrail, this exercise for the inventory row).
- Explicitly notes that SR 11-7 does **not** carve out special guidance for GenAI, RAG, or LLMs — so any GenAI-specific control appeals to a *different* authority (NIST AI RMF Generative AI Profile, FDA GMLP, EU AI Act Article 9 — all cross-referenced to other mod-110 and mod-101 exercises).

Design decisions worth calling out:

- **CarePilot is treated as a high-tier model.** The materiality (triage recommendation feeding clinical decisions) and complexity (RAG retrieval + LLM + prompt + guardrail chain) together place it in the top tier under any reasonable tiering rubric compatible with SR 11-7. A reviewer who wanted to argue a lower tier would need to appeal to the HITL sign-off; the solution addresses that and concludes it reduces residual risk but does not change the *inherent* tier.
- **Effective challenge is modelled as a seat, not a paper review.** SR 11-7 names "critical analysis by objective, informed parties who can identify model limitations and assumptions and produce appropriate changes" — the solution assigns a named reviewer with authority to block, not a checkbox.
- **The MRM artifact is the Model Documentation; the Validation Report is a separate artifact.** SR 11-7 explicitly distinguishes the two. The solution keeps them separate and cites the sections.
- **ML/AI-specific additions are additive, not substitutive.** SR 11-7's monitoring obligation is discharged by mod-109's drift and quality monitors; mod-110-ex04's ML-BOM and mod-110-ex05's signing feed the inventory. The exercise's job is to show that the MRM *shape* still holds for an ML/AI model; the ML/AI-specific controls supplement, they do not replace.

## 2. Implementation — worked answer `carepilot-mrm-sr-11-7.md`

The submission file follows. Paths and artifact names are consistent with the module's `risk-register/2025/CarePilot.yaml` running fiction; a reviewer checks the *shape* and the citations, not the specific identifiers.

```markdown
---
system: CarePilot (hospital triage-support RAG assistant, HITL)
framework: SR 11-7 / OCC 2011-12 (joint Federal Reserve / OCC guidance on model risk management, 4 April 2011)
scope: Apply the SR 11-7 shape to one ML/AI model end-to-end
author: ai-risk-engineer (L25)
reviewers: model-risk-committee (chair), ai-evaluation-engineer (L35), head-of-model-validation (L60)
status: draft-for-review
---

# CarePilot — SR 11-7 Model Risk Management Documentation

## 1. Model tier assessment

### 1.1 Model identification (SR 11-7 §II — Overview of Model Risk Management; Supervisory Letter definition of "model")

- **Definition anchor.** SR 11-7 defines a "model" as "a quantitative method, system, or approach that applies statistical, economic, financial, or mathematical theories, techniques, and assumptions to process input data into quantitative estimates." CarePilot's RAG + LLM pipeline is a quantitative method producing a ranked triage recommendation; it falls inside the definition.
- **Model ID.** `MRM-CAREPILOT-001`, maintained in the model inventory at `model-inventory/2025.yaml#MRM-CAREPILOT-001`.
- **Business owner.** Chief Medical Officer, regional health system.
- **Development owner.** ML engineering (`ai-risk-engineer` L25 + `ai-evaluation-engineer` L35) for the pipeline; clinical informatics for the retrieval corpus.
- **Independent validation owner.** `head-of-model-validation` (L60), reporting outside the AI engineering line — SR 11-7 §V requires validation independence from the development function.

### 1.2 Tiering rationale (SR 11-7 §IV — Guidance on Model Development, Implementation, and Use; §VII — Governance)

- SR 11-7 anchors risk-rating and prioritisation on the **materiality of model use** and the **complexity and uncertainty of the model**.
- **Materiality — High.** CarePilot's output feeds a clinician triage decision. Even with human sign-off, the model shapes the clinician's prior and the differential it anchors on. Cost of a false high-acuity triage (resource misallocation) and a false low-acuity triage (delayed care) are both material.
- **Complexity — High.** The pipeline chains (a) retrieval from a versioned corpus, (b) an LLM with sampling non-determinism, (c) a safety guardrail, (d) a rationale-emitting formatter. Uncertainty in each stage composes multiplicatively; none of the four has a closed-form error model.
- **Tier.** Tier 1 (highest) under the regional health system's MRM tiering rubric. The rubric itself is in `docs/mrm/tiering-rubric.md`; this cell cites the rubric and asserts the tier assignment.
- **Residual-risk note on HITL.** Clinician sign-off lowers *residual* risk but is a *control*, not a tiering-driver — SR 11-7 ties tier to inherent risk, not post-mitigation risk. The tier is set on materiality + complexity; the HITL is cited as a mitigation in §3.

### 1.3 Scope of this document (SR 11-7 §II)

This document covers the SR 11-7 obligations for `MRM-CAREPILOT-001`:

- **Development, implementation, and use** (SR 11-7 §IV)
- **Validation** — conceptual soundness, ongoing monitoring, outcomes analysis (SR 11-7 §V)
- **Governance, policies, and controls** (SR 11-7 §VII)

ML/AI-specific supervisory expectations outside SR 11-7 (NIST AI RMF 1.0 GenAI Profile, EU AI Act Articles 9/14/15/72, FDA GMLP) are mapped in **mod-101** exercises 01, 02, and 04 and are not reproduced here.

## 2. Development, implementation, and use (SR 11-7 §IV)

### 2.1 Conceptual soundness of the design (§IV — "soundness of design, theory, and logic underlying the model")

- **Design artifact.** `docs/systems/carepilot/design.md` — statement of purpose, inputs (clinician-entered chief complaint + structured intake fields + corpus retrieval), outputs (ranked triage categories + rationale), scope boundaries (ED triage only; not for inpatient disposition).
- **Assumption register.** `docs/systems/carepilot/assumptions.md` — explicit list (e.g., "retrieval corpus is authoritative for ED triage within this hospital", "LLM temperature set for low-variance outputs", "clinician reviews every recommendation before patient-facing action"). Each assumption carries a test and a trigger that invalidates it.
- **Limitation register.** `docs/systems/carepilot/limitations.md` — explicit list of what the model is not for (paediatric < 2 years, obstetric emergencies with specific-protocol override, mass-casualty events). Each limitation is enforced by a routing rule at the guardrail layer (mod-108).

### 2.2 Data (§IV — "data and other information used to develop the model")

- **Training-data attestations.** `docs/systems/carepilot/data-card.md` — provenance of the retrieval corpus (hospital clinical handbook revision X, ED triage protocols revision Y), fine-tuning data (none; base model used as-is), evaluation data (held-out ED cases from the hospital's QA database with PHI redacted per mod-107 exercise 05).
- **Representativeness assessment.** `docs/systems/carepilot/data-representativeness.md` — demographic coverage, acuity-mix coverage, chief-complaint coverage, with gaps named and mitigation plans referenced.
- **Change control on data.** Any change to the retrieval corpus triggers a re-validation pass (§5.2 below).

### 2.3 Implementation (§IV — "process and infrastructure of implementing the model")

- **Code artifact.** `src/carepilot/` under the same version-control discipline as production code; every merge carries a signed risk-review stamp (per mod-101 exercise 01 GOVERN 4.1).
- **Change-control contract.** Protected branch; required reviewers include a model-risk delegate; CI runs the evaluation harness and blocks merge on any regression.
- **Model-artifact provenance.** The deployed model binary is signed (mod-110 exercise 05) and tracked with an ML-BOM (mod-110 exercise 04).

### 2.4 Use (§IV — "procedures for appropriate use")

- **Deployment envelope.** Internal network only, SSO-authenticated, no public endpoint.
- **User training.** Every clinician user completes a 30-minute module on CarePilot's scope, failure modes, and override procedure. Completion is logged.
- **Override procedure.** The clinician's sign-off action is always available; overrides are logged with free-text reason and feed the outcomes-analysis stream (§5.3).
- **Boundary enforcement.** The guardrail (mod-108) refuses to emit a recommendation when the input matches any limitation in §2.1; the refusal is itself an auditable artifact.

## 3. Validation (SR 11-7 §V)

### 3.1 Evaluation of conceptual soundness (§V.A)

- **Independent validator.** `head-of-model-validation` (L60), reporting outside the AI engineering line.
- **Scope.** A written evaluation of the design, assumptions, limitations, and data choices in §2.1–§2.2, with explicit challenge of each assumption.
- **Deliverable.** `validation/carepilot/2025-Q1/conceptual-soundness-report.md`, signed by the validator and the model-risk committee chair.
- **Effective-challenge test.** The validator has authority to withhold sign-off; a withheld sign-off blocks the release gate. The committee minutes record any override of the validator's recommendation and the written justification required by SR 11-7 §V ("processes to address… any instances where a model is used… without validation").

### 3.2 Ongoing monitoring (§V.B)

- **Monitoring artifact.** `monitoring/carepilot/` — the drift, quality, and safety monitors wired in mod-109 exercises 02, 03, and 04. Each monitor emits a signal with a contract (per mod-109 exercise 01) and a threshold tied back to the risk register.
- **ML/AI translation note.** SR 11-7 §V.B frames ongoing monitoring around *process verification* (model is being used as intended) and *benchmarking* (model performance against alternatives). For CarePilot the translations are: *process verification* = retrieval-corpus revision matches the deployed version and the prompt template hash matches the approved hash (release-time check + runtime assertion); *benchmarking* = the challenger model in §3.3.
- **Re-validation trigger.** Any monitor breach that persists for the dwell time defined in `risk-register/2025/CarePilot.yaml#monitoring` triggers re-validation per §3.1.

### 3.3 Outcomes analysis including back-testing or benchmarking (§V.C)

- **Back-testing substitute.** Clinical outcomes for a sample of CarePilot-recommended triage decisions are reconciled, retrospectively, with the final clinical disposition at 24 hours and at 7 days. Discordances feed a case-review queue chaired by the clinical-safety officer.
- **Benchmarking.** A *challenger model* — a simpler non-LLM protocol-matching baseline — is run shadow-mode on the same inputs; its triage distribution is compared to CarePilot's monthly and reported to the committee. The baseline exists to make "CarePilot adds what, vs a simpler alternative?" a monthly, measurable question, which is the discipline SR 11-7 §V.C wants.
- **Deliverable.** `validation/carepilot/2025-Q1/outcomes-analysis-report.md` and `validation/carepilot/2025-Q1/benchmarking-report.md`, both signed by the validator.

### 3.4 Where SR 11-7 needs ML/AI translation — stated explicitly

| SR 11-7 concept | Banking-model shape | CarePilot ML/AI translation | Where discharged |
|---|---|---|---|
| Back-testing | Compare model-predicted outcomes to realised outcomes on historical data | Compare CarePilot-recommended acuity to 24h/7d clinical outcome reconciliation | §3.3 + mod-109 ex02 drift monitor |
| Benchmarking | Compare model to an alternative model on the same inputs | Challenger non-LLM protocol-matching baseline run shadow-mode | §3.3 |
| Ongoing monitoring | Detect degradation from stable baseline | Drift (feature + output) + quality + safety-event monitors | §3.2 + mod-109 ex02, ex03 |
| Model change control | Approval gate for parameter changes | Approval gate for corpus / prompt / model-binary / guardrail changes | §2.3 + mod-108 release gate |
| Model inventory | Centralised inventory with owner + tier + last-validation date | Same, augmented with ML-BOM reference and signature verification | §4.3 + mod-110 ex04, ex05 |

## 4. Governance, policies, and controls (SR 11-7 §VII)

### 4.1 Policies and procedures (§VII.A)

- **MRM policy pointer.** The regional health system's MRM policy document at `docs/mrm/policy.md` is the authoritative source for scope, roles, and tiering. This CarePilot MRM document is scoped to one model and references the policy.
- **Change-management procedure.** Documented in `docs/mrm/change-management.md`; referenced here.

### 4.2 Roles and responsibilities (§VII.B — Board and Senior Management; §VII.C — Model Developer, User, and Validator)

| Role | SR 11-7 anchor | CarePilot owner |
|---|---|---|
| Board / senior management | §VII.B | Model-risk committee (reports to the hospital board quarterly) |
| Business owner (model user) | §VII.C | Chief Medical Officer |
| Model developer | §VII.C | `ai-risk-engineer` (L25) + `ai-evaluation-engineer` (L35) |
| Independent validator | §V + §VII.C | `head-of-model-validation` (L60), outside the AI engineering line |
| Internal audit | §VII.D | `internal-audit` team, annual review of the MRM programme as a whole |

### 4.3 Model inventory (§VII.A — "comprehensive inventory of models")

- **Inventory artifact.** `model-inventory/2025.yaml` — one row per model with: model ID, business owner, developer, validator, tier, last-validation date, next-validation due date, ML-BOM pointer, signature-verification status, risk-register link.
- **CarePilot row (indicative shape).**

```yaml
- id: MRM-CAREPILOT-001
  name: CarePilot (hospital triage-support RAG assistant)
  tier: 1
  business_owner: chief-medical-officer
  developer: ai-risk-engineering
  validator: head-of-model-validation
  last_validated: 2025-Q1
  next_validation_due: 2025-Q3
  ml_bom: artifacts/carepilot/ml-bom.cdx.json      # mod-110 ex04
  signed_artifact: artifacts/carepilot/model.sig    # mod-110 ex05
  risk_register_row: risk-register/2025/CarePilot.yaml
  monitoring_dashboard: monitoring/carepilot/       # mod-109 ex02-05
```

### 4.4 Internal audit (§VII.D)

- Annual audit of the MRM programme as a whole (not of CarePilot in isolation) tests whether: (a) every production ML/AI model has an inventory row, (b) every row cites a validation report within the policy's refresh window, (c) every monitor breach that should have triggered re-validation did so.
- Audit report filed at `audit/mrm/<yyyy>.md` with findings and management responses.

## 5. Open items

- **Playbook / policy references.** `docs/mrm/policy.md` and `docs/mrm/tiering-rubric.md` are the internal policy artifacts the solution cites. In a real submission these would exist; in the exercise they are referenced as the file the engineering team would produce.
- **Challenger-model build-out.** The non-LLM protocol-matching baseline in §3.3 is specified as a design commitment; the actual implementation is a separate engineering work item.

## Notes

- Section numbers above cite SR 11-7 as jointly issued by the Federal Reserve (SR 11-7) and the OCC (Bulletin OCC 2011-12), 4 April 2011. The letter is an unchanged joint supervisory communication; where the OCC's Comptroller's Handbook *Model Risk Management* booklet (August 2021, OCC) expands on the same shape, that expansion is referenced by name but does not change the SR 11-7 obligations relied on here.
- SR 11-7 does not have ML/AI-specific carve-outs; the ML/AI-specific supervisory expectations referenced in §1.3 come from NIST AI RMF, FDA GMLP, and EU AI Act and are mapped in mod-101.
```

## 3. Validation steps

1. **Three-part structure present.** The submission has top-level sections matching SR 11-7's three-part shape — Development/Implementation/Use (§IV), Validation (§V), Governance (§VII). A reviewer checks the ToC against SR 11-7's section numbering.
2. **Tiering rationale cites both tier-drivers.** The §1.2 tiering block names both *materiality* and *complexity and uncertainty* and states the tier assignment with a written rationale. A tier assignment without rationale fails review.
3. **Three validation elements as separate subsections.** §3.1 (conceptual soundness), §3.2 (ongoing monitoring), §3.3 (outcomes analysis / benchmarking) are each present with a named validator, an artifact path, and a release-gate consequence.
4. **Effective challenge is an authority, not a checkbox.** §3.1 names the validator seat, the reporting line (outside the development function), and the authority to withhold sign-off. A reviewer greps for the word "block" or "withhold" and confirms the validator has it.
5. **ML/AI translation table present.** §3.4 names at least the five SR 11-7 concepts that need ML/AI translation and the discharge artifact for each. A reviewer confirms each discharge artifact exists (or is pointed to a mod-110 / mod-109 / mod-108 exercise that produces it).
6. **Model inventory row present.** §4.3 includes a YAML row with tier, developer, validator, last-validation date, next-validation due date, ML-BOM pointer, signed-artifact pointer, risk-register link. A row missing any of those fields fails review.
7. **No invented facts.** Any specific CFR / handbook paragraph or page number cited outside SR 11-7 itself appears with a `<!-- verify-citation: ... -->` marker.

## 4. Rubric — review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | SR 11-7 three-part shape preserved | §IV, §V, §VII appear as top-level sections | Sections re-ordered or renamed |
| R2 | Model definition anchored | §1.1 cites SR 11-7's definition verbatim or close paraphrase, applies it to CarePilot | Model definition skipped |
| R3 | Tiering uses both SR 11-7 drivers | §1.2 names materiality AND complexity/uncertainty with a written rationale | Tier asserted without rationale or drawn only from one driver |
| R4 | Three validation elements each present | §3.1 (conceptual soundness), §3.2 (ongoing monitoring), §3.3 (outcomes / benchmarking) are each non-empty and name artifact + validator | Any of the three omitted or collapsed into one section |
| R5 | Effective challenge is authoritative | Validator can withhold sign-off; committee must record override | Validator described only as a reviewer with no authority |
| R6 | ML/AI translation table present | At least 4 SR 11-7 concepts with banking → ML/AI translation and discharge artifact | Table missing or lists concepts without discharge |
| R7 | Model inventory row | §4.3 YAML row with tier, owner, validator, validation dates, ML-BOM, signed artifact, risk-register link | Missing fields; no inventory section |
| R8 | ML/AI-specific authorities cited elsewhere | NIST AI RMF, FDA GMLP, EU AI Act referenced by name, scope note stating they are supplements to SR 11-7 (not replacements) | Those authorities cited as if they replace SR 11-7, or omitted entirely when relevant |
| R9 | No invented facts | Specific CFR / handbook / OCC-bulletin page numbers outside SR 11-7 itself carry `<!-- verify-citation: ... -->` where unverified | Fabricated paragraph numbers or page citations |
| R10 | Running-system continuity | Uses CarePilot from mod-101 exercise 01; model ID and risk-register row cited | Different fictional system introduced without rationale |

Scoring bands: *Pass* — R1–R8 green, R9/R10 honoured; *Pass with revisions* — R6 or R7 fails but R1–R5 are sound; *Fail* — any of R1, R3, R4, R9 fails.

## 5. Common mistakes

- **Dropping the governance section.** Candidates cover development and validation and treat §VII as a one-paragraph afterthought. SR 11-7 weights governance equally — the committee, the independent validator seat, the inventory, and internal audit are the *controls* that make the rest credible. Reviewers fail submissions that gut §VII.
- **Treating effective challenge as a code review.** A peer PR review is not SR 11-7 effective challenge. Effective challenge is independent (reporting outside development), authoritative (can block), and documented. A submission that says "PRs are reviewed by another engineer" misunderstands the control.
- **Collapsing the three validation elements into "we evaluate the model".** Conceptual soundness, ongoing monitoring, and outcomes analysis are *three different jobs* done by partly different people. Collapsing them hides which obligation is unmet.
- **Confusing tiering with residual risk.** HITL sign-off lowers residual risk — it does not lower the inherent tier. A submission that argues CarePilot is "not Tier 1 because clinicians sign off" misreads §VII and the OCC Handbook's tiering guidance.
- **Pretending SR 11-7 has GenAI-specific language.** It does not. GenAI-specific obligations come from NIST AI RMF 1.0's Generative AI Profile, FDA GMLP, and EU AI Act — the solution cites them as supplements, not as substitutes.
- **Omitting the ML/AI translation note.** SR 11-7's back-testing and benchmarking are written for financial-model shapes; ML/AI needs a translation (outcomes reconciliation, challenger model). Without §3.4, the reviewer cannot tell whether the candidate thought about the mismatch.
- **Skipping the model inventory row.** The inventory is the primary artifact an examiner / auditor would ask for first. A solution without a concrete inventory row is unrecoverable.
- **Citing specific OCC Handbook or Fed-SR paragraph numbers that weren't verified.** SR 11-7 is a short joint letter; expanded guidance lives in the OCC's August 2021 *Model Risk Management* booklet in the Comptroller's Handbook. Candidates sometimes cite specific booklet pages from memory; reviewers flag those as unverified unless the citation is retrieved from the primary source.

## 6. References

Only official standards / regulator documents and clearly-labelled module references are relied on.

- **Federal Reserve SR 11-7 / OCC Bulletin 2011-12 — Supervisory Guidance on Model Risk Management.** Joint issuance, 4 April 2011. Primary source for the three-part MRM shape, the model definition, the effective-challenge concept, and the three validation elements.
- **OCC Comptroller's Handbook — *Model Risk Management* booklet.** OCC, August 2021. Expands the SR 11-7 shape; referenced in §1.2 for tiering practice. <!-- verify-citation: when citing specific Handbook pages, confirm the current booklet revision date and paragraph numbering against the OCC publication page. -->
- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — referenced in §1.3 as the ML/AI-specific supplement; the cross-walk lives in mod-101 exercise 01.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — referenced in the mod-110 index as the attack catalogue that the §3 monitoring and §4 governance disciplines rely on; the MRM shape in SR 11-7 is the *control* framework, OWASP ML Top 10 is the *threat* catalogue.
- **MITRE ATLAS.** `https://atlas.mitre.org/` — same role as OWASP ML Top 10: threat catalogue that the monitoring and validation disciplines need to contemplate.
- **Module running system.** CarePilot — hospital-triage-support RAG assistant, introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`; the risk register is at `risk-register/2025/CarePilot.yaml`.
- **Exercise prompt.** `lessons/mod-110-model-risk-management-and-supply-chain/exercises/exercise-01-sr-11-7-shape-applied-to-an-ml-model.md`.

# Exercise 03 — FDA GMLP + PCCP Slice for One Device-Shaped System — Reference Solution

## 1. Solution overview

This exercise applies the FDA's **Good Machine Learning Practice (GMLP) Guiding Principles** (jointly developed by the US FDA, Health Canada, and the UK MHRA, October 2021) together with the FDA's **Predetermined Change Control Plan (PCCP) for AI/ML-Enabled Device Software Functions** — final guidance **December 2024** — to produce a single slice of the device file for a device-shaped sibling of CarePilot.

Because CarePilot (the module's running system) is an *internal clinical decision-support tool* and the regional health system has not declared an intent to place it on the US market as a medical device, the exercise uses a **hypothetical device-shaped sibling**, **CarePilot-Dx**, scoped so that it would fall under the FDA's software-as-a-medical-device (SaMD) framing. The write-up is explicit that CarePilot-Dx is a device-shaped reference target used for the exercise; nothing in this packet declares CarePilot itself a regulated device.

The deliverable is a single markdown document, `carepilot-dx-gmlp-pccp-slice.md`, that reads as a **device-file slice** with three top-level sections:

1. **GMLP guiding-principle map.** Each of the ten GMLP guiding principles is cited verbatim (short-form), the CarePilot-Dx translation is written, the engineering artifact that discharges it is named, and the gap (if any) is marked.
2. **PCCP slice.** A worked PCCP with the three required components: **Description of Modifications**, **Modification Protocol**, **Impact Assessment**. One specific, scoped modification is chosen — "periodic re-training on the hospital's accumulated labelled ED cases" — and the PCCP is drafted around it.
3. **Submission-readiness checklist.** A short checklist naming the artifacts that the device file would attach for a 510(k) / De Novo submission that includes a PCCP, with a `<!-- verify-citation: ... -->` tag on each FDA-document reference that requires a current-page retrieval at submission.

The reference solution:

- Keeps the ten GMLP guiding principles in order (1 through 10) and does not re-order or re-number them.
- Treats the PCCP as the FDA defines it: a device-file section that pre-specifies **what** can change, **how** it can change (methods / tests / acceptance criteria), and **what** the risk assessment of those changes is — so that the pre-specified changes can be implemented **without** a new marketing submission, provided the device change stays within the PCCP.
- Chooses a single, bounded modification type — periodic re-training on new local-population data — rather than trying to pre-authorise arbitrary architecture changes. The PCCP's power is in the specificity.
- States, in writing, that this exercise produces a **slice** of the device file (the GMLP map + one PCCP) and is **not a complete device submission**. Clinical-evaluation, usability-engineering, software-documentation-level, cybersecurity (FDA's September 2023 Premarket Cybersecurity guidance), and risk-management (ISO 14971) obligations are referenced but not reproduced.
- Keeps claims tied to FDA primary sources; where paragraph-level citations are needed, the solution uses `<!-- verify-citation: ... -->` tags so a reviewer can confirm against the current FDA document at submission.

Design decisions worth calling out:

- **Device-shape surrogate.** CarePilot's current deployment envelope (internal clinical decision support, HITL) and the hospital's stated intent keep it outside the device framing. Using a hypothetical CarePilot-Dx preserves the exercise's learning value without making a regulatory claim about CarePilot itself.
- **One modification, well-scoped.** A PCCP that tries to pre-authorise "any future improvement" fails FDA review. The solution shows what a *reviewable* PCCP looks like for a single narrow modification.
- **The PCCP does not replace adverse-event reporting or QMS controls.** It only authorises pre-specified changes without a new submission; everything else (QMS under 21 CFR Part 820, MDR under 21 CFR Part 803, labelling) applies unchanged.

## 2. Implementation — worked answer `carepilot-dx-gmlp-pccp-slice.md`

```markdown
---
system: CarePilot-Dx (hypothetical device-shaped sibling of CarePilot — triage decision-support SaMD)
indication_for_use: (illustrative; not a regulatory claim) "clinical decision support for triage acuity categorisation in the hospital emergency department; intended to be used by trained clinicians; not intended to be the sole basis for triage decisions"
regulatory_framing: US FDA SaMD, hypothetical 510(k) / De Novo pathway with a PCCP
author: ai-risk-engineer (L25)
reviewers: quality-system-management, regulatory-affairs, clinical-safety-officer, head-of-model-validation (L60)
status: draft-for-review
scope_note: "This is an exercise slice — GMLP map + one PCCP. It is NOT a complete device submission."
---

# CarePilot-Dx — FDA GMLP guiding-principle map + PCCP slice

## 0. Scope and what this packet is not

- **In scope.** The ten GMLP guiding principles (FDA / Health Canada / MHRA, October 2021) mapped to CarePilot-Dx engineering artifacts; one PCCP for periodic local-population re-training.
- **Out of scope for this slice** (named for the reviewer's benefit, not deferred away): clinical evaluation per the SaMD clinical-evaluation framework, human-factors / usability engineering per IEC 62366-1, risk management per ISO 14971, premarket cybersecurity per FDA's September 2023 guidance, QMS under 21 CFR Part 820, software documentation level per the FDA's 2023 content-of-premarket-submissions guidance.

## 1. GMLP guiding-principle map

The ten guiding principles below are those jointly issued by the FDA, Health Canada, and the UK MHRA in October 2021 under the title "Good Machine Learning Practice for Medical Device Development: Guiding Principles."

### GMLP 1 — Multi-Disciplinary Expertise Is Leveraged Throughout the Total Product Life Cycle

- **Says (short-form).** The development and oversight team integrates clinical, data-science, software, human-factors, and quality expertise.
- **CarePilot-Dx translation.** The CarePilot-Dx working group has (a) ED clinician leads, (b) triage-nurse representation, (c) ML engineering, (d) a quality / regulatory engineer, (e) a human-factors engineer, (f) the clinical-safety officer.
- **Artifact.** `docs/dx/governance/charter.md`; meeting minutes archived; decision log pinned to the device risk file.
- **Discharge test.** Every design-input / design-output change carries sign-off from at least one non-engineering seat on the working group.

### GMLP 2 — Good Software Engineering and Security Practices Are Implemented

- **Says.** The device software is built with modern software-engineering practice and secure-development-lifecycle practice.
- **CarePilot-Dx translation.** Version control with protected main, mandatory review, CI gating, SBOM per the FDA's September 2023 premarket cybersecurity guidance (SBOM required under section 524B of the FD&C Act for cyber-devices), vulnerability-management procedure, coordinated-disclosure contact.
- **Artifact.** `src/carepilot-dx/`; CI pipeline at `.github/workflows/dx-build.yml`; SBOM at `artifacts/carepilot-dx/sbom.cdx.json` (CycloneDX) + the AI/ML-BOM in mod-110 exercise 04.
- **Discharge test.** Every release produces an SBOM and a signed artifact (mod-110 ex05); every CVE above the threshold set in the risk file is tracked to remediation.

### GMLP 3 — Clinical Study Participants and Data Sets Are Representative of the Intended Patient Population

- **Says.** Datasets used for development and testing reflect the intended-use population.
- **CarePilot-Dx translation.** The training / fine-tuning / evaluation datasets' demographic coverage (age, sex, race/ethnicity, acuity-mix, chief-complaint-mix, insurance status proxy) is documented and compared to the hospital's ED catchment distribution; gaps are enumerated.
- **Artifact.** `docs/dx/data/representativeness.md`; demographic-coverage table reviewed quarterly.
- **Discharge test.** Any under-represented subgroup with > ε% (ε set in the risk file) deviation from catchment distribution triggers a bias / fairness subgroup analysis (mod-106).

### GMLP 4 — Training Data Sets Are Independent of Test Sets

- **Says.** Training and test datasets are independent; data leakage is prevented.
- **CarePilot-Dx translation.** Train / validation / test splits done by patient-encounter ID, not by row; temporal split for the top-level test set (train on year-N, test on year-N+1) to avoid leakage via temporal auto-correlation.
- **Artifact.** `docs/dx/data/split-policy.md`; `eval/dx/splits/` with hashed index files and a leakage-check unit test.
- **Discharge test.** A unit test in `tests/dx/test_no_leakage.py` fails if the intersection of train and test patient-encounter IDs is non-empty.

### GMLP 5 — Selected Reference Datasets Are Based Upon Best Available Methods

- **Says.** Reference ("gold-standard") datasets used for performance claims have a documented labelling methodology and known quality.
- **CarePilot-Dx translation.** The ED-outcomes reference dataset has (a) a labelling SOP, (b) inter-rater reliability on a stratified sample above the pre-set threshold, (c) a change log.
- **Artifact.** `docs/dx/data/reference-dataset-card.md`; IRR report; labelling SOP.
- **Discharge test.** Any performance claim references the dataset version by hash and names the labelling SOP revision.

### GMLP 6 — Model Design Is Tailored to the Available Data and Reflects the Intended Use

- **Says.** Model design is appropriate to the data and the intended use.
- **CarePilot-Dx translation.** Pipeline description (retrieval + LLM + guardrail + formatter) is justified against the intended use; alternatives considered (protocol-matching baseline) are documented and the trade-off recorded.
- **Artifact.** `docs/dx/design/design-input-output.md`; alternatives analysis in `docs/dx/design/alternatives.md`.
- **Discharge test.** Design-input to design-output traceability matrix is complete and sign-off is in place.

### GMLP 7 — Focus Is Placed on the Performance of the Human-AI Team

- **Says.** Performance claims consider the human + AI team, not the AI alone.
- **CarePilot-Dx translation.** Human-factors protocol per IEC 62366-1; a human-in-the-loop performance study is designed before clinical release — clinicians working *with* CarePilot-Dx vs clinicians working without — and both arms are measured on the same primary endpoint (triage concordance with the 24-hour clinical disposition).
- **Artifact.** `docs/dx/clinical/human-ai-team-protocol.md`; eval harness supports both arms in the same scoring pipeline.
- **Discharge test.** Performance claims in the labelling cite the human-AI arm result, not the AI-alone arm.

### GMLP 8 — Testing Demonstrates Device Performance During Clinically Relevant Conditions

- **Says.** Performance testing covers the conditions encountered in clinical use.
- **CarePilot-Dx translation.** The test set spans the operational envelope — shifts, weekend/holiday patterns, diurnal variation, outbreak periods — and is stratified by subgroup.
- **Artifact.** `eval/dx/clinical-conditions-matrix.md`; test-set construction documented and reviewed.
- **Discharge test.** Any clinical-relevant condition absent from the test set is flagged as a labelling limitation.

### GMLP 9 — Users Are Provided Clear, Essential Information

- **Says.** Users receive the information needed for safe and effective use — intended use, performance, limitations, warnings, and the role of the AI.
- **CarePilot-Dx translation.** Labelling (IFU) names (a) the intended use, (b) the clinically-relevant performance, (c) the limitations (paediatric < 2 y, obstetric emergencies, mass-casualty override), (d) the role of clinician sign-off, (e) the mechanism for reporting issues.
- **Artifact.** `docs/dx/labelling/IFU.md`; patient-labelling if applicable; in-product UX strings traced to the IFU.
- **Discharge test.** Usability-engineering study confirms clinicians understand the limitations section (summative-usability acceptance criterion).

### GMLP 10 — Deployed Models Are Monitored for Performance and Re-training Risks Are Managed

- **Says.** Performance is monitored post-deployment and re-training is governed.
- **CarePilot-Dx translation.** Post-market monitoring under 21 CFR Part 803 (MDR) for adverse events + a performance-monitoring dashboard (mod-109 exercises 02/03/05); re-training is governed by the PCCP in §2 of this packet when within scope, and by a new marketing submission when outside the PCCP.
- **Artifact.** `monitoring/dx/` dashboards; `docs/dx/postmarket/monitoring-plan.md`; MDR reporting procedure.
- **Discharge test.** Every monitoring alert that crosses the threshold produces a case for the quality-committee queue; the queue has an SLA.

## 2. PCCP — Predetermined Change Control Plan slice

The PCCP below authorises **one** specific category of modification — *periodic re-training on the hospital's own accumulated labelled ED cases, on a quarterly cadence, with pre-specified acceptance criteria*. It does not authorise architecture changes, new indications for use, new patient subpopulations, new input modalities, or re-training on data from outside the hospital's catchment.

### 2.1 Description of Modifications

**In scope** — this PCCP authorises the following modification, and only this modification:

- **Modification.** Re-fine-tune the deployed model on a quarterly cadence using the hospital's own newly-labelled ED cases from the preceding quarter, holding fixed:
  - the base model and its version,
  - the retrieval corpus boundaries (ED triage protocols, clinical-handbook ED sections),
  - the prompt template and the guardrail configuration,
  - the input modalities (clinician-entered chief complaint + structured intake fields),
  - the output modality (ranked acuity + rationale),
  - the intended use and indication for use.

- **What is not in scope.** Changes to the base model version, architecture, retrieval corpus boundaries, prompt template, guardrail configuration, input modalities, output modalities, or intended use. Any such change exits the PCCP and requires a new submission.

- **Frequency.** Up to four re-trainings per calendar year, each separated by at least sixty days.

- **Change-control gate.** Each re-training is a controlled change under QMS; sign-off by the quality-committee delegate.

### 2.2 Modification Protocol

The modification protocol specifies, prospectively, **how** each re-training is performed and validated so that a reviewer can confirm the change will stay within the PCCP without seeing the specific re-training output.

#### 2.2.1 Data management

- Source: hospital-ED cases from the preceding quarter, labelled per the reference-dataset SOP cited in GMLP 5.
- Inclusion / exclusion criteria: same as the original training-set criteria, documented in `docs/dx/data/inclusion-exclusion.md`; new cases that fall outside these criteria are excluded and noted.
- Demographic-coverage check: the incremental dataset's demographic distribution is compared to the hospital catchment distribution; a deviation above the pre-set ε triggers a bias / fairness subgroup check per GMLP 3.
- Split: new cases are partitioned into a re-training subset and a frozen quarterly evaluation subset; the frozen subset is *never* used for re-training.

#### 2.2.2 Re-training methodology

- Re-fine-tune the deployed model using the pre-specified recipe stored in `training/dx/recipe.yaml` under version control.
- Random seed is pre-recorded; hyper-parameters are pinned; any hyper-parameter change is outside the PCCP.
- Artifact: the re-trained model binary with a signed attestation (mod-110 ex05), ML-BOM (mod-110 ex04), and an evaluation report.

#### 2.2.3 Performance evaluation — acceptance criteria

Each re-training must meet all of the acceptance criteria below *before* any clinical traffic is routed to it. The criteria are set against the **currently-deployed** model as the comparator; the PCCP cites numeric thresholds for the reviewer, which in a real submission would be the ones negotiated during FDA review.

| Criterion | Comparator | Acceptance threshold | Data |
|---|---|---|---|
| Top-level triage concordance with the 24-hour clinical disposition | current deployment | not worse by more than Δ_concordance | frozen quarterly evaluation subset |
| Subgroup concordance | current deployment | not worse by more than Δ_concordance in any subgroup defined in GMLP 3 | frozen quarterly evaluation subset, stratified |
| Calibration — Brier score for acuity categories | current deployment | not worse by more than Δ_calibration | frozen quarterly evaluation subset |
| Safety-signal rate (false low-acuity) | current deployment | not worse by more than Δ_safety | frozen quarterly evaluation subset |
| Human-AI team concordance (summative-usability substitute: clinician override rate) | currently-deployed clinician override rate | not worse by more than Δ_override | prospective shadow-run on the next 30 days of live traffic before go-live |

- Δ values are set and signed off as part of the PCCP; in the device file they are concrete numeric values negotiated with FDA. <!-- verify-citation: in a real submission the specific Δ values and their statistical justification would be confirmed against the latest FDA Guidance on PCCP (final guidance, December 2024). -->

#### 2.2.4 Update procedure

- Re-training occurs in a controlled environment; artifacts are versioned.
- A **shadow-mode** deployment runs the candidate re-trained model in parallel with the currently-deployed model for 30 days of live traffic; divergence is monitored (mod-109 ex02) and the acceptance criteria evaluated prospectively.
- On passing all acceptance criteria *and* on quality-committee sign-off, the candidate replaces the current deployment; the old model is retained for rollback for the length of the retention policy.
- A **rollback plan** is documented: on any post-go-live acceptance-criterion breach within 30 days, the system rolls back to the previously-deployed model automatically and the change is reported as an adverse event per §2.3.

#### 2.2.5 Change transparency

- Each re-training produces a labelling supplement under version control at `docs/dx/labelling/CHANGE-LOG.md` naming the re-training date, the re-training data window, the evaluation results, and any newly-identified limitation.
- Clinicians are notified via the hospital's change-communication channel; the IFU is updated if any performance claim or limitation changes.

### 2.3 Impact Assessment

- **Benefits.** Local-population re-training is expected to maintain or improve performance under natural drift (patient-mix drift, protocol updates, clinician-input-style drift) that would otherwise degrade the currently-deployed model.
- **Risks.**
  - *Performance regression on an unmeasured subgroup.* Mitigated by the stratified acceptance criterion and by shadow-mode monitoring.
  - *Over-fitting to the hospital's own population leading to reduced generalisation.* Scoped-out: CarePilot-Dx is intended for use at the hospital whose data trains it; cross-site generalisation is explicitly outside the PCCP and would require a new submission.
  - *Drift in the labelling SOP affecting the re-training ground truth.* Mitigated by labelling-SOP versioning and the quality-committee gate.
  - *Adversarial-data influence on re-training.* Mitigated by the data-governance controls in §2.2.1 and by the data-pipeline integrity signals from mod-110 ex06.
  - *Cybersecurity regression* in the re-trained model's deployment. Mitigated by SBOM / signing at each release (GMLP 2) and by the FDA September 2023 Premarket Cybersecurity guidance's post-market plan.
- **Overall risk disposition.** With the modification protocol in §2.2 and the shadow-mode gate, the residual risk of a single re-training is acceptable and bounded by the acceptance criteria.

## 3. Submission-readiness checklist (what the device file would attach)

- [ ] GMLP map (this document §1) — signed by the working-group chair.
- [ ] PCCP (this document §2) — signed by quality, regulatory, and clinical-safety.
- [ ] Clinical-evaluation report. <!-- verify-citation: structure and content to be confirmed against the current FDA SaMD / 510(k) clinical-evaluation expectation at submission. -->
- [ ] Human-factors / usability-engineering file per IEC 62366-1.
- [ ] Risk-management file per ISO 14971.
- [ ] Software documentation level per the FDA's 2023 content-of-premarket-submissions for software guidance. <!-- verify-citation: confirm the current FDA software-documentation-level guidance title and date at submission. -->
- [ ] Premarket-cybersecurity file per FDA September 2023 guidance (post-section-524B). <!-- verify-citation: confirm title and date at submission. -->
- [ ] SBOM (CycloneDX) per cyber-device obligations (section 524B of the FD&C Act); ML-BOM per mod-110 ex04.
- [ ] Labelling (IFU) traced to GMLP 9.

## Notes

- GMLP guiding-principle short-form text is paraphrased; the authoritative source is the FDA / Health Canada / MHRA joint document, "Good Machine Learning Practice for Medical Device Development: Guiding Principles," October 2021.
- The PCCP structure (Description of Modifications, Modification Protocol, Impact Assessment) follows the FDA's final guidance "Marketing Submission Recommendations for a Predetermined Change Control Plan for Artificial Intelligence-Enabled Device Software Functions" (final guidance, December 2024).
```

## 3. Validation steps

1. **Scope note present.** The document opens with an explicit statement that the slice does not declare CarePilot a medical device and uses CarePilot-Dx as a device-shaped surrogate.
2. **All ten GMLP guiding principles present and in order.** §1 has ten subsections numbered 1–10, each with Says / Translation / Artifact / Discharge test rows. A missing principle fails.
3. **PCCP has three required components.** §2.1 Description of Modifications, §2.2 Modification Protocol, §2.3 Impact Assessment — all three present and non-empty.
4. **PCCP is scoped to one modification.** §2.1 explicitly enumerates what is in scope and what is not; an unbounded PCCP ("any future improvement") fails.
5. **Acceptance criteria in the Modification Protocol are concrete.** §2.2.3 has a table with comparator, threshold, and data; placeholders are permitted only as `Δ` symbols with a verify-citation marker.
6. **Shadow-mode gate and rollback plan named.** §2.2.4 names shadow-mode, acceptance-criterion evaluation on shadow traffic, and the rollback plan.
7. **Monitoring tied to mod-109.** GMLP 10 and §2.2.4 reference mod-109 exercises for the telemetry.
8. **No invented facts.** Any FDA guidance title, date, or page citation beyond the two primary sources named in Notes carries `<!-- verify-citation: ... -->`.

## 4. Rubric — review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Scope note | Opening scope note names CarePilot-Dx as a device-shaped surrogate and states what is out-of-scope | No scope note; or exercise claims CarePilot itself is a device |
| R2 | Ten GMLP principles present | §1 enumerates all ten, in order, with Says/Translation/Artifact/Discharge test | Any principle missing or re-ordered |
| R3 | GMLP translations cite artifacts | Each principle names a specific engineering artifact path | Translations described in generalities with no artifact |
| R4 | PCCP structure | §2 has Description of Modifications, Modification Protocol, Impact Assessment, each non-empty | Any component missing or collapsed |
| R5 | PCCP scope bounded | §2.1 enumerates in-scope AND out-of-scope changes | Unbounded "any future improvement" PCCP |
| R6 | Acceptance criteria concrete | §2.2.3 table has comparator, threshold, data; thresholds expressed as concrete Δ with justification path | Vague "model must be good" criteria |
| R7 | Shadow-mode gate + rollback | §2.2.4 names shadow-mode window, acceptance-criterion evaluation on shadow traffic, rollback plan | Go-live without shadow validation |
| R8 | Impact assessment substantive | §2.3 enumerates at least four risk categories with mitigations | One-paragraph vague impact |
| R9 | Linkage to other mod-110 exercises | SBOM / signing (ex05), ML-BOM (ex04), ingestion integrity (ex06), monitoring (mod-109) all cited | Device slice floats free of the module's engineering artifacts |
| R10 | No invented facts | FDA guidance titles / dates beyond the two primary sources tagged `<!-- verify-citation: ... -->`; no fabricated 510(k) numbers | Fabricated FDA references |

Scoring bands: *Pass* — R1–R8 green, R9/R10 honoured; *Pass with revisions* — R6 thresholds left as placeholders without `verify-citation`; *Fail* — any of R1, R2, R4, R5.

## 5. Common mistakes

- **Declaring CarePilot a medical device.** CarePilot's current deployment envelope (internal, HITL, not placed on the US market as a device) keeps it outside the device framing. The slice uses CarePilot-Dx as a surrogate and labels it clearly.
- **Unbounded PCCP.** A PCCP that attempts to pre-authorise architecture changes, new indications, or cross-site deployment is not reviewable. The specificity of the Modification Protocol is the point.
- **Collapsing the three PCCP components.** Description of Modifications, Modification Protocol, Impact Assessment are three different artifacts. A submission that fuses them loses the review shape.
- **Vague acceptance criteria.** "Does not get worse" is not an acceptance criterion. Thresholds must be numeric Δ against a named comparator on a named dataset.
- **Skipping shadow-mode.** Promoting a re-trained model straight to live traffic without a shadow-mode window defeats the PCCP's assurance story.
- **Treating GMLP as a checklist, not an engineering plan.** Each guiding principle must land on an artifact. "We agree with this principle" is not a discharge.
- **Confusing the PCCP with QMS or MDR obligations.** The PCCP authorises *pre-specified* changes without a new marketing submission. It does not relieve the manufacturer of 21 CFR Part 820 (QMS), 21 CFR Part 803 (MDR), labelling, or cybersecurity obligations.
- **Omitting GMLP 7 (human-AI team).** Performance is a property of the clinician-plus-AI team, not of the AI alone. A submission that quotes AI-alone metrics as the headline performance claim mis-frames the device.
- **Stale FDA citations.** FDA guidance evolves; the PCCP final guidance was issued in December 2024 and supersedes the April 2023 draft. The solution flags the primary-source citations in Notes and tags everything else with `<!-- verify-citation: ... -->`.

## 6. References

- **FDA / Health Canada / MHRA.** *Good Machine Learning Practice for Medical Device Development: Guiding Principles.* Jointly issued, October 2021. Primary source for the ten guiding principles in §1.
- **US FDA.** *Marketing Submission Recommendations for a Predetermined Change Control Plan for Artificial Intelligence-Enabled Device Software Functions.* Final guidance, December 2024. Primary source for the three-component PCCP structure in §2.
- **US FDA.** *Content of Premarket Submissions for Device Software Functions.* 2023 guidance. Referenced for the software-documentation-level obligation in §0 and §3. <!-- verify-citation: confirm full title and current revision date at submission. -->
- **US FDA.** *Cybersecurity in Medical Devices: Quality System Considerations and Content of Premarket Submissions.* September 2023 (replacing the 2014 guidance). Referenced for GMLP 2 and §3. <!-- verify-citation: confirm full title and current revision date at submission. -->
- **Federal Food, Drug, and Cosmetic Act §524B (cyber-devices).** Legal anchor for the SBOM requirement cited in GMLP 2. <!-- verify-citation: confirm the current statutory text when quoting; added by the Consolidated Appropriations Act, 2023. -->
- **ISO 14971.** Medical-device risk-management standard; referenced for the risk-management file in §0 and §3.
- **IEC 62366-1.** Usability-engineering standard for medical devices; referenced for GMLP 7 and §3.
- **21 CFR Part 820 (QMS) and Part 803 (MDR).** Referenced in GMLP 10 and §2.3 as ongoing obligations unaffected by the PCCP.
- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — referenced for the cross-walk to GOVERN / MAP / MEASURE / MANAGE from mod-101.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — attack catalogue referenced by GMLP 2's secure-development-lifecycle and by §2.3's adversarial-data risk.
- **Module running system.** CarePilot — hospital-triage-support RAG assistant, `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`.
- **Exercise prompt.** `lessons/mod-110-model-risk-management-and-supply-chain/exercises/exercise-03-fda-gmlp-and-pccp-slice-for-one-device.md`.

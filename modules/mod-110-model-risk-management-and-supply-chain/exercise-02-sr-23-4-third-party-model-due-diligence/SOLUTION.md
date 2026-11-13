# Exercise 02 — SR 23-4 Third-Party Model / API Due Diligence — Reference Solution

## 1. Solution overview

This exercise takes the US interagency guidance **SR 23-4** / **OCC Bulletin 2023-17** / **FDIC FIL-29-2023** — *Interagency Guidance on Third-Party Relationships: Risk Management* (jointly issued **6 June 2023**) and uses its lifecycle — **planning → due diligence and third-party selection → contract negotiation → ongoing monitoring → termination** — to structure a due-diligence packet for one third-party ML/AI capability that CarePilot (the module's running system) consumes.

The deliverable is a single markdown document, `carepilot-sr-23-4-llm-vendor.md`, that reads as a signed due-diligence packet for the specific third-party dependency the module uses: the **hosted LLM inference API** that CarePilot's RAG pipeline calls at runtime. The packet (a) applies each of SR 23-4's lifecycle stages, (b) names the technical artifact the engineering team supplies for each stage, (c) names the contract clause family that must be in place, and (d) ties the whole thing back into the model inventory row from mod-110 exercise 01 and the risk register.

The reference solution:

- Keeps SR 23-4's five-stage lifecycle as the top-level structure and does not re-order it.
- Scopes the review to the **hosted LLM API** dependency specifically, and *excludes* the base-model-weights supply chain from this packet (that supply-chain due diligence lives in mod-110 exercise 04 for the ML-BOM and exercise 05 for signing).
- Translates SR 23-4's generic "critical activities" language into ML/AI-specific criticality factors: materiality of use, data sensitivity entering the vendor, model-output substitution cost, and complexity / lack-of-transparency into the vendor's model.
- Treats SR 23-4 as a **third-party risk framework that does not know about ML/AI**. ML/AI-specific due-diligence questions (model card, eval evidence, red-team evidence, AUP/TOS for training on customer data, residency and deletion of inference prompts) are added as a labelled extension, not as a replacement for the agency expectations.
- States, in writing, that SR 23-4 superseded the earlier OCC Bulletin 2013-29 and FRB SR 13-19; the solution cites SR 23-4 as the current agency expectation and references the superseded bulletins only as historical context.
- Connects the ongoing-monitoring section to **mod-109** (drift / quality / safety monitoring), which is where runtime signals from the vendor dependency are collected.

Design decisions worth calling out:

- **One vendor, one packet.** SR 23-4 contemplates risk management proportional to the relationship. A packet that tries to cover "all vendors" dilutes the discipline the guidance expects for a critical activity. The solution is deliberately scoped to one dependency.
- **Criticality classification is a written decision, not an inheritance from the vendor.** SR 23-4 asks the banking organisation, not the vendor, to decide whether the activity is critical. The solution makes the determination with a written rationale.
- **Data-handling diligence lives next to legal, not next to engineering.** Where the LLM vendor's AUP or TOS speaks to training on customer prompts, residency, and deletion, the solution surfaces the question and hands off to counsel for the ruling; engineering supplies the technical facts.
- **Exit strategy is an engineering artifact, not a legal clause alone.** SR 23-4 asks for termination planning; the solution names a *rehearsed* fallback (shadow-mode secondary vendor + protocol-matching baseline), not a paper clause.

## 2. Implementation — worked answer `carepilot-sr-23-4-llm-vendor.md`

```markdown
---
system: CarePilot (hospital triage-support RAG assistant, HITL)
third_party: HostedLLM-Vendor-Z  (fictitious placeholder; real vendor name redacted for this exercise)
scope: Due-diligence packet for CarePilot's runtime dependency on a hosted LLM inference API
framework: SR 23-4 / OCC 2023-17 / FDIC FIL-29-2023 (joint interagency guidance, 6 June 2023)
author: ai-risk-engineer (L25)
reviewers: vendor-risk-committee (chair), chief-privacy-officer, chief-information-security-officer, counsel, head-of-model-validation
status: draft-for-review
---

# CarePilot ↔ HostedLLM-Vendor-Z — SR 23-4 Third-Party Risk Packet

## 0. Preface — scope and why this packet exists

CarePilot's RAG pipeline makes a runtime call to a hosted LLM inference API operated by an external vendor ("HostedLLM-Vendor-Z"). SR 23-4 defines a *third-party relationship* as any business arrangement between a banking organisation (here: the regional health system's cloud-services agreement, which is under the organisation's third-party risk programme) and another entity, by contract or otherwise. The CarePilot ↔ HostedLLM-Vendor-Z relationship is in scope.

This packet is scoped to the **inference-API dependency**. Base-model-weights provenance, supply-chain signing of the inference server binary, and ML-BOM authoring are addressed in **mod-110 exercises 04 and 05** and are referenced here, not reproduced.

## 1. Planning (SR 23-4 §III.A — Planning)

### 1.1 Business purpose

- **Activity.** CarePilot needs an LLM capable of (a) reading short clinical free text, (b) consuming retrieved corpus fragments, and (c) emitting a ranked triage recommendation with a rationale.
- **Alternatives considered.** (a) Self-hosted open-weights LLM — rejected at this time on operational capacity grounds; design-partner fallback retained (see §5 exit strategy). (b) Non-LLM protocol-matching baseline — retained as the *challenger* model under the SR 11-7 shape (mod-110 exercise 01 §3.3) and as the shadow-mode fallback.
- **Decision maker.** Model-risk committee, with sign-off from the clinical-safety officer and the privacy officer.

### 1.2 Inherent risk assessment

- **Operational risk.** The hosted API is in the critical path at runtime; an API outage directly degrades CarePilot. Mitigated by the shadow-mode baseline (§5).
- **Information-security risk.** Clinician-entered free text may contain PHI; a vendor side-channel or breach exposes PHI. Mitigated by (a) the PHI-redaction preprocessor from **mod-107 exercise 05** running *before* the vendor call, (b) contractual data-handling requirements (§3), (c) inference-time logging on the hospital side.
- **Compliance risk.** HIPAA Business Associate status for the vendor; FDA GMLP context for any vendor-side model update; EU AI Act Article 72 post-market monitoring if the system is placed on the EU market.
- **Strategic / concentration risk.** Single-vendor dependency on a market with three plausible alternatives; concentration risk is moderate.
- **Reputational risk.** A vendor incident that becomes public would reach the system via the CarePilot label.

### 1.3 Criticality determination

- **Decision.** This relationship is a **critical activity** under SR 23-4 because (a) the vendor's output directly shapes a clinical triage recommendation in the critical path, (b) loss of the service materially degrades patient care, and (c) sensitive health information flows to the vendor.
- **Consequence of the decision.** The packet below applies the fuller diligence shape SR 23-4 expects for critical activities; a reduced shape for a non-critical activity would compress §2 and §4.

## 2. Due diligence and third-party selection (SR 23-4 §III.B — Due Diligence)

Due-diligence subsections below follow the agency guidance's non-exhaustive list of areas. Each subsection lists the **ask**, the **vendor-supplied artifact**, and the **engineering / governance artifact** that validates or supplements it.

### 2.1 Strategies and goals; legal and regulatory compliance

- **Ask.** Vendor's HIPAA Business Associate posture; SOC 2 Type II; ISO/IEC 27001; where applicable, ISO/IEC 42001 AI-management-system evidence; EU AI Act posture if in scope.
- **Vendor artifact.** Signed BAA; SOC 2 Type II report with the current audit window; any public trust-centre links.
- **Internal artifact.** `vendor-risk/HostedLLM-Vendor-Z/compliance-matrix.md`, with gaps named and compensating controls listed. HIPAA BAA executed and countersigned by counsel.
- **Hand-off.** Counsel owns the legal ruling; engineering supplies the technical facts.

### 2.2 Financial condition; business experience and qualifications

- **Ask.** Audited financials; years in business; reference customers in regulated industries (ideally other healthcare systems); escalation points named.
- **Vendor artifact.** Last-year audited statements; three reference customers with written willingness-to-be-contacted statements.
- **Internal artifact.** `vendor-risk/HostedLLM-Vendor-Z/financial-and-business.md`.

### 2.3 Operations and internal controls — information security

- **Ask.** Security architecture (network segmentation, encryption in transit and at rest, key management), vulnerability-management programme, incident-response playbook with SLAs, supply-chain-security posture for the vendor's own dependencies.
- **Vendor artifact.** SOC 2 Type II control narratives for CC6 (logical access), CC7 (system operations), CC8 (change management); penetration-test executive summary.
- **Internal artifact.** `vendor-risk/HostedLLM-Vendor-Z/security-posture.md`; CISO sign-off on residual risk.

### 2.4 Operations and internal controls — data handling (ML/AI-specific supplement)

This is the SR 23-4 subsection that requires the most ML/AI-specific translation. SR 23-4 does not name LLM-specific questions; the packet answers them here.

- **Ask (AUP / TOS / contract-level).** (a) Does the vendor train on customer prompts, outputs, or RLHF labels? (b) What is the retention policy for inference prompts and outputs? (c) Where (region) are prompts processed and stored? (d) What is the deletion SLA on a documented request?
- **Vendor artifact.** Vendor's enterprise AUP + TOS pages; written confirmation (contract-level, not marketing-page-level) that enterprise customer data is excluded from training and labelling datasets.
- **Internal artifact.** `vendor-risk/HostedLLM-Vendor-Z/data-handling-matrix.md`, with each ask answered verbatim from the contract and each gap flagged.

### 2.5 Operations and internal controls — model documentation (ML/AI-specific supplement)

- **Ask.** (a) Model card for the specific model variant CarePilot calls (training-data provenance to the extent disclosed, evaluation evidence, known limitations). (b) Red-team evidence or system-card equivalent. (c) Content policy / abuse-mitigation posture.
- **Vendor artifact.** Model card URL (archived to internal storage with retrieval date); any public red-team report.
- **Internal artifact.** `vendor-risk/HostedLLM-Vendor-Z/model-documentation.md` — a critical-reading of the vendor artifacts with gaps named.

### 2.6 Resilience — business continuity, disaster recovery, incident response

- **Ask.** RTO/RPO; multi-region posture; incident-notification SLA to customers; past-incident log and remediation evidence.
- **Vendor artifact.** BCDR summary; incident-notification clause from the draft contract.
- **Internal artifact.** `vendor-risk/HostedLLM-Vendor-Z/resilience-and-ir.md`.

### 2.7 Sub-contractors and fourth parties

- **Ask.** Named sub-processor list; the vendor's own third-party risk programme; notice-of-change-of-sub-processor clause.
- **Vendor artifact.** Sub-processor list; vendor's third-party risk policy summary.
- **Internal artifact.** `vendor-risk/HostedLLM-Vendor-Z/fourth-parties.md`.

### 2.8 Insurance

- **Ask.** Cyber-liability coverage limits; E&O coverage; evidence of policies in force naming the regional health system as a loss payee / certificate holder where appropriate.
- **Vendor artifact.** Certificates of insurance.
- **Internal artifact.** `vendor-risk/HostedLLM-Vendor-Z/insurance.md`.

## 3. Contract negotiation (SR 23-4 §III.C — Contract Negotiation)

- **Clause families to require, with a contract-section pointer in `legal/HostedLLM-Vendor-Z/MSA-v3.pdf` — paragraph numbers to confirm at execution <!-- verify-citation: paragraph numbers in the final executed MSA, not the draft. -->**
  1. **Nature and scope** — enumeration of the service, model variants permitted, change-of-model-variant notification.
  2. **Performance measures and benchmarks** — availability SLA; latency SLA; error-rate SLA; measurement methodology and remedy on breach.
  3. **Information security and data handling** — encryption, key management, no-training-on-customer-data commitment, residency, retention, deletion SLA.
  4. **Responsibilities for providing / receiving / retaining information** — audit rights, right-to-examine with notice window.
  5. **Right to audit and remediate** — the banking organisation's right to audit the vendor; vendor's duty to remediate findings within a stated window.
  6. **Business continuity / disaster recovery** — RTO, RPO, incident-notification SLA.
  7. **Indemnification and limitation of liability** — IP indemnity, data-breach indemnity, carve-outs, caps.
  8. **Insurance** — evidence of insurance and notice of material change.
  9. **Dispute resolution** — venue, governing law, escalation ladder.
  10. **Default and termination** — events of default; notice-and-cure period; termination-for-convenience; termination-assistance (data-return + deletion-with-certificate within a stated window).
  11. **Sub-contracting** — notice of new sub-processors, right to object.
  12. **Confidentiality and non-disclosure** — mutual; duration; survival.
  13. **Compliance with laws** — HIPAA Business Associate Agreement by reference; export controls; other applicable laws.

## 4. Ongoing monitoring (SR 23-4 §III.D — Ongoing Monitoring)

- **Operational performance monitoring.** The availability / latency / error-rate SLA metrics are instrumented on the hospital side and mirrored into the CarePilot monitoring dashboard built in **mod-109 exercise 05**. Breach-of-SLA is a signal under the contract in §3 §2.
- **Model-behaviour monitoring.** Drift of CarePilot's output distribution and of red-team sentinel behaviour is monitored per **mod-109 exercises 02 and 03**. A vendor-side silent model update is a plausible cause of drift; monitoring is the detection, the contract clause in §3 §1 is the entitlement to vendor disclosure.
- **Security-event monitoring.** Vendor breach-notification obligations under §3 §3 and §3 §6 are the trigger; the hospital-side incident-response playbook from **mod-111** is the handler.
- **Compliance posture refresh.** SOC 2 Type II refresh annually; HIPAA BAA posture reaffirmed at renewal; vendor's model card and sub-processor list re-reviewed at renewal.
- **Vendor-health signals.** Financial-condition refresh annually; concentration-risk review annually.

## 5. Termination (SR 23-4 §III.E — Termination)

- **Exit strategy.** The shadow-mode non-LLM protocol-matching baseline (mod-110 ex01 §3.3) runs continuously; on a termination event it is the immediate fallback. A secondary hosted LLM vendor is kept contract-ready (dormant MSA + a periodic smoke-test invocation to keep the integration from rotting).
- **Rehearsal cadence.** Fallback cut-over is rehearsed **quarterly** in a game-day; evidence is a signed game-day report at `continuity/carepilot/<yyyy-qq>-cutover-rehearsal.md`.
- **Data return / deletion.** Vendor is contractually required to return or destroy regional-health-system data within the window set in §3 §10 and to provide a certificate of destruction. Engineering requests the certificate on the termination date and files it with the packet.
- **Communication.** Internal communications plan in `continuity/carepilot/termination-comm-plan.md`; patient-facing communication is owned by patient-relations.

## 6. Inventory and sign-off

- **Vendor row in the third-party inventory.** `vendor-inventory/2025.yaml#HostedLLM-Vendor-Z` with fields: vendor name, service, criticality (critical), last diligence date, next diligence due date, BAA status, SLA-breach count, concentration-risk assessment, risk-register links.
- **Linkage to the model inventory.** `model-inventory/2025.yaml#MRM-CAREPILOT-001#third_party_dependencies` lists `HostedLLM-Vendor-Z` with a pointer to this packet.
- **Approvers.** Vendor-risk committee chair, CISO, chief-privacy-officer, counsel, head-of-model-validation (L60), model-risk committee chair.

## Notes

- Section numbers cite the interagency guidance *Final Interagency Guidance on Third-Party Relationships: Risk Management* jointly issued by the Federal Reserve (SR 23-4), the OCC (Bulletin 2023-17), and the FDIC (FIL-29-2023) on 6 June 2023. The guidance supersedes OCC Bulletin 2013-29 and Federal Reserve SR 13-19 and OCC Bulletin 2020-10 on the same topic; this packet treats SR 23-4 as the operative source.
- The ML/AI-specific supplements (§2.4, §2.5) are additive; they are not substituted for any SR 23-4 subsection and do not relieve the organisation of the agency expectations.
```

## 3. Validation steps

1. **Five-stage lifecycle present.** The submission has top-level sections mapping to SR 23-4's Planning, Due Diligence, Contract, Ongoing Monitoring, Termination. A reviewer grep-checks the ToC.
2. **Critical-activity determination written.** §1.3 has an explicit critical-/non-critical determination with a written rationale, and the consequence of the determination (fuller vs reduced due-diligence shape) is stated.
3. **Due-diligence subsections cover all SR 23-4 areas.** §2.1–§2.8 cover compliance, financial condition, security, data handling (ML/AI-specific), model documentation (ML/AI-specific), resilience, sub-contractors, insurance. A reviewer confirms every one has an `Ask` / `Vendor artifact` / `Internal artifact` line.
4. **Contract-clause family list complete.** §3 enumerates the clause families SR 23-4 contemplates (performance measures, data handling, audit rights, resilience, indemnification, termination, sub-contracting, BAA by reference, …). A missing clause family fails review.
5. **Ongoing monitoring hooks to mod-109.** §4 references mod-109 exercises 02/03/05 for the actual telemetry; monitoring is wired, not just promised.
6. **Exit strategy is rehearsed.** §5 names a *rehearsed* fallback (quarterly game-day) and the evidence artifact. A "we have a secondary vendor" line without a rehearsal cadence fails.
7. **ML/AI-specific supplements labelled as supplements.** §2.4 and §2.5 are labelled as ML/AI-specific extensions, not as substitutes. A submission that replaces an SR 23-4 subsection with an LLM-centric one fails.
8. **Inventory row present.** §6 includes the vendor-inventory row with criticality, diligence dates, BAA status, SLA-breach count, concentration-risk assessment, and risk-register links.
9. **No invented facts.** Where specific vendor compliance claims (SOC 2, HIPAA BAA, model-card URL) are cited, the packet shape is agnostic to a specific vendor; any real-vendor assertion added later must carry a dated artifact pointer, not a marketing-page URL.

## 4. Rubric — review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Five-stage lifecycle structure | Planning / Due Diligence / Contract / Monitoring / Termination as top-level sections | Any stage missing or merged into another |
| R2 | Critical-activity determination | §1.3 names the determination with written rationale and names its consequence for the packet shape | Determination skipped or implicit |
| R3 | Due-diligence coverage | §2 covers compliance, security, data handling, model documentation, resilience, sub-processors, insurance, financial condition | Fewer than 6 of the 8 subsections |
| R4 | ML/AI-specific supplements labelled | §2.4 and §2.5 explicitly labelled as supplements to SR 23-4 subsections | LLM-specific questions replace SR 23-4 subsections or are unlabelled |
| R5 | Contract-clause family list | §3 lists at least 10 SR 23-4 clause families with contract-section pointers | Clause families missing or no pointer to the contract |
| R6 | Monitoring wired to mod-109 | §4 references mod-109 exercises 02, 03, 05 for the actual telemetry | Monitoring described as "we will monitor" with no artifact |
| R7 | Exit strategy is rehearsed | §5 names a rehearsal cadence and the evidence artifact | Fallback described but not rehearsed |
| R8 | Inventory linkage | §6 lists both vendor-inventory row and model-inventory `third_party_dependencies` field | Either inventory missing the vendor row |
| R9 | No invented facts | Specific agency-page or vendor-page citations carry `<!-- verify-citation: ... -->` where unverified | Fabricated certifications, BAA status, or SOC 2 scope |
| R10 | SR 23-4 named as current source | Document cites SR 23-4 and notes it supersedes OCC 2013-29 / SR 13-19 | Document cites only the superseded guidance, or omits the supersession note |

Scoring bands: *Pass* — R1–R8 green, R9/R10 honoured; *Pass with revisions* — R3 or R5 misses one item; *Fail* — any of R1, R2, R7, R9.

## 5. Common mistakes

- **Treating SR 13-19 / OCC 2013-29 as the current guidance.** They are superseded by SR 23-4 (6 June 2023). Reviewers reject packets that cite the old bulletins without the supersession note.
- **Letting the vendor set the criticality.** SR 23-4 asks the banking organisation (or health system, under its own third-party programme) to decide. "Vendor says they are critical infrastructure" is not a criticality determination.
- **Due diligence reduced to a questionnaire.** A SOC 2 cover page is not due diligence; reading the control narratives (CC6/CC7/CC8) and listing gaps is. Submissions that paste a SOC 2 logo without a critical-reading fail.
- **Replacing SR 23-4 subsections with LLM-only questions.** The packet needs the SR 23-4 shape in full; LLM-specific questions are additive (data handling, model documentation, red-team evidence).
- **Fallback-on-paper.** A paper fallback that has never been rehearsed is not a termination plan. The game-day is the control, not the MSA clause.
- **Omitting the BAA.** For healthcare, a HIPAA Business Associate Agreement is a precondition to the vendor processing PHI. Submissions that treat it as a nice-to-have fail.
- **Confusing the AUP/TOS with the contract.** Marketing pages or default enterprise terms are not contractually binding without being incorporated by reference into the executed MSA. Submissions that cite only the TOS URL fail.
- **Monitoring-as-promise.** "We will monitor SLA" is not monitoring. Monitoring is a dashboard in mod-109 with thresholds and alert routing in mod-109 exercise 05.
- **Omitting concentration-risk thinking.** One LLM vendor is a concentration-risk exposure; the packet needs to name alternatives and the keep-warm posture of a secondary.
- **Model-card trust.** A vendor model card is evidence the vendor made a claim; it is not evidence the claim is true. The packet's engineering column *critically reads* the model card and names gaps.

## 6. References

- **Federal Reserve SR 23-4 / OCC Bulletin 2023-17 / FDIC FIL-29-2023 — Final Interagency Guidance on Third-Party Relationships: Risk Management.** Jointly issued 6 June 2023. Supersedes OCC Bulletin 2013-29, Federal Reserve SR 13-19, OCC Bulletin 2020-10. Primary source for the five-stage lifecycle and the due-diligence subsections.
- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — referenced for the MAP / MEASURE rows that touch third-party dependencies; cross-walk in mod-101 exercise 01.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — attack catalogue that the §4 ongoing-monitoring discipline watches for.
- **MITRE ATLAS.** `https://atlas.mitre.org/` — threat catalogue that supplements ongoing monitoring and informs the resilience and incident-response subsections.
- **HIPAA Business Associate Agreement (US 45 CFR §164.502(e), §164.504(e); §164.314(a)).** Precondition for the vendor to process PHI. Cited as the ruling authority for the BAA requirement in §2.1 and §3 §13. <!-- verify-citation: paragraph-level citations for the CFR text, and the current OCR Business Associate Contract Provisions webpage, when a specific clause is quoted. -->
- **Module running system.** CarePilot — hospital-triage-support RAG assistant, introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`.
- **Model inventory row.** `MRM-CAREPILOT-001` from **mod-110 exercise 01 §4.3**; this packet populates the `third_party_dependencies` sub-field for the hosted LLM vendor.
- **Monitoring.** `mod-109` exercises 02, 03, 05 — the runtime signals that §4 relies on.
- **Exercise prompt.** `lessons/mod-110-model-risk-management-and-supply-chain/exercises/exercise-02-sr-23-4-third-party-model-due-diligence.md`.

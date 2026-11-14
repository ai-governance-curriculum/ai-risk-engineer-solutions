# Exercise 06 — Certifications Portfolio Planner — Reference Solution

## 1. Solution overview

This exercise is the module's **career-planning** drill: for the AI risk engineer on the CarePilot team (hospital-triage-support RAG assistant from mod-101), author a durable **certifications portfolio plan** that says, in one document, (a) what role gaps a credentialing programme is meant to close, (b) which certifications are candidate fits and which are explicitly rejected, (c) in what order the role-holder will pursue them over a rolling 24-month window, (d) how each credential maps back to the team's evidence artifacts and the organisation's RMF / 42001 posture, and (e) how the plan is reviewed, versioned, and attested. The plan is the thing a new AI risk engineer reads on day one to understand which credentials this role is expected to hold or be working toward, and why.

The reference solution:

- Treats the certifications portfolio as a **gap-closure plan**, not a credential-collection plan. The exercise's trap is to list every AI-adjacent certification on the market and attempt to sequence them; a plan driven by that goal is cosmetic and never finishes. The plan in §2.3 starts from a **role-gap analysis** against the ex-01 charter's roles and the ex-05 evidence crosswalk's framework coverage — the credentials are chosen because they close a specific gap, not because they are available.
- Keeps the scope to **this one role, on this one team, in this one organisation**. The exercise is not to design the organisation's AI-governance training programme (owned by learning-and-development with the head of AI governance) nor to credential every team member (owned by their managers). The plan is for the AI risk engineer (L25) on CarePilot. The scope line in §2.3 states this explicitly.
- Anchors every candidate credential to **three load-bearing attributes**: the role gap it closes, the body-of-knowledge it covers, and the evidence (an exam pass, a renewal cycle, continuing-professional-education hours) that attests it. A credential without a stated gap-closure is rejected.
- Uses an **explicit prioritisation rubric** (§2.4) with five weights — gap-closure fit, framework coverage, renewal burden, time-and-cost realism, and portfolio diversity — rather than a vibe-based ranking. The weights are documented so a reviewer can disagree with the ranking by disagreeing with the weights, not with the author.
- Produces a **24-month sequenced plan** (§2.5) with budgeted hours per quarter, a "no-more-than-two-concurrent-tracks" load limit, and explicit pre-requisites between credentials where they exist.
- Attaches an **attestation and evidence contract** (§2.6) so that each held credential appears in the charter's role-level attestations and in the ex-05 evidence crosswalk's "documented information" row — the plan is accountable to the same artifacts the rest of the module produces.
- Carries a **portfolio disclaimer** (§2.9) so that specific exam formats, hour counts, renewal fees, and body-of-knowledge versions are not reproduced verbatim from any credentialing body's documentation without pinning. Where a specific number or clause would be useful, the plan uses the credential body's **family-level description** and marks the specific detail with `<!-- verify-citation: ... -->` — the same discipline used in ex-05's framework-code citations.
- Treats the plan itself as a **versioned engineering artifact**: it lives in-repo, carries a version and effective date, has a review cadence of its own, and ships with a changelog, so the plan is a maintained document rather than a one-shot deliverable.

Design decisions worth calling out:

- **Why gap-closure first, not credential-first.** A plan that starts with "which certifications exist in the AI-governance space" produces a list that overlaps the role's needs by accident. A plan that starts with "what gaps does this role carry on this team" produces a list that is read by the role's manager, by the AI governance analyst, and by the council, and that is defended during performance review. The gap-closure analysis in §2.2 is the structural anchor; the credentialing is downstream of it.
- **Why role-and-team scope, not person-and-industry scope.** A plan written for "any AI risk engineer in healthcare AI" gets abstract and recommends everything. A plan written for the CarePilot AI risk engineer at the current organisation reflects the actual control environment, the actual regulator the team faces, and the actual gaps named in the ex-01 charter and the ex-05 crosswalk.
- **Why a documented prioritisation rubric, not an opinion.** The alternative is a vibe-ordered list the reviewer cannot argue with. A rubric makes the ordering falsifiable: change a weight, change the order. This also gives the role-holder's manager a clean handle for approving or pushing back without relitigating each candidate.
- **Why a 24-month window.** Shorter than 24 months and the sequencing becomes month-to-month tactics; longer than 24 months and the credential bodies' own curricula have usually revved (new body-of-knowledge, deprecated exam, changed renewal cycle). 24 months is the longest horizon on which specific credentials can be named with any confidence. The plan is **re-planned annually** against the then-current catalogue.
- **Why attestation-and-evidence binding.** A certification plan that does not plug into the team's evidence artifacts is a personal-development document, not a governance artifact. Binding each held credential to the ex-05 crosswalk's "documented information" row and to the charter's role-level attestations makes the portfolio auditable alongside the rest of the team's evidence.
- **Why explicit rejections.** The portfolio is as much about **what the role does not pursue** as what it does. Listing rejected candidates with a one-line rationale prevents the next quarter's "we should add CertX" conversation from restarting the sequencing from scratch — the rejection reason is already written down.
- **Why `<!-- verify-citation: ... -->` discipline applied to credential specifics.** Certification bodies routinely rev their syllabi, exam formats, renewal hours, and pricing. Reproducing a specific number in the plan without pinning the source introduces facts that go stale within quarters. The plan cites the credential at family level and marks specifics for pin-on-publication. This mirrors ex-05's treatment of RMF sub-category codes and 42001 sub-clauses.

## 2. Worked answer or implementation

### 2.1 Repository layout

```
exercise-06-certifications-portfolio-planner/
├── SOLUTION.md                                      # this file
├── portfolio/
│   ├── role-gap-analysis.md                         # the gap-closure anchor
│   ├── candidate-catalogue.md                       # candidate credentials, each scored against the rubric
│   ├── rubric.md                                    # the prioritisation rubric itself
│   ├── 24-month-plan.md                             # the sequenced plan
│   └── rejected-candidates.md                       # explicit rejections with reasons
├── attestations/
│   └── credentials-held.md                          # the role's attested credentials (template; starts empty)
├── templates/
│   └── credential_row_template.md                   # reusable row schema
└── checklists/
    ├── plan_authoring_checklist.md
    └── annual_review_checklist.md
```

### 2.2 Role-gap analysis — `portfolio/role-gap-analysis.md`

The anchor of the whole plan. The question the analysis answers is **"what gaps does the AI risk engineer on CarePilot carry today that a credential could meaningfully close?"**. Every row names a gap, a source (where the gap is documented), and a credential-family that addresses it.

```markdown
---
document_class: certifications-role-gap-analysis
role: AI risk engineer (L25)
scope: CarePilot team
inherits_from:
  - modules/mod-101-ai-risk-frameworks-and-position/exercise-05-role-scope-and-deferral-contract/SOLUTION.md
  - modules/mod-112-cross-functional-ai-risk-program-and-communication/exercise-01-risk-program-charter-for-one-team/SOLUTION.md
  - modules/mod-112-cross-functional-ai-risk-program-and-communication/exercise-05-rmf-and-42001-evidence-crosswalk/SOLUTION.md
version: 1.0
effective_from: 2026-11-13
next_review: 2027-11-13
owner_role: AI risk engineer (self)
reviewer_roles:
  - Tech lead / engineering manager
  - AI governance analyst
status: draft-for-manager-review
---

# AI Risk Engineer (CarePilot) — Role-Gap Analysis for Certifications

## Gap source legend

- **Charter** — gap named in or implied by the ex-01 team charter.
- **Crosswalk** — gap named in the ex-05 evidence crosswalk's gap log (G1–G6) or
  implied by a `<!-- verify-citation: ... -->` marker.
- **Deferral** — gap named in or implied by the ex-05 mod-101 deferral contract
  (for example, a responsibility the role depends on another role for).
- **Observed** — a gap the role-holder has surfaced directly (e.g. after an
  incident retrospective).

## Gaps

| Gap ID | Gap | Source | Why a credential helps | Credential-family candidate |
|---|---|---|---|---|
| G-01 | The role attests artifacts against **NIST AI RMF 1.0** sub-categories but has not read the full AI 100-1 §5 enumeration end-to-end against a structured syllabus. (ex-05 crosswalk gap G4.) | Crosswalk | A sitting-and-passing attestation against an RMF-aligned curriculum is a dated, falsifiable claim of having worked through the whole framework, not just the sub-categories the team happens to reference. | AI-governance credential with explicit NIST AI RMF coverage (e.g. **IAPP AIGP**). |
| G-02 | The role attests artifacts against **ISO/IEC 42001**, but the organisation is not yet certified to 42001 and the role has not sat a 42001-implementer or 42001-auditor curriculum. (ex-05 crosswalk gap G3 — Annex A control pinning not yet complete.) | Crosswalk | A 42001-aligned certification forces a line-by-line read of the clauses and Annex A controls, which is exactly the evidence pinning the crosswalk still needs. | **ISO/IEC 42001 Lead Implementer** and/or **Lead Auditor** (various accredited providers). |
| G-03 | Privacy work in production (PII containment, de-identification) is attested per release, but the role does not hold a primary privacy credential. The deferral contract routes privacy legal interpretation to the privacy officer, but the engineering competence is the role's. | Charter + Deferral | A privacy-specific credential closes the engineering-privacy competence gap; the privacy officer still owns legal interpretation. | **IAPP CIPP / CIPT / CIPM** family (jurisdiction-matched to the hospital's deployment region). |
| G-04 | Red-team and adversarial-ML work is attested through the suite, but there is no credentialed attestation of secure-development or adversarial-ML competence at the role level. | Observed | A security-adjacent credential grounds the red-team work in a maintained, examined body of knowledge rather than resting on the individual's prior experience. | Security credential with an ML / AI module (e.g. a well-known security body's AI-specific credential or an AI-focused red-team certification). |
| G-05 | The role is the primary authorer of the harm model and risk register but has not sat a formal risk-management curriculum that would ground the quantification choices in the register. | Observed | A risk-management credential closes the methodological gap — standard loss-exceedance and quantification vocabulary are not something the role acquired through engineering alone. | Risk-management credential (e.g. **ISO/IEC 31000** foundation-level or an AI-specific risk-management track). |
| G-06 | Incident-response work is attested through the playbook and post-mortems, but the role has not sat a credentialed incident-response curriculum; the deferral contract routes legal-notification work to counsel but engineering response is the role's. | Charter + Deferral | A credential grounds the response practice in an examined curriculum, which matters for regulator-facing incident summaries (mod-111 ex-05). | Incident-response credential family. |
| G-07 | The role is the team's principal reader of OWASP ML Top 10 and MITRE ATLAS but holds no credentialed training on either. | Charter | A credentialed OWASP ML Top 10 / ATLAS training produces a dated attestation that the role has worked through both catalogues. | OWASP / MITRE training programmes (training rather than formal exam; see §2.4 treatment). |
| G-08 | The role interacts regularly with the cloud inference provider and the managed vector-store provider; the deferral contract routes platform-security work to AI infra security, but engineering-side cloud competence is the role's. | Deferral | A cloud-provider AI-practitioner credential closes the engineering-cloud competence gap. | Cloud vendor AI / ML / security practitioner credentials (vendor-matched to the organisation's provider). |

## Gap categories excluded from this plan

- **Executive AI governance** (L60+). Out of role scope. The head of AI governance is the attested role for this category; any credential on this plan does not substitute for the L60 role.
- **General management / leadership credentials**. Out of scope for a technical risk-engineering role.
- **Modality-specific research credentials** (e.g. computer-vision research). Out of role scope: the role does not author modality research; the AI evaluation engineer is the attested methodological role.
- **Jurisdiction-specific regulatory credentials unrelated to the deployment region**. The plan targets the hospital's deployment region; credentials for other regions are tracked as "maybe, on relocation or scope change" (§2.5).
```

### 2.3 Candidate catalogue — `portfolio/candidate-catalogue.md`

Each row is a credential. The schema is deliberately shallow — the depth is in the rubric score (§2.4) and the sequenced plan (§2.5), not in this catalogue.

```markdown
---
document_class: certifications-candidate-catalogue
role: AI risk engineer (L25)
scope: CarePilot team
version: 1.0
effective_from: 2026-11-13
owner_role: AI risk engineer (self)
status: draft-for-manager-review
---

# Candidate Credential Catalogue

Each candidate carries a family-level description. Specific syllabus versions,
exam formats, hour counts, renewal fees, and body-of-knowledge revisions are
marked `<!-- verify-citation: ... -->` and are pinned by the role-holder from
the credential body's current published materials before enrolling.

## Candidate C-AIGP — AI-governance programme certification

- **Family.** AI-governance programme certification for practitioners, with explicit coverage of the NIST AI RMF and (where offered by the body) of ISO/IEC 42001. Example family: the IAPP AIGP (Artificial Intelligence Governance Professional). <!-- verify-citation: confirm the current AIGP body-of-knowledge version and NIST AI RMF coverage from IAPP's published syllabus. -->
- **Gap this closes.** G-01 (NIST AI RMF structured coverage) primarily; G-02 (ISO/IEC 42001) partially.
- **Delivery.** Self-study + proctored exam (typical for the family). <!-- verify-citation: confirm format and sitting availability. -->
- **Body-of-knowledge.** AI governance programme design, AI risk frameworks, legal/regulatory environment for AI, AI lifecycle risk. <!-- verify-citation: confirm the current body-of-knowledge domains from the credential body's materials. -->
- **Renewal.** Periodic CPE / continuing-professional-education hours, renewal fee on a published cycle. <!-- verify-citation: confirm renewal cadence and CPE requirement from the credential body's materials. -->
- **Pre-requisites.** None, in the general case. <!-- verify-citation: confirm any prerequisite or experience requirements from the credential body's materials. -->
- **Attestation evidence.** Exam-pass certificate + enrolment-and-renewal record in the role-level attestations file.

## Candidate C-42001-LI — ISO/IEC 42001 Lead Implementer

- **Family.** Accredited lead-implementer training and examination for ISO/IEC 42001 (AI management system). Offered by several accredited training providers (e.g. PECB, BSI, and others). <!-- verify-citation: confirm available accredited providers and their delivery formats at time of enrolment. -->
- **Gap this closes.** G-02 (ISO/IEC 42001 line-by-line clause and Annex A coverage) primarily.
- **Delivery.** Typically multi-day instructor-led or self-paced virtual, with a proctored exam. <!-- verify-citation: confirm format at the chosen provider. -->
- **Body-of-knowledge.** ISO/IEC 42001 clauses 4–10, Annex A operational controls, AI-management-system implementation methodology. <!-- verify-citation: confirm current syllabus from provider materials. -->
- **Renewal.** Per-provider renewal / CPD cycle. <!-- verify-citation: confirm from provider materials. -->
- **Pre-requisites.** None, in the general case; some providers recommend prior ISO-management-system exposure. <!-- verify-citation: confirm per provider. -->
- **Attestation evidence.** Exam-pass certificate + 42001-implementer renewal log.

## Candidate C-42001-LA — ISO/IEC 42001 Lead Auditor

- **Family.** Accredited lead-auditor training and examination for ISO/IEC 42001. <!-- verify-citation: confirm available accredited providers and their delivery formats at time of enrolment. -->
- **Gap this closes.** G-02 partially. Complements C-42001-LI by adding the external-audit posture.
- **Delivery.** Multi-day instructor-led or virtual with a proctored exam. <!-- verify-citation: confirm format. -->
- **Body-of-knowledge.** AI-management-system auditing methodology, nonconformity treatment, audit evidence standards. <!-- verify-citation: confirm current syllabus. -->
- **Renewal.** Per-provider renewal / CPD cycle. <!-- verify-citation: confirm. -->
- **Pre-requisites.** Many providers list lead-implementer (or equivalent exposure) as expected before lead-auditor. <!-- verify-citation: confirm per provider. -->
- **Attestation evidence.** Exam-pass certificate + auditor renewal log.

## Candidate C-CIPP — Privacy practitioner credential

- **Family.** A jurisdiction-matched privacy credential from a recognised privacy-professional body. Example family: the IAPP CIPP (Certified Information Privacy Professional) / CIPT (technologist) / CIPM (manager). <!-- verify-citation: confirm jurisdiction options and current body-of-knowledge from IAPP materials; confirm the hospital's deployment region to decide the specific CIPP sub-track (e.g. CIPP/E, CIPP/US, etc.). -->
- **Gap this closes.** G-03 (privacy engineering competence).
- **Delivery.** Self-study + proctored exam. <!-- verify-citation: confirm format. -->
- **Body-of-knowledge.** Privacy principles, jurisdiction-specific privacy law, privacy programme management, privacy-by-design. <!-- verify-citation: confirm per track. -->
- **Renewal.** Periodic CPE hours, renewal fee. <!-- verify-citation: confirm. -->
- **Pre-requisites.** None, in the general case. <!-- verify-citation: confirm. -->
- **Attestation evidence.** Exam-pass certificate + renewal log.

## Candidate C-RT — Red-team / adversarial-ML credential

- **Family.** An AI-specific adversarial-security credential (or a general-security credential with a credible AI module). The role-holder evaluates specific offerings from recognised bodies against the rubric before enrolling; the family is intentionally left open because this market is actively developing. <!-- verify-citation: confirm the shortlist of credentials in this family at time of enrolment and pin one from the shortlist. -->
- **Gap this closes.** G-04 (red-team / adversarial-ML).
- **Delivery.** Family-dependent. <!-- verify-citation: confirm. -->
- **Body-of-knowledge.** Adversarial-ML threat models, prompt-injection and jailbreak methodology, model-extraction and membership-inference threats, defensive design. <!-- verify-citation: confirm per credential. -->
- **Renewal.** Family-dependent. <!-- verify-citation: confirm. -->
- **Pre-requisites.** Family-dependent. <!-- verify-citation: confirm. -->
- **Attestation evidence.** Exam-pass or practical-attestation certificate + renewal log.

## Candidate C-RM — Risk-management credential

- **Family.** A risk-management methodology credential, ideally with an AI-specific module. Example family: an **ISO/IEC 31000** foundation-level credential or an AI-specific risk-management track. <!-- verify-citation: confirm provider options and syllabi. -->
- **Gap this closes.** G-05 (risk quantification methodology).
- **Delivery.** Self-study or short instructor-led, proctored exam. <!-- verify-citation: confirm. -->
- **Body-of-knowledge.** Risk principles, risk assessment methodology, treatment and acceptance, loss-exceedance and quantification. <!-- verify-citation: confirm per provider. -->
- **Renewal.** Family-dependent. <!-- verify-citation: confirm. -->
- **Pre-requisites.** None, in the general case. <!-- verify-citation: confirm. -->
- **Attestation evidence.** Exam-pass certificate + renewal log.

## Candidate C-IR — Incident-response credential

- **Family.** An incident-response practitioner credential, preferably one that includes a short AI / ML module or has a parallel AI-incident curriculum. <!-- verify-citation: confirm available credentials in the family and their AI coverage. -->
- **Gap this closes.** G-06 (credentialed incident-response grounding).
- **Delivery.** Family-dependent. <!-- verify-citation: confirm. -->
- **Body-of-knowledge.** Triage, forensics, response coordination, communication, post-incident review. <!-- verify-citation: confirm. -->
- **Renewal.** Family-dependent. <!-- verify-citation: confirm. -->
- **Pre-requisites.** Family-dependent. <!-- verify-citation: confirm. -->
- **Attestation evidence.** Exam-pass certificate + renewal log.

## Candidate C-OWASP — OWASP ML Top 10 credentialed training

- **Family.** Credentialed training on the OWASP Machine Learning Security Top 10. The OWASP project is an official source (`https://owasp.org/www-project-machine-learning-security-top-10/`); individual accredited-training availability is confirmed per the OWASP project's published guidance at time of enrolment. <!-- verify-citation: confirm whether a formal OWASP-badged credentialed training programme on the ML Top 10 is published and available at enrolment time; if not available as a credential, treat as "training with attendance attestation" rather than an exam. -->
- **Gap this closes.** G-07 (OWASP ML Top 10 structured coverage).
- **Delivery.** Family-dependent; may be workshop-plus-attendance-attestation rather than proctored exam. <!-- verify-citation: confirm. -->
- **Renewal.** Family-dependent. <!-- verify-citation: confirm. -->
- **Attestation evidence.** Training-completion attestation + reading-log entry in role attestations.

## Candidate C-ATLAS — MITRE ATLAS credentialed training

- **Family.** Credentialed training on MITRE ATLAS (`https://atlas.mitre.org/`). ATLAS is an official source. Where a MITRE-badged or recognised-third-party-badged training programme exists at time of enrolment, it is the credential; where no formal credential is offered, the role-holder tracks ATLAS as a reading-and-exercise programme with an attendance attestation. <!-- verify-citation: confirm credential availability. -->
- **Gap this closes.** G-07 (MITRE ATLAS structured coverage).
- **Delivery.** Family-dependent. <!-- verify-citation: confirm. -->
- **Renewal.** Family-dependent. <!-- verify-citation: confirm. -->
- **Attestation evidence.** Training-completion attestation + reading-log entry in role attestations.

## Candidate C-CLOUD — Cloud vendor AI / ML / security practitioner credential

- **Family.** A practitioner-level credential from the organisation's cloud-inference provider on their AI / ML / security products, matched to the provider the hospital actually uses. <!-- verify-citation: confirm the organisation's primary cloud-inference provider and the current practitioner-track credentials from that provider. -->
- **Gap this closes.** G-08 (cloud-provider engineering competence).
- **Delivery.** Self-study + proctored exam. <!-- verify-citation: confirm per provider. -->
- **Renewal.** Per-provider cycle. <!-- verify-citation: confirm. -->
- **Attestation evidence.** Exam-pass certificate + renewal log.

## Candidate C-PM — Project/programme-management credential (adjacency candidate)

- **Family.** A programme-management credential, included as an adjacency candidate because risk-programme owners sometimes benefit from a methodological grounding. <!-- verify-citation: confirm whether the organisation's L25 role expectations include programme-management responsibilities sufficient to justify this credential. -->
- **Gap this closes.** Partial; closes no named gap in §2.2 cleanly. Included only as a comparison baseline — see rejection in §2.4 if it does not clear the rubric.
```

### 2.4 Prioritisation rubric — `portfolio/rubric.md`

The rubric is the artifact that makes the ordering falsifiable. Five weights, documented explicitly. The scores are 0–3 per criterion.

```markdown
---
document_class: certifications-prioritisation-rubric
version: 1.0
effective_from: 2026-11-13
owner_role: AI risk engineer (self)
---

# Prioritisation Rubric

## Criteria and weights

| Criterion | Weight | How to score 0 / 1 / 2 / 3 |
|---|---|---|
| Gap-closure fit | 4 | 0 = closes no named role gap. 1 = partially addresses a named gap. 2 = closes one named gap. 3 = closes a primary gap named in the ex-05 crosswalk gap log or the ex-01 charter. |
| Framework coverage (RMF + 42001 alignment) | 3 | 0 = no RMF or 42001 alignment in the body-of-knowledge. 1 = tangential alignment. 2 = covers one of {RMF, 42001} explicitly. 3 = covers both frameworks explicitly. |
| Renewal burden (lower is better) | 2 | 0 = unknown or very heavy renewal (≥ 40 CPE-equivalent hours/year). 1 = heavy (≥ 20 CPE/yr). 2 = moderate (< 20 CPE/yr). 3 = low / amortised across the portfolio. <!-- verify-citation: confirm all CPE numbers per credential before scoring. --> |
| Time-and-cost realism | 2 | 0 = cost > one-quarter of annual professional-development budget or > 80 study-hours. 1 = approaches those limits. 2 = within one-quarter / within 40 hours. 3 = modest (< 20 hours and modest fee). <!-- verify-citation: confirm the organisation's per-role annual professional-development budget. --> |
| Portfolio diversity | 1 | 0 = redundant with an already-planned credential. 1 = adjacent / overlaps substantially. 2 = adds one new domain to the portfolio. 3 = adds two or more new domains and plugs into a gap that would otherwise be uncovered. |

The weights are **debated** at the annual review (§2.8); the scores for each
candidate are debated per-cycle.

## Candidate scoring (round 1, pre-enrolment)

Scores are illustrative round-one values for the candidates in §2.3. The final
score is weight × criterion-score, summed. Ties are broken by the ordering of
the row criteria (gap-closure first).

| Candidate | Gap (×4) | Framework (×3) | Renewal (×2) | Time/cost (×2) | Diversity (×1) | Weighted total |
|---|---|---|---|---|---|---|
| C-AIGP | 3 (closes G-01; partial G-02) → 12 | 3 → 9 | 2 → 4 | 2 → 4 | 3 → 3 | **32** |
| C-42001-LI | 3 (closes G-02) → 12 | 3 → 9 | 2 → 4 | 1 → 2 | 2 → 2 | **29** |
| C-CIPP (region-matched sub-track) | 3 (closes G-03) → 12 | 1 → 3 | 2 → 4 | 2 → 4 | 3 → 3 | **26** |
| C-CLOUD (provider-matched) | 2 (closes G-08) → 8 | 1 → 3 | 2 → 4 | 3 → 6 | 3 → 3 | **24** |
| C-RT | 2 (closes G-04) → 8 | 2 → 6 | 2 → 4 | 1 → 2 | 2 → 2 | **22** |
| C-IR | 2 (closes G-06) → 8 | 1 → 3 | 2 → 4 | 2 → 4 | 2 → 2 | **21** |
| C-RM | 2 (closes G-05) → 8 | 2 → 6 | 2 → 4 | 2 → 4 | 1 → 1 | **23** |
| C-OWASP | 2 (closes G-07) → 8 | 2 → 6 | 3 → 6 | 3 → 6 | 2 → 2 | **28** |
| C-ATLAS | 2 (closes G-07) → 8 | 2 → 6 | 3 → 6 | 3 → 6 | 2 → 2 | **28** |
| C-42001-LA | 2 (complements C-42001-LI on G-02) → 8 | 3 → 9 | 2 → 4 | 1 → 2 | 2 → 2 | **25** |
| C-PM | 0 → 0 | 0 → 0 | 2 → 4 | 2 → 4 | 1 → 1 | **9** |

The scores are not sacred — the role-holder's manager has to agree with the
weights. The scores are updated in the annual review as the role-holder
observes the real cost and renewal burden of credentials actually sat.
```

### 2.5 24-month sequenced plan — `portfolio/24-month-plan.md`

The plan over the rolling 24 months. Two concurrency rules keep the plan realistic.

- **No more than two concurrent tracks.** A track is *in progress* from enrolment until exam-pass or deliberate deferral.
- **No more than ~40 study-hours allocated in any one quarter.** <!-- verify-citation: confirm the role's professional-development time budget against the organisation's published policy. -->

```markdown
---
document_class: certifications-24-month-plan
role: AI risk engineer (L25)
scope: CarePilot team
version: 1.0
effective_from: 2026-11-13
horizon_end: 2028-11-13
next_review: 2027-11-13
owner_role: AI risk engineer (self)
reviewer_roles:
  - Tech lead / engineering manager
  - AI governance analyst
status: draft-for-manager-review
---

# 24-Month Certifications Plan — AI Risk Engineer (CarePilot)

## Horizon

- **Start.** 2026-11-13.
- **End.** 2028-11-13.
- **Re-plan.** Annually (2027-11-13). Any out-of-cycle re-plan is
  explicitly logged in the changelog.

## Sequencing

| Quarter | Primary track | Secondary track | Target milestone |
|---|---|---|---|
| 2026-Q4 | C-OWASP (training; not exam-gated) | — | OWASP ML Top 10 training completed; attendance attestation filed. |
| 2027-Q1 | C-AIGP (study) | C-ATLAS (training; not exam-gated) | AIGP study underway; ATLAS training completed; attendance attestation filed. |
| 2027-Q2 | C-AIGP (exam) | — | AIGP exam sat. Enter either `attested` (pass) or `exam-retry-planned` (fail) state. |
| 2027-Q3 | C-42001-LI (study + exam) | — | 42001 Lead Implementer exam sat. |
| 2027-Q4 | C-CIPP (region-matched sub-track; study) | — | CIPP study underway. |
| 2028-Q1 | C-CIPP (exam) | C-CLOUD (provider-matched; study) | CIPP exam sat; cloud credential study underway. |
| 2028-Q2 | C-CLOUD (exam) | C-RM (study) | Cloud credential exam sat; risk-management credential study underway. |
| 2028-Q3 | C-RM (exam) | — | Risk-management credential exam sat. |
| 2028-Q4 | C-RT (study) | — | Red-team credential study underway; carried into the next 24-month window. |

Candidates not entering the first horizon: C-42001-LA and C-IR. Both are
tracked for the **next** 24-month window — C-42001-LA because it benefits
from first holding C-42001-LI, and C-IR because the rubric placed it
below the current window's capacity. These are not rejected; they are
deferred.

## Explicit deferrals and reasons

| Candidate | Reason for deferral to horizon-2 (2028-11 → 2030-11) |
|---|---|
| C-42001-LA | Lead-auditor benefits from first holding Lead Implementer. Attempting both in the first horizon would violate the two-concurrent-track rule. |
| C-IR | Scored below the current-window capacity. The incident-response grounding currently rests on the mod-111 ex-01 → ex-05 artifacts and will be re-evaluated after one cycle of real-incident response. |

## Load limit audit

| Quarter | Primary hours | Secondary hours | Total | Under 40? |
|---|---|---|---|---|
| 2026-Q4 | ~15 (training) | 0 | 15 | ✅ |
| 2027-Q1 | ~25 (study) | ~10 (training) | 35 | ✅ |
| 2027-Q2 | ~15 (exam period) | 0 | 15 | ✅ |
| 2027-Q3 | ~35 (study + exam) | 0 | 35 | ✅ |
| 2027-Q4 | ~30 (study) | 0 | 30 | ✅ |
| 2028-Q1 | ~15 (exam) | ~20 (study) | 35 | ✅ |
| 2028-Q2 | ~15 (exam) | ~20 (study) | 35 | ✅ |
| 2028-Q3 | ~20 (study + exam) | 0 | 20 | ✅ |
| 2028-Q4 | ~25 (study) | 0 | 25 | ✅ |

All hours are illustrative estimates and are pinned against the credential
body's published study guidance before each quarter starts. <!-- verify-citation: confirm the published study-hour guidance from each credential body before the quarter begins. -->

## Dependencies on the organisation

- Professional-development budget approval per quarter (manager).
- Study-time allocation in the sprint planning (tech lead / EM).
- Attestation filing in the role-level attestation file (self).
- Cross-reference into the ex-05 crosswalk's "documented information" row (self + AI governance analyst).
```

### 2.6 Attestation and evidence contract — `attestations/credentials-held.md`

Each credential, once held, appears here. The file is the single place a reviewer looks to see what the role currently attests. Template rows:

```markdown
---
document_class: role-level-certifications-attestation
role: AI risk engineer (L25)
scope: CarePilot team
version: 1.0
effective_from: 2026-11-13
owner_role: AI risk engineer (self)
status: template-empty-at-plan-inception
---

# Credentials Held — AI Risk Engineer (CarePilot)

(Empty at plan inception. The template below is instantiated at each exam-pass or training-completion event. On update, the ex-05 crosswalk's "documented information" row for this plan is re-attested and the ex-01 charter's role-level attestation is re-dated.)

## Template row

- **Credential.** `<family-level name>` — issuing body / provider `<name>`. <!-- verify-citation: confirm body name and current branding. -->
- **Scope.** The gap this closes from §2.2 (`G-01` / `G-02` / ...).
- **Attested on.** `<YYYY-MM-DD>`.
- **Pin.** Certificate ID (or equivalent) `<value>`; filed at `<path or secure-registry pointer>`.
- **Expiry / renewal.** `<YYYY-MM-DD>` or `<cycle description>`.
- **Renewal plan.** Next CPE / CPD activities scheduled.
- **Cross-references.** This row appears in the ex-05 crosswalk's "documented information" bucket; this attestation is the evidence.
```

### 2.7 Rejected candidates — `portfolio/rejected-candidates.md`

An explicit rejection list prevents next-quarter re-litigation.

```markdown
---
document_class: certifications-rejected-candidates
version: 1.0
owner_role: AI risk engineer (self)
---

# Rejected Candidates

Each entry: credential family, reason for rejection, re-evaluation trigger.

| Candidate | Reason for rejection | Re-evaluate if |
|---|---|---|
| C-PM | Does not close any named role gap; rubric-weighted total 9. | Role scope changes to include programme-management responsibilities (e.g. promotion track with explicit programme-management expectations). |
| General IT-audit credentials | Do not close an AI-specific gap; overlap with C-42001-LA where AI-management-system auditing is already covered. | Role scope changes to include formal IT-audit duties beyond AI. |
| Executive-level AI governance credentials | Target L60 responsibilities; the deferral contract assigns these to the head of AI governance. | Role moves to L50 / L60. |
| Non-deployment-region privacy credentials | Target legal frameworks outside the deployment region. | Hospital relocates or opens a cross-region deployment. |
| Modality-specific research credentials (e.g. computer-vision research) | Target the AI evaluation engineer role, not the AI risk engineer role. | Role scope expands to include modality research. |

The rejection list is **not** a list of credentials the industry should not
offer. It is a list of credentials this role, on this team, in this
organisation, has concluded does not close a named gap *at this time*.
```

### 2.8 Review cadence and attestation — `checklists/annual_review_checklist.md`

```markdown
# Annual Review Checklist — Certifications Portfolio Plan

Run every 12 months (next review 2027-11-13).

- The gap list in §2.2 is current: every gap row is still real; closed gaps are marked `closed` with a pointer to the attestation that closed them.
- Every row in §2.3 has had its `<!-- verify-citation: ... -->` markers checked against the credential body's then-current published materials; stale citations are removed and replaced.
- The rubric weights in §2.4 are debated and either re-affirmed or revised with a changelog entry.
- The candidate scores are re-run with the then-current rubric weights; the sequencing in §2.5 is re-derived from the updated scores.
- The 24-month horizon window is shifted by 12 months (not 24) so that there is always a planned 12 months ahead.
- Credentials held in §2.6 have in-cycle renewal status.
- The rejected-candidates list in §2.7 has re-evaluation triggers checked; any triggered re-evaluation is run.
- The ex-05 crosswalk "documented information" row for this plan has been re-attested.
- The ex-01 charter's role-level attestations have been re-dated.
- The plan's version is bumped; the changelog records the change.
```

### 2.9 Portfolio disclaimer — `portfolio/README.md`

A short disclaimer that is reproduced at the top of each of the portfolio files. One place it reads:

```markdown
# Portfolio Disclaimer

The credential catalogue in this directory names credentialing bodies and
credential families at **structural level**. Specific exam formats, hour
counts, renewal fees, body-of-knowledge versions, and prerequisite rules
change across each credential body's publication cycles. Any specific
number or clause in this plan carries a `<!-- verify-citation: ... -->`
marker; the role-holder pins the specific values against the credential
body's current published materials before enrolling and updates the plan
on the next review cycle.

The plan is **not** financial, legal, or career advice. It is a durable
record of the gap-closure reasoning that one role-holder, on one team,
has used to sequence a credentialing programme. Other role-holders in
other teams in other organisations will have different gaps and so
different portfolios.
```

## 3. Validation steps

1. **Every gap has a credential (or an explicit "no credential proposed").** A reviewer picks three rows from §2.2 and confirms each one is either closed by a candidate in §2.3 or is intentionally left uncovered with a one-line reason.
2. **Every candidate closes a named gap.** A reviewer picks three rows from §2.3 and confirms each one names a specific gap ID from §2.2. Rejection of a candidate with no gap in sight (e.g. C-PM) is in §2.7.
3. **Every specific syllabus / cost / renewal number is marked.** A reviewer picks three rows from §2.3 and confirms every specific number, format, or clause carries a `<!-- verify-citation: ... -->` marker.
4. **Rubric is readable and the scoring is reproducible.** A reviewer takes the §2.4 rubric and independently scores two candidates from §2.3; the scores match within 1 point per criterion.
5. **Sequencing honours the two concurrency rules.** No quarter has more than two concurrent tracks; no quarter has more than ~40 study-hours. The load-limit audit table in §2.5 is consistent.
6. **Rejected-candidates list has re-evaluation triggers.** Every rejection in §2.7 carries a condition under which it is re-evaluated.
7. **Credentials-held file is a template.** The §2.6 file does not pre-attest credentials the role has not yet earned.
8. **Review cadence is set and specific.** §2.8 names a next-review date, a bumped version, and a changelog entry.
9. **Scope is this one role.** The plan does not attempt to credential every team member or write the organisation's training programme.
10. **Framework coverage is explicit.** The RMF / 42001 alignment of each candidate is stated (0–3 in the rubric); the plan does not conflate AI-governance programme credentials with AI-research credentials.
11. **Deferral and inheritance are explicit.** The plan inherits from ex-01 charter, ex-05 crosswalk, and mod-101 ex-05 deferral contract; those inheritances appear in the frontmatter of the gap-analysis file.
12. **No evasion-relevant specifics.** The plan does not reproduce exam questions, specific red-team payloads from the team's red-team suite, or specific monitor thresholds. The credential body's syllabi are cited at family-level only.

## 4. Rubric or review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Gap-first authoring | Each candidate names a gap from §2.2 that it closes | Candidates listed without gap mapping |
| R2 | Documented prioritisation rubric | Weights and scores are explicit; a reviewer can re-score | Ranking without a visible rubric |
| R3 | Specific syllabus / cost / renewal numbers marked | Every specific number carries `<!-- verify-citation: ... -->` | Specific numbers reproduced without pinning |
| R4 | Two-concurrency / 40-hour rules honoured | Load-limit audit shows no quarter in violation | Overlapping tracks or overloaded quarters |
| R5 | Explicit rejection list with re-evaluation triggers | Every rejection names a trigger | Candidates vanish without explanation |
| R6 | Credential-held template empty at inception | §2.6 shows template only, not pre-attested credentials | Credentials attested without evidence |
| R7 | Annual review cadence and changelog | §2.8 names the date; changelog records bumps | No cadence set |
| R8 | Scope is single-role | Plan is for one role on one team; inheritance to broader programmes is explicit | Plan drifts into organisation-wide training |
| R9 | Framework coverage explicit | Each candidate's RMF / 42001 alignment is stated | Framework coverage inferred by vibe |
| R10 | Attestation binding to charter and crosswalk | §2.6 template requires re-attestation of the charter and the crosswalk's documented-information row on each credential event | Credentials earned without binding to team artifacts |
| R11 | Honest about uncredentialed tracks | Where no formal credential exists for a gap (e.g. OWASP-badged programme may not exist), the plan names it as "training with attendance attestation" rather than pretending a certification exists | Fabricated credential families |
| R12 | No evasion-relevant specifics | Red-team payloads, detection thresholds, exam questions are not reproduced | Such specifics leak into the plan |

Scoring bands: *Pass* — R1–R11 green and R12 honoured; *Pass with revisions* — R4 or R7 fails but R1 / R3 / R5 / R8 / R11 pass; *Fail* — any of R1 / R3 / R8 / R11 / R12 fails.

## 5. Common mistakes

- **Credential-first authoring.** The author lists every AI-adjacent certification on the market and sequences them by price or marketing visibility. The rejection in §2.7 for C-PM is the archetype — a credential with no gap mapping is cosmetic. Always start from the role-gap analysis; attach credentials to gaps.
- **Vibe-ordered sequencing.** The sequencing is presented without a visible rubric; the reviewer cannot argue with it, so it goes unchallenged until a cost or capacity surprise arrives mid-plan. Write the rubric; let the ordering fall out of it.
- **Reproducing specific syllabus / cost / renewal numbers without pinning.** Certification bodies rev their syllabi, exam formats, prices, and renewal cycles across quarters. Reproducing a number verbatim is an invented-fact error; marker it, pin it before enrolling.
- **Exceeding the two-concurrency rule.** Three or four concurrent tracks look efficient on paper and fail in practice as study time splits and exam-sits slip. Two-concurrency is a load limit; the plan's load-limit audit is the enforcement.
- **No rejection list.** Without an explicit rejection list, the next quarter's "we should add CertX" conversation re-runs the whole sequencing. Write the rejections with triggers.
- **Pre-attesting credentials.** The credentials-held file (§2.6) must be empty at plan inception; credentials are attested only when the exam-pass or training-completion event has occurred.
- **Scope drift to organisation-wide training.** The plan starts naming "what the team should take" or "what the organisation's L&D should offer". That is a different artifact with a different owner. Re-scope to the one role.
- **Framework-coverage vagueness.** Each candidate's RMF / 42001 alignment is stated in the rubric and in the candidate row. Vague "it aligns with best practice" phrasing is a red flag.
- **No binding to team artifacts.** A certification is earned and never re-appears in the ex-05 crosswalk's "documented information" row or the ex-01 charter's role-level attestations. The attestation-and-evidence contract in §2.6 is the enforcement.
- **Treating the plan as one-shot.** A plan that is not re-planned annually is a plan that is out of date within one cycle of a credential body's syllabus revision. The annual review in §2.8 is load-bearing.
- **Fabricating credential families.** The market for AI-specific credentials is still forming; some gaps may not have a mature credential option. Writing in a credential that does not exist, or that the author has not confirmed is being offered, is an invented-fact error. The treatment in C-OWASP and C-ATLAS — "credential if available; attendance attestation otherwise" — is the honest pattern.
- **Reproducing evasion-relevant specifics.** The red-team body-of-knowledge is cited at family level; specific payloads, suite internals, or detection thresholds stay behind the team's register access control.

## 6. References

Only the official sources listed in the work packet and clearly-labelled local module references are used. No specific credentialing body's materials are reproduced verbatim without a `<!-- verify-citation: ... -->` marker.

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — the framework-coverage criterion in the §2.4 rubric is evaluated against the AI 100-1 §5 enumeration. The AIGP candidate (C-AIGP) and the 42001 implementer / auditor candidates (C-42001-LI / C-42001-LA) each claim NIST AI RMF / ISO/IEC 42001 coverage at family level; the specifics are pinned per `<!-- verify-citation: ... -->` before enrolment.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — the OWASP training candidate (C-OWASP) is cited at the project level; whether any accredited OWASP-badged training programme exists is confirmed from the OWASP project's published guidance at enrolment time.
- **MITRE ATLAS.** `https://atlas.mitre.org/` — the ATLAS training candidate (C-ATLAS) is cited at the project level; credential availability is confirmed from the ATLAS project's published guidance at enrolment time.
- **ISO/IEC 42001 (AI management system).** The 42001 Lead Implementer and Lead Auditor candidates (C-42001-LI, C-42001-LA) reference the standard's publicly-available high-level outline (Clauses 4–10 plus Annex A controls) at family level. Specific sub-clauses and Annex A control IDs carry `<!-- verify-citation: ... -->` markers consistent with ex-05's crosswalk discipline.
- **ISO/IEC 31000 (risk-management principles).** The risk-management candidate (C-RM) is cited at family level; provider options and syllabi carry `<!-- verify-citation: ... -->` markers.
- **Module running system and preceding artifacts.**
  - `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-05-role-scope-and-deferral-contract.md` — the deferral contract that the gap analysis inherits from. The role (AI risk engineer, L25) and its deferral chain are defined there.
  - `modules/mod-101-ai-risk-frameworks-and-position/exercise-05-role-scope-and-deferral-contract/SOLUTION.md` — the worked deferral contract for CarePilot that is inherited in §2.2.
  - `modules/mod-112-cross-functional-ai-risk-program-and-communication/exercise-01-risk-program-charter-for-one-team/SOLUTION.md` — the charter whose role expectations and attestations the credential plan binds to.
  - `modules/mod-112-cross-functional-ai-risk-program-and-communication/exercise-05-rmf-and-42001-evidence-crosswalk/SOLUTION.md` — the crosswalk whose gap log (G1–G6) feeds the role-gap analysis in §2.2 and whose "documented information" row is re-attested on every credential event.
- **Exercise prompt.** `lessons/mod-112-cross-functional-ai-risk-program-and-communication/exercises/exercise-06-certifications-portfolio-planner.md`.

<!-- Credential families, bodies, and specific syllabus / cost / renewal
     numbers carry `<!-- verify-citation: ... --` markers where they have
     not yet been pinned against the credential body's current published
     materials. Pinning is scheduled before each credential's enrolment
     and reviewed in the annual review (§2.8). -->

# Exercise 05 — Regulator-Facing Incident Summary Authoring — Reference Solution

## 1. Solution overview

This exercise is the module's **external-communication capstone**: starting from the internal post-mortem we drafted in ex-02 and the synthetic jailbreak incident we triaged in ex-03 on **CarePilot** (the hospital-triage-support RAG assistant from mod-101), we produce a **regulator-facing incident summary** — the document that leaves the organisation under a notification obligation. The deliverable is (a) a reusable summary **template** with the fields regulators typically request, (b) a **worked summary** against the ex-03 incident (clearly labelled synthetic and constructed for teaching), (c) a **tone and redaction checklist** that bridges internal candour and external disclosure, and (d) an **approval-chain workflow** that ensures nothing leaves the org without legal, privacy, security, communications, and executive sign-off.

The reference solution:

- Treats the regulator audience as **jurisdiction-neutral by default**. We define the fields and the authoring discipline; we do **not** memorise specific article numbers, section numbers, or notification deadlines. Where a specific framework (EU AI Act, FDA software-as-a-medical-device, HIPAA breach notification under US OCR, US state attorneys-general consumer-protection) applies, we flag that counsel owns the jurisdictional mapping and we mark any specific citation with `<!-- verify-citation: ... -->` so the student does not invent.
- Reuses the **ex-03 synthetic jailbreak** as the worked incident. CarePilot's triage summariser was induced, via a crafted multi-turn prompt in an uploaded discharge-note PDF, to produce a medication-dosing suggestion that contradicted the retrieval context and omitted a renal-adjustment caveat. The failure was caught in pre-production canary within four hours and no patient-facing output was delivered — the summary is written as if the organisation has a regulator-notifiable obligation that nonetheless requires disclosure of the near-miss. This is a **teaching construct**; the event is synthetic.
- Closes the loop back to the module: the corrective-action section cites the **ex-04 regression fixture** as the structural fix, and references the **mod-109 drift-monitor threshold update** as the detection control that would catch a recurrence.
- Maps the whole authoring workflow at a high level to the **NIST AI RMF GOVERN** function (accountability, incident reporting, external communication) and names the **OWASP ML Top 10** and **MITRE ATLAS** category families the underlying event belongs to, flagging specific entry codes with `<!-- verify-citation: ... -->` rather than guessing.

Design decisions worth calling out:

- **Why jurisdiction-neutral.** A single template that says "fill in your EU AI Act Article X deadline here" bakes in guesses that will age badly and that are legal advice we are not qualified to give. Instead, the template is **structured by information type** (what the regulator needs to understand the event), and counsel owns the jurisdiction-specific overlay — including the legally correct deadline clock and the right recipient.
- **Why we show redaction as a first-class artifact.** The dominant failure mode of a regulator summary written by engineers is leaking content that shouldn't leave the organisation (named engineers, vendor contract terms, raw adversarial prompts that double as an attack payload, internal metrics given to false-precision). A redaction checklist is the control.
- **Why an approval-chain workflow, not a single "sign-off" box.** A regulator submission is simultaneously a legal statement, a privacy statement, a security statement, a public statement, and an executive commitment. We model that as five gates with explicit exit criteria rather than one amorphous approval.
- **Why we explicitly disclaim legal advice.** The solution teaches the engineering discipline of authoring the summary — it is not and cannot be the legal determination of whether, when, or how to notify. The limitation statement is part of the deliverable.

## 2. Worked answer or implementation

### 2.1 Repository layout

```
exercise-05-regulator-facing-incident-summary-authoring/
├── SOLUTION.md                           # this file
├── templates/
│   ├── regulator_summary.template.yaml   # the reusable template
│   └── jurisdiction_overlay.md           # counsel-owned mapping (framework names only)
├── worked/
│   └── 2026-11-synthetic-jailbreak.md    # the worked summary (clearly labelled synthetic)
├── checklists/
│   ├── tone_and_redaction.md
│   └── approval_chain.md
└── links/
    ├── post_mortem_internal.md           # pointer to ex-02 artifact
    ├── triage_record.md                  # pointer to ex-03 artifact
    └── regression_fixture.md             # pointer to ex-04 artifact
```

### 2.2 The regulator-facing summary template (`templates/regulator_summary.template.yaml`)

The template is YAML so that each field is machine-identifiable for review tooling, but the **free-text fields are authored in prose**. Length guidance per field is given in comments.

```yaml
# templates/regulator_summary.template.yaml
# Reusable regulator-facing incident summary template.
# Jurisdiction-neutral: counsel owns the mapping to a specific regime (EU AI Act, FDA,
# HIPAA, state AG, sector regulator). Do not guess article numbers or deadlines.

schema_version: "1.0"
document_class: "regulator-incident-summary"
confidentiality: "external-restricted"      # released under notification obligation only
is_synthetic: false                          # set true for teaching examples

identification:
  incident_id: ""                            # internal tracker ID, stable across revisions
  system_name: "CarePilot"
  intended_purpose: >
    Hospital-triage-support retrieval-augmented generation (RAG) assistant that
    summarises patient history and surfaces decision-support references for a
    licensed clinician. CarePilot is a decision-support tool; a human clinician
    is the decision-maker of record.
  deployment_context:
    care_setting: ""                         # e.g. inpatient triage, ED intake, outpatient follow-up
    user_population: ""                      # e.g. triage nurses and attending physicians
    geography: ""                            # jurisdictions where the system is in use
    lifecycle_stage: ""                      # pilot / limited release / general availability
  operator:
    legal_entity_name: "<ORG-LEGAL-NAME>"    # placeholder; counsel fills the correct legal entity
    registered_address: "<ORG-ADDRESS>"
  point_of_contact:
    role: "Incident Response Lead"           # role title; no personal name
    email_role_alias: "ai-incident-response@<ORG-DOMAIN>"
    phone_role_alias: "<ORG-INCIDENT-PHONE>"

timeline:
  # All times in ISO-8601 with timezone. Use ranges when exactness is not established.
  occurrence_window:
    earliest: ""                             # best-estimate earliest possible occurrence
    latest: ""                               # best-estimate latest possible occurrence
    basis: ""                                # e.g. "bounded by canary deployment window"
  detection:
    at: ""
    detected_by: ""                          # e.g. "automated canary eval regression"
  internal_declaration:
    at: ""                                   # when the incident was formally opened internally
    severity_at_declaration: ""              # internal severity label at declaration time
  external_notification:
    at: ""                                   # when this document is submitted
    deadline_basis: >
      Deadline determined by counsel against the applicable notification obligation.
      This document is submitted within that window.

affected_population:
  # Use ranges, not false precision. If zero, say "zero confirmed" and show the basis.
  confirmed_count_range: ""                  # e.g. "0 confirmed (pre-production canary only)"
  potentially_exposed_range: ""              # e.g. "0–~250 if the model had been promoted to production for one hour"
  basis_for_estimate: >
    Short paragraph explaining how the range was derived (deployment stage,
    request volume, blast-radius reasoning).

nature_of_harm:
  actual_harm: >
    Paragraph on what actually happened to real users. If no actual user-facing
    delivery occurred, say so explicitly and name the control that prevented it.
  potential_harm: >
    Paragraph on what the plausible worst-case harm would have been had the
    control failed. Qualitative; do not invent incidence numbers.
  harm_category:
    - safety_impact: ""                      # none | near-miss | minor | serious | severe
    - privacy_impact: ""                     # none | limited | material | severe
    - fairness_impact: ""                    # none | suspected-differential | confirmed-differential
    - availability_impact: ""                # none | degraded | outage
  linked_risk_register_rows:
    - "risk-register/2025/CarePilot.yaml#risks[prompt-injection]"
    - "risk-register/2025/CarePilot.yaml#risks[clinical-safety-hallucination]"

root_cause_summary:
  # Sanitised from the internal post-mortem. No named engineers, no internal
  # systems that would double as an attack surface description, no raw
  # adversarial prompt text.
  summary: >
    Prose paragraph suitable for an external audience. Describe the failure
    in terms of system behaviour and control gap, not internal blame.
  contributing_factors:
    - ""                                     # one short phrase per factor
  what_the_investigation_has_not_yet_established: >
    Explicit statement of residual uncertainty. "Under investigation" is
    preferable to a confident guess.

mitigation_summary:
  immediate_actions_taken:
    - action: ""
      completed_at: ""
      owning_role: ""
  planned_actions:
    - action: ""
      target_date_or_milestone: ""
      owning_role: ""
      status: "committed"                    # committed | in-progress | complete

action_items:
  # Explicit statuses for regulator visibility. "Complete" means verifiable.
  - id: ""
    description: ""
    status: "committed"                      # committed | in-progress | complete
    evidence_pointer: ""                     # e.g. internal ticket ID; do not expose internal URLs

assurances:
  regression_fixture_added:
    reference: "ex-04 fixture ID"
    description: >
      A regression fixture that exercises the failure mode has been added to
      the CarePilot pre-release evaluation suite.
  monitoring_thresholds_updated:
    reference: "mod-109 drift monitor"
    description: >
      Production monitoring thresholds have been updated so that a recurrence
      would be detected and trigger incident response before user-facing
      delivery. Thresholds and detection latency are stated in ranges to avoid
      disclosing evasion-relevant specifics.
  governance_reviews_updated: >
    Short note on changes to the pre-release review process, if any.

residual_risk_statement: >
  One paragraph describing the risk that remains after the mitigations, and
  the trigger conditions that would prompt a follow-up notification.

attestations:
  reviewed_by:
    - role: "Legal Counsel"
      at: ""
    - role: "Privacy Officer / Data Protection Officer"
      at: ""
    - role: "Chief Information Security Officer"
      at: ""
    - role: "Head of Communications"
      at: ""
    - role: "Executive Sponsor"
      at: ""
  submitting_role: "Incident Response Lead"
```

**Why YAML with prose fields.** Fixed-shape fields are the ones a regulator (or counsel reviewing against a regime) needs to find quickly — identification, timeline, counts, actions. Prose fields are the ones that require narrative judgement — nature of harm, root cause, mitigations. Mixing the two forms matches how a reviewer actually reads the document.

### 2.3 Jurisdiction-neutral overlay note (`templates/jurisdiction_overlay.md`)

We do **not** attempt to encode specific legal thresholds. We list the frameworks counsel most commonly maps against for an AI-system incident at a hospital operator, with the mapping work explicitly delegated.

- **EU AI Act** — high-risk AI systems have a serious-incident notification obligation. Specific article, trigger definition, and deadline are jurisdiction-specific and counsel-owned. <!-- verify-citation: confirm the exact EU AI Act article governing serious-incident reporting for high-risk AI systems and its current notification window before citing. -->
- **US FDA** — software functioning as a medical device (SaMD) may be subject to Medical Device Reporting. Applicability depends on the system's regulatory classification. <!-- verify-citation: confirm which FDA regulation (21 CFR Part 803 or successor) applies to the specific CarePilot classification before citing. -->
- **US HIPAA — HHS Office for Civil Rights** — if the event constitutes a reportable breach of protected health information at a covered entity, there is a breach notification obligation. Applicability is counsel-determined. <!-- verify-citation: confirm the HIPAA breach-notification threshold and clock before citing. -->
- **US state attorneys-general consumer-protection** — several US states have AI-adjacent consumer-protection obligations. These are state-specific. <!-- verify-citation: confirm the specific state statutes that apply to the operator's footprint before citing. -->
- **Sector regulators** — hospital-licensing boards, professional medical boards, and accrediting bodies may have their own reporting obligations distinct from the above. <!-- verify-citation: confirm the specific sector-regulator obligations in-scope for the operator before citing. -->

The overlay note's job in this repository is to **name the frameworks** and route the jurisdictional detail to counsel. The template remains the same across regimes; counsel fills the deadline clock, the recipient address, and any regime-specific fields (e.g. device identifier for FDA, data-protection registration number for EU).

### 2.4 Worked summary — synthetic (`worked/2026-11-synthetic-jailbreak.md`)

The following is the regulator-facing document as it would be submitted. It is reproduced here as a fenced markdown block so that the student can see the full end-to-end shape. **It is synthetic and constructed for teaching; it does not describe a real event and does not attribute any fictional event to any real organisation.**

```markdown
<!-- worked/2026-11-synthetic-jailbreak.md
     SYNTHETIC — teaching artifact only. Do not treat as a record of a real event.
     Reproduces the structure a regulator-facing submission would take. -->

# CarePilot — AI Incident Summary (Synthetic Teaching Example)

**Document class.** Regulator-facing AI incident summary.
**Confidentiality.** External-restricted. Released under a notification obligation.
**Synthetic.** This document describes a synthetic near-miss constructed for training
purposes. It is not a record of an actual event and is not attributed to any real
organisation.

## 1. Identification

- **Incident ID.** CP-2026-NM-0014 (synthetic).
- **System.** CarePilot.
- **Intended purpose.** Hospital-triage-support retrieval-augmented generation (RAG)
  assistant that summarises patient history and surfaces decision-support references
  for a licensed clinician. CarePilot is a decision-support tool; a human clinician
  is the decision-maker of record for any clinical action.
- **Deployment context.** Pre-production canary environment, bounded to an internal
  test cohort. No patient-facing delivery occurred.
- **Operator.** `<ORG-LEGAL-NAME>`, `<ORG-ADDRESS>` (placeholders; counsel fills).
- **Point of contact.** Incident Response Lead, `ai-incident-response@<ORG-DOMAIN>`,
  `<ORG-INCIDENT-PHONE>`.

## 2. Timeline

All times UTC.

- **Occurrence window.** 2026-11-05 between approximately 09:00 and 13:00 UTC.
  Bounded by the canary deployment window.
- **Detection.** 2026-11-05 at approximately 13:12 UTC, by the automated canary
  evaluation regression (unexpected deviation from reference summary on a
  medication-dosing probe).
- **Internal declaration.** 2026-11-05 at 14:00 UTC, severity *High* at declaration
  (pre-production near-miss with potential for clinical safety impact had the model
  been promoted).
- **External notification.** This document is submitted on 2026-11-13. Submission
  deadline determined by counsel against the applicable obligation and met within
  that window.

## 3. Affected population

- **Confirmed count.** Zero confirmed. No patient-facing output was delivered. The
  canary environment is bounded to an internal test cohort consisting solely of
  synthetic case data.
- **Potentially exposed.** Preliminary estimate: zero to approximately low-hundreds
  of clinical users in the hour following a hypothetical promotion to production,
  had the canary control failed to catch the regression. This range reflects the
  smoothed hourly request volume at the pilot site; it is not a user count.
- **Basis for estimate.** Derived from the canary environment scope and the
  smoothed production request-volume range. Estimated; not audited.

## 4. Nature of harm

- **Actual harm.** No actual harm occurred. No patient-facing output was delivered.
  The canary regression blocked promotion of the affected model version to
  production.
- **Potential harm.** Had the regression not blocked promotion, the affected
  summariser could have produced a medication-dosing suggestion that contradicted
  the retrieved clinical context and omitted a renal-adjustment caveat for a
  subset of inputs involving uploaded discharge-note PDFs with adversarially
  structured content. The clinician-in-the-loop requirement is the final
  safeguard; however, we treat omission of a safety-relevant caveat as a serious
  potential harm category regardless of the clinician override.
- **Harm category.**
  - Safety impact: near-miss.
  - Privacy impact: none identified.
  - Fairness impact: under investigation; no differential impact confirmed.
  - Availability impact: none.
- **Linked risk-register rows.** `risks[prompt-injection]`,
  `risks[clinical-safety-hallucination]`.

## 5. Root cause summary

Preliminary findings, under continuing investigation.

A document-ingest path in the retrieval pipeline accepted structured content from
uploaded PDFs without applying the same instruction-stripping policy applied to
free-text channels. A crafted multi-turn pattern embedded in a document's textual
layer was treated by the summariser as authoritative instruction rather than as
quoted content, causing the model to override its retrieval-grounded reasoning on a
narrow slice of medication-dosing inputs.

Contributing factors:

- A policy gap between the free-text ingestion path (hardened) and the
  document-ingestion path (not hardened to the same standard).
- A regression-fixture gap: the pre-release evaluation suite did not include an
  adversarial document-payload probe equivalent to the free-text probe that is
  routinely run.
- A monitoring-threshold gap for the specific regression signal that fired.

What the investigation has not yet established: whether any other document-based
ingestion path exhibits an equivalent gap, and whether the pattern was discovered
by chance by a benign document author or by an actor conducting reconnaissance.
These remain under investigation.

## 6. Mitigation summary

### Immediate actions taken (complete at time of submission)

- The affected model version has been held in canary and is blocked from
  production promotion.
- The document-ingestion path has been reconfigured to apply the same
  instruction-stripping policy as the free-text ingestion path, pending the
  structural fix in §7.
- A targeted probe set has been added to the pre-release evaluation suite and is
  being run on each candidate build.

### Planned actions

- Structural fix: unify the ingestion paths behind a single policy layer so that
  a future divergence of this form cannot recur silently. Target: within the
  current release-engineering sprint.
- Threshold update to the production drift and regression monitor so that an
  analogous deviation is detected earlier.
- External-communication readiness review for the clinical-user population,
  owned by the Head of Communications.

## 7. Action items

| ID | Description | Status | Evidence pointer (internal) |
|---|---|---|---|
| A1 | Regression fixture added to pre-release suite for the specific failure mode | Complete | ex-04 fixture ID, internal ticket |
| A2 | Unify free-text and document ingestion under a shared policy layer | In-progress | release-engineering sprint, internal ticket |
| A3 | Drift-monitor threshold update for the regression signal | In-progress | mod-109 monitor change ticket |
| A4 | Investigate equivalence of pattern on other document-ingest surfaces | In-progress | security review ticket |
| A5 | External clinical-user communication readiness review | Committed | communications ticket |

## 8. Assurances

- A regression fixture covering the specific failure mode has been added to the
  CarePilot pre-release evaluation suite (ex-04 fixture ID). The fixture is run
  on every candidate build and blocks promotion on failure.
- Production monitoring thresholds have been updated per the mod-109 drift
  monitor configuration so that a recurrence of the regression signal would be
  detected and trigger incident response before user-facing delivery. Specific
  threshold values and detection latencies are withheld to avoid disclosing
  evasion-relevant specifics; ranges are available to the regulator on request
  through counsel.

## 9. Residual risk statement

Residual risk after the mitigations is assessed as *reduced but non-zero*. The
immediate mitigations block the specific failure mode; the structural fix (A2)
addresses the class of failure. Until A2 and A4 are complete, we treat the
document-ingestion surface as elevated-risk. A follow-up notification will be
submitted if investigation A4 identifies an additional affected surface, or if
the structural fix materially changes the risk picture.

## 10. Attestations

This submission has been reviewed by Legal Counsel, the Privacy Officer, the
Chief Information Security Officer, the Head of Communications, and the
Executive Sponsor, each at the role level. Dates recorded in the internal
submission record.

Submitting role: Incident Response Lead.

---

*End of synthetic teaching example.*
```

**Why the worked summary is written to a near-miss.** The pedagogical risk of writing to a "real" harm event is that the student learns to overclaim; the pedagogical risk of writing to a trivial event is that the student learns to underdisclose. A near-miss at pre-production canary is the realistic middle: there is no patient-facing harm, there is a potential-harm narrative that the regulator will want to understand, and there is a corrective-action story anchored to the ex-04 regression fixture.

### 2.5 Tone and redaction checklist (`checklists/tone_and_redaction.md`)

The checklist is applied line-by-line to a draft before it enters the approval chain.

**Redact or re-level (always).**

- Personal names of internal engineers, responders, and reviewers — use role titles.
- Specific SaaS-vendor names and vendor contract terms — refer to "the inference provider" or "a managed vector-store provider" unless counsel directs otherwise.
- Internal metric names, dashboards, Slack channel names, ticket IDs, and internal URLs — refer to the category ("the pre-release evaluation suite", "the drift monitor") and keep the specific pointer in an internal appendix.
- Raw adversarial prompt text, exact payloads, exploit reproductions, and exact detection thresholds — these double as attack guidance. Replace with a category description ("a crafted multi-turn pattern embedded in a document's textual layer").
- Internal numerical metrics beyond ranges — round to the nearest meaningful range, not to a spurious decimal place.
- Named model versions and internal checkpoint hashes — refer to "the affected model version" and keep the specific identifier in an internal appendix.
- Attributions that identify individual customers, patients, clinicians, or facilities beyond the aggregate necessary for the regulator to understand the scope.

**Required to disclose.**

- Nature of harm (actual and potential) in terms a non-engineer can parse.
- Scope of potential exposure (as a range).
- Timeline anchors (occurrence window, detection, internal declaration, external notification).
- Mitigations (immediate and planned) with status.
- Residual risk and the trigger conditions that would prompt a follow-up submission.
- Point of contact (role alias, not a personal address).

**How to express uncertainty.**

- Prefer "preliminary", "estimated range", "under investigation", "not yet established".
- Do **not** use false-precision numbers ("245 users affected") when the underlying estimate is a range.
- Do **not** write "we do not believe there is any further risk" — instead, state the residual risk explicitly and the follow-up trigger.
- Do **not** state causation that the investigation has not yet established. "The investigation has not yet established X" is a disclosable sentence; "X did not happen" is often not yet supportable.

**Tone register.**

- First-person-plural, past tense for events, present tense for current state, future tense for planned actions.
- No internal jargon. "Model", "retrieval-augmented generation", and "regression fixture" are acceptable; team names, project codenames, and internal acronyms are not.
- No marketing language. Avoid "robust", "industry-leading", "state-of-the-art" in a regulator-facing document.
- No blame on individuals. Describe the system behaviour and the control gap.

### 2.6 Approval-chain workflow (`checklists/approval_chain.md`)

Five gates, in order. A regulator submission does not leave the organisation until every gate's exit criteria are satisfied. Each gate is a role; no gate is a personal name.

| # | Gate (role) | Review focus | Exit criteria |
|---|---|---|---|
| G1 | Legal Counsel | Jurisdictional applicability; whether an obligation exists; which obligation; deadline clock; recipient identity; privilege and admissions language | Counsel attests the obligation is correctly identified, the clock is correctly set, and no language creates unintended admissions. Specific regime citations entered at this gate. |
| G2 | Privacy Officer / Data Protection Officer | PHI / PII exposure assessment; data-subject rights overlay; any parallel data-protection authority obligation | Privacy officer attests the privacy-impact section is accurate and that any parallel data-protection notification is routed. |
| G3 | Chief Information Security Officer | Security disclosure — whether the root-cause description reveals evasion-relevant specifics; whether residual attack surface is correctly scoped; whether a parallel security advisory to the operator's own security function is needed | CISO attests that the submission does not operationally aid an attacker and that security-side action items (A4 in the worked example) are correctly captured. |
| G4 | Head of Communications | Audience register; consistency with any planned clinician-user communication and any planned public statement; tone | Communications attests the register is correct and that no inconsistency exists with other planned communications. |
| G5 | Executive Sponsor | Organisational commitment to the action items and residual-risk statement; readiness to resource A1–A5; readiness to brief board / audit committee as appropriate | Executive sponsor attests the organisation stands behind the commitments in the document and will resource them. |

Gate ordering rationale. Legal first because it determines whether, when, and to whom the document is going. Privacy second because the privacy-impact finding may change the scope of the obligation. Security third because the security review may materially change the root-cause wording. Communications fourth because consistency with other channels depends on the final wording. Executive last because an executive attestation is a commitment to resource the action items that only becomes stable once the preceding gates have converged.

Each gate produces a timestamped attestation at the role level; the internal submission record holds the identity, the document holds only the role. A gate may escalate back to the preceding gate — an escalation is recorded in the internal log and the exit criteria must be re-cleared.

### 2.7 Framework mapping

- **NIST AI RMF — GOVERN** (accountability and incident reporting). The authoring workflow operationalises the GOVERN function's expectation that AI incidents are tracked, escalated through accountable roles, and reported externally where an obligation applies. The approval chain in §2.6 is the accountability structure. <!-- verify-citation: confirm the exact NIST AI RMF GOVERN sub-category code (e.g. GOVERN 4.x or 5.x) that most precisely covers external incident reporting before citing a specific sub-category code. -->
- **OWASP Machine Learning Security Top 10.** The underlying event sits in the family of **LLM / ML input-manipulation and prompt-injection failures** enumerated by the OWASP ML Top 10 and its LLM-specific companion project. <!-- verify-citation: cite the exact OWASP ML Top 10 entry code and title (and / or OWASP Top 10 for LLM Applications entry) that covers prompt injection via untrusted document inputs before citing a specific code. -->
- **MITRE ATLAS.** The underlying event sits in the ATLAS tactic family for **ML-system evasion / prompt-injection techniques**; the specific technique ID should be cited from ATLAS directly. <!-- verify-citation: cite the exact ATLAS technique ID for prompt-injection via a document ingestion channel before citing a specific technique ID. -->

### 2.8 Limitation statement (part of the deliverable)

This exercise teaches the **engineering discipline** of authoring a regulator-facing incident summary. It is not legal advice. Whether a notification obligation applies, to which regulator, by what deadline, and in what form is a legal determination that depends on jurisdiction, the specific regulatory classification of the system, the operator's contractual and licensing posture, and the facts of the event. Any real submission must be reviewed and cleared by qualified counsel; counsel owns the jurisdictional overlay and the final text. The worked example in §2.4 is synthetic and is for teaching only — it is not a model for any real submission and must not be used as a template without counsel review.

## 3. Validation steps

1. **Template coverage.** Every section listed in §2.2 is present and non-empty when the template is filled in. A reviewer greps for each top-level YAML key in a candidate submission and confirms none are blank.
2. **No personal names.** `rg -n "(^|\s)[A-Z][a-z]+ [A-Z][a-z]+(\s|$)"` against the candidate document returns only role titles or legal-entity names; any match that looks like a personal name is re-checked and re-levelled to a role.
3. **No raw adversarial payloads.** The candidate document does not contain the raw crafted prompt text from the ex-03 incident. A reviewer confirms by diffing the candidate against the ex-03 internal triage record.
4. **Ranges, not false precision.** Numerical fields (`affected_population`, detection latency if disclosed) are expressed as ranges or qualitative labels. No field states a precise number that the investigation has not supported.
5. **Uncertainty language.** At least one of {"preliminary", "estimated", "under investigation", "not yet established"} appears in `root_cause_summary` or `nature_of_harm` for any claim that is not firmly established.
6. **Approval-chain evidence.** Each of the five gates in §2.6 has a recorded attestation at the role level in the internal submission record, and the document's `attestations.reviewed_by` list matches. No gate is skipped.
7. **Synthetic labelling preserved.** For the teaching artifact, the document opens with a clearly visible "synthetic" banner; a reviewer confirms by reading the first fifteen lines.
8. **Framework citations are either verified or marked.** Every citation of an EU AI Act article, FDA section, HIPAA deadline, state-AG deadline, OWASP ML Top 10 entry code, or ATLAS technique ID is either confirmed from the official source or carries a `<!-- verify-citation: ... -->` marker.
9. **Action-item back-link to ex-04.** The `assurances.regression_fixture_added.reference` field cites the ex-04 fixture, and the ex-04 artifact is linkable from the internal submission record.
10. **Limitation statement present.** §2.8 (or its equivalent in the filled-in artifact) states explicitly that the document is engineering discipline, not legal advice, and that counsel review is required.

## 4. Rubric or review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Template follows the shape in §2.2 | All top-level YAML keys are present in the filled-in artifact | Any top-level key missing, or structure flattened into unstructured prose |
| R2 | Jurisdiction-neutral authorship | Specific article / section / deadline numbers either verified from the official source or marked with `<!-- verify-citation: ... -->` | Specific legal citations present without verification and without the marker |
| R3 | Worked summary clearly synthetic | "Synthetic", "teaching", or equivalent banner is visible in the first screen of the document | Worked summary presented as if it described a real event |
| R4 | No personal names | Only role titles and legal-entity placeholders appear | Any personal name of an engineer, responder, reviewer, patient, or clinician |
| R5 | No raw adversarial payload content | Attack is described by category, not by reproducible payload | Raw prompt text or reproducible exploit content disclosed |
| R6 | Ranges over false precision | Affected-population and other quantitative fields expressed as ranges or qualitative labels where exact numbers are not supported | Spurious precise numbers (e.g. "245 users") the investigation has not supported |
| R7 | Uncertainty language present | At least one of the preferred hedging phrases appears in the root-cause or harm sections where applicable | Confident claims about unestablished causation |
| R8 | Approval-chain evidence | All five gates attested at role level and recorded in the internal submission record | Any gate skipped, or attestations recorded by personal name |
| R9 | Tie-back to ex-04 regression fixture and mod-109 monitoring | `assurances` section cites the ex-04 fixture and the mod-109 monitor update | No linkage to the corrective-action artifacts |
| R10 | Limitation statement present | Document states it is engineering discipline, not legal advice, and that counsel review is required for real submissions | Limitation statement missing |
| R11 | Framework mapping present | NIST AI RMF GOVERN family named; OWASP ML Top 10 and MITRE ATLAS categories named at family level with `<!-- verify-citation: ... -->` where specific codes are not verified | Framework mapping absent, or specific codes invented |
| R12 | No invented facts | Any fact not supported by the official sources or the local module context carries a `<!-- source-pending: ... -->` marker | Fabricated attribution, deadline, or finding |

Scoring bands: *Pass* — R1–R11 green and R12 honoured; *Pass with revisions* — one of R6 / R7 / R9 fails but R1–R5 and R8 pass; *Fail* — any of R2 / R3 / R4 / R5 / R8 / R10 / R12 fails.

## 5. Common mistakes

- **Copy-pasting the internal post-mortem.** The internal post-mortem is written for engineers, with named individuals, raw prompts, internal metrics, vendor names, and ticket IDs. A regulator summary is a different document for a different audience; the redaction checklist in §2.5 is the control.
- **Overclaiming no-harm.** "No patient was harmed" is often supportable for a pre-production near-miss, but "we are confident nothing further will happen" is not. The residual-risk statement is the correct place for honest uncertainty.
- **Underclaiming scope.** Hiding a near-miss behind "no user-facing impact" and omitting the potential-harm narrative misleads the regulator and burns trust the next time there is a real event. The potential-harm paragraph is required.
- **Spurious precision.** "245 users potentially affected" when the real support is a range reads as careful but is false precision; the regulator will discount the whole document. Use ranges.
- **Citing specific legal clauses without verification.** Writing "per EU AI Act Article N, we are submitting within M days" when Article N and M were not verified from the current text of the regulation is a professional error that a reviewer will find. Use the framework name and the `<!-- verify-citation: ... -->` marker until counsel confirms.
- **Leaking evasion-relevant detail.** Publishing the exact prompt or the exact detection threshold tells the next adversary where the fence is. State the category of the attack and keep the specifics in an internal appendix available to the regulator on request through counsel.
- **Naming individuals.** Writing "Engineer A detected the regression at 13:12" exposes an individual to downstream legal and personal risk. Use role titles — "the on-call responder detected the regression".
- **Skipping gates.** The five gates in §2.6 exist because each one surfaces a different class of error. Collapsing to a single executive sign-off leads predictably to privilege errors, privacy-scope errors, or evasion-leak errors.
- **Treating the exercise as legal advice.** The exercise teaches the authoring discipline. Real submissions require counsel review; the limitation statement in §2.8 is part of the deliverable.
- **Forgetting the back-link.** The assurances section without a pointer to the ex-04 regression fixture and the mod-109 monitor update is a statement without evidence. The back-link is what turns "we have fixed it" into "we have fixed it and here is how a reviewer can confirm".

## 6. References

Only the official sources listed in the work packet (and clearly-labelled module references) are used.

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — this exercise operationalises the AI RMF **GOVERN** function for accountability and incident reporting. The authoring workflow (template, redaction, approval chain) is the engineering artifact that satisfies GOVERN-family expectations at the role of incident reporter. <!-- verify-citation: cite the exact GOVERN sub-category code that most precisely covers external incident reporting after reading the current AI RMF Core. -->
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — the OWASP ML Top 10 enumerates the risk families against which the underlying event is classified (input manipulation / prompt injection). <!-- verify-citation: cite the exact OWASP ML Top 10 entry code and title (and the OWASP Top 10 for LLM Applications entry if applicable) for prompt injection via document inputs. -->
- **MITRE ATLAS.** `https://atlas.mitre.org/` — the ATLAS matrix enumerates the tactics and techniques for ML-targeted attacks; the underlying event is categorised against the ATLAS prompt-injection family. <!-- verify-citation: cite the exact ATLAS technique ID for prompt injection via a document ingestion channel. -->
- **Module running system.** CarePilot — hospital-triage-support RAG assistant, introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md` and reused across this module for continuity.
- **Preceding module artifacts cited by the worked summary.**
  - `lessons/mod-111-incident-response-and-postmortems/exercises/exercise-02-blameless-post-mortem-and-five-whys.md` — the internal post-mortem the regulator summary is distilled from.
  - `lessons/mod-111-incident-response-and-postmortems/exercises/exercise-03-ai-specific-incident-triage-and-severity-scoring.md` — the synthetic jailbreak incident the worked summary is written against.
  - `lessons/mod-111-incident-response-and-postmortems/exercises/exercise-04-regression-fixture-authoring-from-incidents.md` — the regression fixture cited as the structural corrective action.
  - `lessons/mod-109-ai-observability-and-drift-monitoring/` — the monitor threshold update cited in the assurances section.
- **Exercise prompt.** `lessons/mod-111-incident-response-and-postmortems/exercises/exercise-05-regulator-facing-incident-summary-authoring.md`.

<!-- The worked summary in §2.4 is synthetic and labelled as such. Specific regulatory
     citations are deliberately deferred to counsel via `<!-- verify-citation: ... --`
     markers rather than guessed. -->

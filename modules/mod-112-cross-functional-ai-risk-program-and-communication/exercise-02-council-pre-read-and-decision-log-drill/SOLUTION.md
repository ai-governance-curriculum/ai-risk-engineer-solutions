# Exercise 02 — Council Pre-Read and Decision-Log Drill — Reference Solution

## 1. Solution overview

This exercise is the **council-facing authorship** drill: the CarePilot team (the running hospital-triage-support RAG assistant from mod-101) is on the AI risk council agenda, and the AI risk engineer must author (a) the **pre-read packet** the council members read before the meeting, and (b) the **decision log** that captures what the council decided and why. These are the two documents that make council meetings work — a council that walks in cold wastes 90 minutes catching up; a council with no decision log leaves no record of why it said yes or no, and the same debate recurs three meetings later.

The reference solution:

- Anchors the drill in a **specific council decision ask** that the CarePilot team realistically brings forward: a **modality-expansion** request (the ex-01 charter's **T1** trigger) — enabling an uploaded-document ingestion path for discharge notes, where the team has already seen a synthetic near-miss (the mod-111 ex-03 jailbreak). This grounds the drill in a decision that has a clear acceptance / rejection shape, non-trivial residual risk, and ties back to artifacts the student has already produced in previous modules.
- Builds the pre-read around **four questions the council must answer**: *What is being asked? What does the harm picture look like? What evidence supports the ask? What is the residual risk and what triggers a return?* Every section of the pre-read serves one of those four questions. Anything that does not serve one of them is cut.
- Publishes the pre-read on the mod-112 ex-01 **charter's five-day pre-read SLA**. The pre-read is dated and versioned; a late pre-read is treated as a procedural defect and is called out in the decision log.
- Captures the decision in a **decision-log entry** — date, ask, decision, rationale, dissents, conditions, triggers for return, attestations — so the decision is auditable by role-level signoff without exposing personal identities beyond what the council record already holds.
- Handles the three realistic council outcomes — **approve**, **approve with conditions**, **decline** — by showing all three in the worked example. A drill that only models approval teaches the author to write pre-reads that pre-cook the answer.
- Keeps the whole workflow **jurisdiction-neutral**: specific legal thresholds are counsel's territory and are flagged with `<!-- verify-citation: ... -->` where cited rather than guessed.

Design decisions worth calling out:

- **Why modality expansion as the ask.** It is the ex-01 charter's cleanest escalation trigger (T1), it has a non-trivial harm surface, and it ties cleanly back to the mod-111 ex-03 synthetic incident and the mod-111 ex-04 regression fixture. The student practises the full loop: team sees near-miss → team builds fixture → team wants the capability on → council has to decide.
- **Why four-question structure.** Councils fail when the pre-read is organised around the team's work (what they did) rather than the council's job (what they need to decide). The four questions are the council's job; the pre-read is organised around them.
- **Why three outcomes shown.** Approve-only shows bias. Decline-only shows cowardice. Approve-with-conditions is the realistic middle, and the student needs to see how conditions are written as discharge-able exit criteria, not vibes.
- **Why role-level attestations.** Decision logs are retrieved during audits, during regulator queries, and during the next council meeting that re-opens the same question. Role-level attestations survive attrition; personal-name attestations do not.

## 2. Worked answer or implementation

### 2.1 Repository layout

```
exercise-02-council-pre-read-and-decision-log-drill/
├── SOLUTION.md                              # this file
├── pre-read/
│   └── 2026-11-20-council-pre-read.md       # the pre-read packet
├── decision-log/
│   └── 2026-11-20-decision-log.md           # the decision log entry
├── templates/
│   ├── pre_read_template.md                 # reusable template
│   └── decision_log_template.md             # reusable template
└── checklists/
    ├── pre_read_authoring_checklist.md
    └── decision_log_authoring_checklist.md
```

### 2.2 The pre-read template — `templates/pre_read_template.md`

```markdown
---
document_class: ai-risk-council-pre-read
council_meeting_date: ""        # YYYY-MM-DD
published_at: ""                # publish ≥ 5 business days before the meeting
scope: single-team-ask
team: ""
author_role: "AI risk engineer"
co_author_role: ""              # tech lead if the ask is a release-gate ask
status: "pre-read-published"    # draft | pre-read-published | superseded
---

# Pre-read — <team> — <one-line ask> — <meeting date>

## A. What is being asked?

One paragraph. The council's first question is "what, specifically, are we
being asked to decide?" Answer it before anything else. State the ask as a
boolean: approve / approve-with-conditions / decline against a specific
capability or scope change. If the ask is non-binary (e.g. "pick one of
three approaches"), name the options and the recommended option up front.

## B. What does the harm picture look like?

One section. The council's second question is "what could go wrong, and how
badly?" Summarise the harm model delta the ask introduces — new rows or
re-levelled rows. Cite the existing harm model file; do not reproduce it.

## C. What evidence supports the ask?

One section per evidence family. The council's third question is "is this
ready to ship?" Name the evaluation suite, the red-team coverage, the
monitor coverage, and the incident history. Each paragraph names a file
path and a headline metric; the full evidence lives in-repo.

## D. What is the residual risk and what triggers a return?

One paragraph on residual risk and one paragraph on the conditions under
which the team comes back to the council. A residual-risk paragraph that
reads "we believe the risk is now zero" is a drafting defect; a residual-
risk paragraph that reads "under conditions X, Y, Z we would re-open" is
usable.

## E. Attachments

List of linked artifacts (charter reference, harm model, risk register
rows, evaluation reports, red-team reports, monitor specs, incident
records). Each attachment is cited by path and version/commit hash.

## F. Decision requested

A single-line statement of the decision asked of the council and the
earliest meaningful date by which the decision is needed.
```

### 2.3 The decision-log template — `templates/decision_log_template.md`

```markdown
---
document_class: ai-risk-council-decision-log-entry
council_meeting_date: ""
entry_id: ""                    # stable ID across revisions, e.g. council-2026-11-20-01
pre_read_reference: ""          # path to the pre-read it decides on
status: "final"                 # draft | final | superseded-by
superseded_by: ""               # entry_id, if any
---

# Decision — <entry_id> — <one-line ask>

## 1. Ask

One paragraph. Reproduces the ask from pre-read §A verbatim (or references
it if space is a concern). A decision log that does not restate the ask is
illegible six months later.

## 2. Decision

One of: *approve*, *approve-with-conditions*, *decline*, *defer*.

- *approve* — the ask proceeds as framed.
- *approve-with-conditions* — the ask proceeds only after the conditions in
  §4 have been met. Conditions are **discharge-able**: each has an exit
  criterion.
- *decline* — the ask does not proceed. The reason is in §3.
- *defer* — the council needs additional information before deciding.
  §4 names exactly what additional information and the target meeting at
  which the decision will be taken.

## 3. Rationale

One or more paragraphs on why the council decided what it decided.
Rationale is the thing that survives a year and that a future council will
read when a similar ask comes forward. Name the evidence that was decisive
and the harm category that was most load-bearing.

## 4. Conditions (if approve-with-conditions) or additional information needed (if defer)

List. Each item has: condition / information request, owning role, exit
criterion, target discharge date.

## 5. Dissents

Named dissents by role (not personal name), each with a one-line summary
of the dissenter's position and whether the dissent was recorded as a
reservation or as a formal objection.

## 6. Triggers for return to the council

List. Each trigger describes a state that would cause the team to come
back to the council — e.g. a monitor tripping, a residual-risk
realisation, a condition failing to discharge by its target date.

## 7. Attestations

- Council chair — attested on <date>.
- Legal counsel — attested on <date>.
- Clinical-safety officer (if clinical impact) — attested on <date>.
- Privacy officer (if privacy impact) — attested on <date>.
- Submitting team's AI risk engineer — attested on <date>.

## 8. Changelog

- <date>: entry created.
- <date>: entry superseded by <entry_id>, reason: …

```

### 2.4 Worked pre-read — `pre-read/2026-11-20-council-pre-read.md`

The pre-read is dated 2026-11-15 for a council meeting on 2026-11-20 — a 5-business-day pre-read window as the ex-01 charter requires.

```markdown
---
document_class: ai-risk-council-pre-read
council_meeting_date: 2026-11-20
published_at: 2026-11-15
scope: single-team-ask
team: CarePilot
author_role: AI risk engineer
co_author_role: Tech lead / engineering manager
status: pre-read-published
---

# Pre-read — CarePilot — enable document-ingestion for discharge notes — 2026-11-20

## A. What is being asked?

The CarePilot team is asking the AI risk council to **approve, with the
conditions listed in §D**, the enablement of a **document-ingestion input
path** for structured discharge notes (PDF only, authored by the admitting
clinician's EHR export) in the triage-support RAG workflow. This is a
**modality-expansion** ask under the T1 escalation trigger in the CarePilot
team's charter (`charter/CarePilot-team-risk-charter.md` §6).

Specifically:

- Scope of the ask: enable PDF discharge-note ingestion in the retrieval
  pipeline for the ED triage flow. No other document types; no image
  attachments; no voice; no agentic tool-use.
- Target go-live: no earlier than 2026-12-15, contingent on discharge of
  the conditions in §D.
- The team's recommendation to the council: *approve with conditions*.

## B. What does the harm picture look like?

The modality adds a harm-model delta of three new rows (full detail in
`docs/systems/carepilot/harm-model.md`, delta rows tagged `v1.1`):

- **H-PI-03 — prompt injection via document textual layer.** An adversarially
  crafted multi-turn pattern embedded in a PDF's textual layer is treated
  by the summariser as authoritative instruction rather than as quoted
  content. Severity: **High**. The team has already observed a synthetic
  near-miss against this failure in pre-production canary; see
  `incidents/carepilot/candidates/2026-11-05-synthetic-canary.md` (mod-111
  ex-03). The near-miss was caught by the canary regression and did not
  reach any patient-facing output.
- **H-PI-04 — retrieval poisoning via document cross-reference.** A
  discharge note that cites a non-existent protocol or a stale protocol
  could cause the summariser to retrieve and surface a stale caveat.
  Severity: **Medium**. Mitigation: retrieval-corpus provenance check on
  every cited reference.
- **H-PR-02 — unintended PHI replay from a prior discharge note.** A
  discharge note contains PHI by definition; a carry-over bug in session
  state could cause PHI from note A to appear in a summary for patient B.
  Severity: **High**. Mitigation: session-isolation hard boundary; no
  cross-session retrieval.

The fairness and privacy slices of the harm model are unaffected at the
category level; the privacy row H-PR-02 is new. The clinical-safety slice
gains H-PI-03 as its load-bearing new entry.

## C. What evidence supports the ask?

### C.1 Evaluation coverage

- **Groundedness and retrieval-alignment** on the shadow eval set
  (`evals/carepilot/shadow/2026-11-R46/`): headline groundedness 94.8% on
  the discharge-note probe set (n=1,420 synthetic probes), unchanged vs.
  the free-text baseline (94.9%). Delta is within the risk-register
  performance envelope.
- **Red-team suite** (mod-104) now includes a document-payload probe class
  pinned at commit `a4f2…c7d1` of the `carepilot-rt-suite` repo, with 118
  attack cases across direct, indirect, and multi-turn payloads. On the
  current candidate model, attack-success-rate is in the range 1–3%
  (ranges, not false-precision) against the target of ≤ 5% defined in
  `risk-register/2025/CarePilot.yaml#security-envelope`.
- **Fairness slices** (mod-106): no material subgroup delta; disparate-
  impact ratio within the envelope floor.
- **Privacy evaluation** (mod-107): PII-leak rate below the envelope
  threshold on the canary cohort; no cross-session leak observed on the
  session-isolation test.

### C.2 Regression-fixture coverage

The mod-111 ex-04 regression fixture ID `rt-doc-injection-2026-11`
exercises the specific failure mode observed in the 2026-11-05 near-miss.
The fixture is pinned in the pre-release suite and blocks promotion on
failure. It has stayed green across the last four candidate builds.

### C.3 Production monitoring

- The mod-109 drift monitor has been extended with a signal for the
  regression pattern (`monitors/carepilot/drift-doc-inject.md`). Threshold
  change: expressed as a range (specific thresholds withheld from the
  pre-read to avoid disclosing evasion-relevant specifics; stored in
  `risk-register/2025/CarePilot.yaml#monitor-doc-inject`).
- HITL override rate is a first-class metric; the on-call playbook has an
  entry for a trend-break on override rate specific to document-ingestion
  inputs.

### C.4 Incident history

- One synthetic near-miss (`2026-11-05`), caught by canary, no
  patient-facing delivery. Post-mortem: `postmortems/carepilot/2026-11-05.md`
  (mod-111 ex-02). The internal post-mortem identified the regression-
  fixture gap that is now closed (A1 in the post-mortem's action
  table). The structural fix (A2: unify free-text and document ingestion
  under a shared policy layer) is the condition the team asks the council
  to make a prerequisite of go-live (§D-C1).

## D. What is the residual risk and what triggers a return?

**Residual risk.** After the conditions in §D-C are discharged, the residual
risk is assessed as *reduced but non-zero*. The specific failure mode
(H-PI-03) is covered by a regression fixture and a dedicated monitor; the
class of failure (document-layer prompt injection against any future
ingest surface) remains an open area that is mitigated at the policy
layer but not yet exhaustively covered by tests. H-PR-02 (PHI replay) is
mitigated by session isolation; the residual probability is bounded by
the session-isolation test's coverage.

**Conditions the team proposes (`approve-with-conditions`).**

- **C1.** Structural fix landed — free-text and document ingestion unified
  behind a shared policy layer. Owner: tech lead / EM. Exit criterion:
  release-engineering sprint merge + green pre-release suite. Target
  discharge: 2026-12-12.
- **C2.** Red-team attack-success-rate for the document-payload probe
  class ≤ 3% on the final candidate build (tightened from the 5%
  envelope floor for the go-live window). Owner: AI risk engineer. Exit
  criterion: signed red-team report. Target discharge: 2026-12-12.
- **C3.** Drift monitor and HITL-override monitor both live in production
  with the on-call playbook updated. Owner: AI infra / MLOps engineer.
  Exit criterion: monitor green on the canary cohort for 7 consecutive
  days before promotion. Target discharge: 2026-12-13.
- **C4.** External-communication readiness review for the clinical-user
  population. Owner: head of communications. Exit criterion:
  communications plan attested. Target discharge: 2026-12-10.
- **C5.** Follow-up council review scheduled 90 days post go-live to
  attest residual risk against observed production behaviour. Owner:
  council chair. Exit criterion: calendar invite accepted. Target
  discharge: 2026-12-18.

**Triggers for return to the council.**

- Any realisation of H-PI-03, H-PI-04, or H-PR-02 in production.
- Red-team attack-success-rate crossing above 3% on any pinned build.
- Drift-monitor or override-rate trend-break lasting more than 48 hours.
- Any extension of the modality (new document type, image, voice,
  agentic action) — triggers a new T1 escalation.
- Any regulated-domain change (new care setting, new patient population,
  new user class) — triggers a T2 escalation.

## E. Attachments

| # | Artifact | Path | Pin |
|---|---|---|---|
| 1 | CarePilot team risk programme charter | `charter/CarePilot-team-risk-charter.md` | v1.0 |
| 2 | Harm model (delta v1.1) | `docs/systems/carepilot/harm-model.md` | commit `b19e…4f30` |
| 3 | Risk register rows | `risk-register/2025/CarePilot.yaml#risks[H-PI-03]` and `#risks[H-PI-04]` and `#risks[H-PR-02]` | 2026-11-14 revision |
| 4 | Shadow eval report | `evals/carepilot/shadow/2026-11-R46/report.json` | commit `c71a…9d2e` |
| 5 | Red-team suite | `carepilot-rt-suite/` | commit `a4f2…c7d1` |
| 6 | Regression fixture | `pre-release-suite/rt-doc-injection-2026-11.yaml` | 2026-11-14 revision |
| 7 | Monitor spec (document injection) | `monitors/carepilot/drift-doc-inject.md` | v1.0 |
| 8 | Post-mortem (synthetic near-miss) | `postmortems/carepilot/2026-11-05.md` | final |
| 9 | Incident triage record | `incidents/carepilot/candidates/2026-11-05-synthetic-canary.md` | final |

## F. Decision requested

The council is asked to **approve, with the conditions C1–C5 in §D**, the
enablement of PDF discharge-note ingestion in the CarePilot triage-support
RAG workflow. The earliest meaningful decision date is **2026-11-20** (this
meeting); the earliest meaningful go-live date is **2026-12-15**, contingent
on discharge of C1–C5.
```

### 2.5 Worked decision log — `decision-log/2026-11-20-decision-log.md`

The decision log reflects a realistic council outcome. For the drill, the reference presents the single entry that the council actually produced; the template is structured so that an *approve*, an *approve-with-conditions*, or a *decline* produces the same shape.

```markdown
---
document_class: ai-risk-council-decision-log-entry
council_meeting_date: 2026-11-20
entry_id: council-2026-11-20-01
pre_read_reference: pre-read/2026-11-20-council-pre-read.md
status: final
superseded_by: ""
---

# Decision — council-2026-11-20-01 — CarePilot document-ingestion enablement

## 1. Ask

The CarePilot team asked the AI risk council to approve, with the conditions
C1–C5 in the pre-read §D, the enablement of a PDF discharge-note ingestion
input path in the triage-support RAG workflow, under the T1 modality-
expansion escalation trigger of the CarePilot team charter.

## 2. Decision

**Approve with conditions.** The council approves the enablement, contingent
on the discharge of conditions C1–C5 as stated in the pre-read §D, with the
amendments to C2 and C5 in §4 below. Go-live is authorised no earlier than
2026-12-15.

## 3. Rationale

The council finds that the team has mapped the three new harm-model rows
(H-PI-03, H-PI-04, H-PR-02) credibly, has evidence of the structural
mitigation in progress (C1), has a regression fixture that specifically
exercises the near-miss failure mode (`rt-doc-injection-2026-11`), has a
production monitor that would catch a recurrence, and has an incident
history that is honest about a synthetic near-miss rather than suppressing
it. The red-team attack-success-rate is within the envelope but the
council agrees with the team's proposed tightening from 5% to 3% for the
go-live window (C2); the council further tightens the follow-up review
cadence from 90 to 60 days post go-live (C5') to reflect the novelty of
the modality. The council finds that the modality is bounded (PDF only,
EHR-authored discharge notes only, no image, no voice, no agentic
tool-use) and that any extension triggers a new T1.

The most load-bearing evidence item was the **mod-111 ex-04 regression
fixture** — the council takes a far stronger position on approval when a
known failure mode is covered by a permanent fixture that blocks
promotion on failure than when it is covered only by a one-off eval
report. The most load-bearing harm row was **H-PI-03**, because the
synthetic near-miss demonstrated the failure was non-theoretical.

## 4. Conditions

| # | Condition | Owner | Exit criterion | Target discharge |
|---|---|---|---|---|
| C1 | Structural fix (unified ingestion policy layer) landed | Tech lead / EM | Release-engineering sprint merge + green pre-release suite | 2026-12-12 |
| C2' (amended) | Red-team attack-success-rate for document-payload probe class **≤ 3%** on the final candidate build; the amendment raises the bar from "≤ 3% proposed" to "≤ 3% required and signed by the AI eval engineer" | AI risk engineer with co-sign by AI eval engineer | Signed red-team report | 2026-12-12 |
| C3 | Drift and HITL-override monitors live in production; on-call playbook updated | AI infra / MLOps engineer | Monitors green on canary cohort for 7 consecutive days before promotion | 2026-12-13 |
| C4 | External-communication readiness review for the clinical-user population | Head of communications | Communications plan attested | 2026-12-10 |
| C5' (amended) | Follow-up council review 60 days post go-live (not 90) to attest residual risk against observed production behaviour | Council chair | Calendar invite accepted | 2026-12-18 |

## 5. Dissents

- **Privacy officer — reservation (not objection).** The privacy officer
  recorded a reservation that H-PR-02 (PHI replay) is marked High severity
  while the mitigation (session isolation) is mostly new code; the
  reservation asks for a second privacy audit at the 60-day review (C5')
  and will be discharged there.
- **Clinical-safety officer — no dissent.** Attested conditions as sufficient.
- **Head of communications — no dissent.**
- **Legal counsel — no dissent.** Attested that the modality expansion
  does not change the regulatory classification; a T2 is not triggered.

## 6. Triggers for return to the council

The team returns to the council when any of the following occurs:

- Any production realisation of H-PI-03, H-PI-04, or H-PR-02.
- Red-team attack-success-rate crossing above 3% on any pinned build.
- Drift-monitor or override-rate trend-break lasting more than 48 hours
  in production.
- Any extension of the modality beyond the approved scope (new document
  type, image, voice, agentic action) — new T1.
- Any regulated-domain change (new care setting, new patient population,
  new user class) — new T2.
- C1–C5' not discharged by the target dates — treated as a request for
  an extension, decided at the next council meeting.
- The 60-day follow-up review (C5') decides whether the residual risk
  posture is sustained, further tightened, or the capability is pulled.

## 7. Attestations

Reviewed and attested by role at the meeting of 2026-11-20.

- Council chair — attested on 2026-11-20.
- Legal counsel — attested on 2026-11-20.
- Clinical-safety officer — attested on 2026-11-20.
- Privacy officer — attested on 2026-11-20 (with reservation, §5).
- Head of communications — attested on 2026-11-20.
- AI governance analyst — attested on 2026-11-20.
- Submitting team's AI risk engineer — attested on 2026-11-20.

## 8. Changelog

- 2026-11-20: entry created and finalised at the meeting.
```

### 2.6 Pre-read authoring checklist — `checklists/pre_read_authoring_checklist.md`

A line-by-line checklist the author runs before publishing the pre-read.

**Deadline.**
- Pre-read published ≥ 5 business days before the meeting. A late pre-read is a procedural defect and should be called out in the decision log.

**Structure.**
- Four sections A–D exist and each answers its question.
- §A states the ask as a boolean or named option-pick.
- §B is a harm-model delta, not a harm-model re-write.
- §C has at least four evidence families (eval, red-team, monitors, incident history) and names a file path + headline metric per family.
- §D's residual-risk paragraph does not claim zero risk.
- §D's conditions are discharge-able — each has an exit criterion, not a vibe.
- §D's triggers for return are mechanical — same discipline as the charter's escalation triggers.

**Content hygiene.**
- No raw adversarial payload text.
- No personal names (beyond attestations, which are at role level).
- No internal URLs or ticket IDs in free-text that would read as evidence; file paths that live in-repo are acceptable.
- No headline product-accuracy numbers passed off as risk metrics.
- All ranges are ranges where the underlying data does not support a precise number.

**Attachments.**
- Every attachment is pinned at a version or commit hash.
- At least one attachment is the charter; at least one is the risk register.

**Decision requested.**
- §F states the ask again as a single line and names the earliest meaningful decision date and the earliest meaningful go-live date.

### 2.7 Decision-log authoring checklist — `checklists/decision_log_authoring_checklist.md`

A line-by-line checklist the author runs within the meeting (for in-session capture) and within 24 hours (for the final entry).

**Capture during the meeting (in-session minutes).**
- The ask (§1) is captured verbatim or by direct reference to the pre-read.
- The decision (§2) is one of the four approved verbs: *approve*, *approve-with-conditions*, *decline*, *defer*. No "approved subject to iteration" — that is *approve-with-conditions*.
- The rationale (§3) is captured at the meeting, not reconstructed afterwards. Reconstruction is a documented failure mode; an in-session scribe is the control.
- Every amendment to a condition is captured with a primed version number (e.g. C2 → C2').
- Dissents (§5) are captured at role level, not personal name. A reservation is distinguished from a formal objection.
- Triggers for return (§6) are captured as mechanical conditions.

**Finalisation within 24 hours.**
- Attestations (§7) collected at role level. If a role-holder cannot attest at the meeting, the attestation date is the later date they attested.
- Changelog (§8) updated.
- Entry ID is stable across revisions. If a revision is material, create a new entry with `supersedes` / `superseded_by`; do not rewrite history.
- Entry is linked back from the pre-read's final version (as a published "decision" reference).

**Audit discipline.**
- A decision log is retrieved during audits, during regulator queries, and during the next council meeting that re-opens the same question. Write it so that a reader six months later with no context understands why the council said what it said.
- A council that approves without recording rationale is a council that will re-debate the same question next quarter.
- A dissent captured only as "no dissent" when there was a reservation is a drafting defect.

## 3. Validation steps

1. **Pre-read published on SLA.** The pre-read's `published_at` is ≥ 5 business days before `council_meeting_date`. A reviewer checks the two dates in the frontmatter.
2. **Four questions answered.** Pre-read sections A–D exist, each addressing its question. A reviewer reads only the section headings and confirms they match "What is being asked? / What does the harm picture look like? / What evidence supports the ask? / What is the residual risk and what triggers a return?"
3. **Ask is a boolean or named-option pick.** §A ends with a sentence of the form "approve / approve-with-conditions / decline against <specific scope>" or "approve option X of {A, B, C}".
4. **Harm delta is a delta, not a re-write.** §B references the existing harm model file and cites specific new or re-levelled rows. The pre-read does not reproduce the full harm model.
5. **Evidence has at least four families.** §C contains at least four sub-sections (evaluation, red-team, monitoring, incident history). Each sub-section cites a file path and a headline metric.
6. **Residual risk is non-zero.** §D's residual-risk paragraph does not claim zero risk; it names a specific residual-risk shape and the triggers for return.
7. **Conditions are discharge-able.** Every condition in §D has: condition text, owner (role), exit criterion, target discharge date.
8. **Decision verbs are restricted.** The decision log's §2 uses one of *approve*, *approve-with-conditions*, *decline*, *defer*. No freeform verbs.
9. **Amendments to conditions are primed.** When the council amends a condition, the log uses `C2 → C2'` notation; the original condition text is preserved.
10. **Dissents are role-level and typed.** §5 distinguishes reservation vs. formal objection; every dissent is by role.
11. **Rationale references evidence.** §3 names at least one evidence item from the pre-read that was load-bearing to the decision. A rationale that reads "the council agreed" with no citation is a drafting defect.
12. **Attestations at role level.** §7 lists roles and dates; personal names are in the internal attestation record, not in the decision log entry.

## 4. Rubric or review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Pre-read on SLA | `published_at` ≥ 5 business days before `council_meeting_date` | Pre-read published late without an explicit procedural note |
| R2 | Four-question structure | §A–§D exist and each answers its question | Pre-read organised around the team's work instead of the council's job |
| R3 | Ask is decidable | §A states ask as boolean / named-option pick | Ask is open-ended discussion item |
| R4 | Harm delta, not re-write | §B is a delta citing specific rows | §B reproduces the harm model |
| R5 | Evidence families complete | ≥ 4 families (eval, red-team, monitoring, incident history) each with path + metric | Missing family or missing path |
| R6 | Residual risk non-zero | §D names a specific residual shape and triggers for return | "No further risk" |
| R7 | Conditions discharge-able | Every condition has text + owner + exit criterion + date | Vague conditions |
| R8 | Decision verb restricted | Log's §2 uses one of the four approved verbs | Freeform verb |
| R9 | Amendments primed | Amendments use `C2 → C2'` notation; original text preserved | Original condition overwritten |
| R10 | Dissents typed and role-level | Reservation vs. formal objection distinguished; dissents by role | Personal names in dissents, or type conflated |
| R11 | Rationale cites evidence | §3 names ≥ 1 evidence item that was load-bearing | Rationale is unsourced |
| R12 | Attestations at role level | §7 lists roles and dates | Personal names in the log |
| R13 | No raw adversarial payload | Attack described by category | Raw prompt text in pre-read or log |
| R14 | Jurisdictional citations verified | Any specific legal citation is verified or marked `<!-- verify-citation: ... -->` | Invented article numbers |

Scoring bands: *Pass* — R1–R13 green and R14 honoured; *Pass with revisions* — R1 or R9 fails but R3–R8 and R11 pass; *Fail* — any of R2 / R3 / R6 / R7 / R8 / R12 fails.

## 5. Common mistakes

- **Organising the pre-read around the team's work, not the council's job.** The pre-read becomes a status report instead of a decision document. Re-organise under the four questions in §A–§D.
- **Burying the ask.** The council spends 20 minutes figuring out what is being asked because §A is a narrative instead of a boolean. State the ask in the first sentence of §A.
- **Reproducing the harm model.** The pre-read is 40 pages because the full harm model is pasted in. Cite the file; show the delta.
- **Only approve shown.** The drill produces a pre-read that cannot result in a decline because the evidence is cherry-picked. A real pre-read supports all three outcomes.
- **Zero-risk residual statement.** "We believe the risk is now fully mitigated" reads to the council as either naive or dishonest. Name the residual shape; name the trigger for return.
- **Vague conditions.** "Improve monitoring before go-live" is not a condition; "Drift monitor live and green on canary cohort for 7 consecutive days by 2026-12-13, owned by AI infra / MLOps engineer" is a condition.
- **Freeform decision verbs.** "Approved pending review", "approved in principle", "approved with iteration" are all drafting defects. Use the four approved verbs.
- **Silent amendments.** The council tightens C2 from 5% to 3%, but the log records only the final condition and loses the delta. Prime the amendment (C2 → C2') and preserve the original text.
- **Reconstructed rationale.** The scribe writes the rationale after the meeting, and it drifts from what was said. Capture in-session; finalise within 24 hours; do not rewrite history.
- **Personal names in the log.** The log reads "Alice chaired; Bob dissented". Use role titles; keep personal names in the internal attestation record.
- **Raw payloads in the pre-read.** The author includes the actual crafted prompt to make the near-miss "concrete". The pre-read is read by the clinical-safety officer, the communications head, and counsel; none of them should be reading attack guidance. Describe the attack by category.
- **No follow-up review on novel modality.** The team approves and never comes back. A novel modality needs a 60–90 day residual-risk follow-up; the council makes that follow-up a condition on the first approval.

## 6. References

Only the official sources listed in the work packet (and clearly-labelled local module references) are used.

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — the pre-read and decision log operationalise the **GOVERN** function's expectations around AI decision accountability, documentation, and escalation. The four-question pre-read structure maps directly to what the GOVERN function asks of accountable roles. <!-- verify-citation: cite the specific GOVERN sub-category code (e.g. GOVERN 2.x on roles/responsibilities, GOVERN 3.x on diverse decision-making) that most precisely covers council-level decision documentation before pinning a specific code. -->
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — the pre-read's harm-model delta (H-PI-03, H-PI-04) sits in the OWASP ML Top 10 **prompt-injection / input-manipulation** category family at the category level. <!-- verify-citation: cite the exact OWASP ML Top 10 entry code (and OWASP Top 10 for LLM Applications entry if applicable) for prompt injection via document inputs before pinning a specific code. -->
- **MITRE ATLAS.** `https://atlas.mitre.org/` — the pre-read's harm delta is cross-walked to the ATLAS **prompt-injection** tactic / technique family at the category level. <!-- verify-citation: cite the exact ATLAS technique ID for prompt injection via a document ingestion channel before pinning a specific ID. -->
- **Module running system and running artifacts.** CarePilot as introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`.
- **Preceding module artifacts cited by the pre-read and decision log.**
  - `lessons/mod-102-harm-modelling-and-risk-taxonomy/` — harm model.
  - `lessons/mod-103-risk-quantification-and-registers/` — risk register.
  - `lessons/mod-104-llm-red-teaming-and-attack-engineering/` — red-team suite and attack-success-rate.
  - `lessons/mod-106-fairness-bias-and-explainability-engineering/` — fairness slices.
  - `lessons/mod-107-privacy-risk-engineering/` — privacy evaluation and PII containment.
  - `lessons/mod-109-continuous-monitoring-and-risk-register-integration/` — drift monitor and HITL override.
  - `lessons/mod-111-incident-response-and-postmortems/exercises/exercise-02-blameless-post-mortem-and-five-whys.md` — post-mortem cited.
  - `lessons/mod-111-incident-response-and-postmortems/exercises/exercise-03-ai-specific-incident-triage-and-severity-scoring.md` — synthetic near-miss.
  - `lessons/mod-111-incident-response-and-postmortems/exercises/exercise-04-regression-fixture-authoring-from-incidents.md` — regression fixture.
- **This module's charter deliverable.** `modules/mod-112-cross-functional-ai-risk-program-and-communication/exercise-01-risk-program-charter-for-one-team/SOLUTION.md` — the escalation triggers (T1 / T2 / T3) referenced by the pre-read and the decision log.
- **Exercise prompt.** `lessons/mod-112-cross-functional-ai-risk-program-and-communication/exercises/exercise-02-council-pre-read-and-decision-log-drill.md`.

<!-- Specific legal citations (regulatory classification determinations) are
     deferred to counsel review via `<!-- verify-citation: ... --` markers; the
     drill is engineering authorship discipline, not legal advice. -->

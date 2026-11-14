# Exercise 01 — Risk Programme Charter for One Team — Reference Solution

## 1. Solution overview

This exercise is the module's **standing-up** capstone: for a single engineering team that ships an AI system — the **CarePilot** team (hospital-triage-support RAG assistant from mod-101) — write the **risk programme charter** that says, in one durable document, (a) what risk work this team owns, (b) who holds which role, (c) which rituals the team runs and at what cadence, (d) which decisions the team makes and which it must escalate, (e) which artifacts it produces as evidence, and (f) how the team plugs into the organisation-wide AI risk council. The charter is the thing a new team member reads on day one to understand how risk work happens here.

The reference solution:

- Keeps the scope to **one team** on purpose. The exercise's trap is to drift into writing the organisation-wide AI governance charter, which is owned by the AI governance analyst / head of AI governance, not by the engineering team. A team charter inherits from the organisation charter; it does not replace it. The deliverable in §2.3 states the inheritance explicitly.
- Structures the charter around the four **AI RMF 1.0 functions** — GOVERN, MAP, MEASURE, MANAGE — because the team's work and evidence already align to those functions from mod-101. GOVERN work maps to the roles, rituals, and escalation paths; MAP/MEASURE/MANAGE map to the artifacts the team produces (harm model, risk register, evals, monitors, incident response). Using the RMF spine means the charter is auditable against the framework the organisation has already adopted.
- Names roles **by role title**, never by personal name. A charter that names Alice, Bob, and Carol breaks the moment Alice takes a new role; a charter that names "the AI risk engineer on the CarePilot team" survives attrition. The per-sprint RACI appendix is the escape hatch where a current personal owner can be recorded, but the charter proper stays at the role level.
- Specifies **decision rights** with an explicit **RACI-style table** and names the three gates where the team **must escalate** — new modality, regulated-domain change, and material residual risk — so the question of "when do we call the council?" has a decidable answer rather than culture.
- Ties every ritual and artifact to the **cadence** that keeps it alive. A ritual with no cadence is aspiration; a ritual with "weekly, Tuesday 15:00 UTC" is a commitment. The cadence column in §2.3 is load-bearing.
- Treats the charter as **a versioned engineering artifact**: it lives in-repo, carries a version, has a review cadence of its own, and ships with a changelog. A charter that is a slide deck in a drive is a charter that is out of date.

Design decisions worth calling out:

- **Why one-team scope.** Teams that attempt to re-charter the whole organisation produce documents nobody reads and that counsel will not stand behind. Teams that charter their own scope and name their inheritance from the org charter produce documents that are used daily.
- **Why RMF spine, not an invented structure.** Two risks from inventing a structure: (a) it does not align to the organisation's existing framework adoption and creates a translation burden at every audit, (b) it over-weights whatever framework the author happens to know best. RMF is cited by the exercise's source list; using it directly avoids both failure modes.
- **Why role titles, not names.** See above. The RACI in §2.3 is the single place the charter has to be edited when a person changes roles, and the per-sprint RACI appendix is the escape hatch for the current owner.
- **Why three explicit escalation triggers, not a culture of judgement.** Culture-only escalation ("escalate when it feels big") is a trailing indicator; by the time it feels big, the decision is already made. Three specific triggers — modality, regulated domain, residual risk — can be checked mechanically and are the triggers the council is actually resourced to handle.

## 2. Worked answer or implementation

### 2.1 Repository layout

```
exercise-01-risk-program-charter-for-one-team/
├── SOLUTION.md                           # this file
├── charter/
│   └── CarePilot-team-risk-charter.md    # the charter itself
├── appendices/
│   ├── raci.md                           # per-sprint personal-owner RACI
│   ├── rituals_calendar.md               # concrete cadence commitments
│   └── escalation_runbook.md             # the three escalation triggers, worked
└── links/
    ├── org_charter_inheritance.md        # pointer to the org AI governance charter
    └── running_artifacts.md              # pointers to harm model, register, evals, monitors
```

### 2.2 How to read the charter

The charter in §2.3 is the submission. It is written in prose because a charter that reads like a specification will not be read by non-engineering stakeholders (the clinical-safety officer, the privacy officer, the executive sponsor) who nonetheless have to attest to it. Fields that need to be machine-readable for audit tooling — roles, rituals, decision rights, escalation triggers — are extracted into the RACI table (§2.3 Section 5) and the rituals calendar (§2.3 Section 4). Every claim ties back to an owner, a cadence, and an evidence artifact.

### 2.3 The charter — `charter/CarePilot-team-risk-charter.md`

```markdown
---
document_class: ai-risk-programme-charter
scope: single-team (CarePilot)
version: 1.0
effective_from: 2026-11-13
next_review: 2027-05-13      # 6-month review cadence
inherits_from: docs/governance/ai-risk-council/charter.md
status: draft-for-council-review
---

# CarePilot Team — AI Risk Programme Charter

## 1. Purpose

The CarePilot team designs, operates, and improves an AI system that participates
in a clinical workflow. That means the team owns risk work — not as an add-on, but
as part of the engineering job. This charter is the durable statement of what that
risk work is, who does it, how it gets done, and how the team connects to the
organisation-wide AI risk council. It is the document a new team member reads on
day one.

The charter inherits from the organisation-wide **AI risk council charter**
(`docs/governance/ai-risk-council/charter.md`). Where the two documents overlap,
the organisation charter governs; this charter adds the team-level specifics.

## 2. Scope

**In scope.**
- The CarePilot system: the RAG assistant, its retrieval corpus, its evaluation
  suite, its production monitors, and its incident response for AI-specific
  failure modes.
- Any downstream pipeline the CarePilot team operates (e.g. the discharge-note
  ingestion pipeline).
- Risk work for any model, prompt, or retrieval-corpus change this team ships.

**Out of scope (owned elsewhere).**
- Organisation-wide AI governance policy — owned by the head of AI governance.
- Legal interpretation of regulatory obligations — owned by counsel.
- Hospital-wide clinical safety policy — owned by the clinical-safety officer and
  the hospital's quality and safety committee.
- Platform-level security of the inference provider — owned by the AI infra
  security team.
- Dataset acquisition and licensing — owned by data governance.

The team **contributes evidence** to these domains and **consumes decisions** from
them, but does not own the decision.

## 3. Roles

Roles are named by title; current personal owners are recorded in the per-sprint
RACI in `appendices/raci.md` so that this charter survives role transitions
without edits.

- **AI risk engineer (L25, charter owner).** Primary owner of this charter and
  of the team's risk artifacts (harm model, risk register, red-team suite,
  regression fixtures, production monitors, incident playbook). Runs the
  team-level risk rituals. Represents the team at the AI risk council.
- **Tech lead / engineering manager.** Accountable for the team's release
  cadence. Co-owner of the release-gate decision with the AI risk engineer.
- **AI evaluation engineer (L35, peer).** Owns evaluation methodology:
  groundedness, retrieval-alignment, red-team suite construction, fairness
  slicing. The team consumes evaluation metrics; it does not author
  methodology.
- **AI infra / MLOps engineer (L25, peer).** Owns the deployment pipeline,
  canary and shadow infrastructure, monitor plumbing. The team contributes
  monitor specifications; infra owns the implementation.
- **Clinical-safety officer (reviewer).** Attests the team's harm model and
  release-gate findings from the clinical-safety perspective. Chairs the
  team-level clinical review.
- **Privacy officer (reviewer).** Attests the team's privacy risk findings and
  owns the privacy-impact assessment overlay.
- **Legal counsel (reviewer).** Owns the regulatory obligation determination
  and signs off on any external disclosure.
- **AI governance analyst (L15, upstream).** Owns the organisation-wide AI
  governance register and the council's standing documents; consumes the
  team's evidence.
- **AI risk council chair (upstream).** Chairs the council to which this team
  escalates.

## 4. Rituals and cadence

The rituals below are the minimum commitments. Cadences are stated in UTC;
local-time anchors are recorded in `appendices/rituals_calendar.md`.

| Ritual | Cadence | Owner | Participants | Output |
|---|---|---|---|---|
| Risk stand-up | Weekly (Tuesday 15:00 UTC) | AI risk engineer | Team engineers, AI eval engineer | Risk-stand-up notes appended to the team log |
| Harm-model review | Each major release, and on any change to users / patients / decision stakes | AI risk engineer | Clinical-safety officer, privacy officer, AI governance analyst | Updated harm model (mod-102 artifact) with dated reviewer signoff |
| Risk-register review | Quarterly | AI risk engineer | Tech lead, AI eval engineer, AI governance analyst | Quarterly register-review minute |
| Red-team cadence | Each release and each retrieval-corpus change | AI risk engineer | AI eval engineer | Signed red-team report (mod-104 artifact) |
| Release gate | Per release candidate | AI risk engineer + tech lead (joint) | AI eval engineer, AI infra / MLOps engineer, clinical-safety officer (for clinical-impact releases) | Release-gate decision record, stored in the release log |
| Monitor-threshold review | Monthly | AI risk engineer | AI infra / MLOps engineer, AI eval engineer | Threshold-change record (mod-109) |
| Incident retro | Per incident declared at severity ≥ High, and quarterly round-up | AI risk engineer | All responders, clinical-safety officer (if clinical impact) | Post-mortem (mod-111 ex-02) and regression-fixture addition (mod-111 ex-04) |
| Council pre-read authorship | Per council meeting the team presents at | AI risk engineer | Tech lead (co-author for release-gate asks) | Council pre-read packet (mod-112 ex-02) |
| Charter review | Every 6 months, or on material change to scope or roles | AI risk engineer | Tech lead, AI governance analyst | New charter version with changelog entry |

A ritual that is skipped twice in a row is an **exception** and is logged as such
in the team's quarterly review. Repeated exceptions escalate to the AI risk
council.

## 5. Decision rights (RACI)

The team makes some decisions; the team contributes to some; the team does not
make others. The table is the decidable answer.

| Decision | AI risk engineer | Tech lead / EM | AI eval engineer | AI infra / MLOps | Clinical-safety officer | Privacy officer | Legal counsel | AI risk council |
|---|---|---|---|---|---|---|---|---|
| Harm-model row — add or re-level | **R / A** | C | C | I | C | C | I | I (council sees quarterly) |
| Risk-register row — add or close | **R / A** | C | C | I | C | C | I | I (council sees quarterly) |
| Release gate — go / no-go | **R** | **A** | C | C | C (clinical-impact releases) | C (privacy-impact releases) | I | I (council sees on escalation trigger) |
| Monitor-threshold change | **R / A** | I | C | C | I | I | I | I |
| Red-team methodology change | C | I | **R / A** | I | I | I | I | I |
| Deployment-pipeline change | C | I | I | **R / A** | I | I | I | I |
| Regulatory-obligation determination | C | I | I | I | I | C | **R / A** | I |
| External disclosure (regulator, press, customer) | C | C | I | I | C | C | **R / A** | **I / Attest** (council attests) |
| Modality expansion (e.g. image, voice, agentic) | C | C | C | C | C | C | C | **R / A** |
| Regulated-domain expansion (new jurisdiction, new care setting) | C | C | C | C | C | C | C | **R / A** |
| Material residual-risk acceptance | C | C | C | C | C | C | C | **R / A** |

Legend. **R** — Responsible (does the work). **A** — Accountable (one role per
row; the role that signs). **C** — Consulted (must be asked before the
decision). **I** — Informed (notified after the decision). Rows with no **A**
assigned are drafting errors.

## 6. Escalation triggers to the AI risk council

The team **must** escalate — i.e. the release gate blocks until the council has
cleared the item — when any of the three triggers below fires. Triggers are
mechanical; the team does not debate whether to escalate.

**T1 — Modality expansion.** The system is extending beyond text-only RAG into a
new modality (image, voice, agentic tool-use, generative file output). Reason:
a new modality carries a new harm surface that the existing harm model and eval
suite do not cover. Minimum council notice: one meeting cycle before the change
lands.

**T2 — Regulated-domain change.** The system is being deployed in a new
jurisdiction, a new care setting (e.g. from ED triage into ICU), a new patient
population, or a new user class (e.g. from licensed clinicians to triage
nurses). Reason: the regulatory classification and the obligation set change;
counsel ownership is necessary. Minimum council notice: two meeting cycles
before the change lands.

**T3 — Material residual-risk acceptance.** The team is proposing to ship with a
known risk at severity ≥ High that remains unmitigated beyond the release — i.e.
the plan is to accept, not mitigate or avoid. Reason: accepting material risk is
an organisational commitment, not a team commitment. Minimum council notice:
the council must attest before the release gate opens; no release-time
acceptance.

A trigger that fires outside these three — e.g. a novel failure mode with no
established category — is a **soft escalation**: the team flags it to the
council chair, but the release gate does not block automatically. The chair
decides whether to treat it as T3-equivalent.

## 7. Evidence artifacts the team produces

The charter is only operational if the artifacts it names actually exist and are
maintained. The team owns:

- **Harm model** (mod-102) — `docs/systems/carepilot/harm-model.md`.
- **Risk register** (mod-103) — `risk-register/2025/CarePilot.yaml`.
- **Red-team suite** and **regression fixtures** (mod-104, mod-111 ex-04) — pinned
  at a commit hash on every release.
- **Fairness evaluation** (mod-106) — slice report per release.
- **Privacy evaluation** (mod-107) — PII containment report per release.
- **Guardrail evaluation** (mod-108) — refusal and false-refusal rates per
  release.
- **Production monitors** (mod-109) — groundedness, retrieval-alignment,
  attack-success-rate, HITL override rate, drift per monitor.
- **Model / system / dataset card risk slices** (mod-112 ex-03) — authored at
  release, archived per release tag.
- **Incident playbook** (mod-111) — incident roles, severity ladder, triage
  runbook, post-mortem template, regulator-summary template.
- **Council pre-read packets** (mod-112 ex-02) — authored when the team is on
  the council agenda.
- **This charter** — reviewed every 6 months.

## 8. Metrics the team reports upward

The AI risk council sees the team's risk picture through four metric families.
Values and thresholds live in `risk-register/2025/CarePilot.yaml`; the charter
names the families and the reporting cadence.

- **Attack-success-rate on the red-team suite** — reported per release.
- **Groundedness and retrieval-alignment on the shadow eval** — reported per
  release and monthly in production.
- **HITL override rate** — reported monthly; a trend-break is a near-miss signal.
- **Incident count by severity** — reported quarterly, plus per-incident at
  severity ≥ High.

The team does **not** report headline accuracy numbers to the council as a
risk metric; headline accuracy is a product metric and belongs in a different
report.

## 9. Relationship to the organisation-wide AI risk council

- The council owns organisation-wide risk appetite, modality-expansion decisions,
  regulated-domain-expansion decisions, and material-residual-risk acceptance.
- The team owns its artifacts, its release gate, and its day-to-day
  decisions.
- The team is represented on the council by the **AI risk engineer**, with the
  tech lead as alternate.
- The team supplies the council with a quarterly risk-register delta, a per-release
  risk summary when the release is on the agenda, and the council pre-read packet
  when the team is presenting.
- The council supplies the team with written decisions on escalation triggers T1
  and T2 (attested by the council chair and legal counsel) and with written
  acceptance of material residual risk on T3.

## 10. Charter maintenance

- The charter is versioned in-repo with a changelog.
- A material change to scope (§2), roles (§3), decision rights (§5), or
  escalation triggers (§6) requires a **council-level** review before merge.
- A non-material change (cadence time-slot, output-file path) is reviewed by the
  tech lead and the AI governance analyst.
- A 6-month review is on the calendar whether or not there is a pending change.

## 11. Changelog

- **1.0 (2026-11-13).** Initial charter.

## 12. Attestations

Reviewed and attested by role at version effective-date.

- AI risk engineer (charter owner) — attested on `<date>`.
- Tech lead / engineering manager — attested on `<date>`.
- Clinical-safety officer — attested on `<date>`.
- Privacy officer — attested on `<date>`.
- Legal counsel — attested on `<date>`.
- AI governance analyst — attested on `<date>`.
- AI risk council chair — attested on `<date>`.
```

### 2.4 RACI appendix — `appendices/raci.md` (illustrative sketch)

The charter's §5 names the RACI at role level. The appendix pins the current personal owner per role, for the current sprint, so that a new team member can find the human on the other end of a question without editing the charter. The appendix is cheap to re-edit; the charter is not.

```markdown
# CarePilot team — per-sprint RACI (current owners)

_Sprint 2026-W46 (2026-11-10 → 2026-11-23)._

- AI risk engineer: `<name or handle>` (alternate: `<name>`)
- Tech lead / EM: `<name>` (alternate: `<name>`)
- AI evaluation engineer (peer): `<name>`
- AI infra / MLOps engineer (peer): `<name>`
- Clinical-safety officer (reviewer): `<name>`
- Privacy officer (reviewer): `<name>`
- Legal counsel (reviewer): `<name>`
- AI governance analyst (upstream): `<name>`
- AI risk council chair (upstream): `<name>`

Rotation note: the AI risk engineer alternate covers the weekly stand-up when
the primary is away; coverage > 1 week triggers a reassignment, recorded in the
team log.
```

### 2.5 Rituals calendar — `appendices/rituals_calendar.md` (illustrative sketch)

```markdown
# CarePilot team — rituals calendar

- **Risk stand-up.** Tuesday 15:00 UTC, 30 minutes. Standing agenda: open
  incidents, open red-team findings, open monitor alerts, upcoming release
  gates, upcoming council items.
- **Harm-model review.** Scheduled with each major release kickoff; also on any
  change to users, patients, or decision stakes.
- **Risk-register review.** Second Thursday of each quarter, 90 minutes.
  Attendees from §4 of the charter.
- **Monitor-threshold review.** First Wednesday of each month, 60 minutes.
- **Release gate.** Per release candidate; convened on-demand by the AI risk
  engineer and the tech lead.
- **Incident retro.** Within 10 business days of incident closure for severity
  ≥ High. Quarterly round-up on the same date as the risk-register review.
- **Council pre-read authorship.** Pre-read packet due 5 business days before
  the council meeting the team presents at.
- **Charter review.** Every 6 months (next: 2027-05-13).
```

### 2.6 Escalation runbook — `appendices/escalation_runbook.md` (illustrative)

The runbook makes the three triggers in §6 operational. Each trigger has a decision tree, a packet the team supplies to the council, and an SLA.

```markdown
# CarePilot team — escalation runbook

## T1 — Modality expansion

**Checks that fire the trigger (any one is sufficient).**
- The system is being asked to accept a new input modality (image, voice,
  uploaded file type beyond the currently-approved set).
- The system is being asked to produce a new output modality (voice output,
  generative file, structured action handed to an agentic tool).
- The system is gaining tool-use capability (call-out to another API that
  produces side-effects).

**Packet to the council.**
- Modality description.
- Harm-model delta: new rows that the new modality introduces.
- Evaluation coverage gap and the plan to close it.
- Monitoring coverage gap and the plan to close it.
- Red-team coverage gap and the plan to close it.
- Residual-risk statement at go-live and the trigger conditions for a
  follow-up review.

**SLA.** Packet 1 meeting cycle before the change lands. Council decision in
writing, attested by the chair.

## T2 — Regulated-domain change

**Checks that fire the trigger.**
- New jurisdiction (new country, new US state, new EU member state).
- New care setting (e.g. ICU, outpatient follow-up, telehealth).
- New patient population (e.g. paediatric).
- New user class (e.g. from licensed clinician to triage nurse, or to patient).

**Packet to the council.**
- Jurisdictional / setting description.
- Counsel determination: which obligation set changes, which obligation set
  is new. Counsel is the author of this section; the team supplies facts.
- Privacy-impact assessment delta.
- Clinical-safety assessment delta.
- Evaluation-set applicability review.

**SLA.** Packet 2 meeting cycles before the change lands. Council decision in
writing, attested by the chair and by counsel.

## T3 — Material residual-risk acceptance

**Checks that fire the trigger.**
- The team is proposing to ship with a risk at severity ≥ High that is
  unmitigated as of go-live, and the plan is to accept rather than mitigate
  or avoid.
- The team is proposing to extend a time-boxed acceptance of a High / Critical
  risk beyond its original time box.

**Packet to the council.**
- The specific risk row (ID, title, severity, current state).
- The mitigation options considered and why they were rejected or deferred.
- The monitor that would detect realisation of the risk in production, with
  threshold and expected detection latency.
- The incident playbook entry that would trigger on detection.
- The acceptance time box (date-bounded; no indefinite acceptance).
- The follow-up trigger conditions.

**SLA.** Council attestation **before** the release gate opens. No
release-time acceptance.
```

## 3. Validation steps

1. **Scope is one team.** The charter's §2 names a single team and names at least four responsibilities that are **out of scope and owned elsewhere**. A reviewer confirms by reading §2 only.
2. **Role titles, not names.** The charter's §3 and §5 contain no personal names; personal names are confined to the appendix. A reviewer greps the charter proper for the three most common names in the team roster and confirms zero hits.
3. **Every ritual has a cadence.** Every row in the §4 rituals table has a non-empty cadence column. No ritual is "as needed" without a trigger definition.
4. **Every decision has one accountable role.** Every row in the §5 RACI table has exactly one **A**. A reviewer counts.
5. **Three escalation triggers are mechanical.** Each of T1, T2, T3 in §6 has checks that can be answered yes/no by a line engineer without a judgement call. A reviewer re-reads §6 and attempts to produce a counter-example that is ambiguous; if they can, the triggers get sharpened.
6. **Artifacts named in §7 are real paths.** For every artifact listed, the reviewer resolves the path (or the mod-NNN exercise reference) and confirms the team either owns it today or has a dated plan to stand it up.
7. **Inheritance from the org charter is stated.** The frontmatter `inherits_from:` field names a file. A reviewer opens that file (or its placeholder) to confirm the inheritance is not a dangling pointer.
8. **Metrics in §8 are not product-accuracy metrics.** No headline accuracy / top-1 / MMLU-style number. All four metric families map to a specific risk artifact.
9. **Changelog is present.** §11 has at least one entry; future amendments are to add entries, not to edit history.
10. **Attestations are at role level.** §12 lists roles and dates; personal-name attestations are in the internal attestation log, not in this charter.

## 4. Rubric or review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Scope is single-team | §2 names one team and ≥4 out-of-scope responsibilities owned elsewhere | Charter re-charters the whole organisation |
| R2 | Inheritance from org charter is explicit | Frontmatter `inherits_from:` resolves to a file or a placeholder | No inheritance statement; charter stands alone |
| R3 | Roles at role title, not name | §3 and §5 contain zero personal names; personal names only in `appendices/raci.md` | Personal names in the charter body |
| R4 | Every ritual has a cadence | Every row in §4 rituals table has a non-empty cadence | Ritual listed without cadence or trigger |
| R5 | Every decision has one accountable role | Every row in §5 RACI has exactly one **A** | Zero-**A** or multi-**A** rows |
| R6 | Escalation triggers are mechanical | Each of T1, T2, T3 can be answered yes/no without a judgement call | Trigger wording requires culture or discretion |
| R7 | Evidence artifacts resolve | Each path in §7 is either a real file or a dated plan to stand it up | Dangling pointers |
| R8 | Metrics upward are risk metrics | §8 names risk metrics only; no headline accuracy numbers | Product-performance metric masquerading as a risk metric |
| R9 | Framework mapping present | RMF 1.0 GOVERN / MAP / MEASURE / MANAGE spine visible; OWASP ML Top 10 and MITRE ATLAS named at the artifact level (red-team suite) | No framework mapping |
| R10 | Charter is versioned and reviewed | Frontmatter has `version`, `effective_from`, `next_review`; §11 has a changelog | No version; no review cadence on the charter itself |
| R11 | Attestations at role level | §12 lists roles and dates | Attestations by personal name only |
| R12 | Appendices present | `appendices/raci.md`, `appendices/rituals_calendar.md`, `appendices/escalation_runbook.md` all present | Appendices missing; charter is just the one file |

Scoring bands: *Pass* — R1–R11 green and R12 honoured; *Pass with revisions* — R10 or R12 fails but R1–R6 pass; *Fail* — any of R1 / R3 / R5 / R6 / R8 fails.

## 5. Common mistakes

- **Writing the org-wide charter.** The team drifts into writing "the organisation's AI governance programme" because that is the more prestigious document. The deliverable is the **team** charter. Re-scope to §2 and name the out-of-scope responsibilities explicitly.
- **Naming individuals in the body.** "Alice chairs the weekly stand-up" breaks the first time Alice changes teams. Use role titles in the charter; pin personal owners in `appendices/raci.md`.
- **Rituals without cadence.** "Ad-hoc risk reviews" is not a ritual; it is an intention. Give every ritual a cadence or a mechanical trigger.
- **RACI rows without an accountable role.** A row where responsibility is "shared" is a row that nobody owns. Pick one **A**; the **R** can be more than one role, but the **A** is singular.
- **Soft escalation triggers.** "Escalate when the risk is significant" is not a trigger; it is a hope. Use mechanical checks (modality, regulated domain, residual risk severity) that can be answered yes/no.
- **Decision-rights drift.** The team claims decision rights it does not have — e.g. "team decides on external disclosure". External disclosure is counsel's call. Keep §5 honest; the council is on the other end of the escalation for a reason.
- **Reporting product metrics as risk metrics.** Shipping the headline accuracy number to the council as the risk picture misleads everyone. Risk metrics are red-team outcomes, drift, HITL override, incident counts.
- **Charter in a slide deck.** A charter that is a slide is a charter that is not versioned and not reviewed. The charter lives in-repo.
- **Skipping the 6-month review.** A charter that is never reviewed drifts away from how the team actually works. The 6-month review is on the calendar whether or not there is a pending change.
- **Forgetting inheritance.** A team charter that does not point to the organisation-wide AI governance charter creates a structural contradiction the next time counsel asks "under whose authority does the team do this?" The inheritance line in the frontmatter is the answer.

## 6. References

Only the official sources listed in the work packet (and clearly-labelled local module references) are used.

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — the charter structures the team's work and evidence around the **GOVERN / MAP / MEASURE / MANAGE** functions; §4 rituals and §7 evidence artifacts map explicitly. <!-- verify-citation: the sub-category codes that most directly cover "accountability roles" (GOVERN 2.x), "incident reporting" (GOVERN 4.x / 5.x), and "oversight of deployed systems" (MANAGE 2.x / 4.x) should be re-read from AI 100-1 §5 before citing specific codes. -->
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — the red-team suite and regression fixtures the charter requires (§7) are cross-walked to OWASP ML Top 10 entries at the artifact level in mod-104 / mod-111 ex-04. <!-- verify-citation: cite the OWASP ML Top 10 entry codes (and OWASP Top 10 for LLM Applications entries where applicable) covered by the team's red-team suite before pinning specific codes. -->
- **MITRE ATLAS.** `https://atlas.mitre.org/` — the red-team suite the charter requires is cross-walked to ATLAS tactics and techniques at the artifact level in mod-104 / mod-111 ex-04. <!-- verify-citation: cite the ATLAS technique IDs covered by the team's red-team suite before pinning specific IDs. -->
- **Module running system.** CarePilot — hospital-triage-support RAG assistant, introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md` and reused across this module for continuity.
- **Preceding module artifacts cited by this charter.**
  - `lessons/mod-102-harm-modelling-and-risk-taxonomy/` — harm model the team owns.
  - `lessons/mod-103-risk-quantification-and-registers/` — risk register the team owns.
  - `lessons/mod-104-llm-red-teaming-and-attack-engineering/` — red-team suite.
  - `lessons/mod-106-fairness-bias-and-explainability-engineering/` — fairness evaluation.
  - `lessons/mod-107-privacy-risk-engineering/` — privacy evaluation.
  - `lessons/mod-108-guardrail-engineering-and-effectiveness/` — guardrail evaluation.
  - `lessons/mod-109-continuous-monitoring-and-risk-register-integration/` — production monitors and threshold reviews.
  - `lessons/mod-111-incident-response-and-postmortems/` — incident playbook, post-mortem process, regression-fixture back-feed.
- **Exercise prompt.** `lessons/mod-112-cross-functional-ai-risk-program-and-communication/exercises/exercise-01-risk-program-charter-for-one-team.md`.

<!-- The charter is engineering-owned; counsel owns the regulatory
     obligation determination and the final text of any external disclosure.
     Specific framework codes are deferred to verification against the current
     official sources rather than guessed. -->

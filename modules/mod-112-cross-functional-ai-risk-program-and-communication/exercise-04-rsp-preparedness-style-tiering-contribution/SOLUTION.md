# Exercise 04 — RSP / Preparedness-Style Tiering Contribution — Reference Solution

## 1. Solution overview

This exercise is the **capability-tiering contribution** drill: for a team that operates under an organisation-wide **Responsible Scaling Policy (RSP) / Preparedness-style** framework — frameworks that bind an organisation to specific commitments at specific capability thresholds — author the artifact that a single system-operating team contributes to the organisation-wide policy. The deliverable is not the organisation's RSP itself (that is counsel-and-executive-owned); it is the **team-level contribution** — the specific capability-indicator observations, pre-mitigation and post-mitigation evidence, and the **tier determination** proposal the team brings to the Responsible Scaling Officer / Preparedness team.

The running system is CarePilot (the hospital-triage-support RAG from mod-101). CarePilot is a domain-specific decision-support assistant, not a frontier general-purpose model, so the tiering exercise is cast against **domain-and-deployment-specific capability indicators** — the risks that would cross a threshold *for this system's intended use* — rather than frontier capability benchmarks. The reference solution is explicit that the approach reuses the **structure** of RSP-style tiering (capability indicator → threshold → commitment → evidence → attestation) but *not* the content of any specific organisation's RSP. Anything that would be a verbatim lift from a published RSP is replaced with a structural placeholder and flagged with `<!-- verify-citation: ... -->` so that a reviewer can anchor the specific wording against the authoritative source the organisation has adopted.

The reference solution:

- Defines a **three-tier** indicator ladder for CarePilot: **T-A** (nominal operations), **T-B** (elevated-risk observations that trigger tightened monitoring and a council pre-brief), and **T-C** (threshold-crossing observations that trigger a mandatory review and a mitigation pause). The ladder is structural; the organisation-wide RSP overlays its own tier names if different. The ladder is **not a frontier-capability ladder** — it is a deployment-context capability-indicator ladder.
- Anchors the ladder to a **small, mechanically-checkable set of indicators**: attack-success-rate on the red-team suite, groundedness and retrieval-alignment, HITL override rate, drift signal on key monitors, incident frequency at severity ≥ High, and near-miss queue depth. Indicator values are **ranges** backed by the risk register's envelope, not spurious precise numbers.
- Pairs each tier with explicit **commitments**: the controls the team operates at that tier, the governance steps that fire at the tier boundary, and the artifacts the team produces to attest the tier.
- Documents the **tier-determination process** — who proposes a tier change, who approves it, how a disagreement is resolved, and how a tier regression (upgrade to a higher tier, i.e. elevated-risk state) relates to the T1 / T2 / T3 escalation triggers from the team charter in ex-01.
- Explicitly **disclaims** (a) that this is not a frontier-capability RSP and does not replace one, (b) that the specific thresholds are team-proposed and are not the organisation's official risk appetite until the Responsible Scaling Officer / Preparedness team attests them, and (c) that the organisation-wide RSP, where it exists, takes precedence over this team artifact.

Design decisions worth calling out:

- **Why deployment-context tiering rather than frontier-capability tiering.** CarePilot is not a general-purpose frontier model; it is a domain-and-deployment-specific RAG system. The capability questions that matter are not "can the model autonomously replicate" but "what rate of adversarial success does the deployed system exhibit in production, and does that rate still support the intended use". The tier definitions are written for that problem.
- **Why structure first, content second.** RSPs and Preparedness-style frameworks are **organisation-owned**; the specific wording of a frontier-lab RSP is not generically applicable to a hospital-operator team. The deliverable adopts the *structure* — capability indicator, threshold, commitment, evidence, attestation — because the structure is what makes the framework operable.
- **Why three tiers, not five or seven.** More tiers produce more boundaries to cross and more boundary-arbitration overhead. Three tiers (nominal, elevated, threshold-crossing) is the minimum that supports a non-trivial decision ladder and keeps the attestation workload within what a team can actually sustain.
- **Why ties to the charter and the decision log.** A capability-indicator observation that crosses a threshold is a council-level event. The charter's T1 / T2 / T3 triggers and the decision log's "triggers for return" section are what this exercise plugs into; without those ties, the tier ladder is theatre.

## 2. Worked answer or implementation

### 2.1 Repository layout

```
exercise-04-rsp-preparedness-style-tiering-contribution/
├── SOLUTION.md                                              # this file
├── tiering/
│   ├── CarePilot-tier-ladder.md                             # the proposed ladder
│   ├── CarePilot-tier-determination.md                      # the proposed tier for v1.1
│   └── CarePilot-mitigation-pause-runbook.md                # what to do when T-C fires
├── indicators/
│   ├── indicators_catalog.md                                # indicator list, source, cadence
│   └── indicators_values_v1.1.md                            # observed values for v1.1
├── templates/
│   └── tier_determination_template.md                       # reusable template
└── checklists/
    ├── proposal_authoring_checklist.md
    └── tier_boundary_review_checklist.md
```

### 2.2 Indicators catalogue — `indicators/indicators_catalog.md`

The indicators are kept small on purpose. Each one is **measurable**, **monitored in production**, **auditable** from an evidence file, and **tied to a risk-register envelope**.

| ID | Indicator | Source / evidence file | Cadence | Register envelope |
|---|---|---|---|---|
| I-RT | Red-team attack-success-rate on the current pinned suite | `evidence/measure-2.7/<release>/report.json` | Per release and per corpus change | `risk-register/2025/CarePilot.yaml#security-envelope` |
| I-GR | Groundedness on shadow eval set | `evals/carepilot/shadow/<release>/report.json` | Per release and monthly in production | `risk-register/2025/CarePilot.yaml#performance-envelope` |
| I-RA | Retrieval-alignment on shadow eval set | Same as above | Same | Same |
| I-OR | HITL override rate (production) | `monitors/carepilot/hitl-override.md` | Monthly | `risk-register/2025/CarePilot.yaml#performance-envelope` |
| I-DR | Drift signal on the document-injection monitor | `monitors/carepilot/drift-doc-inject.md` | Weekly | `risk-register/2025/CarePilot.yaml#monitor-doc-inject` |
| I-IC | Incident count at severity ≥ High | `incidents/carepilot/` | Quarterly rollup; per-incident for severity ≥ High | `risk-register/2025/CarePilot.yaml#incident-envelope` |
| I-NM | Near-miss queue depth | `incidents/carepilot/candidates/` | Weekly | `risk-register/2025/CarePilot.yaml#near-miss-envelope` |

Three deliberate omissions and the reason:

- **Headline accuracy / top-1 benchmark score.** This is a product metric, not a risk indicator. Including it would train the team to tier the system on vanity metrics.
- **Absolute training-data size or model parameter count.** For a deployment-context RAG system, this is not a capability indicator — the upstream model is a constant we do not control.
- **Specific detection thresholds for the drift monitor.** The catalogue cites the envelope location in the register; it does not reproduce the threshold here, because the thresholds are evasion-relevant and belong behind the register's access control.

### 2.3 Tier ladder — `tiering/CarePilot-tier-ladder.md`

Three tiers. Every tier definition is **mechanical**: an on-call engineer can determine the current tier from the indicator readings without a judgement call.

```markdown
---
document_class: rsp-style-tier-ladder (team contribution)
scope: CarePilot (hospital-triage-support RAG)
inherits_from: docs/governance/responsible-scaling/organisation-rsp.md   # placeholder; see §4
version: 0.1-proposal
effective_from_if_approved: 2026-12-15
status: draft-for-responsible-scaling-officer-review
author_role: AI risk engineer
---

# CarePilot — RSP/Preparedness-Style Tier Ladder (Team Contribution)

## 1. Scope and relationship to the organisation-wide RSP

This document is a **team-level contribution** to the organisation's
Responsible Scaling Policy / Preparedness-style framework. It is not the
organisation's RSP. Where the organisation-wide RSP exists, that document
governs; this document proposes the team's tier ladder and its mapping to
the organisation-wide tier names.

The ladder is a **deployment-context** ladder for a domain-specific RAG
system, not a frontier-capability ladder. Frontier-capability tiering is
out of scope.

## 2. Tiers

### T-A — Nominal

**Definition.** All indicators within their risk-register envelope floor
and no trend-break observed over the last review cadence.

**Entry criteria (all must hold).**
- I-RT (red-team attack-success-rate): within envelope.
- I-GR, I-RA (groundedness, retrieval-alignment): within envelope.
- I-OR (HITL override rate): within envelope and no 7-day trend-break.
- I-DR (drift signal on doc-injection monitor): within envelope.
- I-IC (incident count at severity ≥ High): zero new incidents in the
  rolling 30-day window.
- I-NM (near-miss queue depth): within envelope; no queue-depth growth
  beyond the floor.

**Commitments at this tier.**
- Standard controls per the team charter §7 remain in force.
- Release gates proceed under the standard process (ex-01 §5 RACI).
- Council pre-reads are scheduled on the standard cadence.

**Artifacts attested at this tier.**
- Per-release risk slice (ex-03) attested at role level.
- Monthly indicator rollup (`indicators/indicators_values_<release>.md`)
  attested by the AI risk engineer.

### T-B — Elevated

**Definition.** Any indicator moves out of the envelope floor or an
indicator trend-break is observed, but **no** indicator crosses the
threshold that fires T-C.

**Entry criteria (any one triggers T-B).**
- I-RT exceeds the envelope floor but remains below the T-C threshold
  (ranges in `risk-register/2025/CarePilot.yaml#security-envelope`).
- I-GR, I-RA drops below the envelope floor by less than the T-C delta.
- I-OR trend-break persists more than 48 hours.
- I-DR trend-break persists more than 48 hours.
- I-NM queue depth growth exceeds the envelope growth rate.

**Commitments at this tier.**
- **Mitigations tighten within 5 business days.** The AI risk engineer
  files a tightened-mitigation plan with the tech lead; the plan names
  the specific indicator, the mitigation, and the exit criterion to
  return to T-A.
- **Monitoring tightens** — polling frequency on the affected indicator
  increases per the mod-109 runbook; on-call awareness-paragraph updated.
- **Council pre-brief within one meeting cycle.** The team chair-briefs
  the AI risk council chair before the next council meeting; the briefing
  is a one-page addendum to the standing pre-read.
- **Release-gate posture changes** — releases proceed only if the release
  candidate's indicator readings show no further deterioration vs. the
  current T-B observation.

**Artifacts attested at this tier.**
- Everything from T-A, plus:
- Tightened-mitigation plan (`tiering/CarePilot-tier-determination.md`
  updated to v<n>.1).
- Addendum pre-read to the council (`pre-read/<date>-tier-b-addendum.md`).
- Weekly indicator rollup (cadence tightened from monthly to weekly for
  the affected indicator).

**Exit back to T-A.** All indicators return to envelope and hold for the
exit window defined in `risk-register/2025/CarePilot.yaml#tier-exit-envelope`.

### T-C — Threshold-crossing

**Definition.** One or more indicators cross the T-C threshold recorded in
the register envelope. Crossing is a council-level event and triggers a
**mitigation pause**.

**Entry criteria (any one triggers T-C).**
- I-RT crosses the T-C threshold (range recorded in the register).
- I-GR, I-RA drops by more than the T-C delta.
- I-OR trend-break persists more than the T-C persistence window.
- I-DR crosses the T-C threshold.
- I-IC: any incident at severity ≥ High.
- I-NM: queue depth exceeds the T-C queue-depth threshold.

**Commitments at this tier.**
- **Mitigation pause.** New deployments and new modality or domain
  expansions are **paused** until T-C is cleared. The pause is a
  release-gate-level block; the tech lead cannot override.
- **Council review within one meeting cycle.** The council convenes (or
  the chair calls a focused T-C review meeting) to decide whether to
  (a) sustain T-C with a tightened mitigation plan and a date-bounded
  return to T-B, (b) accept the posture as the new baseline (requires
  explicit council attestation under the ex-01 T3 trigger — material
  residual-risk acceptance), or (c) deprecate or restrict the system.
- **Incident investigation.** If T-C fired on I-IC or I-NM, the incident
  / near-miss investigation per mod-111 is activated.
- **External-notification posture review.** Legal counsel determines
  whether an external notification obligation is triggered.

**Artifacts attested at this tier.**
- Everything from T-A and T-B, plus:
- T-C declaration record (`tiering/CarePilot-tier-determination.md`
  updated to v<n>.0 with the T-C stamp).
- Council review minute with a council-chair attestation and a counsel
  attestation (the two attestations are both required).
- Mitigation-pause runbook invocation record (see §2.5 below).
- Incident record (if applicable) per mod-111.

**Exit back to T-B.** Council attests the mitigation plan is sufficient
and that indicators have returned to the T-B band for the exit window.

**Exit back to T-A.** Normal T-B → T-A exit criteria after a sustained
window at T-B.

## 3. Tier-determination process

1. **Proposal.** The AI risk engineer authors a `tier-determination`
   document per release (or at any tier boundary crossing) proposing the
   tier and the evidence. The template is in
   `templates/tier_determination_template.md`.
2. **Peer review.** The AI evaluation engineer and the AI infra / MLOps
   engineer independently confirm the indicator readings.
3. **Tech-lead attestation.** The tech lead attests the determination at
   release gate.
4. **Responsible Scaling Officer / Preparedness team attestation.** The
   RSO (or the organisation's Preparedness team under whatever name it
   is given) attests the determination. This attestation is what makes
   it the organisation's official tier for CarePilot.
5. **Disagreement.** If the RSO disagrees with the team's proposal, the
   disagreement is escalated to the AI risk council; the council's
   attested position governs.

## 4. Inheritance from the organisation-wide RSP

Where an organisation-wide RSP exists, this ladder's tier names are
mapped to the organisation's tier names in
`docs/governance/responsible-scaling/organisation-rsp.md` (placeholder).
If no organisation-wide RSP exists at the time of this contribution,
this ladder operates as a team-level commitment with the inheritance
left as `<!-- verify-citation: pending organisation-wide RSP document
reference and tier-name mapping -->`.

## 5. Explicit disclaimers

- **This is not a frontier-capability RSP.** A frontier-model RSP
  governs training and deployment thresholds for general-purpose
  models at scale; that is not CarePilot's situation and is not this
  ladder's purpose.
- **Thresholds in this document are the team's proposal.** They are not
  the organisation's official risk appetite until attested by the RSO
  and (for T-C's "accept as new baseline" option) by the council
  under the T3 material-residual-risk-acceptance trigger from the team
  charter.
- **This ladder does not replace legal obligation determination.**
  Counsel owns regulatory classification and external-notification
  obligations.
- **Specific indicator thresholds and detection latencies are held in
  the register, not in this document.** Reproducing them here would
  double as evasion guidance.
```

### 2.4 Tier-determination template — `templates/tier_determination_template.md`

```markdown
---
document_class: rsp-style-tier-determination
scope: CarePilot
release: ""                               # release tag or `operational-<date>`
proposed_tier: ""                         # T-A | T-B | T-C
previous_tier: ""
transition: ""                            # same | T-A→T-B | T-B→T-A | T-A→T-C | T-B→T-C | T-C→T-B | T-C→T-A
indicator_values_pin: ""                  # `indicators/indicators_values_<release>.md` at commit
register_pin: ""                          # `risk-register/2025/CarePilot.yaml` at commit
author_role: AI risk engineer
reviewer_roles:
  - AI evaluation engineer
  - AI infra / MLOps engineer
  - Tech lead / engineering manager
  - Responsible Scaling Officer (or equivalent)
status: draft-for-rso-review
---

# Tier determination — <release> — proposed tier <T-A|T-B|T-C>

## 1. Proposal

One paragraph. State the proposed tier, the transition from the previous
tier (if any), and the earliest meaningful effective-from date.

## 2. Indicator readings

Table. For each indicator in `indicators_catalog.md`, the current
reading, the envelope floor, and the tier that reading falls into.

| ID | Reading | Envelope floor | Reading-tier |
|---|---|---|---|
| I-RT | <range> | <range> | T-? |
| I-GR | <range> | <range> | T-? |
| I-RA | <range> | <range> | T-? |
| I-OR | <range> | <range> | T-? |
| I-DR | <range> | <range> | T-? |
| I-IC | <count> | <count> | T-? |
| I-NM | <depth-range> | <depth-range> | T-? |

## 3. Rationale

Short paragraph. Why this tier is proposed, which indicators drove the
tier, and which indicators' positions are the next most likely to force a
tier change.

## 4. Mitigations and commitments in force at this tier

Bullet list. The controls that are in force at the proposed tier (from
§2.3 of the ladder).

## 5. Evidence

Pointers. Every indicator reading has a pointer to the evidence file at
a pinned version/commit.

## 6. Residual-risk statement

One paragraph. Always non-zero. Names what residual risk remains and the
conditions that would cause a tier change.

## 7. Dissents (if any)

Role-level. Reservation vs. formal objection distinguished.

## 8. Attestations

- AI risk engineer — attested on `<date>`.
- AI evaluation engineer — attested on `<date>`.
- AI infra / MLOps engineer — attested on `<date>`.
- Tech lead / engineering manager — attested on `<date>`.
- Responsible Scaling Officer (or equivalent) — attested on `<date>`.
- Council chair (required for T-C entry and T-C exit) — attested on `<date>`.
- Legal counsel (required for T-C entry) — attested on `<date>`.

## 9. Changelog

- <date>: created.
- <date>: superseded by <subsequent-determination>.
```

### 2.5 Worked tier determination — `tiering/CarePilot-tier-determination.md`

A worked example for CarePilot v1.1 just after the 2026-12-15 go-live.

```markdown
---
document_class: rsp-style-tier-determination
scope: CarePilot
release: v1.1
proposed_tier: T-A
previous_tier: (new release)
transition: initial-after-go-live
indicator_values_pin: "indicators/indicators_values_v1.1.md @ e118…9cc0"
register_pin: "risk-register/2025/CarePilot.yaml @ 2026-11-14"
author_role: AI risk engineer
reviewer_roles:
  - AI evaluation engineer
  - AI infra / MLOps engineer
  - Tech lead / engineering manager
  - Responsible Scaling Officer (or equivalent)
status: draft-for-rso-review
---

# Tier determination — v1.1 — proposed tier T-A

## 1. Proposal

The CarePilot team proposes **T-A — Nominal** for CarePilot v1.1 effective
2026-12-15, the go-live date approved under the mod-112 ex-02 decision
log (`council-2026-11-20-01`). All indicators are within envelope on the
pre-go-live canary and on the final shadow eval; no trend-break is
observed.

## 2. Indicator readings

| ID | Reading | Envelope floor | Reading-tier |
|---|---|---|---|
| I-RT (attack-success-rate, doc-payload probe class) | 1–3% | ≤ 3% (go-live window) | T-A |
| I-GR (groundedness, shadow eval) | 94–95% | ≥ 94% | T-A |
| I-RA (retrieval-alignment, shadow eval) | 96–97% | ≥ 95% | T-A |
| I-OR (HITL override rate, canary cohort) | within band | within band | T-A |
| I-DR (drift signal, doc-injection monitor) | within envelope | within envelope | T-A |
| I-IC (incident count at severity ≥ High) | 0 (new incidents since 2026-11-05 near-miss, which is a pre-release event) | 0 rolling 30-day | T-A |
| I-NM (near-miss queue depth) | within envelope floor | within envelope floor | T-A |

## 3. Rationale

The near-miss on 2026-11-05 (synthetic, caught by canary, no patient-
facing delivery) was the event that drove the original council
escalation under T1. The post-incident mitigations (A1–A5 in the
post-mortem; C1–C5' in the council decision log) have been discharged:
regression fixture `rt-doc-injection-2026-11` is pinned and green, the
structural unification of free-text and document ingestion has landed,
the dedicated drift monitor is live, and the HITL-override monitor on
the canary cohort is clean for ≥ 7 days.

The indicators most likely to force a tier change in the next 30 days
are I-DR (drift on the document-injection monitor, because the document-
ingestion path is new in production) and I-NM (near-miss queue depth, if
clinician-reported anomalies grow). The team flags both as the leading
candidates and will tighten the cadence on those two to weekly reviews
for the 60-day post-go-live window.

## 4. Mitigations and commitments in force at T-A

- Standard controls per team charter §7 (harm model, risk register,
  red-team suite, regression fixtures, monitors, incident playbook).
- Release gates proceed under the standard process.
- Monthly indicator rollup to the AI risk council as part of the
  standing report.

## 5. Evidence

- I-RT: `evidence/measure-2.7/v1.1/report.json` at commit `a4f2…c7d1`.
- I-GR, I-RA: `evals/carepilot/shadow/2026-11-R46/report.json` at commit `c71a…9d2e`.
- I-OR: `monitors/carepilot/hitl-override.md` v1.3; canary cohort
  baseline at `monitors/carepilot/canary/2026-12-13/`.
- I-DR: `monitors/carepilot/drift-doc-inject.md` v1.0; canary cohort at
  `monitors/carepilot/drift-doc-inject/canary/2026-12-13/`.
- I-IC: `incidents/carepilot/` rollup at `incidents/carepilot/rollups/2026-12-14.md`.
- I-NM: `incidents/carepilot/candidates/` depth-report at `incidents/carepilot/candidates/rollup/2026-12-14.md`.

## 6. Residual-risk statement

Residual risk is non-zero and is reduced from pre-mitigation. The leading
residual-risk shape is the class of document-layer prompt injection on
future ingest surfaces; the structural mitigation (unified ingestion
policy layer) addresses the class at the policy layer but not
exhaustively. The 60-day post-go-live council review (C5' in the
decision log) will attest whether this residual posture is sustained.

## 7. Dissents

None recorded at proposal. Privacy officer's reservation (recorded in
decision log §5) remains open and is discharged at the 60-day review,
not here.

## 8. Attestations

- AI risk engineer — attested on 2026-12-14.
- AI evaluation engineer — attested on 2026-12-14.
- AI infra / MLOps engineer — attested on 2026-12-14.
- Tech lead / engineering manager — attested on 2026-12-14.
- Responsible Scaling Officer (or equivalent) — attested on 2026-12-15.

## 9. Changelog

- 2026-12-14: v1.0 — created as T-A initial determination for v1.1 go-live.
```

### 2.6 Mitigation-pause runbook — `tiering/CarePilot-mitigation-pause-runbook.md`

The runbook that fires when T-C is declared. Short on purpose; T-C is a high-stakes state and the runbook should be read-at-the-bedside operable.

```markdown
# CarePilot — Mitigation-Pause Runbook (T-C)

Fires immediately when T-C is declared by the AI risk engineer or by the
on-call responder. T-C lasts until council attestation that the posture
is cleared.

## Step 1 — Block new changes

- Release gates for CarePilot are **blocked**. The tech lead cannot
  override; the override authority is the council chair, in writing.
- New modality or domain expansion asks (ex-01 T1 / T2 triggers) are
  **paused**; any in-flight council pre-read is withdrawn and marked
  "paused pending T-C clearance".

## Step 2 — Notify

- AI risk engineer → AI risk council chair, tech lead, RSO.
- Tech lead → engineering on-call and incident commander (if the T-C
  fire is linked to an incident).
- Council chair → legal counsel, clinical-safety officer, privacy
  officer, head of communications.

## Step 3 — Freeze state

- Current model version, prompt, retrieval corpus, and monitor
  configuration are frozen to the state at T-C declaration time.
- The indicator-values snapshot at T-C declaration is archived to
  `indicators/indicators_values_<release>_tc_<yyyy-mm-dd>.md`.

## Step 4 — Investigate

- If T-C fired on an indicator (I-RT, I-GR, I-RA, I-OR, I-DR, I-NM),
  the AI risk engineer leads a focused investigation.
- If T-C fired on I-IC (an incident at severity ≥ High), the mod-111
  incident response is activated; the T-C runbook runs in parallel.

## Step 5 — Convene council review

- Within one meeting cycle of declaration.
- Pre-read: a one-page T-C-review pre-read (mod-112 ex-02 pre-read
  shape, scoped to the T-C fire and the proposed resolution).
- Decision: one of
  - **Sustain T-C with a tightened mitigation plan**, with a date-
    bounded return to T-B.
  - **Accept the posture as the new baseline**, under the team
    charter's T3 material-residual-risk-acceptance trigger, requiring
    explicit council attestation.
  - **Deprecate or restrict the system** — scope reduction or
    withdrawal.

## Step 6 — External-notification check

- Legal counsel determines whether an external notification obligation
  is triggered. If yes, the mod-111 ex-05 regulator-summary workflow is
  activated.
```

### 2.7 Proposal authoring checklist — `checklists/proposal_authoring_checklist.md`

- Proposal cites exactly one tier (T-A, T-B, T-C) and the transition from the previous tier.
- Every indicator in the catalogue is listed in §2 with a reading, a floor, and a reading-tier.
- Readings are expressed as ranges where the underlying data supports a range.
- Every reading has a pinned evidence path.
- Mitigations in force at the proposed tier match the ladder's §2 commitments.
- Residual-risk paragraph is present and non-zero.
- Dissents are captured at role level.
- Attestations are at role level, not personal name.
- The tier-determination is signed by the Responsible Scaling Officer for it to be the organisation's tier.

### 2.8 Tier-boundary review checklist — `checklists/tier_boundary_review_checklist.md`

Fires on any transition (T-A→T-B, T-A→T-C, T-B→T-C, T-C→T-B, T-B→T-A).

- The indicator that crossed the boundary is named explicitly.
- The evidence for the crossing is pinned and attested by a role other than the author (peer review, per the ladder §3).
- The next-tier commitments are activated (monitoring cadence, pre-brief, mitigation pause as applicable).
- The council chair is notified at T-B; the council is convened at T-C.
- The tier-determination document is updated — not overwritten (prime the version, not the content).
- Counsel determines any external-notification applicability on T-C.
- A planned exit date / exit criterion is recorded for every entry to T-B or T-C.

## 3. Validation steps

1. **Ladder has exactly three tiers.** The ladder document contains sections for T-A, T-B, and T-C. More tiers would require a re-justification.
2. **Each tier definition is mechanical.** A reviewer picks two sample indicator readings per tier and determines the tier without a judgement call.
3. **Every indicator is tied to a register envelope.** The catalogue's "Register envelope" column points to a valid risk-register path for every indicator.
4. **No vanity metrics.** The catalogue excludes headline accuracy, benchmark scores, and model-size metrics; the §2.2 omissions are explicit.
5. **Tier transition triggers the next-tier commitments.** For T-B and T-C, the ladder names specific actions that fire on entry; the §2.8 checklist verifies the actions fired.
6. **Thresholds are ranges, not spurious precise numbers.** Indicator readings in the worked determination are ranges.
7. **Thresholds are not reproduced.** The ladder cites the register envelope; it does not reproduce specific thresholds that would double as evasion guidance.
8. **T-C triggers a mitigation pause.** The ladder names the pause and names the authority to override (council chair, in writing).
9. **Tie to charter triggers.** T-C "accept the posture as new baseline" is explicitly tied to the ex-01 T3 material-residual-risk-acceptance trigger.
10. **Disclaimers present.** The ladder states (a) this is not a frontier RSP, (b) thresholds are team-proposed, (c) counsel owns external notification.
11. **Attestations at role level.** Every attestation (ladder, determination, runbook) is at role level; personal names are in the internal attestation record.
12. **Inheritance declared.** The ladder's `inherits_from:` field names the organisation-wide RSP document (or marks the absence with `<!-- verify-citation: ... -->`).

## 4. Rubric or review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Three-tier ladder present | Exactly three tiers (T-A, T-B, T-C) with mechanical entry criteria | Different tier count or non-mechanical definitions |
| R2 | Indicators tied to register | Every catalogue indicator has an envelope path | Indicator without a register anchor |
| R3 | No vanity metrics | Headline accuracy, benchmark scores, model size are not indicators | Vanity metric present |
| R4 | Tier commitments explicit | Each tier lists its controls, cadence, and artifacts | Tier defined without commitments |
| R5 | Ranges over false precision | Indicator readings are ranges where the data supports ranges | Spurious precise numbers |
| R6 | Thresholds held in the register | Ladder cites envelope locations; does not reproduce evasion-relevant thresholds | Specific thresholds reproduced in the ladder body |
| R7 | T-C pause and override explicit | T-C triggers a mitigation pause and names the sole override authority (council chair, in writing) | T-C has no pause or has ambiguous override |
| R8 | Tie to charter triggers | T-C's "accept as baseline" option is tied to the ex-01 T3 trigger | Tie missing |
| R9 | Disclaimers present | Three disclaimers (not a frontier RSP; thresholds team-proposed; counsel owns notification) | Disclaimers missing |
| R10 | Attestations at role level | Role-level attestations throughout | Personal names in attestations |
| R11 | Inheritance declared | `inherits_from:` resolves or is marked with `<!-- verify-citation: ... -->` | Dangling inheritance claim |
| R12 | Boundary-crossing audit runs | The §2.8 checklist can be run on a worked transition without ambiguity | Checklist cannot be applied deterministically |

Scoring bands: *Pass* — R1–R11 green and R12 honoured; *Pass with revisions* — R5 fails but R1–R4 and R6–R10 pass; *Fail* — any of R1 / R2 / R3 / R6 / R7 / R9 fails.

## 5. Common mistakes

- **Confusing frontier-capability tiering with deployment-context tiering.** The team adopts a frontier-lab RSP's language verbatim (autonomous replication thresholds, persuasion benchmarks) and applies it to a hospital-triage RAG system. The result is incoherent. Use the **structure**, not the content.
- **More tiers than the team can attest.** Five or seven tiers produce boundary-arbitration overhead and attestation fatigue. Three tiers is the minimum useful ladder; stay there until there is clear evidence another tier is required.
- **Vanity-metric indicators.** Tiering the system on headline accuracy or benchmark scores teaches the team that risk is a product metric. Risk indicators are red-team outcomes, drift, override rate, incident frequency.
- **Reproducing exact thresholds.** Putting the exact drift-monitor threshold in the ladder document is evasion guidance for an adversary. Cite the register envelope; keep the threshold behind the register's access control.
- **Non-mechanical tier definitions.** "T-B is when things feel elevated" is not a definition. A definition is "T-B is when any indicator exceeds the envelope floor but no indicator crosses the T-C threshold".
- **T-C with no pause.** Declaring T-C and continuing releases defeats the ladder. T-C must have a release-gate-level block and a single override authority in writing.
- **Over-reliance on judgement at the boundary.** The boundary-crossing review becomes a debate because the ladder was written with soft criteria. Sharpen the ladder.
- **Ladder with no tie to the charter.** A tier change that is not an escalation trigger is theatre. T-C's acceptance path must tie to the ex-01 T3 trigger, and T-B's elevated posture must tie to the council pre-brief cadence.
- **Omitting the disclaimers.** The ladder reads as the organisation's position when it is a team proposal. Keep the three disclaimers; they are what the RSO reads first.
- **Personal names in attestations.** Role-level attestations survive attrition; personal names do not. Attestation log holds personal names behind the role.
- **Treating tier changes as paperwork.** T-B and T-C change the team's operating posture. If the posture does not actually change at the boundary, the ladder is not operational.

## 6. References

Only the official sources listed in the work packet (and clearly-labelled local module references) are used.

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — the tier ladder operationalises RMF **MEASURE** (indicator measurement and envelope) and **MANAGE** (response to risks the MAP function identified as high priority). The ladder's commitments at each tier map to MANAGE sub-category responses. <!-- verify-citation: cite the specific MEASURE and MANAGE sub-categories most precisely covering ongoing indicator tracking and the trigger-and-respond pattern used at T-B / T-C before pinning specific codes. -->
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — the indicator catalogue's I-RT (red-team attack-success-rate) is cross-walked to the OWASP ML Top 10 entries covered by the red-team suite at the artifact level. <!-- verify-citation: cite the OWASP ML Top 10 entry codes the red-team suite exercises before pinning specific codes. -->
- **MITRE ATLAS.** `https://atlas.mitre.org/` — I-RT and I-DR are cross-walked to the ATLAS tactics / techniques exercised by the red-team suite and the drift monitor. <!-- verify-citation: cite the ATLAS technique IDs covered by the suite and the monitor before pinning specific IDs. -->
- **Responsible Scaling Policy / Preparedness-style framework (organisation-wide).** The exercise's RSP / Preparedness structure is adopted from the frame the organisation has in place. Specific wording of any published frontier-lab RSP is not reproduced here; the ladder's `inherits_from:` field points to the organisation-wide RSP, with `<!-- verify-citation: pending organisation-wide RSP document reference and tier-name mapping -->` where the organisation has not yet published one.
- **Module running system.** CarePilot, as introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`.
- **Preceding and sibling module artifacts.**
  - `lessons/mod-103-risk-quantification-and-registers/` — risk register envelope structure.
  - `lessons/mod-104-llm-red-teaming-and-attack-engineering/` — red-team suite producing I-RT.
  - `lessons/mod-109-continuous-monitoring-and-risk-register-integration/` — drift monitor and threshold-review runbook that I-DR and I-OR rely on.
  - `lessons/mod-111-incident-response-and-postmortems/` — incident response that T-C invokes.
  - `modules/mod-112-cross-functional-ai-risk-program-and-communication/exercise-01-risk-program-charter-for-one-team/SOLUTION.md` — charter's T1 / T2 / T3 triggers.
  - `modules/mod-112-cross-functional-ai-risk-program-and-communication/exercise-02-council-pre-read-and-decision-log-drill/SOLUTION.md` — pre-read and decision-log shapes used by the ladder.
- **Exercise prompt.** `lessons/mod-112-cross-functional-ai-risk-program-and-communication/exercises/exercise-04-rsp-preparedness-style-tiering-contribution.md`.

<!-- Specific thresholds are held in the risk register's envelope sections,
     not reproduced in this document; this is a deliberate protection
     against the ladder doubling as evasion guidance. Specific frontier-lab
     RSP wording is not reproduced; the ladder adopts structure only. -->

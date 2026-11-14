# Exercise 05 — RMF and ISO/IEC 42001 Evidence Crosswalk — Reference Solution

## 1. Solution overview

This exercise is the **evidence-crosswalk** drill: for the CarePilot team (the hospital-triage-support RAG from mod-101), produce a crosswalk that shows, for each evidence artifact the team maintains, (a) which **NIST AI RMF 1.0** sub-category the artifact addresses and (b) which **ISO/IEC 42001** AI-management-system clause the artifact helps satisfy. The deliverable is a durable document an auditor can read bottom-up (from an artifact to the framework positions it supports) and top-down (from a framework sub-category or clause to the artifacts that evidence it), with explicit owners, pins, and gaps.

The reference solution:

- Anchors the crosswalk in the **artifact set the team already produces** — the artifacts named in the ex-01 charter §7 (harm model, risk register, red-team suite, regression fixtures, fairness, privacy, guardrail, monitors, cards, incident playbook, pre-read packets, decision logs, tier determinations, charter itself). This keeps the crosswalk anchored to what exists, not what we wish existed.
- Maps each artifact to **both** frameworks in a single table so that the crosswalk is **bidirectional**. The same table can be read as "artifact → frameworks" (top-down from an auditor's finding) or re-sorted to "framework → artifact" (top-down from a regulatory ask). The reference includes the re-sorted view as §2.4 and §2.5.
- Uses the **NIST AI RMF 1.0 sub-category labels** from the publicly-released AI 100-1 §5 enumeration. Specific sub-category codes (GOVERN 1.1, MAP 1.1, MEASURE 2.3, etc.) are cited at the sub-category level; where the student is unsure which precise sub-category applies, the row carries `<!-- verify-citation: ... -->` and names the function-family it belongs to. This is the same discipline applied in mod-101's ex-01 reference solution; the crosswalk inherits it.
- Uses the **ISO/IEC 42001 clause structure** as the standard's publicly-available high-level outline: 4 (context of the organization), 5 (leadership), 6 (planning, including the AI-system impact assessment), 7 (support), 8 (operation), 9 (performance evaluation), 10 (improvement), plus the standard's Annex A operational controls. The crosswalk cites these at the clause-family level; where the student is unsure which sub-clause (e.g. 6.1.4 vs. 6.1.5) a specific artifact maps to, the row carries `<!-- verify-citation: ... -->` and names the parent clause. The exercise prompt names ISO/IEC 42001 as an "official standard" but the clause numbering is cited structurally rather than quoted verbatim.
- Explicitly flags the **gaps** — the sub-categories and clauses for which the team does **not** currently have evidence. A crosswalk that is all green reads as either comprehensive or unfalsifiable; a crosswalk that names its gaps reads as credible.
- Keeps the crosswalk itself **versioned and audited**: the document has a frontmatter version, an owner role, a review cadence, and a changelog, so the crosswalk is a maintained artifact rather than a one-shot deliverable.

Design decisions worth calling out:

- **Why artifact-first, not framework-first.** Framework-first crosswalks degrade into "we have an artifact somewhere that probably addresses this sub-category". Artifact-first crosswalks start from an evidence file the team actually maintains and attach the framework positions to it. The result is auditable and the mapping is verifiable.
- **Why both frameworks in one table.** The organisation runs audits against both RMF (adoption posture) and 42001 (AI management system). Maintaining two parallel crosswalks produces drift between them; one table is the control.
- **Why sub-category-code and clause-level citation with `<!-- verify-citation: ... -->` markers.** The source policy is to not invent facts. Sub-category codes and sub-clause numbers age with publication revisions. Marking the specific code where it is not verified lets the crosswalk ship without guessing.
- **Why gaps are first-class.** An auditor who can see what the team does not have attested is far better-informed than one who sees only what the team has. Gaps feed the quarterly planning review.

## 2. Worked answer or implementation

### 2.1 Repository layout

```
exercise-05-rmf-and-42001-evidence-crosswalk/
├── SOLUTION.md                             # this file
├── crosswalk/
│   ├── CarePilot-evidence-crosswalk.md     # the main artifact-first table
│   ├── CarePilot-rmf-reverse-index.md      # auto-derived "RMF sub-category → artifacts"
│   └── CarePilot-42001-reverse-index.md    # auto-derived "42001 clause → artifacts"
├── gaps/
│   └── CarePilot-crosswalk-gap-log.md      # the first-class gap log
├── templates/
│   └── crosswalk_row_template.md           # reusable row schema
└── checklists/
    ├── crosswalk_review_checklist.md
    └── gap_triage_checklist.md
```

### 2.2 Crosswalk row schema — `templates/crosswalk_row_template.md`

Each row is a single artifact. The schema is deliberately shallow — the depth is in the evidence files behind each row, not in this document.

```markdown
- **Artifact.** `<path-in-repo>` (owner role, cadence).
  **What it is.** One sentence.
  **Pin.** `<version or commit>` (if versioned).
  **RMF sub-category/categories.** `<FN>.<N.M>` — short name of the sub-category. (Add `<!-- verify-citation: confirm precise sub-category code from AI 100-1 §5 -->` if unverified.)
  **ISO/IEC 42001 clause(s).** `<clause number>` — short name of the clause. (Add `<!-- verify-citation: confirm precise sub-clause from the current ISO/IEC 42001 text -->` if unverified.)
  **Why this mapping.** One sentence connecting the artifact to the framework position(s).
  **Evidence quality.** `attested | draft | gap` + one-sentence rationale.
```

Three deliberate conventions:

- **One artifact can map to multiple framework positions.** Comma-separate within the row.
- **A framework position can be covered by multiple artifacts.** The reverse indices in §2.4 and §2.5 show this.
- **An artifact that only addresses a framework position at the generic "we comply" level is not evidence.** Evidence must be specific to the artifact; if not, the row is downgraded to `draft` and the gap is logged.

### 2.3 Main crosswalk — `crosswalk/CarePilot-evidence-crosswalk.md`

```markdown
---
document_class: evidence-crosswalk
scope: CarePilot (hospital-triage-support RAG)
frameworks: [NIST AI RMF 1.0 (AI 100-1), ISO/IEC 42001]
version: 1.0
effective_from: 2026-11-13
next_review: 2027-02-13     # quarterly review
owner_role: AI risk engineer
reviewer_roles:
  - AI governance analyst
  - Tech lead / engineering manager
  - Clinical-safety officer (clinical-safety rows)
  - Privacy officer (privacy rows)
  - Legal counsel (regulatory rows only)
status: draft-for-governance-review
---

# CarePilot — RMF & ISO/IEC 42001 Evidence Crosswalk

Artifact-first. Every row is one artifact the team maintains, with the
framework positions it supports and the quality of the evidence today.

Framework code / clause specificity is marked `<!-- verify-citation: ... -->`
where the exact code has not been read back from the current official
source; the family and the sub-category / clause short-name are given
so that a reviewer can verify and pin.

## Governance and programme artifacts

- **Artifact.** `modules/mod-112-cross-functional-ai-risk-program-and-communication/exercise-01-risk-program-charter-for-one-team/SOLUTION.md` (team charter) — owner: AI risk engineer; cadence: 6-month review.
  **What it is.** The CarePilot team's AI risk programme charter — roles, rituals, decision rights, escalation triggers.
  **Pin.** v1.0.
  **RMF.** `GOVERN.2.1` (roles and responsibilities are documented), `GOVERN.4.1` (practices to foster safety-first mindset), `GOVERN.5.1` (processes for AI risks related to third-party inputs). <!-- verify-citation: confirm GOVERN sub-category codes 2.x / 4.x / 5.x best matches against AI 100-1 §5. -->
  **42001.** Clause 5 (leadership), Clause 6 (planning), Clause 7 (support — competence and roles), Annex A controls on AI roles and responsibilities. <!-- verify-citation: confirm specific sub-clause numbers and Annex A control IDs from the current ISO/IEC 42001 text. -->
  **Why this mapping.** The charter is the single place roles, rituals, and decision rights are documented; it is the artifact an auditor would expect to find for both RMF GOVERN-function expectations and 42001's leadership and planning clauses.
  **Evidence quality.** `attested`.

- **Artifact.** `docs/governance/ai-risk-council/charter.md` (organisation-wide AI risk council charter, inherited; placeholder) — owner: AI governance analyst; cadence: annual.
  **What it is.** The organisation-wide AI risk council's charter.
  **Pin.** `<org-managed>`.
  **RMF.** `GOVERN.2.1`, `GOVERN.3.1` (decision-making informed by a diverse team). <!-- verify-citation: confirm GOVERN.3.1 wording. -->
  **42001.** Clause 5 (leadership), Clause 9 (performance evaluation — internal audit cadence). <!-- verify-citation: confirm sub-clauses from current ISO/IEC 42001 text. -->
  **Why this mapping.** Inherited from the organisation; the team charter in §1 above cites this document as the governing authority.
  **Evidence quality.** `attested` (by the organisation; the team consumes it).

- **Artifact.** `modules/mod-112-cross-functional-ai-risk-program-and-communication/exercise-02-council-pre-read-and-decision-log-drill/SOLUTION.md` (council pre-read and decision-log templates and worked example) — owner: AI risk engineer; cadence: per council meeting.
  **What it is.** The pre-read and decision-log shapes the team uses when presenting to the council.
  **Pin.** v1.0.
  **RMF.** `GOVERN.4.1`, `GOVERN.1.2` (processes for documenting AI decisions). <!-- verify-citation: confirm GOVERN sub-categories for decision documentation. -->
  **42001.** Clause 5 (leadership), Clause 9 (performance evaluation), Clause 10 (improvement — decision-driven changes). <!-- verify-citation: confirm sub-clauses. -->
  **Why this mapping.** The pre-read + decision-log pair is the governance-decision trail the frameworks expect to be documented and retained.
  **Evidence quality.** `attested`.

## Mapping artifacts — harm, risk, context

- **Artifact.** `docs/systems/carepilot/harm-model.md` — owner: AI risk engineer (with clinical-safety officer review); cadence: per major release and on context change.
  **What it is.** The CarePilot harm model (mod-102).
  **Pin.** commit `b19e…4f30`.
  **RMF.** `MAP.1.1` (context established and understood), `MAP.1.5` (organizational risk tolerances), `MAP.5.1` (likelihood and magnitude of each identified risk). <!-- verify-citation: confirm MAP sub-categories from AI 100-1 §5. -->
  **42001.** Clause 6 (planning — AI-system impact assessment). <!-- verify-citation: confirm the specific sub-clause and corresponding Annex A control(s) for the AI-system impact assessment. -->
  **Why this mapping.** The harm model is the AI-system impact assessment for CarePilot; it is the artifact the auditor will read to understand what harms are in scope and at what severity.
  **Evidence quality.** `attested`.

- **Artifact.** `risk-register/2025/CarePilot.yaml` — owner: AI risk engineer; cadence: quarterly.
  **What it is.** The CarePilot risk register (mod-103) with per-risk likelihood, magnitude, owner, mitigation, monitor, and envelope.
  **Pin.** 2026-11-14 revision.
  **RMF.** `MAP.5.1`, `MANAGE.1.1` (prioritized risks have been mitigated or accepted), `MANAGE.1.3` (responses are developed). <!-- verify-citation: confirm MAP.5.1 and MANAGE.1.x sub-categories. -->
  **42001.** Clause 6 (planning — risk treatment), Clause 9 (performance evaluation — risk monitoring). <!-- verify-citation: confirm sub-clauses. -->
  **Why this mapping.** The register is the authoritative mapping of risks to owners, mitigations, and monitoring; it is the artifact 42001 clause 6 risk-treatment and RMF MANAGE expect to find.
  **Evidence quality.** `attested`.

- **Artifact.** `docs/systems/carepilot/context.md` — owner: AI risk engineer with clinical-safety officer reviewer; cadence: per major release.
  **What it is.** The CarePilot deployment context brief — clinical setting, users, patients, decision stakes, lifecycle stage.
  **Pin.** commit tied to current release.
  **RMF.** `MAP.1.1` (context). <!-- verify-citation: confirm MAP.1.1 wording. -->
  **42001.** Clause 4 (context of the organization). <!-- verify-citation: confirm sub-clause. -->
  **Why this mapping.** Clause 4 of ISO management system standards is where "context" lives; RMF MAP.1.1 is the sibling in the AI RMF.
  **Evidence quality.** `attested`.

## Measurement artifacts

- **Artifact.** `evals/carepilot/shadow/<release>/report.json` — owner: AI evaluation engineer; cadence: per release and monthly in production.
  **What it is.** The shadow-eval report — groundedness and retrieval-alignment on traffic-replay.
  **Pin.** per release.
  **RMF.** `MEASURE.2.3` (performance in deployment conditions), `MEASURE.1.1` (approaches and metrics selected). <!-- verify-citation: confirm MEASURE.2.3 wording. -->
  **42001.** Clause 9 (performance evaluation). <!-- verify-citation: confirm sub-clause. -->
  **Why this mapping.** The shadow-eval is the "demonstrated for conditions similar to deployment" evidence that RMF MEASURE.2.3 expects, and the quantitative performance evidence 42001 clause 9 expects.
  **Evidence quality.** `attested`.

- **Artifact.** Red-team suite `carepilot-rt-suite` + `evidence/measure-2.7/<release>/report.json` — owner: AI risk engineer with AI evaluation engineer peer; cadence: per release and per retrieval-corpus change.
  **What it is.** The LLM red-team suite (mod-104), pinned at a commit, with signed per-release report.
  **Pin.** commit `a4f2…c7d1`.
  **RMF.** `MEASURE.2.7` (security and resilience evaluated). <!-- verify-citation: confirm MEASURE.2.7 wording. -->
  **42001.** Clause 9 (performance evaluation — adversarial testing); Annex A operational controls on AI testing and verification. <!-- verify-citation: confirm Annex A control IDs. -->
  **Why this mapping.** The suite is the primary evidence for the security-and-resilience sub-category and for the operational-control expectations of 42001.
  **Evidence quality.** `attested`.

- **Artifact.** `evidence/measure-2.11/<release>/fairness-report.json` — owner: AI evaluation engineer with clinical-safety co-sign; cadence: per release.
  **What it is.** The subgroup / fairness slice report (mod-106).
  **Pin.** per release.
  **RMF.** `MEASURE.2.11` (fairness and bias evaluated). <!-- verify-citation: confirm MEASURE.2.11 wording. -->
  **42001.** Clause 9 (performance evaluation). <!-- verify-citation: confirm sub-clause and relevant Annex A controls. -->
  **Why this mapping.** Fairness evaluation is the explicit MEASURE.2.11 sub-category evidence; the 42001 clause 9 position is as a specific quality attribute.
  **Evidence quality.** `attested`.

- **Artifact.** `evidence/measure-2.10/<release>/privacy-report.json` — owner: AI evaluation engineer with privacy-officer co-sign; cadence: per release.
  **What it is.** The privacy-evaluation report — PII containment, session-isolation, de-identification (mod-107).
  **Pin.** per release.
  **RMF.** `MEASURE.2.10` (privacy risk assessed), related `MAP.4.1` (legal and regulatory requirements). <!-- verify-citation: confirm MEASURE.2.10 wording and MAP.4.1 wording. -->
  **42001.** Clause 6 (planning — risk treatment for privacy risks), Clause 9 (performance evaluation). <!-- verify-citation: confirm sub-clauses. -->
  **Why this mapping.** Privacy is both a MEASURE position (evidence it has been evaluated) and a MAP.4.1 position (legal / regulatory requirements understood).
  **Evidence quality.** `attested`.

- **Artifact.** `evidence/measure-2.12/<release>/guardrail-report.json` — owner: AI evaluation engineer with AI risk engineer peer; cadence: per release and per guardrail change.
  **What it is.** Guardrail evaluation — refusal under adversarial prompt, false-refusal rate (mod-108).
  **Pin.** per release.
  **RMF.** `MEASURE.2.7` (security and resilience) and `MEASURE.2.3` (performance in deployment conditions). <!-- verify-citation: confirm MEASURE sub-categories. -->
  **42001.** Clause 9 (performance evaluation). <!-- verify-citation: confirm sub-clauses. -->
  **Why this mapping.** Guardrail effectiveness is partly security and partly deployment-performance; both positions are evidenced by the same artifact.
  **Evidence quality.** `attested`.

## Monitoring artifacts

- **Artifact.** `monitors/carepilot/*.md` (groundedness, retrieval-alignment, HITL override, drift on doc-injection, drift on retrieval, guardrail drift) — owner: AI infra / MLOps engineer with AI risk engineer peer; cadence: live (polling frequencies in the monitor specs).
  **What it is.** The production monitoring specifications (mod-109).
  **Pin.** per monitor version.
  **RMF.** `MEASURE.3.1` (approaches and metrics for continued tracking), `MANAGE.4.1` (monitoring and documenting). <!-- verify-citation: confirm MEASURE.3.1 and MANAGE.4.1 wording. -->
  **42001.** Clause 9 (performance evaluation), Clause 8 (operation — operational controls on monitoring). <!-- verify-citation: confirm sub-clauses and Annex A control IDs. -->
  **Why this mapping.** Monitoring is the operational and performance-evaluation evidence; both frameworks name it explicitly as an ongoing-tracking control.
  **Evidence quality.** `attested`.

## Management and response artifacts

- **Artifact.** `incidents/carepilot/` (incident records) and `postmortems/carepilot/` (post-mortems) — owner: AI risk engineer; cadence: per incident and quarterly rollup.
  **What it is.** Incident records and post-mortems (mod-111 ex-02 / ex-03).
  **Pin.** per incident.
  **RMF.** `MANAGE.4.2` (mechanisms for incident response), `GOVERN.4.3` (processes for documenting AI-related errors and incidents). <!-- verify-citation: confirm MANAGE.4.2 and GOVERN.4.3 wording. -->
  **42001.** Clause 10 (improvement — nonconformity and corrective action). <!-- verify-citation: confirm sub-clauses. -->
  **Why this mapping.** Incidents and post-mortems are the nonconformity-and-corrective-action evidence for 42001 clause 10 and the incident-response and documentation evidence for RMF MANAGE.4.2 and GOVERN.4.3.
  **Evidence quality.** `attested`.

- **Artifact.** `pre-release-suite/rt-doc-injection-2026-11.yaml` (and similar regression fixtures) — owner: AI risk engineer with AI evaluation engineer; cadence: added on every near-miss and High-severity incident; green on every candidate build.
  **What it is.** Permanent regression fixtures from incidents (mod-111 ex-04).
  **Pin.** per fixture revision.
  **RMF.** `MANAGE.1.1` (prioritized risks have been mitigated), `MEASURE.2.7` (security and resilience). <!-- verify-citation: confirm MANAGE.1.1 wording. -->
  **42001.** Clause 10 (improvement — corrective action evidenced in pre-release gates). <!-- verify-citation: confirm sub-clauses. -->
  **Why this mapping.** A regression fixture is the specific corrective-action evidence that closes an incident's loop; it is the thing an auditor will look for as "evidence that this failure mode cannot silently recur".
  **Evidence quality.** `attested`.

- **Artifact.** `modules/mod-112-cross-functional-ai-risk-program-and-communication/exercise-03-model-card-risk-slice-authoring/SOLUTION.md` — model, system, and dataset card risk slices — owner: AI risk engineer; cadence: per release and on dataset change.
  **What it is.** The three card risk slices attested at release.
  **Pin.** per release.
  **RMF.** `GOVERN.4.2` (publishing AI documentation), `MAP.5.2` (information documented), `MEASURE.4.1` (results are documented and communicated). <!-- verify-citation: confirm sub-category wording. -->
  **42001.** Clause 7 (support — documented information), Annex A controls on AI documentation. <!-- verify-citation: confirm Annex A control IDs. -->
  **Why this mapping.** The cards are the external-communication documentation covering the model, the deployed product, and the dataset; both frameworks expect this documentation.
  **Evidence quality.** `attested`.

- **Artifact.** `modules/mod-112-cross-functional-ai-risk-program-and-communication/exercise-04-rsp-preparedness-style-tiering-contribution/SOLUTION.md` — tier ladder and tier determinations — owner: AI risk engineer with RSO attestation; cadence: per release and per tier boundary.
  **What it is.** The deployment-context tier ladder and the per-release tier determinations.
  **Pin.** per determination.
  **RMF.** `MANAGE.1.1`, `MANAGE.2.4` (continuous improvement), `GOVERN.1.3` (processes for defining risk tolerance). <!-- verify-citation: confirm sub-category wording. -->
  **42001.** Clause 6 (planning — risk treatment and acceptance criteria), Clause 9 (performance evaluation). <!-- verify-citation: confirm sub-clauses. -->
  **Why this mapping.** The tier ladder is the operational expression of the organisation's risk tolerance; the tier determinations are the evidence that the current posture is consistent with that tolerance.
  **Evidence quality.** `attested` (team); `attested-pending-RSO` (RSO attestation required for the determinations to be the organisation's official tier).

- **Artifact.** `legal-register/CarePilot/2025.yaml` — owner: AI governance analyst with counsel sign-off; cadence: quarterly.
  **What it is.** The regulatory-obligation register.
  **Pin.** 2025 annual.
  **RMF.** `GOVERN.1.1` (legal and regulatory requirements are understood and managed). <!-- verify-citation: confirm GOVERN.1.1 wording. -->
  **42001.** Clause 4 (context — needs and expectations of interested parties), Clause 6 (planning — legal requirements). <!-- verify-citation: confirm sub-clauses. -->
  **Why this mapping.** The regulatory register is where the organisation records what obligations apply to CarePilot; both frameworks require this to be documented.
  **Evidence quality.** `attested` (by counsel; the team consumes it).

## Supplier / supply-chain artifacts

- **Artifact.** `vendor-register/CarePilot/<vendor>.yaml` (inference provider; managed vector-store provider; other third-party components) — owner: AI governance analyst with AI infra / MLOps peer; cadence: annually and on vendor change.
  **What it is.** Per-vendor register — scope of supply, contracted terms, incident-notification clauses, SOC2 / ISO42001 or equivalent artifacts the vendor supplies.
  **Pin.** per vendor revision.
  **RMF.** `GOVERN.5.1` (processes and procedures for AI risks related to third-party inputs), `MAP.4.1` (legal and regulatory requirements). <!-- verify-citation: confirm GOVERN.5.1 wording. -->
  **42001.** Clause 8 (operation — control of externally provided processes), relevant Annex A controls on supply chain. <!-- verify-citation: confirm sub-clauses and Annex A control IDs. -->
  **Why this mapping.** Supply-chain controls are an explicit framework position in both.
  **Evidence quality.** `draft` — the vendor register exists but the per-vendor breach-notification clause survey has not been attested for all vendors yet (see gap log).

## Improvement artifacts

- **Artifact.** `reports/governance/carepilot/<yyyy-mm>.md` — monthly risk-posture summary — owner: AI risk engineer; cadence: monthly.
  **What it is.** Monthly roll-up of indicator readings and open-risk status for the council.
  **Pin.** per month.
  **RMF.** `MANAGE.4.3` (continuous improvement), `MEASURE.4.3` (results are communicated). <!-- verify-citation: confirm sub-category wording. -->
  **42001.** Clause 9 (performance evaluation — management review input), Clause 10 (improvement). <!-- verify-citation: confirm sub-clauses. -->
  **Why this mapping.** The monthly roll-up is the management-review input that drives improvement; both frameworks name this as a position.
  **Evidence quality.** `attested`.

- **Artifact.** `reports/governance/carepilot/annual-management-review/<yyyy>.md` — owner: AI risk engineer with council chair attestation; cadence: annually.
  **What it is.** The annual management review record.
  **Pin.** per year.
  **RMF.** `GOVERN.1.4` (effectiveness of processes is evaluated). <!-- verify-citation: confirm GOVERN.1.4 wording. -->
  **42001.** Clause 9 (performance evaluation — management review). <!-- verify-citation: confirm sub-clauses. -->
  **Why this mapping.** The management review is a core 42001 clause 9 expectation and is the RMF GOVERN effectiveness-evaluation position.
  **Evidence quality.** `draft` — the review template exists; the first annual review is scheduled for 2027-01 (see gap log).

## Documented-information / records artifacts

- **Artifact.** Role-level attestations across the charter, cards, determinations, decision logs, and post-mortems — owner: each attesting role; cadence: per artifact event.
  **What it is.** The attestation records.
  **Pin.** per attestation event.
  **RMF.** `GOVERN.2.1` (roles), `GOVERN.4.2` (publishing documentation). <!-- verify-citation: confirm sub-category wording. -->
  **42001.** Clause 7 (support — documented information). <!-- verify-citation: confirm sub-clauses. -->
  **Why this mapping.** Documented information held at role level is the records-management evidence.
  **Evidence quality.** `attested`.

- **Artifact.** This crosswalk — owner: AI risk engineer; cadence: quarterly.
  **What it is.** The evidence crosswalk.
  **Pin.** v1.0.
  **RMF.** `GOVERN.1.2` (processes for documenting AI decisions and mapping to organisational requirements). <!-- verify-citation: confirm GOVERN.1.2 wording. -->
  **42001.** Clause 7 (support — documented information). <!-- verify-citation: confirm sub-clauses. -->
  **Why this mapping.** The crosswalk is the self-documenting record of how the team evidences each framework position; it is both an RMF-adoption artifact and a 42001 documented-information artifact.
  **Evidence quality.** `attested`.
```

### 2.4 Reverse index — RMF → artifacts — `crosswalk/CarePilot-rmf-reverse-index.md`

Sorted by function and sub-category for top-down reading from a regulatory or audit ask. The first column cites the sub-category code as named in §2.3; the mark `<!-- verify-citation: ... -->` is inherited from the row.

| Sub-category | Artifact(s) evidencing |
|---|---|
| GOVERN.1.1 | `legal-register/CarePilot/2025.yaml` |
| GOVERN.1.2 | Pre-read + decision-log (ex-02); this crosswalk |
| GOVERN.1.3 | Tier ladder and tier determinations (ex-04) |
| GOVERN.1.4 | Annual management review record |
| GOVERN.2.1 | Team charter (ex-01); organisation-wide AI risk council charter; role-level attestations |
| GOVERN.3.1 | Organisation-wide AI risk council charter |
| GOVERN.4.1 | Team charter (ex-01); pre-read and decision log (ex-02) |
| GOVERN.4.2 | Model / system / dataset card risk slices (ex-03); role-level attestations |
| GOVERN.4.3 | Incidents and post-mortems (mod-111) |
| GOVERN.5.1 | Team charter (ex-01); vendor register |
| MAP.1.1 | Harm model; deployment-context brief |
| MAP.1.5 | Harm model |
| MAP.4.1 | Privacy report; vendor register; legal register |
| MAP.5.1 | Harm model; risk register |
| MAP.5.2 | Model / system / dataset card risk slices (ex-03) |
| MEASURE.1.1 | Shadow eval report |
| MEASURE.2.3 | Shadow eval report; guardrail report |
| MEASURE.2.7 | Red-team suite and report; regression fixtures; guardrail report |
| MEASURE.2.10 | Privacy report |
| MEASURE.2.11 | Fairness report |
| MEASURE.3.1 | Production monitors |
| MEASURE.4.1 | Model / system / dataset card risk slices (ex-03) |
| MEASURE.4.3 | Monthly risk-posture summary |
| MANAGE.1.1 | Risk register; regression fixtures; tier determinations (ex-04) |
| MANAGE.1.3 | Risk register |
| MANAGE.2.4 | Tier ladder and tier determinations (ex-04) |
| MANAGE.4.1 | Production monitors |
| MANAGE.4.2 | Incident playbook and incidents (mod-111) |
| MANAGE.4.3 | Monthly risk-posture summary |

Rows where this team does not currently attest coverage are logged as gaps in §2.6.

### 2.5 Reverse index — ISO/IEC 42001 → artifacts — `crosswalk/CarePilot-42001-reverse-index.md`

Sorted by clause family. Sub-clauses are at the family level; `<!-- verify-citation: ... -->` marks the family-to-sub-clause binding where unverified.

| Clause family | Artifact(s) evidencing |
|---|---|
| Clause 4 (context) | Deployment-context brief; legal register |
| Clause 5 (leadership) | Team charter; organisation-wide AI risk council charter; pre-read and decision log |
| Clause 6 (planning) | Harm model; risk register; tier ladder and tier determinations; privacy report; legal register |
| Clause 7 (support — documented information and competence) | Model / system / dataset card risk slices; role-level attestations; this crosswalk |
| Clause 8 (operation — operational controls) | Production monitors; vendor register |
| Clause 9 (performance evaluation) | Shadow eval; red-team suite; fairness report; privacy report; guardrail report; production monitors; monthly risk-posture summary; annual management review record |
| Clause 10 (improvement) | Incidents and post-mortems; regression fixtures; tier determinations; monthly risk-posture summary |
| Annex A operational controls | Multiple — rows above carry `<!-- verify-citation: confirm Annex A control IDs -->` tags; the gap log tracks per-control attestation. |

### 2.6 Gap log — `gaps/CarePilot-crosswalk-gap-log.md`

The crosswalk is honest about what is **not** yet attested.

| # | Gap | Framework position | Owner | Plan | Target |
|---|---|---|---|---|---|
| G1 | Vendor register breach-notification-clause survey not attested for all vendors | RMF GOVERN.5.1; 42001 Clause 8 | AI governance analyst | Add per-vendor breach-notification-clause row and attest in Q1-2027 vendor-register review | 2027-01-31 |
| G2 | Annual management review has not yet occurred (team is less than 12 months into the current cadence) | RMF GOVERN.1.4; 42001 Clause 9 | AI risk engineer with council chair | Scheduled for 2027-01 | 2027-01-31 |
| G3 | Annex A operational-control ID pinning not yet completed | 42001 Annex A | AI risk engineer with AI governance analyst | Read the current ISO/IEC 42001 text and pin each Annex A control referenced in §2.5; update rows and remove `<!-- verify-citation: ... -->` markers | Q1-2027 |
| G4 | Sub-category code pinning against AI 100-1 §5 not yet completed for all rows | RMF (all functions) | AI risk engineer | Read the current AI 100-1 §5 enumeration and pin each sub-category code; update rows and remove `<!-- verify-citation: ... -->` markers | Q1-2027 |
| G5 | Second-provider evaluation for the inference provider not attested | RMF GOVERN.5.1; 42001 Clause 8 | Tech lead / EM | Secondary-provider scoping memo + initial evaluation plan | Q2-2027 |
| G6 | GOVERN.3.1 (diverse decision-making) attested only at the org-wide council; team-level diversity attestation absent | RMF GOVERN.3.1 | AI governance analyst | Team-level diversity attestation written into the charter's 6-month review | 2027-05-13 |

Gaps are reviewed at the quarterly crosswalk review and feed the team's planning backlog.

### 2.7 Crosswalk review checklist — `checklists/crosswalk_review_checklist.md`

- Every artifact in the ex-01 charter §7 appears as a row in the crosswalk.
- Every row has: path, owner role, cadence, pin, RMF mapping, 42001 mapping, why-this-mapping sentence, evidence quality.
- Every RMF or 42001 code that is cited specifically is either verified from the current official source text or marked `<!-- verify-citation: ... -->`.
- Every "generic compliance" wording is removed; each why-this-mapping sentence names the specific aspect of the artifact that evidences the framework position.
- Evidence quality for each row is honestly assigned — `attested | draft | gap` — and gaps are listed in §2.6.
- The reverse indices (§2.4 and §2.5) are consistent with the main table; a row that cites a sub-category / clause must appear under that sub-category / clause in the reverse index.
- The crosswalk itself appears as a row in the crosswalk (as it does in §2.3's final row).
- The gap log has an owner and a target date for every entry.

### 2.8 Gap-triage checklist — `checklists/gap_triage_checklist.md`

Run during the quarterly crosswalk review.

- Each gap's owner has been contacted; the target date is realistic.
- Closed gaps are moved to a resolved-gaps section with a resolution date and pointer to the attested artifact.
- New gaps discovered in the quarter are added with an owner and a target date.
- Gaps that have slipped twice feed the council-level planning review — a gap that keeps slipping is a signal of misassigned ownership or insufficient resourcing.
- The crosswalk's version is bumped if any row's framework mapping, evidence quality, or pin changes; the changelog records the change.

## 3. Validation steps

1. **Every charter-listed artifact is a crosswalk row.** A reviewer picks three artifacts at random from the ex-01 charter §7 and confirms they appear as rows in the crosswalk.
2. **Every row has all seven fields.** path, owner, cadence, pin, RMF mapping, 42001 mapping, why-this-mapping, evidence quality.
3. **Specific codes are verified or marked.** Every specific sub-category code or sub-clause is either verified against the current official source or carries `<!-- verify-citation: ... -->`.
4. **No generic "complies with" language.** The why-this-mapping sentence names the specific aspect of the artifact that evidences the framework position.
5. **Evidence quality is honestly assigned.** A reviewer picks two rows marked `attested` and confirms the attestation is dated and role-level; picks two rows marked `draft` and confirms the gap log explains the status.
6. **Gap log has owners and dates.** Every gap has an owner role and a target date.
7. **Reverse indices are consistent.** Every sub-category / clause cited in the main table appears in the reverse index with the same artifact(s).
8. **Self-reference present.** The crosswalk itself appears as a row.
9. **Version and changelog present.** The frontmatter has version, effective-from, next-review, owner role; the changelog records at least the initial version.
10. **Scope is one team's artifacts.** The crosswalk does not drift into attesting organisation-wide artifacts that belong to the AI governance analyst or counsel; artifacts inherited from the organisation are explicitly marked as such.

## 4. Rubric or review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Artifact-first structure | Rows are artifacts; each row maps to framework positions | Framework-first structure that says "RMF 1.1: an artifact exists somewhere" |
| R2 | Both frameworks mapped | Every row has an RMF mapping and a 42001 mapping | Row missing one framework |
| R3 | Specific codes verified or marked | Every specific code is verified or marked with `<!-- verify-citation: ... -->` | Specific codes invented without marker |
| R4 | Why-this-mapping sentence specific | Each row's rationale sentence names a specific aspect of the artifact | Generic "complies with" phrasing |
| R5 | Evidence quality honestly assigned | Rows marked `attested` have dated role-level attestation; `draft` and `gap` rows explain why in the gap log | Rows marked `attested` when evidence is incomplete |
| R6 | Reverse indices consistent | Every mapping in the main table appears in the reverse index with the same artifact | Drift between main table and reverse indices |
| R7 | Gap log has owners and dates | Every gap has an owner role and a target date | Dangling gaps |
| R8 | Self-reference present | The crosswalk itself is a row | Missing self-reference |
| R9 | Versioned and reviewed | Frontmatter has version, effective-from, next-review; changelog present | No version or review cadence |
| R10 | Scope is single-team | Inherited artifacts are explicitly marked; team charter artifacts dominate | Crosswalk attests organisation-wide artifacts as the team's |
| R11 | Cadences explicit | Every row's cadence is stated in cadence form | Rows without cadence |
| R12 | No evasion-relevant specifics in the crosswalk | Specific detection thresholds, exploit payloads, evasion-sensitive numbers are not in the crosswalk body | Such specifics reproduced in the crosswalk |

Scoring bands: *Pass* — R1–R11 green and R12 honoured; *Pass with revisions* — R5 or R7 fails but R1–R4, R6, R8 pass; *Fail* — any of R1 / R3 / R4 / R10 / R12 fails.

## 5. Common mistakes

- **Framework-first authoring.** The author writes "RMF GOVERN 1.1: we have policies" with no specific artifact. The row is unfalsifiable. Always start from an artifact; attach the framework position to it.
- **Generic rationale.** "This artifact helps us comply with the sub-category" says nothing. The why-this-mapping sentence must name a specific aspect of the artifact.
- **Invented codes.** Writing "RMF GOVERN 1.1" or "ISO 42001 clause 6.1.4" when the specific code was not verified from the current source is a professional error. Use the sub-category / clause short name with `<!-- verify-citation: ... -->` until verified.
- **All rows marked `attested`.** Nothing is ever all green. A crosswalk with no gaps reads as unfalsifiable.
- **Gaps without owners.** A gap that no role owns is a gap that never closes. Every gap has an owner and a date.
- **Reverse-index drift.** The reverse indices are hand-maintained alongside the main table; without the §2.7 review check, they drift. Re-derive on every version bump.
- **Scope drift.** The crosswalk starts attesting organisation-wide artifacts (the council charter, the legal register, the organisation-wide RSP) as the team's. Mark inherited artifacts as inherited.
- **Missing self-reference.** The crosswalk is itself a documented-information artifact under 42001 clause 7. The row that says so is the crosswalk's own self-audit.
- **Reproducing evasion-relevant specifics.** Specific detection thresholds, exploit payload text, or specific monitor latency numbers belong behind the register's access control, not in the crosswalk.
- **Treating the crosswalk as one-shot.** A crosswalk that is not maintained is worse than no crosswalk, because it misleads an auditor. Quarterly review, versioned bumps, changelog entries — the crosswalk is a live document.

## 6. References

Only the official sources listed in the work packet (and clearly-labelled local module references) are used.

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — the crosswalk is anchored to the **GOVERN / MAP / MEASURE / MANAGE** functions and sub-categories enumerated in AI 100-1 §5. Sub-category code citations in §2.3 and §2.4 carry `<!-- verify-citation: ... -->` markers where the precise code has not been read back from the current AI 100-1 §5 enumeration. <!-- verify-citation: on the next pass, read AI 100-1 §5 and remove markers for pinned codes; log remaining uncertainty as a gap in §2.6 (G4). -->
- **ISO/IEC 42001 (AI management system).** The 42001 clause structure cited in §2.3 and §2.5 follows the standard's publicly-available high-level outline (Clauses 4–10 plus Annex A controls). Specific sub-clauses and Annex A control IDs carry `<!-- verify-citation: ... -->` markers where unverified. Precise sub-clause and Annex A control pinning is scheduled in gap G3 of §2.6.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — the red-team suite and regression-fixture rows in the crosswalk inherit the OWASP ML Top 10 cross-walk from mod-104 / mod-111 ex-04 at the artifact level.
- **MITRE ATLAS.** `https://atlas.mitre.org/` — the red-team suite row inherits the ATLAS cross-walk from mod-104 at the artifact level.
- **Module running system.** CarePilot, as introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`. The mod-101 ex-01 solution also cites and uses AI 100-1 §5 as the primary source of sub-category text; this crosswalk inherits that discipline.
- **Preceding and sibling module artifacts cited in the crosswalk.**
  - `lessons/mod-102-harm-modelling-and-risk-taxonomy/`, `lessons/mod-103-risk-quantification-and-registers/`, `lessons/mod-104-llm-red-teaming-and-attack-engineering/`, `lessons/mod-106-fairness-bias-and-explainability-engineering/`, `lessons/mod-107-privacy-risk-engineering/`, `lessons/mod-108-guardrail-engineering-and-effectiveness/`, `lessons/mod-109-continuous-monitoring-and-risk-register-integration/`, `lessons/mod-110-model-risk-management-and-supply-chain/`, `lessons/mod-111-incident-response-and-postmortems/`.
  - `modules/mod-112-cross-functional-ai-risk-program-and-communication/exercise-01-risk-program-charter-for-one-team/SOLUTION.md`.
  - `modules/mod-112-cross-functional-ai-risk-program-and-communication/exercise-02-council-pre-read-and-decision-log-drill/SOLUTION.md`.
  - `modules/mod-112-cross-functional-ai-risk-program-and-communication/exercise-03-model-card-risk-slice-authoring/SOLUTION.md`.
  - `modules/mod-112-cross-functional-ai-risk-program-and-communication/exercise-04-rsp-preparedness-style-tiering-contribution/SOLUTION.md`.
- **Exercise prompt.** `lessons/mod-112-cross-functional-ai-risk-program-and-communication/exercises/exercise-05-rmf-and-42001-evidence-crosswalk.md`.

<!-- Specific sub-category codes (RMF) and sub-clauses / Annex A control
     IDs (42001) carry `<!-- verify-citation: ... --` markers where they
     have not yet been read back from the current official source; pinning
     is scheduled in the crosswalk's gap log (G3, G4). -->

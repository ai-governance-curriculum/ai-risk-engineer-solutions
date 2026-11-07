# Exercise 01 — NIST AI RMF Obligation → Engineering Artifact Map — Reference Solution

## 1. Solution overview

This reference walks a worked `mod-101-ex01-rmf-map.md` for the module's fictional running system, **CarePilot** — a hospital-triage support assistant for a mid-sized regional health system, RAG-based over triage protocols, human-in-the-loop before every recommendation (per the exercise prompt).

The deliverable is a single markdown document with five top-level sections (running system, Playbook version cited, sub-category map, open items, peer hand-offs). The map covers **13 sub-categories**: 3 GOVERN, 3 MAP, 4 MEASURE, 3 MANAGE — comfortably above the ≥10 / ≥3 MEASURE / ≥3 MANAGE floor set by the acceptance criteria. Every row answers the five-sentence engineering read from chapter 01 (says / means / test / pass / evidence) and is either fully populated or marked **OPEN** with a linked risk-register TODO shape.

The core design decisions are:

- **Running system is CarePilot.** It is generative (RAG), has a non-trivial human-in-the-loop story (clinician sign-off before every recommendation reaches the patient), is exactly the kind of Annex III high-risk analogue the EU AI Act is written for, and is used identically in exercises 02–05 so the artifact set converges.
- **Sub-categories are drawn from the published AI 100-1 enumeration** (cited verbatim in the row, with a `to-confirm-at-submission` note left in the single cell where a Playbook action string would be quoted — Playbook action text is revisable on `airc.nist.gov` and is outside this solution's verifiable source set).
- **Thresholds are defined where they live** — in the CarePilot risk register (`risk-register/2025/CarePilot.yaml`), not in the map. The map *cites* the threshold; the register *sets* it. This is the discipline chapter 01 teaches.
- **Peer ownership is explicit.** Where `ai-eval-engineer` (L30) or `ai-infra-mlops` (L25 peer) owns the methodology, the row says so as a `hand-off → …` line. The map does not pretend L25 owns L30 methodology.

## 2. Implementation — worked answer — `mod-101-ex01-rmf-map.md`

The file below is the submission. Paths are illustrative (the exercise explicitly allows plausible paths for not-yet-built artifacts); the *shape* is what the reviewer checks.

```markdown
---
system: CarePilot (hospital triage-support RAG assistant, HITL)
framework: NIST AI RMF 1.0 (publication AI 100-1, January 2023)
playbook: NIST AI RMF Playbook, hosted on airc.nist.gov/AI_RMF_Knowledge_Base/Playbook
playbook_version_cited: "to-confirm-at-submission — record the airc.nist.gov Playbook entry revision date on the day this map is signed"
author: ai-risk-engineer (L25)
reviewers: AI risk council (chair), ai-evaluation-engineer (L35), ai-governance-analyst (L15)
status: draft-for-review
---

# CarePilot — NIST AI RMF 1.0 Obligation → Engineering Artifact Map

## 1. Running system

**CarePilot** is a hospital-triage support assistant for a mid-sized regional health system. It is RAG-based over the hospital's triage protocols and the ED-specific sections of the clinical handbook. A clinician sits on every recommendation before it reaches the patient (human-in-the-loop sign-off). Deployment: internal only, inside the hospital network, authenticated via SSO; no public endpoint.

Public documentation: fictional system, documented in `docs/systems/carepilot/README.md` in this repo.

## 2. Playbook version cited

- AI 100-1 (AI RMF 1.0), published January 2023. The core and sub-category list are cited from AI 100-1 §5 and the AI RMF Playbook hosted on `airc.nist.gov/AI_RMF_Knowledge_Base/Playbook`. Sub-category outcome statements in the table below follow the AI 100-1 enumeration; verbatim action text from the Playbook is not reproduced in this map (paraphrase convention, see [notes](#notes)).
- Playbook revision retrieved: *to-confirm-at-submission* — record the airc.nist.gov revision date on the day this map is signed.

## 3. Sub-category map

For each sub-category, the five-sentence read per chapter 01 is: *Says / Means / Test / Pass / Evidence*. Where a peer owns methodology, a `hand-off →` line states the peer and the metric consumed.

### GOVERN 1.1 — Legal and regulatory requirements involving AI are understood, managed, and documented

- *Says:* Legal and regulatory requirements involving AI are understood, managed, and documented (AI 100-1, GOVERN function).
- *Means:* for CarePilot, the applicable set is HIPAA (US), the FDA Good Machine-Learning Practice guiding principles (clinical decision-support context), and — if the hospital or any parent entity places the system on the EU market — Articles 9, 10, 14, 15, 72 of the EU AI Act (exercise 04 maps these).
- *Test:* legal register in `legal-register/CarePilot/2025.yaml`, reviewed quarterly by the `ai-governance-analyst` (L15) with sign-off by counsel; engineering supplies the technical facts (not the legal ruling).
- *Pass:* every regulatory obligation listed has a named internal owner, a target review date within 90 days, and a pointer to the engineering artifact that discharges it (exercises 02–04 populate those pointers).
- *Evidence:* `legal-register/CarePilot/2025.yaml`, signed by counsel and the head of AI governance (L60).
- *hand-off → `ai-governance-analyst` (L15) for the register itself; L25 engineering authors the technical-fields columns only.*

### GOVERN 3.1 — Decision-making related to mapping, measuring, and managing AI risks throughout the lifecycle is informed by a diverse team

- *Says:* Decision-making related to mapping, measuring, and managing AI risks throughout the lifecycle is informed by a diverse team (AI 100-1, GOVERN function).
- *Means:* CarePilot's harm model, risk register, and release-gate decisions are reviewed by an AI risk council whose membership includes ED clinicians, nursing leadership, patient-advocacy representation, IT security, legal, and engineering.
- *Test:* council charter in `docs/governance/ai-risk-council/charter.md`; meeting minutes archived; decision log pinned to the risk register.
- *Pass:* every release-gate decision for CarePilot has a quorum-signed minute entry; absences on a quorum-critical seat trigger a 48-hour re-vote.
- *Evidence:* `docs/governance/ai-risk-council/minutes/<yyyy-mm-dd>.md` per meeting; decisions back-referenced from `risk-register/2025/CarePilot.yaml`.

### GOVERN 4.1 — Organizational practices are in place to foster a critical thinking and safety-first mindset

- *Says:* Organizational practices are in place to foster a critical thinking and safety-first mindset in the design, development, deployment, and uses of AI systems to minimize negative impacts (AI 100-1, GOVERN function).
- *Means:* for CarePilot, this shows up as a pre-commit *risk review stamp* — any change to the retrieval corpus, the prompt, or the model carries a named reviewer from the AI risk council and a link to the eval run that cleared it.
- *Test:* a CI check verifies that changes under `src/carepilot/` have a signed-off risk-review stamp before merge; otherwise the pipeline blocks.
- *Pass:* 100% of merged PRs touching the scoped paths carry a stamp; the monthly audit (part of mod-112) samples ≥20 PRs and reports exceptions.
- *Evidence:* CI log per PR; monthly audit report at `reports/governance/carepilot/<yyyy-mm>.md`.

### MAP 1.1 — Context is established and understood

- *Says:* Context is established and understood (AI 100-1, MAP function).
- *Means:* for CarePilot, context = clinical setting (US regional hospital ED), users (ED clinicians, triage nurses), patients (ED patients of all demographics present in the hospital catchment), decision stakes (triage classification feeding clinician triage decision; wrong triage can cascade to patient harm), deployment envelope (internal network, SSO-authenticated, HITL).
- *Test:* context brief in `docs/systems/carepilot/context.md`, reviewed by the clinical-safety officer on every major release; reviewed by the harm-model authoring exercise (mod-102) on every risk-register update.
- *Pass:* context brief carries a signed-off reviewer and is dated within the last release window; any material change to users, patients, or decision stakes triggers a harm-model refresh (mod-102).
- *Evidence:* `docs/systems/carepilot/context.md`.
- *hand-off → `ai-governance-analyst` (L15) for intake / inventory fields; L25 engineering authors the technical envelope section.*

### MAP 2.3 — Scientific integrity and TEVV considerations are identified and documented

- *Says:* Scientific integrity and test, evaluation, verification, and validation (TEVV) considerations are identified and documented (AI 100-1, MAP function).
- *Means:* for CarePilot, TEVV = (a) the triage-protocol retrieval-alignment eval set, (b) the clinician-adjudicated groundedness eval set, (c) the LLM red-team suite (direct + indirect prompt injection), (d) the fairness slice set across patient demographics, (e) the privacy eval set on synthesised PHI; each with its lineage, authorship, and limitations stated.
- *Test:* the TEVV plan at `docs/systems/carepilot/tevv-plan.md` names every eval, who owns its methodology, its provenance, and its known limitations. The red-team suite is pinned to a commit hash at every release (mod-104).
- *Pass:* every claim in the model card (`model-cards/carepilot/<release>.md`) traces back to an eval named in the TEVV plan. Reviewer with no prior exposure can reproduce the eval in a sandbox from the plan alone.
- *Evidence:* `docs/systems/carepilot/tevv-plan.md`, signed by `ai-evaluation-engineer` (L35) and the AI risk council chair.
- *hand-off → `ai-eval-engineer` (L30) for the groundedness and retrieval-alignment methodology; consumed metrics = groundedness, retrieval-alignment.*

### MAP 5.1 — Likelihood and magnitude of each identified risk based on expected use are examined

- *Says:* Likelihood and magnitude of each identified risk based on expected use is examined (AI 100-1, MAP function).
- *Means:* for CarePilot, each risk row in `risk-register/2025/CarePilot.yaml` carries a *likelihood* rating (Low/Med/High/Critical, per the hospital's existing risk scale) and a *magnitude* rating against patient-safety severity levels S1–S4; both are set at design time by the harm-model author and re-examined whenever a mod-109 monitor signals material drift.
- *Test:* the risk register schema (mod-103) enforces the two columns; a release-gate check (mod-109) fails if any monitored risk has no current-quarter likelihood/magnitude assessment.
- *Pass:* every risk row in the register has non-empty likelihood and magnitude fields whose last-review date is within the quarter; drift beyond the review window creates an OPEN item.
- *Evidence:* `risk-register/2025/CarePilot.yaml`, with the quarterly review archived at `risk-register/reviews/CarePilot/<yyyy-qN>.md`.

### MEASURE 1.1 — Approaches and metrics for measurement of AI risks enumerated during the MAP function are selected for implementation

- *Says:* Approaches and metrics for measurement of AI risks enumerated during the MAP function are selected for implementation (AI 100-1, MEASURE function).
- *Means:* every risk row in the CarePilot register names at least one measured metric: groundedness, retrieval-alignment, attack-success-rate on the red-team suite, fairness slice delta, PII-leak rate, drift per monitor, and the HITL override rate. A `risk-metric-map.md` holds the traceability.
- *Test:* a repo-level CI check at release time verifies that every risk ID in the register has ≥1 metric ID linked back to an eval run file; unresolved risk IDs block the release.
- *Pass:* `risk-metric-map.md` has 100% coverage of register rows; the CI check is green on every candidate build.
- *Evidence:* `risk-metric-map.md`, versioned alongside the register.

### MEASURE 2.3 — AI system performance or assurance criteria are measured qualitatively or quantitatively and demonstrated for conditions similar to deployment

- *Says:* AI system performance or assurance criteria are measured qualitatively or quantitatively and demonstrated for conditions similar to deployment setting(s) (AI 100-1, MEASURE function).
- *Means:* for CarePilot, "conditions similar to deployment" = traffic replay of de-identified triage-consult transcripts drawn from the shadow log, routed through the candidate model and compared to the current production model on groundedness, retrieval-alignment, and clinician-adjudicated correctness.
- *Test:* shadow-eval harness at `evals/carepilot/shadow/` runs on every candidate; sample size ≥ the risk-register-defined floor per release.
- *Pass:* metric delta vs. the offline eval remains within the budget set in `risk-register/2025/CarePilot.yaml#performance-envelope`; delta exceeding budget blocks release (or forces a canary with tightened monitoring per mod-109).
- *Evidence:* `evidence/measure-2.3/<release-tag>/shadow-eval-report.json`, signed by the CI pipeline.
- *hand-off → `ai-eval-engineer` (L30) for the shadow-eval harness methodology; consumed metrics = groundedness, retrieval-alignment, clinician-adjudicated correctness.*

### MEASURE 2.7 — AI system security and resilience are evaluated and documented

- *Says:* AI system security and resilience — as identified in the MAP function — are evaluated and documented (AI 100-1, MEASURE function).
- *Means:* for CarePilot, resilience = refusal under adversarial prompting (direct and indirect prompt injection via a poisoned retrieval document), refusal on out-of-policy queries (e.g., self-harm content unrelated to triage), and PII containment (no leakage of other patients' records across sessions). Security maps to OWASP Machine Learning Security Top 10 and MITRE ATLAS techniques relevant to RAG assistants.
- *Test:* a garak + PyRIT + AgentDojo-style indirect-injection suite pinned at a versioned commit hash, run on every model candidate and on every retriever-corpus change. The suite coverage is cross-walked to MITRE ATLAS tactics/techniques (e.g., *ML Supply Chain Compromise*, *Evade ML Model*) and to the OWASP ML Top 10 risks that apply to RAG (notably OWASP ML Top 10 *Input Manipulation Attack* and *Output Integrity Attack*).
- *Pass:* attack-success-rate ≤ the target defined in `risk-register/2025/CarePilot.yaml#security-envelope`; every confirmed finding becomes a permanent regression fixture (mod-111) and the fixture must stay green.
- *Evidence:* signed red-team report at `evidence/measure-2.7/<release-tag>/report.json`; suite commit hash and config hash referenced from the release's evidence manifest.
- *hand-off → `security-learning` / `ai-infra-security` (L35) for deep platform defence and judge-supply-chain protection; engineering owns suite authorship and the metric.*

### MEASURE 2.11 — Fairness and bias are evaluated and results are documented

- *Says:* Fairness and bias — as identified in the MAP function — are evaluated and results are documented (AI 100-1, MEASURE function).
- *Means:* for CarePilot, "fairness" means the triage-classification distribution and the groundedness metric do not differ materially by patient demographic attributes defined in the harm model (age bucket, sex, ethnicity grouping used by the hospital, insurance status as a proxy-stratification control).
- *Test:* Fairlearn slice report on a clinician-adjudicated eval set; disparate-impact ratio on the triage-classification label distribution; subgroup groundedness delta.
- *Pass:* subgroup metric band and disparate-impact floor as defined in `risk-register/2025/CarePilot.yaml#fairness-envelope`; the envelope's numeric values are set jointly by the clinical-safety officer and the AI risk council. Mod-106 deepens the methodology.
- *Evidence:* `evidence/measure-2.11/<release-tag>/fairness-report.json`, signed by `ai-eval-engineer` (L30) methodology and co-signed by clinical safety.
- *hand-off → `ai-eval-engineer` (L30) for slice-methodology robustness (intersectional slicing; see mod-106); consumed artifact = fairness-report.json.*

### MEASURE 3.2 — Risk tracking approaches are considered for settings where AI risks are difficult to assess

- *Says:* Risk tracking approaches are considered for settings where AI risks are difficult to assess using currently available measurement techniques, or where metrics are not yet available (AI 100-1, MEASURE function).
- *Means:* for CarePilot, this names (a) near-miss reporting — every HITL override is a near-miss signal, (b) emergent-harm reporting — clinician-reported patterns that no current eval set catches, and (c) incident-candidate queueing for cases where the HITL caught something the eval would not have.
- *Test:* near-miss hook in the trace store; HITL override rate is a first-class metric in mod-109's monitoring.
- *Pass:* every guardrail hit above severity S2 is logged as an incident candidate; near-miss review cadence in the risk register is honoured; unresolved candidates older than the SLA defined in the register are exceptions.
- *Evidence:* near-miss dashboard spec at `monitors/carepilot/near-miss.md`; incident-candidate queue in `incidents/carepilot/candidates/` (mod-111 owns the queue schema).

### MANAGE 1.3 — Responses to the AI risks deemed high priority are developed, planned, and documented

- *Says:* Responses to the AI risks deemed high priority as identified by the MAP function are developed, planned, and documented. Risk response options can include mitigating, transferring, avoiding, or accepting (AI 100-1, MANAGE function).
- *Means:* every risk in the CarePilot register at severity High or Critical has a named response (mitigate/transfer/avoid/accept) with an owner and a target completion date; mitigations trace to specific controls (guardrails from mod-108, monitor spec from mod-109, incident hook from mod-111).
- *Test:* the risk-register schema enforces non-empty response + owner + due date for High and Critical rows; the quarterly review process (mod-103) verifies that due dates have not slipped.
- *Pass:* 0 High/Critical rows with slipped responses as of release gate; any slipped response forces an OPEN flag and an escalation to the AI risk council.
- *Evidence:* `risk-register/2025/CarePilot.yaml` plus `risk-register/reviews/CarePilot/<yyyy-qN>.md`.

### MANAGE 2.2 — Mechanisms are in place and applied to sustain the value of deployed AI systems

- *Says:* Mechanisms are in place and applied to sustain the value of deployed AI systems (AI 100-1, MANAGE function).
- *Means:* for CarePilot, this is the permanent regression-fixture loop — every confirmed incident, HITL-caught near-miss escalated by the clinical-safety officer, or red-team finding gets turned into a fixture in `regressions/carepilot/` that must stay green on every subsequent release.
- *Test:* CI runs the fixture set on every build; failure blocks the release; a monthly review (mod-111) checks fixture coverage against the incident log for drift.
- *Pass:* fixture set green on every release build; no fixture removed without AI risk council approval; incident-to-fixture conversion rate reported monthly.
- *Evidence:* `regressions/carepilot/` history; monthly report at `reports/regression-fixtures/carepilot/<yyyy-mm>.md`.

### MANAGE 4.1 — Post-deployment AI system monitoring plans are implemented, including mechanisms for capturing and evaluating input from users and other relevant AI actors

- *Says:* Post-deployment AI system monitoring plans are implemented, including mechanisms for capturing and evaluating input from users and other relevant AI actors, appeal and override, decommissioning, incident response, recovery, and change management (AI 100-1, MANAGE function).
- *Means:* for CarePilot, mod-109 wires drift, safety-monitor signal, HITL override rate, near-miss queue depth, and jailbreak / prompt-injection detections into the risk register; the output doubles as the Article 72 post-market surveillance report (exercise 04).
- *Test:* monitor specs at `monitors/carepilot/` reviewed quarterly; signal-to-register pipeline tested end-to-end on every release.
- *Pass:* every monitored signal has an alert threshold and a named receiver; incidents detected via monitor feed the mod-111 playbook within the SLA defined in the register.
- *Evidence:* post-market surveillance report at `reports/pms/carepilot/<quarter>.md`; monitor specs and alert configs under `monitors/carepilot/`.

## 4. Open items

The following rows are either partially populated (still OPEN) or have an acknowledged gap that is tracked as a TODO in `risk-register/2025/CarePilot.yaml`.

| Row | Field | What is open | Target close | Owner | Risk-register TODO ID |
|---|---|---|---|---|---|
| GOVERN 1.1 | Legal register pointers | EU AI Act Articles 9/10/14/15/72 pointers are due from exercise 04; placeholder in legal register. | *to-confirm-at-submission — set after exercise 04 is submitted* | `ai-governance-analyst` (L15) | TODO-RR-G11-01 |
| MAP 2.3 | TEVV plan — privacy eval | PII-leak eval plan pending mod-107 methodology selection. | Before privacy-eval methodology is ratified in mod-107 | ai-risk-engineer (L25) with `ai-eval-engineer` (L30) consult | TODO-RR-M23-01 |
| MEASURE 2.7 | Suite coverage | AgentDojo-style indirect-injection suite integration pending; direct-injection suite via garak/PyRIT is in. | Before mod-104 submission | ai-risk-engineer (L25) | TODO-RR-MS27-01 |
| MEASURE 2.11 | Intersectional fairness slicing | Single-attribute slices in; intersectional methodology pending mod-106. | Before mod-106 submission | ai-risk-engineer (L25) + `ai-eval-engineer` (L30) | TODO-RR-MS211-01 |
| MANAGE 4.1 | HITL override rate alert threshold | Monitor exists; alert threshold not yet calibrated against baseline. | Before first production release | ai-risk-engineer (L25) with clinical safety | TODO-RR-MN41-01 |

## 5. Peer hand-offs

Metrics and methodologies CarePilot consumes from peer roles.

| Metric / methodology | Consumed from | At level | Where consumed | Artifact |
|---|---|---|---|---|
| Groundedness | `ai-eval-engineer` | L30 (AI Engineering family) | MEASURE 2.3, MEASURE 2.11, MAP 2.3 | `evals/carepilot/groundedness/` + `evidence/measure-2.3/<release>/shadow-eval-report.json` |
| Retrieval-alignment | `ai-eval-engineer` | L30 | MEASURE 2.3, MAP 2.3 | same report |
| Fairness slice methodology (intersectional) | `ai-eval-engineer` | L30 (with cross-training into `model-evaluation-engineer` statistical calibration) | MEASURE 2.11 | `evidence/measure-2.11/<release>/fairness-report.json` |
| Deployment-gate pipeline implementation | `ai-infra-mlops` | L25 (Infra family peer) | MANAGE 2.2, MANAGE 4.1 | `.github/workflows/carepilot-release.yml` + `evidence/release-gate/<release>/manifest.json` |
| Platform-scale ML security (judge supply chain, eval-set exfiltration prevention) | `security-learning` / `ai-infra-security` | L35 | MEASURE 2.7 | `security/ml/carepilot/ARCHITECTURE.md` (ownership shared; L35 signs) |
| Release-assurance sign-off and audit-trail packaging | `ai-evaluation-engineer` | L35 (Governance family next-up) | overall release | `releases/carepilot/<release>/assurance-packet.pdf` |

## Notes

- Paraphrase convention: the *outcome statement* for each sub-category above is a short-form reference to the AI 100-1 sub-category and its heading; the AI RMF Playbook's action strings are *not* reproduced verbatim in this submission. A reviewer who needs the Playbook text should open the Playbook entry and cross-reference by the sub-category ID.
- "signer" = the role or named owner who puts their name on the artifact, not necessarily the tool that produces it.
- Path conventions follow chapter 01 and subsequent chapters in this module.
```

## 3. Validation steps

A reviewer should be able to confirm the submission in under 15 minutes:

1. Count the sub-categories: expect ≥10 total, ≥3 MEASURE, ≥3 MANAGE. This solution has 13 total, 4 MEASURE, 3 MANAGE, 3 GOVERN, 3 MAP.
2. For each row, check that *Says / Means / Test / Pass / Evidence* are all populated, OR that the row is in the OPEN items table with a linked risk-register TODO ID.
3. For each threshold mentioned, verify it cites a source (`risk-register/2025/CarePilot.yaml#...`). No thresholds are set inline — they are all pointed at the register.
4. For each artifact, verify it has a path and either a signer or an owner. Hand-offs cite the peer and the consumed metric by name.
5. The Playbook version line is present at the top of the submission (with a `to-confirm-at-submission` placeholder on the dated revision — this is a *verifiable later* flag, not an invented fact).
6. Read the sub-category map end-to-end as a reviewer with no prior exposure to CarePilot — you should understand what tests run, what passes, and what evidence is produced for each row.

## 4. Rubric / review checklist

Scoring is pass/fail at the row level; the overall submission is accepted if **all** row-level checks pass and the structural checks pass. Review this alongside the acceptance criteria printed in the exercise file.

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Running system + public documentation link present | Named system; one-paragraph description; link (even if to a repo-internal doc) | Missing or ambiguous system scope |
| R2 | Playbook version cited at top | Version/date line present (a `to-confirm-at-submission` placeholder on the revision date is acceptable) | Not cited |
| R3 | ≥10 sub-categories total | Row count ≥10 | <10 |
| R4 | ≥3 MEASURE sub-categories | 3+ MEASURE rows | <3 MEASURE |
| R5 | ≥3 MANAGE sub-categories | 3+ MANAGE rows | <3 MANAGE |
| R6 | Each row has all five fields OR is marked OPEN | Fully populated or linked OPEN row | Row partially filled without OPEN marker |
| R7 | Thresholds cite their source | Threshold text includes a register/chapter/standard pointer | Inline numeric thresholds with no source |
| R8 | Artifacts have paths and signers/owners | Path present + owner present per artifact | Missing path or owner |
| R9 | Peer hand-offs explicit | Rows that depend on L30 methodology include a `hand-off →` line and are summarised in section 5 | Peer dependencies hidden inside L25 ownership |
| R10 | Open items table present with target dates | Section 4 lists every OPEN row with target close date and owner | OPEN rows mentioned inline but not aggregated |
| R11 | Peer hand-off table present | Section 5 lists every consumed metric/methodology | Peers mentioned inline only |
| R12 | Under-15-minute readability | A reviewer unfamiliar with CarePilot can, per row, state "test X, pass Y, evidence Z" in one sentence | Rows require interpretation or re-reading |

**Scoring bands.**

- *Pass* — R1–R11 all green; R12 green.
- *Pass with revisions* — one row violates R6 or R7 but the overall shape is right; revise the row and resubmit.
- *Fail* — any of R3–R5 fails (not enough rows or wrong distribution), or R1/R2 missing, or R9 silently absorbs peer ownership into L25 rows.

## 5. Common mistakes

- **Collapsing MEASURE and MANAGE into a single section.** The functions have different owners (MEASURE is this role's; MANAGE is co-owned). Collapsing them hides the co-ownership.
- **Setting thresholds inline instead of in the register.** The map must *cite* a threshold; the risk register *holds* the threshold. If a reviewer sees `attack-success-rate ≤ 0.05` in the map without a register pointer, that is a finding.
- **Skipping sub-categories as "n/a".** An out-of-scope sub-category needs a one-sentence reason, not silence. "MEASURE 2.11 — n/a" is rejected; "MEASURE 2.11 — CarePilot is a clinical decision-support assistant with patient-demographic exposure; the slice must be produced even though the model itself is a RAG over triage protocols — therefore IN scope; see row above" is accepted.
- **Inventing artifact paths and signing roles.** Paths in this exercise are *plausible*, not real. But signing roles must map to actual levels on the ladder (L15 analyst, L30 eval peer, L35 release-assurance, etc.). Signing by an invented role is a finding.
- **Quoting Playbook action text verbatim without citation of the Playbook revision date.** The AI RMF Playbook on `airc.nist.gov` is revisable; a quoted action string without a revision date cannot be verified. Paraphrase and cite the sub-category ID.
- **Treating `hand-off →` as "someone else's problem."** The L25 engineer still *consumes* the metric and *sets the threshold*. The hand-off states the peer owns the methodology — not the gating decision.
- **Open items without target dates or owners.** An OPEN row with no owner and no date is invisible to the risk register and is rejected.

## 6. References

Only the official sources listed in the work packet (and local exercise context) are used.

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** Four-function structure (GOVERN / MAP / MEASURE / MANAGE), sub-category enumeration. `https://www.nist.gov/itl/ai-risk-management-framework`
- **NIST AI RMF Playbook.** Hosted on the AI Resource Center at `airc.nist.gov/AI_RMF_Knowledge_Base/Playbook` — the day-to-day sub-category action reference. Playbook revision date is version-stamped in the submission.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — referenced from MEASURE 2.7 as the OWASP ML Top 10 risks applicable to RAG (input-manipulation and output-integrity classes).
- **MITRE ATLAS.** `https://atlas.mitre.org/` — referenced from MEASURE 2.7 as the tactic/technique catalogue the red-team suite is cross-walked to.
- **Chapter 01 of this module** — `lessons/mod-101-ai-risk-frameworks-and-position/01-nist-ai-rmf-as-an-engineer.md` — the five-sentence read format and the running translation-table shape.
- **Chapter 06 of this module** — `lessons/mod-101-ai-risk-frameworks-and-position/06-role-scope-on-the-level-ladder.md` — the ladder levels used for peer hand-offs.
- Exercise prompt — `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md` — including the fictional `CarePilot` running system.

<!-- Playbook action-text revision dates are left as `to-confirm-at-submission` placeholders; they will be set at submission time from the airc.nist.gov Playbook entry metadata. -->

# Exercise 05 — Role Scope and Deferral Contract — Reference Solution

## 1. Solution overview

This reference walks a worked `mod-101-ex05-deferral-contract.md` for **CarePilot** under a plausible hospital-system org sketch. CarePilot is the running system carried across exercises 01–04; the deferral contract pins everything those exercises produced to a role on the AI Governance level ladder (chapter 06).

The deliverable is a single markdown document with the eight sections called for by the exercise prompt:

1. Running system + org sketch
2. Owned obligations
3. Consumed obligations
4. Sideways hand-offs
5. Escalations upward
6. Out of scope
7. Role-boundary risks
8. Sign-off

Design decisions:

- **The org sketch is a realistic mid-sized regional hospital's AI-governance surface** — mature enough to have `ai-governance-analyst` (L15), `ml-engineer` (L20), `ai-eval-engineer` (L30), and `ai-infra-mlops` (L25) staffed, but *short on L35+ roles*. This matches the pattern chapter 06 names: "many mature enterprises have this and are still short on levels 35+." The role-boundary-risks section is non-empty precisely because of that staffing shape.
- **Every row cites an artifact path from exercises 01–04**, by design. No abstract ownership language.
- **"TBD" appears nowhere.** Where ownership is genuinely unclear, it is filed as a role-boundary risk and the mitigation is named.
- **Section 7 (role-boundary risks) gets the most editorial attention** — chapter 06 and the exercise prompt both call it the most valuable part of the contract for the management chain.
- **Section 8 (sign-off) is written as a paragraph an executive could actually put their name to**, with a crisp "what you are accepting" statement.

## 2. Implementation — worked answer — `mod-101-ex05-deferral-contract.md`

```markdown
---
system: CarePilot (hospital triage-support RAG assistant, HITL)
role_under_contract: ai-risk-engineer (L25, AI Governance family)
cross-ref:
  - mod-101-ex01-rmf-map.md
  - mod-101-ex02-applicability.md
  - mod-101-ex03-iso-map.md
  - mod-101-ex04-eu-ai-act-map.md
author: ai-risk-engineer (L25)
reviewers: head-of-ai-governance (L60), AI risk council chair, CMIO, VP of IT
status: draft-for-signature
---

# CarePilot — AI Risk Engineer (L25) Deferral Contract

## 1. Running system + org sketch

**Running system.** CarePilot is a hospital-triage support assistant for a mid-sized regional health system, RAG-based over triage protocols, with a clinician on every recommendation before it reaches the patient. Internal only, SSO-authenticated, deployed inside the hospital network. Full description in `docs/systems/carepilot/README.md`.

**Org sketch.** The health system has ~ 2,500 employees and runs an IT organisation structured along familiar hospital-IT lines: a CMIO's office (clinical informatics), an IT-security team, a data-governance function, and a vendor-management team. AI governance is a *shared surface* across the CMIO's office (owns clinical safety) and IT (owns infrastructure, security, and now AI risk). The AI risk council is co-chaired by the CMIO and the head of IT risk.

**Current staffing map by level for the AI-governance surface.**

| Level | Role | Named team / seat |
|---|---|---|
| L10 | `ai-infra-junior-engineer` equivalents | Platform engineering team (multiple) |
| L15 | `ai-governance-analyst` | AI governance analyst — one seat, inside the CMIO's office |
| L20 | `ml-engineer` | Two ML engineers in the data-science team |
| L25 | `ai-risk-engineer` (this contract) | One seat — reports to head of IT risk, dotted-line to CMIO |
| L25 (peer) | `ai-infra-mlops` | One seat — platform engineering |
| L30 (peer) | `ai-eval-engineer` | One seat — AI engineering (recently filled) |
| L30 (peer) | `model-evaluation-engineer` | *Not staffed* — eval methodology is cross-trained from the ML-engineer team |
| L35 | `ai-evaluation-engineer` (release-assurance, audit-trail) | *Not staffed* |
| L35 | `security-learning` / `ai-infra-security` | *Not staffed* — IT security owns classical application security but not ML-specific platform security |
| L40 | `agentic-safety-engineer` (frontier / dangerous-capability) | *Not staffed* — not expected to be staffed at this hospital's scale |
| L50 | `senior-ai-governance-architect` | *Not staffed* — the head of AI governance covers L50 and L60 jointly |
| L60 | `head-of-ai-governance` | Head of IT risk doubles as acting head of AI governance |
| L70 | `chief-ai-officer` | *Not staffed* |

The shape — well-staffed L10–L30, thinning L35+, hat-stacking at L60 — is typical of the sector. The role-boundary-risks section (§7) is where that shape surfaces as named risk.

## 2. Owned obligations

Every row is an obligation this L25 role owns on CarePilot. Framework pointers tie back to the exercise 01–04 maps.

| Obligation | Framework | Artifact | Path | Cadence |
|---|---|---|---|---|
| Author the harm model for CarePilot | AI RMF MAP 1.x / 2.x; AI 600-1 context | Harm-model document (mod-102) | `docs/harm-model/carepilot.md` | Every major release + quarterly review |
| Maintain the CarePilot risk register | AI RMF MEASURE 1.1 / MANAGE 1.3; ISO 42001 clause 6.1; EU AI Act Article 9 | Risk register | `risk-register/2025/CarePilot.yaml` | Continuous; quarterly review archived |
| Author the LLM red-team suite and run it per release | AI RMF MEASURE 2.7; AI 600-1 Risk 9; ISO 42001 A.6.2.4; EU AI Act Article 15 (cybersecurity) | Red-team report + pinned suite | `evidence/measure-2.7/<release>/report.json` + `suites/garak-pyrit-indirect/<commit>` | Every model/retriever candidate; monthly ad-hoc probe |
| Author adversarial-ML evaluations | AI RMF MEASURE 2.7; EU AI Act Article 15 (robustness) | Adversarial-ML report (mod-105) | `evidence/mod-105/<release>/report.json` | Every candidate |
| Author fairness / bias eval plan | AI RMF MEASURE 2.11; AI 600-1 Risk 6; EU AI Act Article 10 | Fairness slice plan + envelope in the risk register | `evals/carepilot/fairness/slices-<version>` + register | Every candidate; reviewed with clinical safety quarterly |
| Author privacy-risk eval plan | AI RMF MEASURE 2.10 (errata); AI 600-1 Risk 4; EU AI Act Article 10 | Privacy eval set + MIA baseline (mod-107) | `evidence/privacy/<release>/` | Every candidate |
| Specify guardrail configs and release-gate thresholds | AI RMF MANAGE 2.x; AI 600-1 Risks 2,3,9,10; ISO 42001 A.8.2; EU AI Act Articles 14, 15 | Guardrail configs | `guardrails/carepilot/*.yaml` | Every candidate |
| Wire monitoring signals into risk register | AI RMF MANAGE 4.1; ISO 42001 clause 9.1; EU AI Act Article 72 | Monitor specs | `monitors/carepilot/*.md` | Continuous |
| Produce the post-market surveillance report | AI RMF MANAGE 4.1; EU AI Act Article 72 | PMS report | `reports/pms/carepilot/<quarter>.md` | Quarterly |
| Run AI-specific incident post-mortems and feed fixtures | AI RMF MANAGE 2.2; ISO 42001 clause 10.2; EU AI Act Article 73 interaction | Post-mortem docs + fixtures | `incidents/carepilot/`, `regressions/carepilot/` | On every AI-specific incident |
| Author technical sections of the AI-system impact assessment | ISO/IEC 42005; EU AI Act Article 27 (stretch) | Impact assessment technical sections | `docs/impact-assessments/carepilot/` | Every major release |
| Supply the technical fields of the registration record | EU AI Act Article 61/49/71 (registration) | Field index | `docs/registration/carepilot/fields.md` | At each market-placement event |
| Author the supply-chain evidence inventory for CarePilot | AI 600-1 Risk 12; ISO 42001 A.10.2; mod-110 | ML-BOM + vendor attestations | `bom/carepilot/<release>/mlbom.json`, `vendor-attestations/<vendor>/<release>.json` | Every release |

## 3. Consumed obligations

Rows this L25 role depends on but does not own. Every row includes a fallback (what this role does when the upstream is late).

| Obligation | Owner level | Owner role | Artifact consumed | Fallback if upstream is late |
|---|---|---|---|---|
| CarePilot intake and inventory record | L15 | `ai-governance-analyst` (CMIO's office) | System inventory entry; stakeholder list | Pull last-known intake from inventory; file a one-line delta into the risk register as an OPEN item; escalate to head of AI governance if gap persists > 1 release |
| Legal register and regulatory tracking | L15 | `ai-governance-analyst` (with counsel) | `legal-register/CarePilot/2025.yaml` | Use the previous quarter's register; file counsel-pending items as OPEN; do not proceed with release if a new mandatory obligation is unanswered |
| First-draft impact assessment | L15 | `ai-governance-analyst` | Draft impact assessment | Provide the technical sections against the context brief; mark the governance sections as "awaiting L15 draft" |
| Classical ML / DL tooling fluency | L20 / L10 | `ml-engineer` / platform engineering | PyTorch / HF / MLflow / registry conventions | Use the existing platform conventions; do not substitute (L25 does not re-pave pipelines) |
| Model-registry entries for CarePilot models | L25 peer | `ai-infra-mlops` | Registry entry with provenance fields | Block the release-gate until a registry entry is present; do not bypass the gate |
| App-side evaluation plumbing (traces, judges, RAG eval) | L30 peer | `ai-eval-engineer` | `evals/carepilot/` methodology | Use the previous release's eval harness; do not re-author eval plumbing |
| Statistical methodology / judge-vs-human calibration | L30 peer | `model-evaluation-engineer` (currently *not staffed*) | Calibration record | Cross-train from ML-engineer team (role-boundary risk RB-01 below); file as risk until seat filled |

## 4. Sideways hand-offs

Hand-offs to peer roles. Each row names the interface (what this role specifies vs. what the peer implements) and the artifact that closes the interface.

| Peer role | Level | Interface this role specifies | Peer implements | Artifact that closes the interface |
|---|---|---|---|---|
| `ai-eval-engineer` | L30 (AI Engineering) | Metric envelope (floor/band) + slice set + evaluation cadence | Traces, judges, RAG eval harness, groundedness/retrieval-alignment methodology | `evals/carepilot/<metric>/` + consumed scorecards under `evidence/measure-*/` |
| `model-evaluation-engineer` | L30 (ML Engineering) *(currently not staffed)* | Statistical confidence bars; benchmark design | Benchmark execution; calibration | `reports/calibration/carepilot/<release>.md` — currently cross-trained from ML engineering (RB-01) |
| `ai-infra-mlops` | L25 (Infra peer) | Release-gate contract: inputs (eval artifacts, scan results), thresholds, failure modes | CI/CD pipeline implementation; registry gate; canary infra | `.github/workflows/carepilot-release.yml` + `releases/carepilot/<release>/manifest.json` |
| IT security (classical application security) | adjacent | Standard app-sec checks against the CarePilot deployment | Static analysis, SAST/DAST, container scan | Existing IT security scorecard entries; no AI-specific coverage |
| Clinical safety officer | adjacent (clinical governance) | Definitions of safety-escalation subsets, fairness-envelope magnitudes | Clinical adjudication of eval sets; safety-SOP card content | `docs/clinical-safety/carepilot/` co-authored docs |
| Product / UX | adjacent | Technical content of oversight design (signals, override paths) | UI; user research; training-material style | `docs/oversight/carepilot.md` co-authored with product |

## 5. Escalations upward

Each row names the trigger (when escalation is initiated) and what this role supplies to the escalation target.

| Escalation target | Level | Trigger | What this role supplies |
|---|---|---|---|
| `ai-evaluation-engineer` (release-assurance, audit-trail) | L35 *(not staffed; see RB-02)* | Any release where the risk register carries an unresolved High/Critical row; any regulator-facing audit event | Signed assurance packet (`releases/carepilot/<release>/assurance-packet.pdf` shape) — currently landing on the AI risk council |
| `agentic-safety-engineer` (frontier / dangerous-capability) | L40 *(not staffed; not expected to be staffed at this scale)* | If CarePilot ever adds agentic tool-use or expands scope beyond clinical decision-support | A scope-change memo citing the trigger and the risk the new scope adds; escalation path leaves the hospital (vendor or specialised consultancy) |
| `senior-ai-governance-architect` (control-library architecture; SoA authorship) | L50 *(not staffed; covered by head of AI governance in a hat-stack; RB-03)* | Any cross-system control decision; any SoA change; any new regulation entering scope | Engineering input and evidence summaries; a one-pager per new control the architect is evaluating |
| `head-of-ai-governance` (program leadership, board reporting) | L60 (hat-stacked with head of IT risk) | Quarterly cadence; any serious incident; any out-of-cycle regulator engagement | PMS summary, the quarterly risk-register review, a one-page board-shaped executive summary |
| `chief-ai-officer` (executive) | L70 *(not staffed; CIO/CISO covers executive AI asks)* | Board-level AI governance questions; material changes in AI exposure | Via head of AI governance; no direct interface |

## 6. Out of scope

| Out-of-scope item | Who owns it | Interface this role maintains |
|---|---|---|
| Legal opinion (HIPAA, EU AI Act classification, state privacy law) | Health-system counsel (external firm on retainer: Firm X) + CMIO's compliance officer | Supply technical facts on request; do not render legal opinion |
| SOC-side DFIR (network compromise, credential theft, endpoint response) | IT security SOC (in-house) + managed MDR vendor | Feed AI-specific incident signal (jailbreak success, model exfiltration attempt, poisoned-model ingest alert) into the SOC's incident queue; do not run the SOC |
| Clinical decision-making | Attending clinicians + CMIO's office | Provide technical evidence that supports (or does not support) a given clinical decision path; do not make clinical calls |
| Patient-safety incident adjudication (clinical) | Patient-safety committee under the chief medical officer | Supply AI-specific technical evidence; do not adjudicate clinical severity |
| Vendor contract negotiation | Vendor-management team + counsel | Supply the technical-requirements section of the RFP / contract; do not sign |

## 7. Role-boundary risks

The interesting part of this contract. Each risk below is a place where the current staffing map lands something on the wrong person, or where no one owns a surface the organisation needs owned.

### RB-01 — No L30 `model-evaluation-engineer`

**Risk.** Statistical calibration of judges, benchmark-design rigour, and judge-vs-human calibration are currently cross-trained from the ML-engineer team. The ML engineers are capable, but calibration methodology is *not* their primary responsibility; releases where confidence bars are tight (small eval sets, close fairness-band margins) may ship without proper statistical treatment.

**Current mitigation.** Cross-training arrangement is documented; close-margin releases get escalated to the AI risk council for a judgement call.

**Target mitigation.** Fill the L30 seat, or formalise a vendor relationship with a calibration-methodology consultancy.

**Who is affected.** `ai-eval-engineer` (L30) and this L25 role share the load; both sign off on the workaround.

### RB-02 — No L35 `ai-evaluation-engineer` (release-assurance, audit-trail)

**Risk.** Release-assurance sign-off and audit-trail packaging for regulator-facing engagement are currently landing on the AI risk council collectively. The council is not constituted as a release-assurance engineering function; its members are senior but not the full-time release-assurance engineers the chapter 06 ladder expects. For external audits (ISO 42001, a hypothetical EU AI Act Article 15 conformity assessment) this is a visible weakness.

**Current mitigation.** This L25 role packages a candidate assurance packet in a format the AI risk council can sign; a dry-run audit is scheduled before any external engagement.

**Target mitigation.** Fill the L35 seat, or contract a specialist firm for regulator-facing engagements.

**Who is affected.** This L25 role carries methodology responsibility it was not designed to carry; the health system's regulator-facing posture is weaker than its internal engineering practice.

### RB-03 — L50 and L60 hat-stacked on the head of IT risk

**Risk.** The head of IT risk doubles as acting head of AI governance (L60) and informally covers senior-architect (L50) responsibilities. Hat-stacking across three distinct levels makes it hard to produce a Statement of Applicability and a cross-jurisdiction control library at the shape the standards expect.

**Current mitigation.** External consulting firm Y is retained for SoA drafting and for cross-jurisdiction reconciliation on demand; the head of IT risk reviews and signs.

**Target mitigation.** Split the role into distinct L50 and L60 seats, or make the external consulting retainer a formal co-signer on the SoA.

### RB-04 — No `security-learning` / `ai-infra-security` (L35)

**Risk.** Classical application security is well-covered by IT security; AI-specific platform security (judge supply chain, eval-set exfiltration prevention, MLSecOps) is not covered. Residual risk shows up in Risks 9 and 12 of exercise 02.

**Current mitigation.** This L25 role covers adversarial-ML and red-team at working depth (mod-104 and mod-105 outputs); platform-scale MLSecOps is explicitly out of scope and is tracked as a residual-risk statement in the register.

**Target mitigation.** Fill the L35 security seat, or move platform-scale MLSecOps to a managed-service provider with contractual depth.

### RB-05 — `ai-governance-analyst` (L15) is one seat

**Risk.** A single L15 analyst covers intake, inventory, regulatory tracking, impact-assessment first drafts, and framework-crosswalk legwork for CarePilot *and* any other AI system the hospital adopts. The role is a single point of failure; absence creates delays that fall silently on this L25 role.

**Current mitigation.** Fallback paths in Section 3 above; escalate to head of AI governance when the delay exceeds one release cycle.

**Target mitigation.** Fund a second L15 analyst (and/or reserve external contractor capacity).

## 8. Sign-off

By signing this contract, the head of IT risk (acting head of AI governance, L60) accepts:

1. That the L25 ai-risk-engineer is accountable for the obligations listed in §2 at the cadences stated, against the artifacts at the paths cited.
2. That the consumed obligations in §3 remain the owners' responsibility; delays trigger the fallbacks documented and the escalation paths in §5.
3. That the sideways hand-offs in §4 are honoured as *interface contracts*, not as flexible ownership boundaries.
4. That the escalations in §5 are initiated at the triggers stated, not at discretionary moments.
5. That the out-of-scope items in §6 are not silently absorbed by the L25 role.
6. **Most importantly: that the role-boundary risks in §7 are acknowledged as named organisational risks, not as hidden expectations on the L25 role.** The RB rows are open items on the organisation's governance roadmap, not on this L25 role's quarterly deliverables.

Signed:

- head of IT risk (acting head of AI governance, L60): ___________________ date: __________
- CMIO: ___________________ date: __________
- ai-risk-engineer (L25): ___________________ date: __________

This contract is refreshed quarterly and immediately on: (a) any change in CarePilot's scope or classification, (b) any new regulation entering scope, (c) any filled or vacated role in the staffing map.

## Stretch (optional): RACI matrix

| Obligation | R (does the work) | A (accountable) | C (consulted) | I (informed) |
|---|---|---|---|---|
| Risk register maintenance | L25 | AI risk council | L15 analyst, clinical safety | L60 |
| Red-team suite | L25 | AI risk council | L30 eval peer, (absent L35 security) | L60 |
| Fairness slice set | L25 | AI risk council | L30 eval peer, clinical safety | L60 |
| Oversight design doc | L25 (technical sections), product (UI) | AI risk council | CMIO, clinical safety | L60 |
| PMS report | L25 | AI risk council | L30 eval peer, L25 MLOps peer | L60, counsel |
| Supply-chain ML-BOM | L25 | L60 (currently; target L50) | Vendor management | L60 |
| SoA authorship | (absent L50, covered by L60 + external firm) | L60 | L25, L35 (absent) | AI risk council |
```

## 3. Validation steps

1. Count sections: all eight are present (running system + org sketch, owned, consumed, sideways, escalations, out of scope, role-boundary risks, sign-off).
2. Every row in every table cites a specific artifact path — spot-check against the paths used in exercises 01–04.
3. §7 (role-boundary risks) is non-empty and names ≥1 gap with a mitigation each. This solution names five.
4. Every consumed-obligation row has a fallback (§3).
5. Every escalation row has a trigger (§5).
6. Every out-of-scope row names the actual receiver (§6).
7. "TBD" does not appear anywhere. (Grep for it before submission.)
8. §8 is signable — written as a paragraph with an explicit "what you are accepting" list and signature lines.

## 4. Rubric / review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | All eight sections present | 8 sections populated | Any section missing |
| R2 | Every row cites a specific artifact | Paths resolve to exercises 01–04 (or mod-102/103/… planned) | Abstract ownership language |
| R3 | Every consumed row has a fallback | Fallback sentence per row | Fallback missing |
| R4 | Every escalation row has a trigger | Trigger sentence per row | Silent escalation |
| R5 | Every out-of-scope row names the receiver | Named team / firm / person | "Someone else" |
| R6 | Role-boundary risks section non-empty | ≥1 named risk with ≥1 mitigation | Empty / "no risks" |
| R7 | No "TBD" anywhere | 0 occurrences | 1+ |
| R8 | Sign-off paragraph signable | Explicit "accepting" list + signature lines | Unsignable vagueness |
| R9 | Readable under 15 minutes by a technical executive | Clear structure; summary available in headings alone | Requires re-reading |
| R10 | Staffing map by level present | §1 includes a level/role/seat table | Level map missing |
| R11 | Owned obligations tie back to exercises 01–04 | Framework pointer per row | No framework pointer |

**Scoring bands.** Pass = R1–R11 green; Pass with revisions = one row missing a fallback or trigger; Fail = R6 empty, R7 contains "TBD," or R8 unsignable.

## 5. Common mistakes

- **Empty role-boundary-risks section.** The common failure mode is "everything is fine; everyone is in place." Chapter 06 is explicit: a "no risks" answer is usually a missed observation. If every ladder level is staffed and healthy, name the individuals or teams and justify — do not leave the section blank.
- **Using "TBD" to mean "I don't know yet."** Replace it with either (a) a role-boundary risk row (if it's a staffing gap), or (b) an OPEN item in the register (if it's a work item).
- **Abstract ownership language.** "I own risk assessment" is not a row; the row is *"I own the CarePilot risk register at `risk-register/2025/CarePilot.yaml`, updated continuously and reviewed quarterly."* Every row needs the artifact path.
- **Consumed rows with no fallback.** Chapter 06 and the exercise prompt both require it. "The analyst will produce it" with no fallback is the classic shape that falls on the L25 role when the analyst is unavailable.
- **Escalation rows with no trigger.** "I escalate to L50" with no trigger is silent escalation and defeats the contract's purpose.
- **Pretending levels are staffed when they're not.** Hat-stacking is normal; pretending the stack does not exist is the finding. Name the stack in the staffing map and file the role-boundary risk.
- **Sign-off written as a disclaimer instead of a commitment.** The executive is signing commitment to §2–§7. Make the commitment explicit; do not write defensive language.

## 6. References

- **Chapter 06 of this module** — `lessons/mod-101-ai-risk-frameworks-and-position/06-role-scope-on-the-level-ladder.md` — the ladder, the four-column deferral contract excerpt, the "what you own / consume / hand-off / escalate" shape, and the anti-pattern catalogue.
- **Exercise 01, 02, 03, 04 reference solutions** (same repository) — the artifact paths cited throughout this contract.
- **NIST AI RMF 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — the MAP / MEASURE / MANAGE function names used to identify which obligations the L25 role owns vs. co-owns.
- **ISO/IEC 42001:2023.** Overview: `https://www.iso.org/standard/81230.html` — the clause citations in §2.
- **Regulation (EU) 2024/1689 — European AI Act.** `https://eur-lex.europa.eu/eli/reg/2024/1689/oj` — Articles cited in §2.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — framework pointer for mod-104/105 obligations.
- **MITRE ATLAS.** `https://atlas.mitre.org/` — framework pointer for mod-104 obligations.
- Exercise prompt — `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-05-role-scope-and-deferral-contract.md`.

<!-- No external individual or team is identified by name; "Firm X", "Firm Y", and role labels are used instead, consistent with the exercise prompt's advice to abstract to roles and teams. -->

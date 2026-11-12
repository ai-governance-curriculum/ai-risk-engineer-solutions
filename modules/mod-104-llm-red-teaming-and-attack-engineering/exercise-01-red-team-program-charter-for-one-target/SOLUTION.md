# Exercise 01 — Red-Team Program Charter For One Target (CarePilot) — Reference Solution

## 1. Solution overview

This reference walks a worked **one-page red-team program charter** for the module's running system, **CarePilot** — a hospital-triage support RAG assistant for a mid-sized regional health system, with clinician human-in-the-loop (HITL) sign-off on every recommendation, internal network only, SSO-authenticated, no public endpoint. The deliverable is a single markdown page (approximately one printed page; ~100–120 lines of prose in the charter block itself) that a release-assurance reviewer can read, sign, and hand to the next person running an attack.

Four design decisions drive the whole charter, and the reviewer should see them before reading the sections:

- **Charter, not pen-test report.** A one-off pen-test confirms a snapshot. CarePilot's corpus changes, the model vendor patches behavior, the retriever index is re-built on protocol updates, and the tool-call surface may grow. Each of those events can introduce (or regress) a prompt-injection, insecure-output-handling, or excessive-agency finding. The charter is the *repeatable program* — a reusable suite, a stable ASR denominator, and a cadence — that catches drift. This posture is consistent with NIST AI RMF 1.0's MEASURE function and the Generative AI Profile's emphasis on continuous measurement of GAI risks (NIST AI 600-1).
- **Synthetic PHI only.** HIPAA minimum-necessary and minimisation principles mean the red-team cannot use real patient data for adversarial prompts or retriever-poisoning payloads, even in staging. Synthetic PHI (schema-faithful, demographically plausible, generated under a dedicated data-contract) is the only acceptable test fuel. Transcripts are themselves sensitive — they contain the attack strings that work — and are stored encrypted in a dedicated channel with access gated to the red-team roster.
- **Staging only, never production.** Blast-radius is the reason. A successful excessive-agency or insecure-plugin-design exploit on the production tool-call interface could place a real order against a real patient record. Staging has the same model + retriever + tool-call code paths but a synthetic corpus and a mocked EHR; it is the only safe target.
- **Cross-walked to OWASP + MITRE ATLAS.** Each objective in the charter names an OWASP Top 10 for LLM Applications risk (prompt injection, sensitive information disclosure, insecure output handling, excessive agency, insecure plugin design, overreliance, training data poisoning, model DoS, supply chain, model theft) and, where applicable, an OWASP Machine Learning Security Top 10 entry and a MITRE ATLAS tactic/technique. The cross-walk is what makes every red-team finding traceable *back to the CarePilot risk register* (`risk-register/2025/CarePilot.yaml`) and *forward to* mod-109 shadow-eval regression fixtures and mod-111 incident tickets.

Thresholds (attack-success-rate floors per harm class, severity-to-release mapping) are **cited** here and **set** in the risk register. The charter does not re-define them; the risk register is the single source of truth per mod-101's discipline.

## 2. Worked answer or implementation — `docs/red-team/CarePilot/charter.md`

The file below is the submission. A reviewer should be able to lift this block, change the system name and the register path, and have a working charter for a different target.

````markdown
---
system: CarePilot (hospital triage-support RAG assistant, HITL)
charter_type: red-team program (repeatable), not one-shot pen-test
author: ai-risk-engineer (L25)
signed_off_by: ai-evaluation-engineer / release-assurance (L35)
consulted: security-learning / ai-infra-security (L35) — judge supply chain
risk_register: risk-register/2025/CarePilot.yaml
cross_walk:
  - OWASP Top 10 for LLM Applications
  - OWASP Machine Learning Security Top 10
  - MITRE ATLAS
status: draft-for-signoff
version: 0.1.0
---

# CarePilot — Red-Team Program Charter

## Why a charter, not a one-off pen-test

A one-off pen-test confirms CarePilot's behavior at a single commit, with a single retriever index, against a single model snapshot. CarePilot changes along three axes that a one-off cannot cover:

1. **Corpus drift.** Triage protocols and ED handbook sections are re-ingested on a quarterly cadence (and ad-hoc on urgent updates). Each ingest is a fresh opportunity to introduce an **indirect prompt injection** via a poisoned source or a formatting trick that survives chunking.
2. **Model drift.** The underlying LLM is versioned by the vendor; a point release can change refusal behavior, instruction-following, or tool-call reliability.
3. **Surface drift.** The tool-call interface may grow (e.g., a new "look up admission history" tool). Each new tool is a new **excessive agency** and **insecure plugin design** surface.

The charter is the repeatable program — a versioned suite, a stable ASR denominator, a cadence, and an escalation path — that catches those drifts. Findings become **regression fixtures** so the same attack cannot silently return after a corpus or model change. This posture aligns with NIST AI RMF 1.0's MEASURE function (continuous, repeated measurement across the lifecycle) and the NIST GenAI Profile (NIST AI 600-1).

## (a) Scope

**System under test.** CarePilot — hospital-triage support RAG assistant. Clinician HITL sign-off on every recommendation.

**Deployment envelope (staging).** Internal network only; SSO-authenticated red-team roster; mirrors production topology; isolated from production monitoring and paging.

**Versions pinned per engagement.**
- Model: `<vendor>/<model-id>@<version-pin>` (recorded at engagement start; re-pinned on each cadence event).
- Retriever: `<embedding-model>@<version>` + index build hash.
- Tool-call manifest: `tools/manifest.yaml@<commit-sha>`.
- Red-team suite: `red-team/suite.yaml` — config hash is one of the required deliverables (see section d).

**Data in play.**
- Synthetic PHI only (schema-faithful, generated under `data/synthetic/CarePilot/contract.md`).
- Synthetic corpus mirroring the triage protocol + ED handbook structure; real clinical content is replaced with plausible-but-fictional equivalents. No real patient data, no real clinician notes, no real admission records.
- Mocked EHR behind the tool-call interface; it accepts the production API shape but writes to a scratch store.

**Surfaces in scope.**
1. **Chat UI** — direct prompt injection, overreliance-inducing wording, sensitive-info-disclosure probes, model-DoS style (long-context, pathological-token) inputs.
2. **Retriever corpus ingestion** — indirect prompt injection via poisoned documents, formatting / markdown / HTML-comment smuggling, embedding-collision attacks on retrieval ranking.
3. **Tool-call interface** — excessive agency, insecure plugin design, insecure output handling (where a tool's return value is rendered back without sanitisation).
4. **Judge / eval pipeline** (consulted with L35 security-learning) — supply-chain exposure of any judge models or scoring prompts used by the red-team suite itself.

**Out of scope (explicit).**
- Production tenant, production EHR, production paging.
- Real patient data in any form (including de-identified exports — too risky under HIPAA re-identification concerns for this use case).
- Social-engineering of hospital staff. (Handled by the enterprise security team under a separate authorisation.)
- Physical / network-layer attacks on the hosting infrastructure. (Owned by `ai-infra-security` L35 under a separate program.)
- Model-weight extraction attacks against vendor-hosted endpoints when such tests would violate the vendor terms of service — legal review required before any such probe is enabled.

## (b) Objectives tied to the risk register

The red team attempts to elicit harm in the following classes. Each row names the OWASP Top 10 for LLM Applications risk, the OWASP ML Top 10 entry where applicable, and the MITRE ATLAS tactic(s) it cross-walks to. Thresholds are *cited* here and *set* in `risk-register/2025/CarePilot.yaml`.

| # | Harm class (OWASP LLM Top 10) | OWASP ML Top 10 cross-walk | MITRE ATLAS tactic(s) | Risk-register entry |
|---|---|---|---|---|
| 1 | Prompt injection (direct) | — | ATLAS: ML Attack Staging / LLM Prompt Injection | `carepilot.risk.prompt-injection.direct` |
| 2 | Prompt injection (indirect, via retriever) | ML02 Data Poisoning (overlap) | ATLAS: Initial Access via LLM-accessible resources | `carepilot.risk.prompt-injection.indirect` |
| 3 | Sensitive information disclosure | — | ATLAS: Exfiltration via LLM output | `carepilot.risk.info-disclosure` |
| 4 | Insecure output handling | — | ATLAS: Defense Evasion through unsanitised output | `carepilot.risk.insecure-output` |
| 5 | Excessive agency | — | ATLAS: Impact via overly-privileged tool-use | `carepilot.risk.excessive-agency` |
| 6 | Insecure plugin design | — | ATLAS: Execution via tool-call | `carepilot.risk.insecure-plugin` |
| 7 | Overreliance | — | ATLAS: Impact via plausible-but-wrong output | `carepilot.risk.overreliance` |
| 8 | Training data poisoning / corpus poisoning | ML02 Data Poisoning | ATLAS: Resource Development via corpus manipulation | `carepilot.risk.corpus-poisoning` |
| 9 | Model DoS (long-context, pathological inputs) | ML04 Membership Inference (unrelated); covered under LLM Top 10 | ATLAS: Impact via resource exhaustion | `carepilot.risk.model-dos` |
| 10 | Supply chain (model, retriever, judge, tool-call libs) | ML06 ML Supply Chain Attacks | ATLAS: Supply Chain Compromise | `carepilot.risk.supply-chain` |
| 11 | Model theft | ML05 Model Theft | ATLAS: Exfiltration of model artefacts | `carepilot.risk.model-theft` |

Patient-safety–adjacent classes (1, 2, 3, 5, 7, 8) are the priority set for every engagement. Classes 9, 10, 11 are covered on the quarterly deep-dive cadence and whenever the dependency graph materially changes.

<!-- verify-at-pin: confirm final ML Top 10 numbering in the version pinned by the program; the OWASP ML Top 10 is maintained and may renumber between engagements. The authority-of-record is https://owasp.org/www-project-machine-learning-security-top-10/ -->

## (c) Rules of engagement

**Who may run attacks.** A named red-team roster, maintained by L35 release-assurance, with each member having completed the internal red-team safety briefing and signed the data-handling acknowledgement. No ad-hoc participation.

**Where.** Staging environment only. Production endpoints are blocked at the network layer for the red-team roster's service accounts. Any attempt to run a red-team suite against a production URL fails closed.

**Data hygiene.**
- Synthetic PHI only; the synthetic generator contract is reviewed by the AI risk council on each major release.
- Red-team *transcripts* (prompts + responses + tool-call traces) are themselves sensitive — they contain the attack strings that succeeded — and are stored encrypted in a dedicated channel (`storage://red-team/CarePilot/<engagement-id>/`), access gated to the red-team roster + L35 release-assurance + security-learning on request.
- Transcripts are **not** forwarded to the production observability stack. This is a hard boundary: red-team findings travel through the red-team report, not through production logs.

**Legal review.** Any new attack family that touches vendor terms of service (e.g., model-extraction style probes against a hosted endpoint) requires written legal sign-off before enablement. The default posture is conservative — if in doubt, do not run.

**Separation from production monitoring.** The staging environment emits its own telemetry stream, tagged and routed separately from production. Red-team runs do not page production on-call; they page the red-team channel.

**Retention policy.**
- Transcripts: retained 24 months, then rotated out per the data-retention policy set by `ai-governance-analyst` (L15) in the governance register.
- Reproducer prompts promoted to regression fixtures: retained for the life of the system (they are the regression contract).
- Signed red-team reports: retained per the legal-hold policy of the hospital.

<!-- verify-at-pin: confirm the 24-month figure against the hospital's final data-retention policy; the governance register is the source of truth, not this charter. -->

## (d) Deliverable set

Every engagement produces, at minimum:

1. **Red-team suite config hash.** The exact `red-team/suite.yaml` (prompt sets, attack families enabled, judge configuration, seed) hashed and recorded in the engagement manifest. Without this, ASR is not comparable across runs.
2. **Attack-success-rate (ASR) per harm class.** Numerator = confirmed successful attacks in the class; denominator = attempts in the class as encoded by the suite. ASR is reported per surface (chat UI / retriever / tool-call) *and* aggregated.
3. **Confirmed-finding set.** For each finding: a reproducer prompt (or prompt chain, or poisoned-document payload), the model + retriever + tool-manifest versions at which it reproduces, the harm class, the severity (S1–S4 per mod-101's patient-safety scale), and a narrative of observed impact.
4. **Regression fixtures.** Each confirmed finding ships as a fixture into the shadow-eval suite (mod-109) so the same attack is run on every subsequent release. A finding without a fixture is not considered closed.
5. **Signed red-team report.** One-document summary: scope delta vs. previous engagement, ASR table, confirmed findings, fixtures added, open items, and sign-offs. L25 authors; L35 release-assurance signs.

Tooling. The program uses open-source attack frameworks where they fit: **garak** (NVIDIA/garak) for breadth probes against the chat surface; **PyRIT** (Azure/PyRIT) for orchestrated multi-turn and jailbreak families; **AgentDojo** (ethz-spylab/agentdojo) and **InjecAgent** (uiuc-kang-lab/InjecAgent) for the tool-call and indirect-injection surfaces; **Inspect** (UKGovernmentBEIS/inspect_ai) for authored red-team evals that need to live alongside the capability-eval suite. Tool choice per surface is documented in the suite config, not in this charter.

## (e) Acceptance thresholds and escalation

Thresholds are **set** in `risk-register/2025/CarePilot.yaml` under the `security_envelope` block and **cited** here. The charter ties engagement outcomes to release-gate behavior:

- **ASR ≤ register-defined floor per harm class.** For each harm class in section (b), the register defines a maximum tolerated ASR (e.g., `prompt-injection.direct.asr_max: <value>`). An engagement that comes in over the floor on any patient-safety-adjacent class (1, 2, 3, 5, 7, 8) blocks release.
- **Severity-to-release mapping (mod-101 scale).**
  - **S1** (direct patient-safety harm demonstrated, e.g., tool-call causes an incorrect order against the mocked EHR) → **halts release**. Fix + full re-run before gate can be re-opened. Opens a mod-111 incident ticket with same-day SLA.
  - **S2** (strong plausibility of patient-safety harm, e.g., confirmed indirect injection that would mislead a HITL clinician) → **halts release**. Fix + regression fixture + partial re-run.
  - **S3** (meaningful risk, no direct patient-safety path, e.g., sensitive-info disclosure of a synthetic PHI field in a benign-looking query) → **opens mod-111 incident ticket** with the SLA defined in the register (does not necessarily halt release; decision is release-assurance's).
  - **S4** (low-impact, but repeatable, e.g., model-DoS via pathological input causing a timeout but not a wrong answer) → **opens mod-111 incident ticket** with the standard SLA; ships as a backlog item.

All severities, regardless of release impact, ship a **regression fixture**.

## (f) Cadence

The program runs on three triggers:

1. **Pre-release gate.** Every CarePilot release runs the current red-team suite in staging. ASR table + any S1/S2 finding feeds the release-assurance gate.
2. **Corpus-change gate.** Any corpus re-ingest — quarterly scheduled or urgent — triggers at least the indirect-injection and corpus-poisoning families before the new index is promoted to production.
3. **Quarterly deep-dive.** A dedicated engagement each quarter that explores new attack families, re-tunes the suite, refreshes the judge configuration, and covers the lower-frequency classes (9, 10, 11).

The program is **coordinated with shadow eval (mod-109).** Regression fixtures from red-team findings live in the shadow-eval suite; the shadow-eval cadence picks them up on every release. Red-team ASR deltas are reported alongside shadow-eval capability deltas in the release readiness review.

## (g) Roles

- **L25 `ai-risk-engineer` — author.** Authors the charter, maintains the suite config, drives engagements, writes the red-team report.
- **L35 `ai-evaluation-engineer` / release-assurance — signs.** Signs the charter version in force; signs each engagement's final report; owns the release gate decision.
- **L35 `security-learning` / `ai-infra-security` — consulted.** Consulted on the judge supply chain (which judge models, where they run, how prompt-injectable *they* are), on any attack family that touches credentialed systems, and on the retention / encryption posture for transcripts.
- **L15 `ai-governance-analyst` — informed.** Informed on charter changes and retention policy; owns the governance register that this charter cites.
- **L30 `ai-eval-engineer` — informed / collaborator.** Maintains the shadow-eval surface that regression fixtures flow into.
- **L25 `ai-infra-mlops` — collaborator.** Maintains the staging environment the red team operates in.

## Sign-off

- Author: `<ai-risk-engineer>` — `<date>`
- Release assurance: `<ai-evaluation-engineer-L35>` — `<date>`
- Security consulted: `<security-learning-L35>` — `<date>`
- Governance informed: `<ai-governance-analyst-L15>` — `<date>`
````

## 3. Validation steps

A reviewer can validate this charter in under fifteen minutes by walking the following numbered steps.

1. **Scope diff.** Read section (a). Confirm the deployment envelope, the four surfaces, the versions-pinned list, and the explicit out-of-scope list. If any item in the real engagement is not covered by one of those four categories, the scope is incomplete.
2. **Risk-register traceability.** For each row in section (b), open `risk-register/2025/CarePilot.yaml` and confirm the referenced entry (`carepilot.risk.<slug>`) actually exists and has an `asr_max` (or equivalent) field. A charter that cites non-existent register entries fails this step.
3. **Cross-walk sanity.** For each OWASP LLM Top 10 name used in section (b), confirm the spelling matches the current OWASP Top 10 for LLM Applications list by name. Confirm every MITRE ATLAS tactic named is a real ATLAS tactic (https://atlas.mitre.org/). Flag any mismatch.
4. **Rules-of-engagement completeness.** Section (c) must answer *who*, *where*, *what data*, *what legal*, *what telemetry*, *what retention*. Six questions, six answers. Any missing answer blocks sign-off.
5. **Deliverable set is auditable.** Section (d) must name the five deliverables in a form a release-assurance reviewer could check without the author in the room: suite config hash is a file hash, ASR is a number per class, confirmed findings have reproducer prompts, regression fixtures have shadow-eval paths, the report is a signed PDF or signed markdown. If any deliverable is described in prose only, fail the step.
6. **Threshold-to-severity ladder is internally consistent.** Section (e) must map every severity S1–S4 to a release behavior (halt / ticket) *and* every ticketing route must point at mod-111 with an SLA citation. Re-read and check there are no orphan severities.
7. **Cadence covers all three drift axes.** Section (f) must name a trigger for corpus drift, a trigger for release-time drift (model + code), and a periodic deep-dive. If any axis is missing, the program will miss drift in that axis.
8. **Roles are RACI-shaped.** Section (g) must name an author, a sign-off, a consulted party, and informed parties. If any of the four RACI slots is empty, fail the step.

## 4. Rubric or review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| 1 | Charter framing | Section 1 / overview states explicitly that this is a repeatable program, not a one-off pen-test, and names the three drift axes (corpus / model / surface) | Charter reads as a one-off engagement document, or drift axes are not identified |
| 2 | Scope completeness | Section (a) names system, deployment envelope, pinned versions, data, four surfaces, and an explicit out-of-scope list | Any of those six items missing, or out-of-scope list is absent / implicit |
| 3 | OWASP LLM Top 10 coverage | Section (b) names each in-scope harm class using the current OWASP Top 10 for LLM Applications terminology | Harm classes invented or named inconsistently with OWASP |
| 4 | OWASP ML Top 10 cross-walk | Where applicable (poisoning, supply chain, model theft) the ML Top 10 entry is named | ML Top 10 mentioned nowhere, or forced in where it does not fit |
| 5 | MITRE ATLAS cross-walk | Each harm class has an ATLAS tactic or technique cited | ATLAS not referenced or referenced with invented IDs |
| 6 | Register traceability | Every objective row points at a real `carepilot.risk.<slug>` entry in the risk register | Charter sets thresholds inline instead of citing the register |
| 7 | Rules of engagement | Section (c) answers who, where, data hygiene, legal, telemetry separation, retention | Any one of the six missing |
| 8 | Deliverable auditability | Section (d) names the five deliverables in machine-checkable form (hash, number, reproducer, fixture path, signed report) | Deliverables described only as prose |
| 9 | Severity ladder | Section (e) maps S1–S4 to release behavior and mod-111 ticketing | Ladder omits a severity or decouples severity from release behavior |
| 10 | Cadence | Pre-release + corpus-change + quarterly deep-dive all present; coordination with mod-109 shadow eval named | Only one trigger named, or shadow-eval coordination missing |
| 11 | Roles | Author (L25), sign-off (L35 release-assurance), consulted (L35 security-learning) all named with levels | Levels absent, or sign-off is self-signed by L25 |
| 12 | One-page discipline | Charter fits on roughly one printed page / is terse enough for a release gate to read in under five minutes | Charter sprawls into an eval design document |

**Scoring bands.**

- **Pass** — all 12 criteria pass, no `<!-- verify-at-pin: -->` tags on load-bearing claims.
- **Pass with revisions** — up to two criteria fail on form (e.g., missing level on a role, a terminology drift against OWASP) *or* up to two `<!-- verify-at-pin: -->` tags on non-load-bearing items. Author revises; reviewer re-checks only the flagged rows.
- **Fail** — three or more criteria fail, or any of criteria 2, 6, 7, 8, 9 fails. Scope, register traceability, rules of engagement, deliverables, and severity ladder are load-bearing; a charter that fails any of them is not safe to sign.

## 5. Common mistakes

- **Treating the charter as a pen-test statement of work.** The charter must survive multiple engagements. If section (a) names a specific set of commits instead of a *version-pinning policy*, the charter will go stale after the first release.
- **Setting thresholds inline instead of citing the risk register.** The register is the single source of truth (per mod-101). A charter that hard-codes an `asr_max: 0.05` will drift out of sync with the register the first time the register is updated; the charter must cite the register path and let the number live there.
- **Omitting the retriever and tool-call surfaces.** Only testing the chat UI misses indirect prompt injection (the retriever) and excessive agency / insecure plugin design (the tool-call interface). All three surfaces must be in scope or the program will systematically under-report.
- **Using real patient data "just in staging."** Synthetic PHI is non-negotiable under HIPAA minimisation principles. "Staging only" is not a defense if the data is real; the posture must be synthetic-only, enforced by the data contract, not by convention.
- **Routing red-team transcripts through production observability.** Transcripts contain working attack strings. Shipping them into the production log stream (where more people have access) widens the blast radius of a successful attack. Dedicated encrypted channel, roster-gated.
- **Not shipping regression fixtures.** A finding without a fixture is a finding that will come back. The shadow-eval suite (mod-109) is where fixtures live; coordination with mod-109 is a charter obligation, not a nice-to-have.
- **Collapsing severities.** Treating all findings as "high / medium / low" instead of the mod-101 S1–S4 patient-safety scale loses the release-gate signal. S1 and S2 halt; S3 and S4 ticket. Collapsing into a three-level scale breaks the mapping.
- **Self-signing.** L25 authors; L35 release-assurance signs. A charter signed by the author alone does not clear the segregation-of-duties requirement that the release gate depends on.

## 6. References

- NIST AI Risk Management Framework 1.0 (AI 100-1) — https://www.nist.gov/itl/ai-risk-management-framework
- NIST AI RMF Generative AI Profile (NIST AI 600-1) — https://www.nist.gov/itl/ai-risk-management-framework
- OWASP Top 10 for Large Language Model Applications — referenced by name; authoritative list hosted by OWASP
- OWASP Machine Learning Security Top 10 — https://owasp.org/www-project-machine-learning-security-top-10/
- MITRE ATLAS — https://atlas.mitre.org/
- garak (red-team probes) — https://github.com/leondz/garak (now maintained at NVIDIA/garak)
- PyRIT (red-team orchestrator) — https://github.com/Azure/PyRIT
- Inspect (eval authoring, used for authored red-team evals) — https://github.com/UKGovernmentBEIS/inspect_ai
- AgentDojo (tool-call / indirect-injection benchmark) — https://github.com/ethz-spylab/agentdojo
- InjecAgent (indirect prompt-injection benchmark) — https://github.com/uiuc-kang-lab/InjecAgent

*Practitioner implementation example (not an authority):* VeriSwarm — referenced only as an example of how teams operationalise a repeatable red-team program; its claims are not cited as evidence in this charter.

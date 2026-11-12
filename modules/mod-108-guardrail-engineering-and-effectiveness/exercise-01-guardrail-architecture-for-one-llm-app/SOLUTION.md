# Exercise 01 — Guardrail Architecture for One LLM Application (CarePilot) — Reference Solution

## 1. Solution overview

This exercise produces the **design artefact** every subsequent mod-108 exercise maps to. It is a design, not code — the slot set, the coverage table binding slots to external frameworks, the composition walk-through of three harm scenarios, and the handoff shape the mod-103 register will later consume (after exercise 05 measurement).

The running system is **CarePilot** — a hospital-triage support RAG assistant with clinician human-in-the-loop (HITL) sign-off on every recommendation, internal network only, SSO-authenticated, no public endpoint. CarePilot is the thread system used across mods 101–107 in this repository, so the slot set, the mod-102 harm classes, and the mod-103 register row IDs cited below are already shared vocabulary.

Four design decisions drive the whole deliverable:

- **Harm model first, slots second.** The slots are instantiated to *address* harm-model rows. Starting from the slot list produces an architecture that looks complete but silently leaves scenarios uncovered. The scenario-slot matrix (§2.5) is the honest coverage read.
- **Five slots, with absences stated.** Chapter 01's five-slot set is instantiated in full for CarePilot. Every slot has a stated responsibility, a stated failure mode, and a candidate mechanism. A slot CarePilot does not need (hypothetically — e.g., a pure-chat system without retrieval) would be explicitly marked absent with a cross-reference to how the harm classes are handled elsewhere.
- **Tool-invocation policy is deterministic, not a classifier.** Chapter 02's "when rule-based fits" framing applies. Slot 3 is **always** an allow-list plus argument schema in custom middleware; "Llama Guard on the tool call" is a design bug.
- **Design commitments, not shipped code.** Mechanism cells carry candidate names (NeMo Guardrails input rail + Llama Guard action, Guardrails AI output guard + Presidio validator, etc.) but no residual-risk credit — credit requires measurement, which exercise 05 produces.

The six artefacts below live under `risk/mod-108/carepilot/architecture/` in the running-system workspace and are indexed from `mod-108-ex01-slots.md`.

## 2. Implementation — worked answer across the six architecture artefacts

### 2.1 Architecture diagram (`slots.md`)

ASCII sketch — a rendered PNG lives alongside in the real deliverable; the sketch is the communication aid, not the specification. Request flows left-to-right, response right-to-left.

```
                                     ┌─ Retrieval filter (slot 2)
                                     │  NeMo GR retrieval rail + ShieldGemma
                                     │  + Presidio retrieval scan (exercise 04)
                                     v
user ─► Input filter (slot 1) ─► retriever ─► model ─► Output validator (slot 4) ─► Refusal shaper (slot 5) ─► caller
        NeMo GR input rail                                 Guardrails AI guard +      Templated refusal content;
        + Llama Guard action                               Llama Guard + ShieldGemma  no classifier here.
        + signature denylist                               + Presidio output scan
                                     ▲
                                     │
                                     │ Tool-invocation policy (slot 3)
                                     │ Custom FastAPI middleware:
                                     │ allow-list, argument schema, per-tool quota.
```

### 2.2 Slot specification (`slot-spec.md`)

One section per slot. Responsibility (one sentence), failure mode (one sentence from chapter 01 or application-specific), chosen mechanism candidate (one line), version-pinning notes (which library / model / vendor version this will bind to when the mechanism is shipped in exercises 02–04).

- **Slot 1 — Input filter.**
  - *Responsibility.* Block or transform caller inputs that match known-bad signatures (prompt-injection templates, jailbreak scaffolds) or that the input-side classifier flags as unsafe before CarePilot's main model runs.
  - *Failure mode.* Fail-open on classifier error — if Llama Guard errors, the input must be rejected, not passed through. (Chapter 01 §"Silent single-point-of-failure anti-pattern", case 2.)
  - *Mechanism candidate.* NeMo Guardrails input rail; signature denylist in Colang; Llama Guard action behind the rail.
  - *Version pins (bound in exercise 02–03).* NeMo Guardrails pinned; Colang pinned; Llama Guard model version and taxonomy version pinned.

- **Slot 2 — Retrieval filter.**
  - *Responsibility.* Score and sanitise each retrieved chunk before it is assembled into the model prompt — catches indirect prompt injection (OWASP LLM01), sensitive-information disclosure via retrieved PII (OWASP LLM06-era category; see §2.3 version note), and poisoned chunks surfaced by embedding-ranking attacks.
  - *Failure mode.* Under-scoping — a retrieval filter that only looks for injection markers misses PII and poisoned-content cases.
  - *Mechanism candidate.* NeMo Guardrails retrieval rail with (a) ShieldGemma action scoring per-policy probabilities (`injection-likely`, `unsafe`, `sensitive-info`), (b) a fast Presidio analyser pass (exercise 04). Chunks failing any threshold drop from assembly or are transformed.
  - *Version pins (bound in exercise 03–04).* ShieldGemma checkpoint + policy set; Presidio analyser + anonymiser; spaCy model.

- **Slot 3 — Tool-invocation policy.**
  - *Responsibility.* Enforce an allow-list of destinations, an argument schema on every tool call, and a per-tool rate / quota before any tool action can execute (mocked EHR lookup, order-draft submission for clinician review).
  - *Failure mode.* Debug bypass — an `X-Debug: true` or test-tenant flag that silently disables the policy and is not logged / rate-limited / network-gated.
  - *Mechanism candidate.* Custom FastAPI middleware (a dependency wrapping the tool-executor function); allow-list and argument-schema validation in code owned by the CarePilot team; per-tool quota and rate limit in the same middleware.
  - *Version pins (bound in exercise 02).* Middleware dependencies pinned; the git commit hash of the middleware becomes a `control_version` value in exercise 05's measurement rows.

- **Slot 4 — Output validator.**
  - *Responsibility.* Score the model's response against two independent classifiers (safety taxonomy + per-policy probability) and against a Presidio scan (per-field policy on the structured `answer` / `citations` JSON), and reject or sanitise before the response moves to slot 5.
  - *Failure mode.* Two-variants-of-the-same-model-at-two-thresholds masquerading as defence-in-depth. Chapter 03 §"Composing classifier + rule + retrieval" is explicit: two thresholds on one classifier is one control, not two.
  - *Mechanism candidate.* Guardrails AI guard on the response body; per-field validators; Llama Guard + ShieldGemma both must-pass (different training data); Presidio output scan as a Guardrails AI validator.
  - *Version pins (bound in exercise 02–04).* Guardrails AI version; every Hub validator version; Llama Guard + ShieldGemma checkpoints; Presidio version.

- **Slot 5 — Safe-completion / refusal shaping.**
  - *Responsibility.* Produce the actual bytes the caller sees when any upstream slot rejects — a refusal template per named category that does not restate the disallowed content and does not leak internal policy.
  - *Failure mode.* "Policy-leak refusal" — a refusal that quotes back the triggering phrase or names the specific rule that fired, feeding an attacker a search gradient. (Chapter 01 §"The slot set".)
  - *Mechanism candidate.* NeMo Guardrails output rail with templated refusal content; no classifier; refusal strings reviewed against a chapter-01 refusal-string audit (stretch goal).
  - *Version pins (bound in exercise 02).* NeMo Guardrails config version; refusal-template file git commit hash.

CarePilot has retrieval and limited tool calls (both via the HITL-reviewed order path), so slots 2 and 3 are both present. No slot is absent; the "why absent" paragraph is a null for this system.

### 2.3 Coverage table (`coverage.md`)

Framework versions pinned at the top of the artefact so the reader never has to guess:

- **OWASP Top 10 for Large Language Model Applications** — 2025 edition identifiers used throughout (`LLM01:2025 Prompt Injection`, `LLM02:2025 Sensitive Information Disclosure`, etc.). Register the exact revision with its access date at submission time. <!-- verify-citation: confirm the current 2025-edition category codes and titles against the published list at `https://genai.owasp.org/llm-top-10/` on the day the artefact is signed; the codes shifted from the 2023 edition (e.g., Sensitive Information Disclosure moved category). -->
- **MITRE ATLAS** — matrix version pinned at authoring time with access date; techniques cited by ATLAS ID (`AML.T….`). <!-- verify-citation: pin matrix version and tactic/technique IDs against `https://atlas.mitre.org/` on the day of signing. -->
- **Google SAIF** — pillar set pinned (six-pillar framing in use at authoring). <!-- verify-citation: pin the SAIF pillar names and count against `https://safety.google/cybersecurity-advancements/saif/` on the day of signing. -->
- **NIST AI RMF** — version 1.0 (AI 100-1, January 2023), with the Generative AI Profile (AI 600-1) used for the MEASURE / MANAGE tie-back. (`https://www.nist.gov/itl/ai-risk-management-framework`.)

| Slot | OWASP LLM Top 10 (2025) | MITRE ATLAS tactic(s) | Google SAIF pillar(s) | Chosen mechanism (candidate) | Named failure mode |
|---|---|---|---|---|---|
| 1 Input filter | LLM01 Prompt Injection; LLM05 Improper Output Handling (partial — upstream shaping); — for LLM02/LLM06/LLM08/LLM10 at this slot | Initial Access (ATLAS AML.TA.—); ML Model Access <!-- verify-citation --> | Detection & Response; Model & Data Security | NeMo GR input rail + Llama Guard action + signature denylist | Fail-open on classifier error |
| 2 Retrieval filter | LLM01 Prompt Injection (indirect); LLM02 Sensitive Information Disclosure (retrieved PII); LLM08 Vector / Embedding Weaknesses | ML Supply Chain Compromise; Discovery; Initial Access <!-- verify-citation --> | Data Governance; Detection & Response | NeMo GR retrieval rail + ShieldGemma + Presidio retrieval scan | Under-scoping (single-category check) |
| 3 Tool-invocation policy | LLM06 Excessive Agency (2025 renumbering — formerly LLM08:2023); LLM05 Improper Output Handling (tool-arg validation); — for LLM01/LLM02/LLM04/LLM10 at this slot | ML-Enabled Product or Service abuse; Impact <!-- verify-citation --> | Secure by Default; Automation | Custom FastAPI middleware (allow-list + argument schema + quota) | Debug bypass |
| 4 Output validator | LLM02 Sensitive Information Disclosure (output PII); LLM05 Improper Output Handling; LLM09 Misinformation (partial — safety-side only) | Model Evasion; Exfiltration <!-- verify-citation --> | Detection & Response; Model & Data Security | Guardrails AI guard + Llama Guard + ShieldGemma + Presidio output scan | Two-variants-of-the-same-model masquerading as defence-in-depth |
| 5 Refusal shaping | LLM09 Misinformation (partial — refusal content hygiene); — for the majority of categories | Defensive Evasion (defender-side) <!-- verify-citation --> | Human-Centric Transparency | NeMo GR output rail + templated refusal content | Policy-leak refusal that restates the disallowed content |

Empty cells are written as "—" so the reader knows the gap was considered (chapter 01 §"Slot × attack-class coverage table"). Every `verify-citation` marker blocks auto-merge until the exact category codes, ATLAS IDs, and SAIF pillar names are personally confirmed on the official sources at signing time.

### 2.4 Composition walk-through (`composition.md`)

The three scenarios are chosen from CarePilot's mod-102 harm model. Row IDs below are the mod-102-level identifiers the running system carries; see `risk-register/2025/CarePilot.yaml` for the current harm-row set.

#### Scenario A — indirect prompt injection via a poisoned triage protocol (`harm-ipi-triage-protocol`, two slots address)

A poisoned PDF entering CarePilot's retrieval corpus contains instructions that attempt to flip the model into recommending an unreviewed drug dosage. Slots 2 and 4 both address the scenario.

- **Slot 2 defeated alone.** The poisoned chunk survives retrieval. The output-side Llama Guard + ShieldGemma + Presidio validators still have a chance — the model may still synthesise a response that one of the output classifiers flags. Residual moves from "measured (slot 2 + slot 4) residual" to "measured slot-4-only residual" — not inherent.
- **Slot 4 defeated alone.** The retrieval filter catches and drops the poisoned chunk; the model never sees the injected instructions. Residual stays close to the slot-2-only number for the rest of the scenario coverage.
- **Both defeated.** The harm lands on the clinician HITL review path (last-line defence outside the slot set) — residual degrades toward inherent but not to inherent because HITL is a separate control. This is the shape chapter 01's defence-in-depth test wants: two independent slots, each with a meaningfully different residual-collapse profile, and a last-line human control.
- **No silent SPOF.** Slot 2 and slot 4 use different mechanisms (ShieldGemma retrieval policy scoring vs. Llama Guard + ShieldGemma + Presidio on output), so a bypass of one classifier family does not simultaneously defeat both.

#### Scenario B — PII exfiltration via model confabulation (`harm-pii-output`, one slot addresses in current design — gap filed)

A benign-looking user turn triggers the model to synthesise a plausible-looking patient name or medical record number that is not in the retrieved context. Slot 4's Presidio output scan is the single addressing slot (Llama Guard and ShieldGemma do not score for CarePilot-specific MRN format).

- **Current residual collapse.** If slot 4's Presidio pass is defeated (or has a recall hole on CarePilot-MRN), the residual collapses to inherent.
- **Treatment (added).** A second mechanism is added at slot 4: a cheap regex guard for the CarePilot-MRN format and a per-caller entitlement check on `PERSON` entities, both authored in Guardrails AI as a custom validator. Composition is now Presidio *or* regex-entitlement must flag; the two validators run in parallel and either rejects. The residual-collapse profile now looks like scenario A.
- **Why this is not "two thresholds on one model".** Presidio uses spaCy NER + regex context; the regex-entitlement validator is pure pattern + ACL. Different mechanisms, so the chapter 03 anti-pattern does not apply.

#### Scenario C — silent single-point-of-failure anti-pattern probe (`harm-tool-call-bypass`)

Scenario-C targets the tool-invocation policy. CarePilot exposes an `escalate_to_pager` tool (rate-limited; allow-list; argument schema). The chapter 01 anti-patterns most plausibly biting this slot:

- **Fail-open on error (case 2).** If the FastAPI middleware raises on a schema-validation error and the exception handler re-emits the tool call, the slot is bypassed silently. **Design discipline.** Middleware raises a typed `ToolPolicyRejection`; the request path catches it and returns a slot-5 refusal with a logged `request_id`. A unit test asserts that any middleware exception produces a refusal, not a pass-through.
- **Debug bypass (case 4).** If the middleware honours an `X-Debug: true` header for internal test callers without logging, the bypass is invisible to the mod-109 monitor. **Design discipline.** Debug flag is only read when the caller's IP sits inside an explicit allow-list, is logged to the trace with `debug_bypass=true`, is rate-limited independently, and raises a monitor alarm on every use.
- **Shared-model bypass (case 1) is not latent here** because the middleware does not call the classifier model at all.
- **Streaming bypass (case 3) is scenario-specific** — only relevant for streaming output; CarePilot's HITL design defers rendering, so slot-4 streaming-bypass is not latent at submission, but the stretch goal of §"The streaming path" is still worth writing if a streaming endpoint is added later.

The most plausible anti-pattern for CarePilot is **fail-open on error** at slot 3 — the discipline above is the fix, and the exercise-02 fixture harness is where the fix is proven.

### 2.5 Scenario-slot matrix (`scenario-slot-matrix.md`)

One row per harm-model scenario. Cells: `Y` the slot addresses the scenario; `N` it does not; `partial` with a one-sentence note if the slot catches a subset. The matrix below shows the pattern; populate every CarePilot mod-102 row in the real artefact.

| mod-102 scenario (ID) | Slot 1 Input | Slot 2 Retrieval | Slot 3 Tool policy | Slot 4 Output | Slot 5 Refusal shaping |
|---|---|---|---|---|---|
| `harm-direct-injection` | Y | partial (indirect only) | N | Y | partial (refusal content) |
| `harm-ipi-triage-protocol` | N | Y | N | Y | partial |
| `harm-pii-output` | N | partial (ingest-time) | N | Y (after treatment B) | partial |
| `harm-excessive-agency-order` | partial (shape-check on user turn) | N | Y | Y (tool-arg echo scan) | partial |
| `harm-model-dos-long-context` | Y (length guard in input rail) | N | N | N | N |
| `harm-sensitive-retrieval-disclosure` | N | Y | N | Y (per-caller entitlement) | partial |
| `harm-refusal-policy-leak` | N | N | N | N | Y |
| `harm-citation-fabrication` | N | partial (cite-source verification) | N | Y (Guardrails AI per-field URL check) | partial |

Any row whose entire slot vector is `N` is a **control-defeated row** for the register — flag it explicitly. In the matrix above, no row is entirely `N`; if the real system produces one, it is written here and referenced in the register handoff (§2.6).

### 2.6 Register handoff note (`register-handoff.md`)

For every scenario where the coverage table adds a new mechanism candidate, name which mod-103 register row's residual and control-defeated columns will be updated **after** exercise 05 measurement. Do not update the register yet — residual credit requires measurement.

| mod-103 register row | mod-102 scenario | New mechanism candidate | Measured by |
|---|---|---|---|
| `risks[prompt-injection-direct]` | `harm-direct-injection` | Slot 1 Llama Guard + signature denylist; slot 4 dual classifier + Presidio | exercise 05 measurement |
| `risks[indirect-prompt-injection]` | `harm-ipi-triage-protocol` | Slot 2 ShieldGemma + Presidio; slot 4 dual classifier | exercise 05 |
| `risks[pii-output]` | `harm-pii-output` | Slot 4 Presidio + CarePilot-MRN regex entitlement check | exercise 04 (initial calibration), exercise 05 (production measurement) |
| `risks[excessive-agency]` | `harm-excessive-agency-order` | Slot 3 middleware allow-list + arg schema + quota | exercise 02 fixture pass; exercise 05 production measurement |
| `risks[model-dos]` | `harm-model-dos-long-context` | Slot 1 length guard + rate limit | exercise 02 fixture pass; exercise 05 |
| `risks[retrieval-pii-exposure]` | `harm-sensitive-retrieval-disclosure` | Slot 2 Presidio retrieval scan + RBAC entitlement filter | exercise 04 calibration + exercise 05 |
| `risks[refusal-leak]` | `harm-refusal-policy-leak` | Slot 5 templated refusals (refusal-string audit — stretch) | exercise 02 fixture; manual review |
| `risks[citation-fabrication]` | `harm-citation-fabrication` | Slot 4 Guardrails AI URL-shape validator + per-field policy | exercise 02 fixture; exercise 05 |

**Rule.** Residual credit is a measurement, not a design claim. The handoff note queues the rows; exercise 05 writes the measured number into them. Any row whose measured residual sits above the mod-103 appetite is flagged for mod-112 escalation.

## 3. Validation steps

1. **All six files exist.** `slots.md`, `slot-spec.md`, `coverage.md`, `composition.md`, `scenario-slot-matrix.md`, `register-handoff.md` are all present under `risk/mod-108/carepilot/architecture/` and indexed from `mod-108-ex01-slots.md`.
2. **Every slot carries responsibility, failure mode, mechanism candidate.** Reviewer greps each slot section in `slot-spec.md` for these three items; every section carries all three.
3. **Coverage table pins framework versions at the header.** OWASP LLM Top 10 revision, MITRE ATLAS matrix version, SAIF pillar set, NIST AI RMF reference all named at the top of `coverage.md`.
4. **Empty cells are "—", not blank.** Reviewer scans every row of the coverage table and the scenario-slot matrix — every cell is either a code / Y / N / partial / "—"; never blank.
5. **Three composition scenarios named by ID.** Scenarios A, B, C each cite a specific mod-102 harm-row ID and walk the defeat sequence.
6. **Scenario-slot matrix flags uncovered rows.** Reviewer confirms any all-`N` row is flagged as a control-defeated row for the register.
7. **Register handoff queues, does not pre-credit.** Reviewer confirms no residual-risk number has been written into `risk-register/2025/CarePilot.yaml` yet — the handoff only names the rows that exercise 05 will update.
8. **No naked "high/medium/low".** Reviewer greps the whole deliverable for `high`, `medium`, `low`; any occurrence either cites a harm-scenario ID / OWASP code / ATLAS tactic / SAIF pillar or is a defended one-sentence note.
9. **Version-pin markers present.** Every mechanism candidate names the library / model / vendor whose version will be pinned when the mechanism ships; the pins themselves land in exercises 02–04.
10. **No measured coverage claims.** Reviewer greps for the words `measured`, `tested`, `evaluated`; the only occurrences are descriptions of what exercise 05 will do, not claims about the current state.

## 4. Rubric — review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Five slots instantiated or explicit absence | Every slot has a spec section or a written justification for absence with cross-reference | A slot is missing without comment, or absence paragraph is vague |
| R2 | Coverage table complete and version-pinned | Every slot row populated; every framework version pinned at the header; empty cells "—" | Any cell blank; framework versions missing or stale |
| R3 | Three composition scenarios walked | Scenarios A, B, C each cite a mod-102 row ID and walk the defeat sequence explicitly | Scenario cited by category only; defeat sequence vague |
| R4 | Scenario-slot matrix present | One row per harm-model scenario; cells Y / N / partial / note | Matrix missing or incomplete |
| R5 | Register handoff queues, does not pre-credit | Handoff names the mod-103 rows and the measurement exercise; no residual number written into the register | Residual number updated before exercise 05 measurement |
| R6 | Tool-invocation policy is deterministic | Slot 3 mechanism is allow-list + argument schema + quota in custom middleware | Slot 3 mechanism is classifier-based |
| R7 | Output slot uses independent mechanisms | Slot 4 names two different training-data families (Llama Guard + ShieldGemma) plus Presidio, not two thresholds on one model | Slot 4 mechanism is "same classifier at two thresholds" |
| R8 | No naked ratings | Every qualitative claim cites a row ID / code / pillar / sentence | A `high` / `medium` / `low` appears uncited |
| R9 | No invented facts | Every cell that cannot be verified today carries `<!-- verify-citation: ... -->` | Fabricated ATLAS IDs, OWASP codes, or SAIF pillar names |
| R10 | Cross-module handshake flagged | Register handoff and scenario matrix explicitly name the mod-103 rows and queue mod-105 / mod-107 / mod-109 handshakes where relevant | No cross-module tie-back |

Scoring bands: *Pass* — R1–R9 green, R10 at least honoured on the register handoff; *Pass with revisions* — R10 partial, or R2 missing one framework version; *Fail* — any of R1 / R6 / R7 / R9 fails.

## 5. Common mistakes

- **Starting from the slots, not the harm model.** The slot list is complete but the scenario-slot matrix has rows with all-`N` coverage. Start from mod-102.
- **Slot 3 classifier.** "Llama Guard on the tool call" is a design bug. Slot 3 is always a deterministic allow-list + argument schema in custom code owned by the application team.
- **Slot 4 is "Llama Guard at threshold 0.6 plus Llama Guard at threshold 0.8".** That is one control. Chapter 03 §"Composing classifier + rule + retrieval" requires two mechanisms with *different training data* at the output slot — the Llama Guard + ShieldGemma pair, or a classifier + a rule.
- **Slot 5 is treated as "the classifier refusal path".** Slot 5 is the *content the caller sees* when a classifier refuses. If the refusal string restates the triggering phrase ("I can't help you draft a phishing email because of [the phishing email text]…"), the refusal is itself a leak.
- **Coverage table cells left blank.** A blank cell means "the author forgot," not "the slot does not address this class." Write "—" so the reader knows the empty cell was considered.
- **Residual-risk credit written in exercise 01.** The register handoff is a *queue*, not an update. Residual credit requires exercise 05 measurement.
- **Framework versions unpinned.** OWASP LLM Top 10 v2023 and v2025 use different category codes; MITRE ATLAS and SAIF both evolve. Any citation that does not pin the version is a citation that will be wrong in six months.
- **Single-slot scenarios not called out.** Scenario B's shape (one addressing slot) is the exact condition the chapter 01 composition test wants to catch — if the only addressing slot is defeated, the residual collapses to inherent. The matrix must call these out so exercise 02–04 can add a second mechanism.
- **Confusing design commitment with shipped code.** The mechanism cells are candidates. Stubs do not exist yet; measurements do not exist yet; the exercise produces the *design* that subsequent exercises will prove.
- **Silent-SPOF discipline skipped.** All four chapter 01 anti-patterns deserve a probe. Even on a well-designed stack, one is latent — the composition walk-through's payoff is naming which one.

## 6. References

Only the official sources listed in the work packet (and clearly-labelled project documentation) are used. Every citation whose code / ID / pillar name cannot be verified today carries `<!-- verify-citation: ... -->` to block auto-merge until the signing engineer confirms the exact identifiers.

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework`. The handoff shape (design artefact → measurement → register update) aligns with MEASURE 2.11 (measurement of effectiveness) and MANAGE 2.2 (sustained controls). The AI RMF Generative AI Profile (AI 600-1) is the shape used for the composition walk-through.
- **OWASP Top 10 for Large Language Model Applications.** `https://genai.owasp.org/llm-top-10/`. Version pinned in the coverage table header at signing time. <!-- verify-citation: pin the 2025-edition category codes used in the table against the live list on the day of signing; the 2023 → 2025 renumbering is well documented but the exact titles and codes must be current. -->
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/`. Covers model-side risks (data poisoning, inference-time attacks) that intersect with the retrieval and output slots. <!-- verify-citation: cite the exact ML Top 10 entry codes and titles against the live list on the day of signing. -->
- **MITRE ATLAS.** `https://atlas.mitre.org/`. Tactic and technique IDs cited in the coverage table are placeholders until pinned at signing. <!-- verify-citation: pin matrix version and the exact tactic / technique IDs for each slot row. -->
- **Google SAIF.** `https://safety.google/cybersecurity-advancements/saif/`. Pillar set pinned at signing. <!-- verify-citation: pin the SAIF pillar names and count against the live page on the day of signing. -->
- **Module running system.** CarePilot — hospital triage-support RAG assistant, introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`; carried through mods 101–107 in the solutions in this repo.
- **mod-102 harm model** for CarePilot — `lessons/mod-102-harm-modelling-and-risk-taxonomy/` and the running-system harm rows in `risk-register/2025/CarePilot.yaml`.
- **mod-103 register** for CarePilot — `risk-register/2025/CarePilot.yaml`; the register handoff writes queued rows here after exercise 05 measurement.
- **Exercise prompt.** `lessons/mod-108-guardrail-engineering-and-effectiveness/exercises/exercise-01-guardrail-architecture-for-one-llm-app.md`.

<!-- All coverage-table citations (OWASP LLM codes, ATLAS IDs, SAIF pillar names) are version-pinned at signing; the verify-citation markers are intentional and block auto-merge until a human confirms the exact identifiers against the live sources. -->

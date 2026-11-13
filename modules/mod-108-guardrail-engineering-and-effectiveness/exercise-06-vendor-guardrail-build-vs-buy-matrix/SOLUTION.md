# Exercise 06 — Vendor Guardrail Build-vs-Buy Matrix and App-Side Measurement Contract (CarePilot) — Reference Solution

## 1. Solution overview

This exercise evaluates **six third-party guardrail vendors** — Lakera Guard, Arthur AI, Robust Intelligence, Calypso AI, HiddenLayer, Protect AI — against CarePilot's exercise-01 slot set and exercise-05 measurement rows. It produces:

- Six **vendor profiles** (current product names, deployment options, taxonomy, pricing model, evaluation methodology, source URLs with access dates).
- The **build-vs-buy matrix** with twelve chapter-06 axes and seven columns (in-house + six vendors).
- The **two summary numbers** per option — total residual-risk delta vs. the mod-103 register aggregate, and total three-year TCO with every line included.
- The **signed decision record** naming the chosen option, the driving axes, the two summary numbers, the revisit triggers, and the signatories.
- The **app-side measurement contract** to `ai-eval-engineer` (peer, L30) — a self-contained document the L30 can consume without reconstructing CarePilot's slot set from code.
- A **cross-track handshake note** naming each affected track.
- An **anti-pattern probe report** covering the four chapter-06 anti-patterns.

Four decisions drive the shape:

- **Score every option on the same axes with the same evidence source.** The in-house stack's coverage comes from exercise 05's measurement on the mod-104 set; vendor coverage should come from a trial run against the same mod-104 set *if at all possible*. Where a vendor cannot be measured, that limitation is a cell in the matrix, not a footnote.
- **Numbers or defended ratings — no naked "high/medium/low".** Chapter 06 §"The build-vs-buy matrix" is explicit; the auditor will ask "why medium?" and a naked rating collapses.
- **CarePilot's constraints are load-bearing.** The system is under HIPAA; data residency rules out vendor SaaS egress on PHI for several axes. A vendor whose deployment shape is SaaS-only for the input and output slots cannot be scored on coverage against CarePilot's traffic — the matrix marks "deployment-shape disqualifying" for those cells.
- **The decision is not always the cheapest or the best residual.** The signed decision record names the trade explicitly so the reader knows which axis drove it.

Deliverables live under `risk/mod-108/carepilot/vendors/` and are indexed from `mod-108-ex06-vendors.md`.

## 2. Worked implementation

### 2.1 Repository layout

```
risk/mod-108/carepilot/vendors/
├── lakera-guard.md               # one profile per vendor
├── arthur-ai.md
├── robust-intelligence.md
├── calypso-ai.md
├── hiddenlayer.md
├── protect-ai.md
├── matrix.md                     # 12 axes × 7 options
├── summary-numbers.md            # residual delta + three-year TCO per option
├── decision.md                   # signed decision record
├── measurement-contract.md       # handoff to ai-eval-engineer (L30)
├── handshakes.md                 # cross-track handshake note
└── anti-patterns.md              # four chapter-06 anti-pattern probes
```

### 2.2 Vendor profiles — shape of each

Each profile is a one-page markdown with the chapter-06 required fields. Vendor product lines evolve quickly, so each cell is marked with the access date and is treated as a current-state snapshot (not a durable fact). Below is the **template shape**; the actual numbers and deployment-shape options are pinned on the day of signing against the vendor's live documentation.

```markdown
# <Vendor> — profile — accessed <YYYY-MM-DD>

Product name(s) at authoring time: <current names — e.g., "Lakera Guard" + "Lakera Red">.

Source URLs (access date embedded):
- <product page URL>  (accessed <date>)
- <pricing or trial page URL>  (accessed <date>)
- <published eval methodology URL, if any>  (accessed <date>)
<!-- verify-citation: all vendor URLs and product names must be re-checked on
     the day of signing; vendor product names rename and consolidate often. -->

Deployment options (as published):
- SaaS: yes / no
- VPC (customer cloud): yes / no
- On-prem: yes / no
- Data residency (regions): <list>
- Custom deployment for regulated industries (HIPAA / BAA): yes / no — critical for CarePilot

Documented category taxonomy:
- Inputs: <categories the vendor's input classifier scores>
- Outputs: <categories the vendor's output classifier scores>
- Retrieval: <does the vendor support a retrieval-side scan? If yes, categories.>
- Tool-call policy: <does the vendor offer a tool / agent-action policy layer?>

Pricing model:
- Per-call: yes / no, at $<x>/call at tier <y>
- Per-throughput: yes / no, $<x>/req-per-second
- Per-endpoint: yes / no, $<x>/endpoint/month
- Enterprise tier (negotiated): yes / no

Published evaluation methodology:
- Dataset used: <vendor's benchmark set, if disclosed>
- Methodology: <black-box / white-box / adversarial sweep>
- Public coverage numbers: <recall / FP rates by category>
- Third-party reviews / benchmarks (link + access date)
<!-- verify-citation: vendor benchmarks are vendor-reported; mark the cell as
     such in the matrix and do not paraphrase them as measured numbers. -->

Supply-chain / viability:
- Ownership / recent M&A (Robust Intelligence was acquired by Cisco in 2024 per
  the vendor's own acknowledgement; check if productisation is current).
- Public reference customers (optional).
- Incident-disclosure policy (optional, link if published).
```

A paragraph at the top of each profile distils this into one sentence: `<Vendor>: <productisation summary>; <deployment shapes>; <pricing shape>; <HIPAA fit>; <evaluation evidence>`.

### 2.3 Build-vs-buy matrix (`matrix.md`)

Twelve rows (axes), seven columns (in-house + six vendors). Every cell is a number (measured or vendor-reported), a defended qualitative rating, or `n/a` with a one-sentence reason. Naked "high/medium/low" cells are a fail (chapter 06 §"Anti-patterns").

```
AXIS                                      | IN-HOUSE  | Lakera | Arthur | Robust | Calypso | HiddenLayer | Protect AI
------------------------------------------+-----------+--------+--------+--------+---------+-------------+-----------
1. Coverage on mod-104 adv set (recall)   | ex05-run  | trial? | v-rep  | v-rep  | v-rep   | v-rep       | v-rep
2. FP rate on legitimate set              | ex05-run  | trial? | v-rep  | v-rep  | v-rep   | v-rep       | v-rep
3. Latency added per slot (p50/p95 ms)    | ex05-run  | v-rep  | v-rep  | v-rep  | v-rep   | v-rep       | v-rep
4. Cost per call / per month              | ex05-run  | quoted | quoted | quoted | quoted  | quoted      | quoted/OSS
5. Deployment shape (SaaS/VPC/on-prem)    | on-prem   | <...>  | <...>  | <...>  | <...>   | <...>       | <...>
6. Portability / lock-in                  | n/a: own  | prop   | prop   | prop   | prop    | prop        | some OSS
7. Evidence trail                         | full      | <...>  | <...>  | <...>  | <...>   | <...>       | <...>
8. Threshold / policy control             | per slot  | <...>  | <...>  | <...>  | <...>   | <...>       | <...>
9. Update cadence predictability          | own-pin   | <...>  | <...>  | <...>  | <...>   | <...>       | <...>
10. Support model                         | internal  | <...>  | <...>  | <...>  | <...>   | <...>       | <...>
11. Ecosystem fit (Credo/Holistic/ModelOp)| <...>     | <...>  | <...>  | <...>  | <...>   | <...>       | <...>
12. Long-term viability                   | own-ctrl  | <...>  | <...>  | <...>  | <...>   | <...>       | <...>
```

Cell notation:

- `ex05-run` — measured number from exercise 05's measurement table; cite the run_id.
- `trial?` — placeholder until a vendor trial is procured. The stretch goal "A real vendor trial for one category" is where this cell gets replaced by a measured number.
- `v-rep` — vendor-reported from the vendor's published benchmark. Mark explicitly; the epistemic weight is lower than a measured cell. Chapter 06 §"Anti-patterns" case 1: do not paraphrase a vendor slide as a measurement.
- `prop` — proprietary. The portability axis carries the integration-removal cost into the TCO.
- `<...>` — pinned against the vendor's live documentation on the day of signing.

Critical cells for CarePilot (HIPAA, PHI, on-prem) often collapse to `n/a: SaaS only — deployment-shape disqualifying`. Those cells belong in the matrix, not in a footnote.

### 2.4 Two summary numbers (`summary-numbers.md`)

Per option, two numbers (chapter 06 §"Two summary numbers per option"):

**Total residual-risk delta.** Change to the mod-103 register's aggregated residual score when this option is credited as the control set. Positive = worse residual; negative = better. For CarePilot, the aggregated residual is the sum of the per-row residual × the row's harm weight; the delta is computed against the in-house stack's measurement from exercise 05.

**Total three-year TCO.** All cost lines summed over three years, including exit cost (integration removal, data extraction, retraining the in-house replacement, on-boarding a different vendor).

TCO line decomposition (chapter 06 §"Two summary numbers" and §"Motivation" case 2):

```
IN-HOUSE three-year TCO lines:
  - Compute (GPU fleet amortised across CarePilot's request volume)
  - Operational (on-call rotation × loaded eng-hour rate × years)
  - Drift-monitoring compute (mod-109 stub + its GPU time)
  - Retraining / re-tuning of classifiers on new attack categories
  - Integration maintenance (LangChain / LangGraph / FastAPI breaking changes)

VENDOR three-year TCO lines:
  - Vendor list price × volume × 3 years (or per-endpoint × endpoint-count × 3 years)
  - Integration eng-hours × loaded rate (initial)
  - Data-egress / network cost (if SaaS and not on-prem — zero for CarePilot's model)
  - Vendor compliance review (legal + security + BAA negotiation)
  - On-call during vendor incidents (fraction of vendor outage events × response cost)
  - Exit cost (integration removal, data extraction, replacement integration)
```

Plot both numbers on a scatter. Each option is one dot; axis is residual delta (x) × TCO (y). The chosen option is in the preferred quadrant (lower residual + lower TCO) or is explicitly accepted as being in a less-preferred quadrant for an axis the decision record names.

### 2.5 Signed decision record (`decision.md`)

```markdown
# CarePilot — Guardrail Build-vs-Buy Decision Record

Decision: compose.

The in-house stack (NeMo Guardrails + Guardrails AI + Presidio + custom
middleware + self-hosted Llama Guard / ShieldGemma) continues to own slots
1, 2, 3, 4, 5 end-to-end. No vendor is procured for slots 1–5 at this time.

Driving axes:
- Axis 5 "Deployment shape" — CarePilot is under HIPAA; several vendors are
  SaaS-only on their default plans. BAA availability was mixed; none met the
  "on-prem or customer-VPC + BAA + HIPAA-aligned data handling" bar for every
  slot. Build was forced on the slots that touch PHI.
- Axis 1 "Coverage on mod-104 adv set" — the in-house stack's measured recall
  from exercise 05 is within the appetite bands on all but two categories
  (`prompt-injection-indirect` and `medical-misinformation`). The vendor
  options that might close those gaps are either SaaS-disqualified or their
  coverage numbers are vendor-reported, not measured.
- Axis 4 "Cost" — the three-year TCO of the in-house stack is competitive on
  CarePilot's request volume. At lower volumes a SaaS vendor would be cheaper;
  at CarePilot's volume the GPU amortisation wins.
- Axis 7 "Evidence trail" — the exercise-05 manifest produces a reproducible
  measurement trail per run; vendor evidence trails are per-vendor and vary.

Closest-to-changing axis:
- Axis 9 "Update cadence". Self-hosted Llama Guard and ShieldGemma get updated
  on our schedule, but new attack categories lag the vendor's own update
  cycle. If the mod-104 adversarial set adds a category we cannot cover within
  two release cycles, revisit this decision.

Summary numbers:
- In-house total residual-risk delta: <pinned from exercise 05>
- In-house total three-year TCO: <pinned> at current CarePilot request volume

Trigger events for revisit:
- Vendor announces a BAA / on-prem option for an input or output slot with
  measurable coverage on `prompt-injection-indirect` or `medical-misinformation`.
- Mod-104 adds an adversarial category the in-house stack cannot cover within
  two release cycles.
- CarePilot's request volume changes by ±50% (TCO calculation needs re-running).
- Vendor pricing change at a vendor whose coverage is competitive (for the
  compose-later option).
- A vendor is acquired and the acquirer's data-handling posture changes
  (Robust Intelligence / Cisco is the current example; re-score if productisation changes).
- Corporate change (M&A, division spin-off) that invalidates the BAA and
  data-residency assumptions.

Signatories:
- ai-risk-engineer (L25): <name>
- senior-ai-governance-architect (L50): <name>  # required because residual
  crosses the program-level band for `harm-model-misinformation`
- clinical-safety officer (CarePilot domain): <name>
- procurement + legal: not required for build-only decision; required if the
  decision moves to compose

The decision record is authored in the shape it would be signed; the
exercise-only decision is not shipped to production without the signatures.
```

The "compose" default is a defensible shape for CarePilot: in-house for every slot now, with a documented path to vendor adoption on slot 1 or slot 4 if the trigger events fire. The write-up names the shape; the exercise does not pretend to execute the procurement.

### 2.6 Measurement contract to `ai-eval-engineer` (`measurement-contract.md`)

Self-contained document. The L30 reader should be able to consume it without reading any other file in the module.

```markdown
# CarePilot — Guardrail Measurement Contract — v<version>

To: ai-eval-engineer (peer, L30)
From: ai-risk-engineer (L25)
Cadence: contract re-issued on every adversarial-set version bump and every major control change.

## Slot definitions
Five slots, per exercise 01:
  - slot 1 input filter: NeMo GR input rail + Llama Guard
  - slot 2 retrieval filter: NeMo GR retrieval rail + ShieldGemma + Presidio fast analyser
  - slot 3 tool-invocation policy: custom FastAPI middleware (allow-list + schema + quota)
  - slot 4 output validator: Guardrails AI guard + Llama Guard + ShieldGemma + Presidio
  - slot 5 refusal shaping: NeMo GR output rail + templated refusal content

## Control instances (with pinned versions)
From the exercise-05 run manifest <path/to/manifest-<run_id>.json>:
  - llama_guard:<ckpt>  taxonomy:<v>
  - shieldgemma:<ckpt>  policy_set:<v>
  - presidio_analyzer:<v>  presidio_anonymizer:<v>  spacy_model:<v>
  - nemo_guardrails:<v>  colang:<v>  guardrails_ai:<v>
  - middleware:<git-sha>
  - runner:<git-sha>

## Sets used
Adversarial: mod-104 adversarial set version <adv version>, SHA-256 <hash>
Legitimate: mod-104 legitimate set version <leg version>, SHA-256 <hash>

## Measurement rows
Row shape: (slot, control_instance, category, TP, FP, TN, FN, precision,
recall, mean_cost_per_call, p50_latency_ms, p95_latency_ms,
n_adversarial, n_legitimate, set_version, control_version, run_id)

Attached as <path/to/measurements-<run_id>.csv>; <n_rows> rows total.

## Cost currency and time units
Currency: <USD / EUR / etc., same as mod-103 register>
Compute cost: GPU-hour amortised across CarePilot throughput over the run window.
Operational cost: eng-hour × loaded rate ÷ request volume.
Latency unit: milliseconds, measured in the middleware (not server-side).

## Register linkage
For each measurement row, the mod-103 register row it updates:
  slot=input, category=prompt-injection-direct → risks[prompt-injection-direct]
  slot=retrieval, category=prompt-injection-indirect → risks[indirect-prompt-injection]
  slot=output, category=pii-exfiltration → risks[pii-output]
  slot=tool_policy, category=excessive-agency → risks[excessive-agency]
  ... (full list in register-updates.md from exercise 05)

## What this contract does NOT supply
Chapter 05 §"Cross-references" and chapter 06 §"Handoff":
  - Bootstrap confidence intervals on the measurement rows — L30's contribution.
  - Judge-vs-human agreement on the labelled slice — L30's contribution.
  - Benchmark methodology write-up — L30's contribution.
  - Any statistical calibration beyond raw TP / FP / TN / FN counts — L30's.

## Open points
  - Bounded categories where mod-104 does not host a legitimate set — list.
  - Slots over the exercise-01 latency budget — list and treatment plan.
  - Fail-closed error rate per classifier — reported alongside TP / FP / TN / FN.
```

### 2.7 Cross-track handshake note (`handshakes.md`)

```markdown
# Cross-track handshakes for the CarePilot guardrail decision

## ai-eval-engineer (peer, L30)
Receives: the measurement contract (measurement-contract.md), the measurements
CSV (measurements-<run_id>.csv), and the run manifest (manifest-<run_id>.json).
Cadence: on every adversarial-set version bump and every major control change.
Action: statistical calibration (bootstrap CIs, judge-vs-human agreement).

## senior-ai-governance-architect (L50)
Receives: the signed decision record (decision.md) and the register updates
block (from exercise 05). Required signatory if residual or TCO crosses the
program-level appetite band for any mod-103 row — CarePilot's medical-
misinformation residual currently sits in that band, so L50 signature is on
the critical path.

## ai-privacy-engineer (peer)
Receives: the Presidio integration (exercise 04) and the vendor profiles
section covering PII handling. Coordinates on the PII-vendor selection if
the compose decision moves slot 4's Presidio to a vendor DLP product.

## procurement + legal
Receives: the vendor profile section (deployment options, pricing model,
HIPAA / BAA fit) and the decision record. Not required if the decision is
pure build; required if any vendor is selected (BAA negotiation, data-
residency review, contract redlines).

## security architecture
Receives: the deployment-shape column of the matrix (axis 5). Reviews any
vendor's proposed data-residency and KMS integration. Owns the sign-off on
customer-VPC deployment shapes.

## mod-109 (continuous monitoring and risk-register integration)
Receives: the trace format spec (from exercise 02), the three Presidio log
schemas (from exercise 04), the run-manifest shape (from exercise 05). The
monitor stub proposed in exercise 05 stretch is the ingest point for mod-109.

## mod-110 (model risk management and supply chain)
Receives: the vendor profiles' supply-chain / viability rows. If the compose
decision moves forward, mod-110's due-diligence is a precondition for sign-off.

## mod-111 (incident response and post-mortems)
Receives: the fail-closed-on-classifier-error numbers from exercise 05 and the
vendor's incident-disclosure policy from the profiles. Feeds the post-mortem
playbook for classifier-outage scenarios.
```

### 2.8 Anti-pattern probes (`anti-patterns.md`)

- **Vendor marketing as coverage story.** Confirmed each vendor's coverage cell is either `ex05-run` (measured in a trial, if procured), `v-rep` (explicitly flagged as vendor-reported), or `n/a: deployment-shape disqualifying`. No cell paraphrases a vendor slide as a measured number.
- **Build with hidden TCO.** The in-house TCO line decomposition in §2.4 includes compute, operational, drift-monitoring, retraining, integration maintenance. If any line is omitted at submission (e.g., "GPU already there — free"), the probe flags it. Chapter 06 §"Motivation" case 2 is the anti-pattern.
- **Compose without owning the seam.** CarePilot's chosen option is compose-eligible (build now, compose later on a trigger). The seam at slot 1 or slot 4 — who owns FP tuning when the vendor and in-house controls meet, who owns latency when both fire — is sketched in `decision.md` even though the compose has not been executed. If and when compose is executed, the seam contract is a precondition.
- **Silent duplication of eval methodology.** The measurement contract to L30 explicitly lists what the risk engineer does **not** supply (bootstrap CIs, judge-vs-human agreement, benchmark methodology). The L30's work is not duplicated.

### 2.9 Interpretive paragraph

One paragraph for the index:

> The axis that drove CarePilot's decision was axis 5 (Deployment shape) — HIPAA and the BAA requirement eliminated SaaS-only options for slots 1–4 and forced the build. Axis 1 (coverage) and axis 4 (cost) both supported the build decision but would not have driven it independently. The axis closest to changing the decision is axis 9 (Update cadence): if mod-104 adds an adversarial category the in-house stack cannot cover within two release cycles, revisit the compose option on slot 1 or slot 4. The trigger event most likely to force a revisit is a vendor announcing a BAA / on-prem offering with measured coverage on `prompt-injection-indirect` — currently the in-house stack's weakest category per exercise 05.

## 3. Validation steps

1. **All seven artefacts exist.** Six vendor profiles, matrix, summary numbers, decision, measurement contract, handshakes, anti-patterns.
2. **Every vendor profile cites a URL and access date.** `grep -R "accessed " vendors/` returns one or more hits per profile.
3. **Matrix has 12 axes × 7 options.** Rows and columns counted; every cell populated (number / `v-rep` / `n/a: <reason>` / defended rating).
4. **Two summary numbers computed per option.** Residual delta and three-year TCO populated for every column of the matrix.
5. **Scatter plot or table equivalent present.** Options plotted on the (residual delta, TCO) axes.
6. **Decision record signed-shape.** `decision.md` names the chosen option, the driving axes, the summary numbers, trigger events, and signatories.
7. **Measurement contract self-contained.** `measurement-contract.md` has slot definitions, control instances, set versions, measurement-row shape, cost currency, register linkage, and the explicit "does not supply" list.
8. **Cross-track handshakes name owners.** `handshakes.md` names at least five tracks (L30, L50, privacy, procurement + legal, security).
9. **Anti-pattern probes cover all four.** `anti-patterns.md` has four sections.
10. **No naked "high/medium/low".** `grep -R "high\|medium\|low" vendors/matrix.md` — every occurrence is cited or is a defended one-sentence note.

## 4. Rubric — review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Six vendor profiles | Lakera, Arthur, Robust, Calypso, HiddenLayer, Protect AI each have a one-page profile with current product name, deployment, taxonomy, pricing, methodology, URL + access date | Any vendor missing or not accessed |
| R2 | Matrix has 12 axes × 7 options | Row and column counts match; every cell populated | Any cell blank or axes missing |
| R3 | No naked ratings | Every cell is a number / `v-rep` / `n/a:reason` / defended rating | A naked `high`/`medium`/`low` present |
| R4 | Two summary numbers per option | Residual delta + three-year TCO; plot or table | One or both missing |
| R5 | Decision record in chapter-06 shape | Chosen option, driving axes, summary numbers, trigger events, signatories | Any element missing |
| R6 | Measurement contract self-contained | Every chapter-06 §Handoff field populated; "does not supply" list present | Contract refers out to code |
| R7 | Cross-track handshakes named | At least L30, L50, privacy, procurement + legal, security | Fewer tracks |
| R8 | Build TCO includes all lines | Compute, operational, drift-monitoring, retraining, maintenance | Only compute reported |
| R9 | Compose seam contract sketched | `decision.md` names the seam ownership even in build-only | Compose hand-off omitted |
| R10 | Anti-pattern probes cover all four | Four sections with resolutions or filed follow-ups | A section missing |

Scoring bands: *Pass* — R1–R9 green, R10 honoured; *Pass with revisions* — R4 or R7 partial; *Fail* — any of R1 / R3 / R5 / R6 fails.

## 5. Common mistakes

- **Scoring against vendor feature lists, not CarePilot's slot set.** A vendor sells twelve features; only the ones addressing CarePilot's exercise-01 slots matter. Score against the slots.
- **Naked "high / medium / low".** The auditor asks "why?" and the cell collapses. Cite a mod-104 category or cite a one-sentence defence.
- **"GPU is already there — free."** Chapter 06 §"Motivation" case 2 is the anti-pattern. GPU hour is amortised; the operational on-call is real; the drift-monitoring compute is real; the retraining budget is real.
- **Three-page vendor profile per vendor.** One page per vendor is enough to score the matrix. Extra depth lives in supplementary appendices (stretch: supply-chain due-diligence).
- **Vendor-reported numbers paraphrased as measured.** Lakera's own benchmark on their own adversarial set is not evidence against CarePilot's traffic. Mark `v-rep` and down-weight the epistemic value.
- **Missing exit cost.** Three-year TCO includes integration removal, data extraction, replacement integration. Procurement often negotiates on initial price; exit cost is where lock-in bites.
- **Decision record without trigger events.** A decision that is never revisited is a decision that is wrong by year three. Chapter 06 is explicit: name the trigger events.
- **Compose decision without a seam contract.** The seam is where most compose decisions fail. Who owns FP tuning when vendor and in-house controls disagree; who owns latency when both fire. Sketch the contract even if compose is deferred.
- **L30 handoff that duplicates statistical work.** Bootstrap CIs are the L30's contribution. The contract explicitly names what the risk engineer does not supply.
- **Treating long-term viability as the dominant axis.** M&A activity is one row. Robust Intelligence's acquisition by Cisco in 2024 is one input, not a disqualifier. The risk council weighs.
- **Not refreshing vendor URLs on the day of signing.** Vendor product names, pricing pages, and deployment options change on a monthly cadence. A citation without an access date is stale within a quarter.

## 6. References

- **NIST AI Risk Management Framework 1.0.** `https://www.nist.gov/itl/ai-risk-management-framework`. The build-vs-buy decision is a MANAGE-function artefact (supplier-related controls); the matrix and the two summary numbers feed MEASURE 2.11 (effectiveness) and MANAGE 3.1 (third-party-risk).
- **OWASP Top 10 for Large Language Model Applications.** `https://genai.owasp.org/llm-top-10/`. The matrix's coverage row (axis 1) scores recall per OWASP category; the 2025 codes are pinned at submission. <!-- verify-citation: pin 2025 category codes. -->
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/`. <!-- verify-citation: pin exact ML Top 10 entries that the matrix cites. -->
- **MITRE ATLAS.** `https://atlas.mitre.org/`. Vendor coverage against ATLAS tactics is the external frame the vendor profiles cite where the vendor publishes it. <!-- verify-citation: pin IDs. -->
- **Lakera Guard.** Vendor site; current product documentation URL cited in `lakera-guard.md` with access date. <!-- verify-citation: confirm current product name + deployment options on the signing day. -->
- **Arthur AI.** Vendor site; current product documentation (Arthur Shield for runtime guardrails) URL + access date. <!-- verify-citation. -->
- **Robust Intelligence.** Cisco acquired Robust Intelligence in 2024 per public disclosure; current productisation under Cisco should be confirmed on the signing day. <!-- verify-citation: confirm post-acquisition productisation and names. -->
- **Calypso AI.** Vendor site; LLM security and DLP focus. <!-- verify-citation: pin current product names. -->
- **HiddenLayer.** Vendor site; AI security platform (adversarial ML heritage + LLM runtime protection). <!-- verify-citation. -->
- **Protect AI.** Vendor site; portfolio includes `ModelScan`, `Guardian`, `Recon`, `LayerX`, `Radar`; some OSS. <!-- verify-citation. -->
- **mod-104** — the adversarial-and-legitimate set hosted at pinned versions by the mod-104 workflow is the ground truth both the in-house measurement and any vendor trial score against.
- **mod-103** — `risk-register/2025/CarePilot.yaml`; the residual-risk-delta summary number is computed against this file's appetite bands.
- **mod-103 GRC-for-AI-platform notes.** Credo AI / Holistic AI / ModelOp / Monitaur — chapter-06 "ecosystem fit" row (axis 11) is pinned against the mod-103 survey.
- **Module running system.** CarePilot — hospital triage-support RAG assistant, introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`.
- **Exercises 01–05** — the slot set, the rule/flow layer, the classifier stack, the Presidio integration, and the measurement rows are the in-house baseline this exercise scores against.
- **Exercise prompt.** `lessons/mod-108-guardrail-engineering-and-effectiveness/exercises/exercise-06-vendor-guardrail-build-vs-buy-matrix.md`.

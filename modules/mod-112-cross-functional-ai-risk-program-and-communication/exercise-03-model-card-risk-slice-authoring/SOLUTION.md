# Exercise 03 — Model / System / Dataset Card Risk-Slice Authoring — Reference Solution

## 1. Solution overview

This exercise is the **documentation-under-governance** drill: for the running CarePilot system (hospital-triage-support RAG from mod-101), author the **risk slice** of three distinct card artifacts — a **model card** (describes a model), a **system card** (describes a deployed AI-enabled product), and a **dataset card** (describes a dataset the model consumes) — and show how the three stay consistent with each other and with the team's harm model and risk register.

The deliverable is **three worked risk slices** (one per card), a **cross-card consistency check** that an auditor can run, and a short **authoring convention** that the team uses for every release.

The reference solution:

- Treats the three cards as **distinct documents with distinct audiences**: a model card is for the model consumer (the team itself, a downstream integrator, or a model risk reviewer); a system card is for the operational consumer (the clinician user, the hospital procurement reviewer, the regulator reading a product-level summary); a dataset card is for the data consumer (someone deciding whether this dataset is fit for a particular use). The risk slice of each is scoped to what *that* audience needs to decide.
- Focuses the deliverable on the **risk slice**. A full model / system / dataset card has many other sections (model architecture, training recipe, evaluation methodology, intended use, licence). This exercise teaches the risk-slice authoring discipline; other sections are referenced but not reproduced.
- Keeps the three risk slices **grounded in the harm model and the risk register**. Every risk-slice statement traces back to a row in `docs/systems/carepilot/harm-model.md` or a row in `risk-register/2025/CarePilot.yaml`. If a card claims a risk that is not in the register, the register is wrong, or the card is wrong — the inconsistency is the signal.
- Writes risk slices that **name the limitation honestly**. The dominant failure mode is marketing-grade cards that read as risk-free; the authoring convention is explicit that a card without named limitations is a drafting defect.
- Treats the slice-level cross-walk to **NIST AI RMF** (GOVERN / MAP / MEASURE / MANAGE), **OWASP ML Top 10**, and **MITRE ATLAS** as **framework tags on the risk statement**, not as a separate section. Tags let a reviewer trace a specific risk-slice row to the framework position it corresponds to; this is what makes the card auditable against a framework the organisation has adopted.

Design decisions worth calling out:

- **Why three distinct cards, not one.** The impulse to produce "one card to rule them all" fails because the three audiences read different sections, make different decisions, and need different granularity. A hospital procurement reviewer does not need the model's training-data provenance; a model reviewer does. A dataset reviewer does not need the production monitoring posture; the system reviewer does.
- **Why risk slices only.** A full card is a multi-author artifact spanning methodology, metrics, architecture, and licence. The exercise prompt is the *risk slice* — the sections that document what could go wrong and how the team has addressed it. Over-scoping produces documents nobody finishes.
- **Why grounded in the harm model and the register.** Cards drift when they are a parallel authoring surface. The discipline here is: the risk slice is a *projection* of the harm model and the register onto an audience; it is never the authoritative source.
- **Why framework tags inline.** Keeping the framework mapping inline (as tags on a row) means an audit can grep the card for a sub-category code and get a hit; keeping it in a separate appendix means the reviewer has to switch contexts.

## 2. Worked answer or implementation

### 2.1 Repository layout

```
exercise-03-model-card-risk-slice-authoring/
├── SOLUTION.md                                      # this file
├── model-card/
│   └── carepilot-v1.1-model-card.risk-slice.md
├── system-card/
│   └── carepilot-v1.1-system-card.risk-slice.md
├── dataset-card/
│   └── carepilot-discharge-notes-synthetic-v1.dataset-card.risk-slice.md
├── templates/
│   ├── model_card_risk_slice_template.md
│   ├── system_card_risk_slice_template.md
│   └── dataset_card_risk_slice_template.md
└── checks/
    └── cross_card_consistency.md
```

### 2.2 Authoring convention (applies to all three cards)

Every risk-slice row in every card follows the same shape so a reviewer can scan.

```markdown
- **<risk-row-title>.**
  _Audience read:_ <one-sentence summary for this card's audience.>
  _Harm-model anchor:_ `<harm-model-row-id>` in `docs/systems/carepilot/harm-model.md`.
  _Risk-register anchor:_ `risk-register/2025/CarePilot.yaml#risks[<id>]`.
  _Evidence_: `<file path(s)>` at `<version/commit>`; headline metric `<name>` in range `<range>`.
  _Current mitigation_: one sentence.
  _Residual risk_: one sentence; never "zero".
  _Framework tags_: `RMF: <function>.<sub-category>`; `OWASP ML Top 10: <entry>`; `MITRE ATLAS: <technique id>`. (Use `<!-- verify-citation: ... -->` where the exact code has not been confirmed from the official source.)
```

**Why the shape.** It forces: an anchor to a single row of truth (harm model and register), an evidence pointer that is pinned at a version, an explicit residual statement, and a framework tag that is auditable. Rows that do not fit the shape are the rows that need rework.

**Rules of the slice.**
- Every row in the model-card slice maps to at least one harm-model row **about the model itself**. Rows that are about the *product deployment* belong in the system-card slice.
- Every row in the system-card slice maps to a harm-model row that has at least one deployment-context precondition (user population, care setting, geography, lifecycle stage).
- Every row in the dataset-card slice maps to a dataset-level harm — bias, missingness, PII, licensing, provenance — not to a model-level harm.
- Headline metrics appear as **ranges** unless the underlying data supports a precise number. False precision is a drafting defect.
- Specific threshold values that would aid an attacker (detection latency, exact denylist patterns) are kept as **ranges or categorical labels**; the register holds the precise value.
- Each card's risk slice names what the card does **not** cover and points to the sister card(s) that do.

### 2.3 Model-card risk slice — `model-card/carepilot-v1.1-model-card.risk-slice.md`

```markdown
---
document_class: model-card.risk-slice
model: carepilot-triage-summariser
model_version: v1.1
model_version_commit: "c71a…9d2e"
effective_from: 2026-12-15
supersedes: v1.0
author_role: AI risk engineer
reviewer_roles:
  - AI evaluation engineer
  - Tech lead / engineering manager
  - Clinical-safety officer (for clinical-impact release)
status: pre-release-draft
harm_model_pin: "docs/systems/carepilot/harm-model.md @ b19e…4f30"
register_pin: "risk-register/2025/CarePilot.yaml @ 2026-11-14"
---

# CarePilot — Triage Summariser Model — Risk Slice (v1.1)

## 1. Scope of this slice

This slice documents **model-level** risks for the triage-summariser model
that the CarePilot system uses, at the pinned version. It does **not**
document deployment risks (see the system-card risk slice at
`system-card/carepilot-v1.1-system-card.risk-slice.md`) or dataset-level
risks (see the dataset-card risk slice at
`dataset-card/carepilot-discharge-notes-synthetic-v1.dataset-card.risk-slice.md`).

## 2. Intended use (recap)

- Summarise a patient's triage context, grounded in a retrieved snippet of
  the hospital's triage protocols and the ED-specific clinical handbook,
  for a licensed clinician as final decision-maker. The model is a
  decision-support instrument.

## 3. Out-of-scope uses

- Not for autonomous dispensing, autonomous dosing, or any action that
  bypasses a clinician in the loop.
- Not for population-level analytics or billing determinations.
- Not for use outside the ED triage flow within the hospital network at
  the lifecycle stage pinned above.

## 4. Known model-level risks

- **Hallucinated clinical caveat omission.**
  _Audience read:_ Under rare input patterns, the model has produced
  summaries that omit a safety-relevant caveat present in the retrieval
  context. Observed in pre-production canary (synthetic).
  _Harm-model anchor:_ `H-CS-01` (clinical-safety hallucination).
  _Risk-register anchor:_ `risk-register/2025/CarePilot.yaml#risks[clinical-safety-hallucination]`.
  _Evidence_: shadow-eval report `evals/carepilot/shadow/2026-11-R46/report.json` at commit `c71a…9d2e`; groundedness in the range 94–95% on the clinical-adjudicated eval set; retrieval-alignment in the range 96–97%.
  _Current mitigation_: regression fixture `rt-doc-injection-2026-11`; HITL sign-off requirement unchanged; groundedness floor enforced at release gate.
  _Residual risk_: non-zero; the clinician-in-the-loop requirement is the final safeguard.
  _Framework tags_: `RMF: MEASURE.2.3`; `OWASP ML Top 10: <!-- verify-citation: confirm the OWASP ML Top 10 / LLM Top 10 entry for output-integrity / hallucination -->`; `MITRE ATLAS: <!-- verify-citation: confirm the ATLAS technique for ML-output manipulation most applicable -->`.

- **Prompt injection via document textual layer.**
  _Audience read:_ A crafted multi-turn pattern embedded in the textual
  layer of an uploaded document has been shown (in a synthetic near-miss)
  to cause the summariser to override its retrieval-grounded reasoning.
  _Harm-model anchor:_ `H-PI-03`.
  _Risk-register anchor:_ `risk-register/2025/CarePilot.yaml#risks[prompt-injection]`.
  _Evidence_: red-team suite `carepilot-rt-suite` at commit `a4f2…c7d1`; attack-success-rate on the document-payload probe class in the range 1–3% against a target of ≤ 3% for the v1.1 go-live window.
  _Current mitigation_: shared ingestion policy layer (free-text and document paths unified); regression fixture pinned.
  _Residual risk_: non-zero; the class of failure (any future ingest surface) remains an open area mitigated at the policy layer but not exhaustively tested.
  _Framework tags_: `RMF: MEASURE.2.7`; `OWASP ML Top 10: <!-- verify-citation: prompt injection entry -->`; `MITRE ATLAS: <!-- verify-citation: prompt injection via document ingestion technique ID -->`.

- **Retrieval-alignment drift under corpus update.**
  _Audience read:_ When the hospital updates its triage protocols, the
  retrieval index can briefly return stale or misaligned snippets; the
  model may surface a caveat that is no longer current.
  _Harm-model anchor:_ `H-RA-01` (retrieval alignment).
  _Risk-register anchor:_ `risk-register/2025/CarePilot.yaml#risks[retrieval-alignment]`.
  _Evidence_: drift monitor `monitors/carepilot/drift-retrieval.md`; retrieval-alignment production metric reported monthly.
  _Current mitigation_: re-index gating at corpus update; retrieval-alignment delta blocks promotion if delta > envelope.
  _Residual risk_: time-boxed; detection latency in the range of hours on the current monitor.
  _Framework tags_: `RMF: MEASURE.3.1`; `OWASP ML Top 10: <!-- verify-citation: data-quality / supply-chain entry -->`; `MITRE ATLAS: <!-- verify-citation: ML supply-chain or data manipulation technique ID -->`.

- **Fairness slice differential on under-represented subgroup.**
  _Audience read:_ No material differential observed at v1.1 across the
  harm-model-defined subgroups; the risk remains open because sample
  sizes in some subgroups are small.
  _Harm-model anchor:_ `H-FR-01`.
  _Risk-register anchor:_ `risk-register/2025/CarePilot.yaml#risks[fairness-triage]`.
  _Evidence_: fairness report `evidence/measure-2.11/v1.1/fairness-report.json`; subgroup groundedness delta within envelope; disparate-impact ratio within floor.
  _Current mitigation_: slice report run at every release; subgroup floor enforced.
  _Residual risk_: statistical uncertainty on small subgroups; pending larger slice volume.
  _Framework tags_: `RMF: MEASURE.2.11`; `OWASP ML Top 10: <!-- verify-citation: bias-related entry -->`; `MITRE ATLAS: <!-- verify-citation: not primarily ATLAS-mapped -->`.

- **Privacy — PII in model output.**
  _Audience read:_ No PII leak observed on the pre-release privacy eval
  set; the model's training did not include patient records, but the
  retrieval and session paths carry PII by definition.
  _Harm-model anchor:_ `H-PR-01` and `H-PR-02` (session isolation).
  _Risk-register anchor:_ `risk-register/2025/CarePilot.yaml#risks[pii-leak]` and `risk-register/2025/CarePilot.yaml#risks[session-isolation]`.
  _Evidence_: privacy report `evidence/measure-2.10/v1.1/privacy-report.json`; PII-leak rate below envelope.
  _Current mitigation_: session-isolation boundary; PII-scrub on structured output; mod-107 evaluation pinned.
  _Residual risk_: non-zero; a session-isolation audit is scheduled at the 60-day post-go-live review (per the mod-112 ex-02 decision log).
  _Framework tags_: `RMF: MEASURE.2.10`; `OWASP ML Top 10: <!-- verify-citation: sensitive-information disclosure entry -->`; `MITRE ATLAS: <!-- verify-citation: inference-time data exposure technique ID -->`.

## 5. Known limitations

- The model is **not** designed for adversarial users. The HITL sign-off
  requirement is load-bearing; downstream users must not be routed
  around the HITL.
- The model's evaluation coverage on the document-ingestion path is
  **newer** than on the free-text path. The risk picture is reviewed at
  the 60-day post-go-live council review.
- Metrics are **reported as ranges** where the underlying support is a
  range; false-precision numbers are a drafting defect that this slice
  avoids by convention.

## 6. What this slice does not cover

- Deployment context (care setting, user population, geography, lifecycle
  stage): see the system-card risk slice.
- Dataset provenance, bias, missingness, licensing: see the dataset-card
  risk slice.
- Legal / regulatory classification: counsel-owned; see the system-card
  risk slice §4.

## 7. Cross-card references

- System card: `system-card/carepilot-v1.1-system-card.risk-slice.md` at this release tag.
- Dataset card: `dataset-card/carepilot-discharge-notes-synthetic-v1.dataset-card.risk-slice.md` at this release tag.
- Harm model: pinned above.
- Risk register: pinned above.

## 8. Attestations

- AI risk engineer — attested on `<date>`.
- AI evaluation engineer — attested on `<date>`.
- Tech lead / engineering manager — attested on `<date>`.
- Clinical-safety officer — attested on `<date>`.
```

### 2.4 System-card risk slice — `system-card/carepilot-v1.1-system-card.risk-slice.md`

```markdown
---
document_class: system-card.risk-slice
system: CarePilot
system_version: v1.1 (triage-support RAG with discharge-note ingestion)
effective_from: 2026-12-15
supersedes: v1.0
author_role: AI risk engineer
reviewer_roles:
  - Tech lead / engineering manager
  - AI evaluation engineer
  - AI infra / MLOps engineer
  - Clinical-safety officer
  - Privacy officer
  - Head of communications
  - Legal counsel (classification determination)
status: pre-release-draft
harm_model_pin: "docs/systems/carepilot/harm-model.md @ b19e…4f30"
register_pin: "risk-register/2025/CarePilot.yaml @ 2026-11-14"
---

# CarePilot — System Card — Risk Slice (v1.1)

## 1. Scope of this slice

This slice documents **product-deployment-level** risks of the CarePilot
system as deployed at the hospital in the ED triage flow, at v1.1 which
adds the discharge-note ingestion modality. It does **not** document
model-level risks (see the model card) or dataset-level risks (see the
dataset card).

## 2. Deployment context

- Care setting: hospital ED triage flow (inpatient, adult; paediatric is
  out of scope).
- User population: licensed clinicians (attending physicians, triage
  nurses) authenticated via SSO.
- Geography: one regional US health system; one jurisdiction.
- Lifecycle stage: limited release (post-pilot, pre-general-availability).
- Human-in-the-loop: clinician sign-off is a required gate on every
  recommendation reaching the patient.

## 3. Known deployment risks

- **Over-reliance on AI-generated summary by rushed clinician.**
  _Audience read:_ Under time pressure, a clinician may accept the
  summary without reading the retrieval source; a wrong summary becomes
  action.
  _Harm-model anchor:_ `H-OA-01` (over-reliance).
  _Risk-register anchor:_ `risk-register/2025/CarePilot.yaml#risks[over-reliance]`.
  _Evidence_: HITL-override-rate production metric reported monthly; near-miss queue reviewed quarterly.
  _Current mitigation_: UI design requires explicit source-click before sign-off; monthly over-reliance training session; clinical-safety sign-off on the UI at every major release.
  _Residual risk_: non-zero; the risk is monitored, not eliminated.
  _Framework tags_: `RMF: MANAGE.4.3`; `OWASP ML Top 10: <!-- verify-citation: over-reliance entry -->`; `MITRE ATLAS: <!-- verify-citation: not primarily ATLAS-mapped -->`.

- **Regulatory classification change on care-setting expansion.**
  _Audience read:_ If the hospital deploys CarePilot in a new care
  setting (ICU) or a new patient population (paediatric), the regulatory
  classification may change and additional obligations may apply.
  _Harm-model anchor:_ `H-RG-01`.
  _Risk-register anchor:_ `risk-register/2025/CarePilot.yaml#risks[regulatory-classification]`.
  _Evidence_: counsel determination (current scope).
  _Current mitigation_: T2 escalation trigger in the CarePilot team charter forces a council-and-counsel review before any care-setting or population change.
  _Residual risk_: zero within the pinned scope; non-zero on any extension.
  _Framework tags_: `RMF: GOVERN.1.1`; `OWASP ML Top 10: <!-- verify-citation: governance-related entry -->`; `MITRE ATLAS: <!-- verify-citation: not primarily ATLAS-mapped -->`.

- **Operational — single-provider dependency.**
  _Audience read:_ The system depends on a single inference provider and
  a single managed vector-store provider; a provider-side outage or
  policy change could degrade service or change the trust boundary.
  _Harm-model anchor:_ `H-OP-01` (supply-chain / third-party).
  _Risk-register anchor:_ `risk-register/2025/CarePilot.yaml#risks[supply-chain]`.
  _Evidence_: availability metrics reported quarterly; SLA tracking.
  _Current mitigation_: incident playbook has a provider-outage runbook; vendor contracts carry breach-notification terms.
  _Residual risk_: non-zero; a secondary-provider evaluation is on the roadmap.
  _Framework tags_: `RMF: MAP.4.1`; `OWASP ML Top 10: <!-- verify-citation: supply-chain entry -->`; `MITRE ATLAS: <!-- verify-citation: ML supply chain compromise technique ID -->`.

- **Operational — guardrail drift in production.**
  _Audience read:_ Guardrails (refusal on out-of-policy queries, PII scrub
  on structured outputs) can drift as the underlying model, the retrieval
  corpus, or the user-input distribution changes.
  _Harm-model anchor:_ `H-GR-01` (guardrail effectiveness).
  _Risk-register anchor:_ `risk-register/2025/CarePilot.yaml#risks[guardrail-drift]`.
  _Evidence_: guardrail false-refusal and refusal-under-adversarial-prompt metrics reported monthly (mod-108).
  _Current mitigation_: monthly guardrail drift monitor; threshold update process under mod-109.
  _Residual risk_: non-zero; drift is a continuous phenomenon.
  _Framework tags_: `RMF: MEASURE.3.1`; `OWASP ML Top 10: <!-- verify-citation: guardrail-evasion entry -->`; `MITRE ATLAS: <!-- verify-citation: ML model evasion technique ID -->`.

- **Operational — incident-response readiness decay.**
  _Audience read:_ The incident-response readiness (on-call coverage,
  regression-fixture breadth, post-mortem review cadence) can decay in
  periods of low incident volume.
  _Harm-model anchor:_ `H-IR-01` (incident readiness).
  _Risk-register anchor:_ `risk-register/2025/CarePilot.yaml#risks[incident-readiness]`.
  _Evidence_: mod-111 playbook; quarterly readiness drill; regression-fixture green-rate reported monthly.
  _Current mitigation_: mandatory quarterly readiness drill; regression-fixture addition on every near-miss.
  _Residual risk_: non-zero; the drill cadence is the control.
  _Framework tags_: `RMF: MANAGE.2.4`; `OWASP ML Top 10: <!-- verify-citation: not primarily OWASP-mapped -->`; `MITRE ATLAS: <!-- verify-citation: not primarily ATLAS-mapped -->`.

## 4. Regulatory classification (counsel-owned)

Counsel owns this section; the team supplies facts. The current pinned
classification is recorded in `legal-register/CarePilot/2025.yaml` and
is reviewed quarterly by counsel. Any T2-triggering change (new
jurisdiction, new care setting, new patient population, new user class)
forces a new determination before the change lands.

## 5. Known limitations

- The system is scoped to a **single hospital, single jurisdiction, single
  care setting, single user class**. Any extension is out of scope at v1.1
  and triggers the charter's T1 or T2.
- Clinical-safety outcome metrics are **leading indicators** (groundedness,
  override rate, near-miss queue); long-horizon outcome-level metrics
  (patient outcomes correlated to triage-support use) are not in scope at
  this lifecycle stage and are owned by the hospital quality-and-safety
  committee.

## 6. What this slice does not cover

- Model-level risks: see the model card.
- Dataset-level risks: see the dataset card.

## 7. Cross-card references

- Model card: `model-card/carepilot-v1.1-model-card.risk-slice.md` at this release tag.
- Dataset card: `dataset-card/carepilot-discharge-notes-synthetic-v1.dataset-card.risk-slice.md` at this release tag.
- Harm model, risk register, legal register: pinned above.

## 8. Attestations

- AI risk engineer — attested on `<date>`.
- Tech lead / engineering manager — attested on `<date>`.
- AI evaluation engineer — attested on `<date>`.
- AI infra / MLOps engineer — attested on `<date>`.
- Clinical-safety officer — attested on `<date>`.
- Privacy officer — attested on `<date>`.
- Head of communications — attested on `<date>`.
- Legal counsel (§4 only) — attested on `<date>`.
```

### 2.5 Dataset-card risk slice — `dataset-card/carepilot-discharge-notes-synthetic-v1.dataset-card.risk-slice.md`

```markdown
---
document_class: dataset-card.risk-slice
dataset: carepilot-discharge-notes-synthetic
dataset_version: v1
dataset_version_commit: "d803…11a2"
effective_from: 2026-11-05
author_role: AI risk engineer
reviewer_roles:
  - AI evaluation engineer
  - Data governance owner
  - Privacy officer
  - Clinical-safety officer
status: pre-release-draft
harm_model_pin: "docs/systems/carepilot/harm-model.md @ b19e…4f30"
register_pin: "risk-register/2025/CarePilot.yaml @ 2026-11-14"
---

# CarePilot — Discharge-Notes Synthetic Dataset — Risk Slice (v1)

## 1. Scope of this slice

This slice documents **dataset-level** risks for the synthetic
discharge-notes evaluation dataset that is used to evaluate the
document-ingestion modality. It does **not** document model-level risks
or product-deployment risks (see the model and system cards).

## 2. What the dataset is

- **Purpose.** An evaluation dataset for the discharge-note ingestion
  path. Not a training dataset. Used to compute groundedness,
  retrieval-alignment, adversarial attack-success-rate, and PII-leak on
  the document-ingestion path.
- **Composition.** Synthesised discharge-note PDFs modelled on the
  structure of the hospital's EHR export format, with no PHI from any
  real patient. Includes a benign probe set, an adversarial probe set
  (document-payload prompt injections), and a stale-protocol-citation
  probe set.
- **Size and sources.** v1 contains the counts recorded in
  `datasets/carepilot-discharge-notes-synthetic/v1/manifest.yaml`;
  sources are entirely synthetic.
- **Licensing.** Internal-use only; not redistributable. A per-release
  manifest records the licence of every bundled resource.

## 3. Known dataset-level risks

- **Synthetic distribution drift from real EHR output.**
  _Audience read:_ The synthetic data is modelled on the hospital's EHR
  export format; the model's production performance on real discharge
  notes may differ.
  _Harm-model anchor:_ `H-DQ-01` (dataset–deployment distribution gap).
  _Risk-register anchor:_ `risk-register/2025/CarePilot.yaml#risks[dataset-distribution-gap]`.
  _Evidence_: dataset-to-production distribution comparison report `datasets/carepilot-discharge-notes-synthetic/v1/distribution-comparison.md`.
  _Current mitigation_: the production shadow-eval cohort is used to validate dataset coverage; a 60-day post-go-live re-measurement is scheduled.
  _Residual risk_: non-zero; the mitigation is monitoring, not elimination.
  _Framework tags_: `RMF: MEASURE.2.3`; `OWASP ML Top 10: <!-- verify-citation: data-quality entry -->`; `MITRE ATLAS: <!-- verify-citation: not primarily ATLAS-mapped -->`.

- **Adversarial-probe coverage gap.**
  _Audience read:_ The adversarial probe set exercises the known failure
  mode (document-payload prompt injection); it cannot cover unknown
  variants. Attack-success-rate on the known class is in the range 1–3%.
  _Harm-model anchor:_ `H-PI-03`.
  _Risk-register anchor:_ `risk-register/2025/CarePilot.yaml#risks[prompt-injection]`.
  _Evidence_: adversarial probe-set manifest; mod-104 methodology; coverage note at `datasets/carepilot-discharge-notes-synthetic/v1/adversarial-coverage.md`.
  _Current mitigation_: probe-set is updated every red-team cycle; unknown-variant discovery pathways (near-miss hook, incident back-feed) are in place.
  _Residual risk_: non-zero and unknown-variant-bounded.
  _Framework tags_: `RMF: MEASURE.2.7`; `OWASP ML Top 10: <!-- verify-citation: prompt-injection entry -->`; `MITRE ATLAS: <!-- verify-citation: prompt-injection via document ingestion technique ID -->`.

- **Subgroup representation.**
  _Audience read:_ The synthetic discharge notes are seeded from the
  hospital's ED patient-mix distribution; subgroups that are rare in the
  real cohort are rare in the synthetic set. Statistical power on small
  subgroups is correspondingly lower.
  _Harm-model anchor:_ `H-FR-01` (fairness representation).
  _Risk-register anchor:_ `risk-register/2025/CarePilot.yaml#risks[fairness-triage]`.
  _Evidence_: subgroup-count table at `datasets/carepilot-discharge-notes-synthetic/v1/subgroup-counts.md`.
  _Current mitigation_: stratified sampling against the hospital's recorded ED patient-mix; up-sampling for subgroups below a minimum-count floor; limitation stated in the model card's fairness row.
  _Residual risk_: statistical uncertainty on small subgroups; same as the model-card row.
  _Framework tags_: `RMF: MEASURE.2.11`; `OWASP ML Top 10: <!-- verify-citation: bias-related entry -->`; `MITRE ATLAS: <!-- verify-citation: not primarily ATLAS-mapped -->`.

- **Synthetic-PII hygiene.**
  _Audience read:_ Although the dataset contains no PHI by construction,
  synthetic PII can inadvertently approximate real individuals (named
  entity collisions, re-identification via combination).
  _Harm-model anchor:_ `H-PR-03` (synthetic-PII hygiene).
  _Risk-register anchor:_ `risk-register/2025/CarePilot.yaml#risks[synthetic-pii]`.
  _Evidence_: PII-generation audit at `datasets/carepilot-discharge-notes-synthetic/v1/pii-audit.md`.
  _Current mitigation_: name-list generation against a published synthetic-identity schema; re-identification audit before release; privacy-officer sign-off.
  _Residual risk_: low; residual risk documented in the audit file.
  _Framework tags_: `RMF: MEASURE.2.10`; `OWASP ML Top 10: <!-- verify-citation: sensitive-information entry -->`; `MITRE ATLAS: <!-- verify-citation: not primarily ATLAS-mapped -->`.

- **Provenance and licensing.**
  _Audience read:_ The dataset is fully synthetic; no external corpus is
  bundled. A manifest records every bundled resource with a licence
  pointer.
  _Harm-model anchor:_ `H-DQ-02` (dataset licensing).
  _Risk-register anchor:_ `risk-register/2025/CarePilot.yaml#risks[dataset-license]`.
  _Evidence_: `datasets/carepilot-discharge-notes-synthetic/v1/manifest.yaml`.
  _Current mitigation_: manifest discipline at release; counsel-owned licence review on any bundled external resource.
  _Residual risk_: zero within the pinned manifest; non-zero on any future addition of an externally-sourced resource (which triggers a new counsel review).
  _Framework tags_: `RMF: GOVERN.1.1`; `OWASP ML Top 10: <!-- verify-citation: data-provenance entry -->`; `MITRE ATLAS: <!-- verify-citation: ML supply-chain technique ID -->`.

## 4. Known limitations

- The dataset is **evaluation-only**; it is not training data.
- The dataset is **synthetic**; production performance is monitored
  against real traffic and is reported in the system-card risk slice.
- The dataset is **versioned**; downstream card artifacts cite the
  dataset commit hash.

## 5. What this slice does not cover

- Model-level behaviour (see the model card).
- Deployment-context risks (see the system card).

## 6. Cross-card references

- Model card: `model-card/carepilot-v1.1-model-card.risk-slice.md` at this release tag.
- System card: `system-card/carepilot-v1.1-system-card.risk-slice.md` at this release tag.
- Harm model and risk register: pinned above.

## 7. Attestations

- AI risk engineer — attested on `<date>`.
- AI evaluation engineer — attested on `<date>`.
- Data governance owner — attested on `<date>`.
- Privacy officer — attested on `<date>`.
- Clinical-safety officer — attested on `<date>`.
```

### 2.6 Cross-card consistency check — `checks/cross_card_consistency.md`

The three cards must stay consistent with each other and with the harm model / register at every release. The check is a reviewer workflow (not a full-blown linter), but every step is mechanical.

```markdown
# Cross-card consistency check — CarePilot (any release)

Run before a release gate opens. Each step produces a pass/fail.

1. **Harm-model pin match.** All three cards' frontmatter `harm_model_pin`
   fields are identical and resolve to the same commit of the harm model.

2. **Register pin match.** All three cards' frontmatter `register_pin`
   fields are identical.

3. **Row anchors resolve.** Every row's `Harm-model anchor` ID exists in
   the pinned harm model. Every row's `Risk-register anchor` exists in
   the pinned register.

4. **Slice assignment correct.**
   - Rows in the model-card slice map to harm-model rows about the model
     (model behaviour, model evaluation, model metrics).
   - Rows in the system-card slice map to harm-model rows with a
     deployment-context precondition.
   - Rows in the dataset-card slice map to harm-model rows about the
     dataset (quality, coverage, provenance, licensing).

5. **No same-row in two slices.** A given harm-model row anchor is the
   primary anchor in at most one slice. Secondary references from a
   sister slice are via the "What this slice does not cover" section and
   the cross-card reference list.

6. **Residual-risk statements present and non-zero.** For every row in
   every slice, a residual-risk sentence exists and does not read "zero".
   The two acceptable zero-residual statements are (a) "zero within the
   pinned scope" (as in the system card §3 regulatory row) and (b)
   "zero within the pinned manifest" (as in the dataset card §3
   licensing row). Both carry an explicit statement that an extension
   would re-open the row.

7. **Framework tags present.** Every row in every slice has an `RMF`
   tag, an `OWASP ML Top 10` tag, and a `MITRE ATLAS` tag. Where the
   exact code has not been verified from the official source, the tag
   carries a `<!-- verify-citation: ... -->` marker rather than a
   guessed code.

8. **Evidence pins resolve.** Every row's `Evidence` pointer resolves to
   a file at the pinned version/commit. Dangling evidence pointers are a
   release-blocking defect.

9. **Attestations at role level.** All three cards' attestations list is
   at role level; personal names are in the internal attestation record,
   not in the card.

10. **Date coherence.** All three cards' `effective_from` dates are
    consistent with the release-gate date; supersession is handled by
    the `supersedes` field.

Any step that fails is logged as a card-coherence defect and blocks the
release gate until resolved.
```

## 3. Validation steps

1. **Three cards exist.** Model-card, system-card, and dataset-card risk slices are present at the pinned version.
2. **Slice assignment correct.** For each of three sample harm-model rows, the reviewer confirms the row is referenced as the primary anchor in exactly one slice.
3. **Every row follows the shape.** A reviewer picks three rows at random from each card and confirms the authoring convention's five lines (harm anchor, register anchor, evidence, mitigation, residual, tags) are all populated.
4. **No "zero-risk" residual.** A reviewer greps for `Residual risk: zero` across all three cards; hits must be qualified ("zero within the pinned scope", "zero within the pinned manifest") and must state the re-opening trigger.
5. **Framework tags present.** Every row has `RMF`, `OWASP ML Top 10`, and `MITRE ATLAS` tags. Unverified codes are marked.
6. **Pins resolve.** `harm_model_pin`, `register_pin`, `model_version_commit`, and `dataset_version_commit` resolve to files at the pinned revision.
7. **Cross-card consistency check runs clean.** The ten-step check in §2.6 produces no fails.
8. **Attestations at role level.** All three cards' attestations list roles, not personal names.
9. **False precision removed.** Numerical metrics are ranges where the underlying data supports ranges; precise numbers appear only where they are supportable from the evidence file.
10. **Counsel section flagged.** The system card's §4 (regulatory classification) is explicitly counsel-owned and refers to `legal-register/CarePilot/2025.yaml`; no article numbers or deadlines are invented.

## 4. Rubric or review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Three cards present | Model, system, dataset risk slices all present at the pinned version | Any card missing |
| R2 | Slice assignment correct | Each harm-model row has one primary slice | Same row primary-anchored in two slices |
| R3 | Authoring convention followed | Every row has the five lines (anchor, anchor, evidence, mitigation, residual, tags) | Row missing any of the five |
| R4 | Residual risk honest | Every row states a residual that is non-zero or qualified | "Zero risk" without qualification |
| R5 | Framework tags present | Every row has RMF + OWASP ML Top 10 + ATLAS tags; unverified codes marked | Missing tag or invented code without marker |
| R6 | Pins resolve | All frontmatter pins resolve to files | Dangling pin |
| R7 | Cross-card consistency check clean | Ten-step check in §2.6 passes | Any fail |
| R8 | Attestations at role level | Role-level attestations in each card | Personal names in the card |
| R9 | False precision removed | Ranges used where underlying data supports a range | Spurious precise numbers |
| R10 | Counsel section flagged | System card §4 is counsel-owned and refers to the legal register | Invented article numbers or deadlines |
| R11 | Cross-card references bidirectional | Each card's cross-card section lists the other two with the same release tag | One-way or inconsistent cross-references |
| R12 | Harm-model-and-register as single source of truth | No risk statement in any card that is not anchored in the harm model and the register | Card asserts a risk that is not in the register |

Scoring bands: *Pass* — R1–R11 green and R12 honoured; *Pass with revisions* — R9 fails but R1–R8 pass; *Fail* — any of R1 / R2 / R4 / R7 / R12 fails.

## 5. Common mistakes

- **Writing one card to rule them all.** A single document that tries to be all three fails every audience. Produce three cards.
- **Reproducing the harm model in the card.** The card becomes the authoritative source and drifts from the register. The card is a projection of the harm model and the register; it is not the source.
- **Marketing voice.** "State-of-the-art", "robust", "industry-leading" in a risk slice is a red flag. Risk slices are honest about what could go wrong.
- **Zero-residual claims.** "No risk" is almost always wrong. Qualify ("zero within the pinned scope") and state the re-opening trigger.
- **Slice drift.** The model card picks up deployment-context rows; the system card picks up model-architecture rows. Enforce the slice assignment with the §2.6 check.
- **Framework tags as an appendix.** Keeping tags off-row makes auditing a context-switch. Keep tags inline on each row.
- **Invented framework codes.** Writing "RMF: GOVERN 4.3" when the code was not verified is a professional error. Use `<!-- verify-citation: ... -->` until confirmed.
- **Personal names in attestations.** Role-level attestations survive attrition. Personal names live in the internal attestation record.
- **No dates on attestations.** An attestation without a date is not an attestation.
- **Cross-card references drift.** The model card cites the system card at v1.0 while the system card is at v1.1. Enforce release-tag match in the §2.6 check.
- **Dataset card covers model behaviour.** The dataset card drifts into "the model can be jailbroken on this dataset" (model-card content). The dataset card is about the dataset; the vulnerability is model-side.
- **System card invents legal citations.** The system card writes "we are regulated under Article X of the AI Act" without counsel. Counsel owns the regulatory classification section; the card points to the legal register.

## 6. References

Only the official sources listed in the work packet (and clearly-labelled local module references) are used.

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — the risk slices use RMF **GOVERN / MAP / MEASURE / MANAGE** sub-categories as framework tags on each row, so that an auditor can trace a card row to a framework position. <!-- verify-citation: the exact sub-category codes most precisely matching model-behaviour risks (MEASURE.2.x family) and deployment-context risks (MANAGE.2.x / 4.x family) should be re-read from AI 100-1 §5 before pinning the tags. -->
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — card rows carry OWASP ML Top 10 tags for categories the row corresponds to (prompt injection, output integrity, data quality, sensitive-information disclosure, supply-chain). <!-- verify-citation: cite the exact OWASP ML Top 10 entry codes (and OWASP Top 10 for LLM Applications entries where applicable) before pinning specific codes. -->
- **MITRE ATLAS.** `https://atlas.mitre.org/` — card rows carry ATLAS technique tags where the row corresponds to an ATLAS-mapped attack (prompt injection via document ingestion, ML supply-chain compromise, model evasion). <!-- verify-citation: cite the exact ATLAS technique IDs before pinning specific IDs. -->
- **Module running system.** CarePilot, as introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`.
- **Preceding module artifacts the cards anchor to.**
  - `lessons/mod-102-harm-modelling-and-risk-taxonomy/` — harm model.
  - `lessons/mod-103-risk-quantification-and-registers/` — risk register.
  - `lessons/mod-104-llm-red-teaming-and-attack-engineering/` — red-team suite.
  - `lessons/mod-106-fairness-bias-and-explainability-engineering/` — fairness evaluation.
  - `lessons/mod-107-privacy-risk-engineering/` — privacy evaluation.
  - `lessons/mod-108-guardrail-engineering-and-effectiveness/` — guardrail evaluation.
  - `lessons/mod-109-continuous-monitoring-and-risk-register-integration/` — production monitors.
  - `lessons/mod-111-incident-response-and-postmortems/` — incident playbook, near-miss handling, regression fixtures.
- **This module's charter deliverable.** `modules/mod-112-cross-functional-ai-risk-program-and-communication/exercise-01-risk-program-charter-for-one-team/SOLUTION.md` — T1 / T2 escalation triggers.
- **Exercise prompt.** `lessons/mod-112-cross-functional-ai-risk-program-and-communication/exercises/exercise-03-model-card-risk-slice-authoring.md`.

<!-- The risk slices are engineering-authored. Regulatory-classification
     wording in the system card is counsel-owned; specific framework codes
     are deferred to verification against the current official sources
     via `<!-- verify-citation: ... --` markers rather than guessed. -->

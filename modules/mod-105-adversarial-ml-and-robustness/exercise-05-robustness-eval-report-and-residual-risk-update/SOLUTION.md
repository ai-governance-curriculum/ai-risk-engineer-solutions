# Exercise 05 — Robustness-Eval Report and Residual-Risk Register Update — Reference Solution

## 1. Solution overview

This reference walks a worked `mod-105-ex05-robustness-report/` for **CarePilot** — a signed release-gate artifact that rolls up the results of mod-105 ex-02, ex-03, and ex-04 (plus the cross-tag from ex-01 and the LLM red-team suite from mod-104) into a single robustness-eval report and a corresponding diff to the CarePilot risk register. The deliverable has two files:

1. `robustness-eval-report.md` — a release-gate report structured as (A) threat-model recap, (B) per-attack-class results with before/after numbers and envelope comparisons, (C) defence posture summary, (D) gates (pass / pass-with-conditions / fail per row), and (E) the explicit residual-risk list that feeds the register update.
2. `risk-register-diff.yaml` — the YAML diff to apply to `risk-register/2025/CarePilot.yaml` for the adversarial-ML rows. This is produced by the panel, reviewed by the AI risk council, and signed.

Design decisions:

- **The report is the only release-gate artifact that spans all five exercises.** Ex-01 produced the cross-tag; ex-02, ex-03, ex-04 produced measurements; ex-05 produces the *gate*. If a reviewer reads nothing else in mod-105, they read this.
- **Every row in the register diff is backed by a measurement.** No row claims "mitigated" without an attack-run artifact that shows post-defence numbers meeting the envelope. No row claims "accepted" without a sign-off from the clinical-safety officer and the AI risk council chair.
- **Residual risk is explicit and durable.** Rows that do not meet the envelope do not silently reset — they become residual-risk rows with owner, due date, and the monitor (mod-109) that will track them.
- **The report writes to NIST AI RMF MEASURE 2.7.** The CarePilot mod-101 ex-01 map named the sub-category; this report is the evidence it cites.
- **Gates are pass / pass-with-conditions / fail per attack row, not a single overall verdict.** The release-gate pipeline (mod-109) composes row verdicts; a single overall "red" verdict would hide which attack class failed.
- **No invented envelope numbers.** The register-envelope floors are either cited from `risk-register/2025/CarePilot.yaml` or marked `<!-- submission-todo: ... -->`, which blocks auto-merge and forces the submitter to pull the register value before sign-off.

## 2. Implementation — worked artifact

### 2.1 `robustness-eval-report.md`

```markdown
---
system: CarePilot (hospital triage-support RAG assistant, HITL)
release: carepilot-2026.04.00-rc.1   # illustrative — the pipeline stamps this
report_id: carepilot-robustness-2026-04-001
author: ai-risk-engineer (L25)
reviewers: ai-evaluation-engineer (L35), security-learning / ai-infra-security (L35), clinical-safety-officer, ai-risk-council (chair)
inputs:
  cross_tag: modules/mod-105-adversarial-ml-and-robustness/exercise-01-nist-ai-100-2-to-atlas-to-owasp-cross-tag/SOLUTION.md
  evasion_panel_run: artifacts/mod-105-ex02/aggregated.json
  poisoning_panel_run: artifacts/mod-105-ex03/aggregated.json
  extraction_panel_run: artifacts/mod-105-ex04/aggregated.json
  llm_red_team_run: artifacts/mod-104/.../aggregated.json
rmf_subcategory: MEASURE 2.7
status: draft-for-release-gate
---

# CarePilot — Robustness-Eval Report (Release {release})

## A. Threat-model recap

CarePilot's adversarial-ML exposure is cross-tagged in `mod-105-ex01-cross-tag.md`. For this release, the in-scope rows are:

| Row | Attack class | CarePilot applicability | Exercise that measured it |
|---|---|---|---|
| E1, E2 | Evasion — PredAI (white-box ceiling + black-box realistic) | CONDITIONAL — auxiliary classifier only | ex-02 |
| E3, E4 | Evasion — GenAI — prompt injection (direct + indirect, RAG) | IN | mod-104 red-team suite |
| E5 | Evasion — NLP | CONDITIONAL — intake classifier (disabled this release) | ex-02 (reports clean, not released) |
| P1, P2 | Poisoning — targeted + availability | CONDITIONAL — auxiliary classifier only | ex-03 |
| P3 | Backdoor / trojan | CONDITIONAL — auxiliary classifier only | ex-03 |
| P4 | Supply-chain (ML) | IN — procurement scope | mod-110 |
| X1 | Extraction — scaffolding / corpus | CONDITIONAL — IN for the scaffolding and corpus; vendor for base LLM | ex-04 |
| X2, X3 | MIA + attribute inversion | CONDITIONAL — auxiliary classifier only | ex-04 |
| X5 | System-prompt leakage | IN | mod-104 |
| A1 | Sponge / DoS | CONDITIONAL — only if public-facing; CarePilot is SSO-gated internal | ex-04 (rate-limit measurement) |
| A2 | Abuse / out-of-scope use | IN | mod-104 + mod-108 |

Base LLM extraction/MIA/inversion is a vendor concern (cross-referenced only; this report does not bind a verdict on vendor-side exposures).

## B. Per-attack-class results

Each row quotes (i) the threat model, (ii) the observed metric on the defended endpoint (or on the current build, for training-time attacks), (iii) the register envelope floor, (iv) the row verdict.

### B.1 Evasion — PredAI

| Attack | Threat model | Metric | Observed | Envelope floor | Verdict |
|---|---|---|---|---|---|
| ART PGD (white-box ceiling) | L∞, ε = 0.1, 40 iter | ASR | {ex02.art-pgd.asr:.3f} | *ceiling — informational* | — |
| ART HopSkipJump (black-box, realistic) | L2, max_eval = 1000 | ASR | {ex02.art-hsj.asr:.3f} | `adv-evasion-predai#envelope.black_box_asr_cap` | {verdict} |
| ART Square (black-box, score) | L∞, ε = 0.1, iter = 5000 | ASR | {ex02.art-sq.asr:.3f} | same | {verdict} |

### B.2 Evasion — NLP (if intake classifier is released)

| Attack | Threat model | Metric | Observed | Envelope floor | Verdict |
|---|---|---|---|---|---|
| TextFooler (TA) | black-box score, query_cap = 500, default constraints | ASR | {ex02.ta-tf.asr:.3f} | `adv-evasion-nlp#envelope.asr_cap` | {verdict} |
| PWWS (TA) | same | ASR | {ex02.ta-pw.asr:.3f} | same | {verdict} |
| DeepWordBug (TA) | same | ASR | {ex02.ta-dw.asr:.3f} | same | {verdict} |
| BAE (TA) | same | ASR | {ex02.ta-bae.asr:.3f} | same | {verdict} |

(This section is marked **not released** when the intake classifier is disabled.)

### B.3 Evasion — GenAI — prompt injection (cross-ref mod-104)

| Attack family | Source | Metric | Observed | Envelope floor | Verdict |
|---|---|---|---|---|---|
| Direct prompt injection | garak + PyRIT | attack-success-rate | {mod104.direct.asr:.3f} | `prompt-injection-direct#envelope.asr_cap` | {verdict} |
| Indirect prompt injection (RAG) | AgentDojo-style | attack-success-rate | {mod104.indirect.asr:.3f} | `prompt-injection-indirect#envelope.asr_cap` | {verdict} |

### B.4 Training-time attacks (ex-03, auxiliary classifier only)

| Attack | Metric | Observed | Envelope floor | Verdict |
|---|---|---|---|---|
| Label-flip availability (5%) | clean-acc delta vs baseline | {ex03.label_flip.delta:+.3f} | `data-poisoning-availability#envelope.clean_acc_drop_cap` | {verdict} |
| Clean-label targeted (2%) | targeted flip rate | {ex03.clean_label.targeted_flip_rate:.3f} | `data-poisoning-targeted#envelope.flip_rate_cap` | {verdict} |
| BadNets backdoor (3%) | backdoor ASR | {ex03.badnets.backdoor_asr:.3f} | `backdoor-trojan#envelope.asr_cap` | {verdict} |

Detection-side (development-time) metrics for the baselines are recorded in the ex-03 run artifact; they are informational here — a defence that catches 90% of poisoning in dev does not guarantee runtime safety. The register row is sized on the attack-success side, with the detection recall as a confidence modifier.

### B.5 Privacy attacks — extraction, MIA, inversion (ex-04)

| Attack | Metric | Undefended | Defended | Envelope floor | Verdict |
|---|---|---|---|---|---|
| KnockoffNets extraction | substitute agreement | {ex04.kn.undef.agreement:.3f} | {ex04.kn.def.agreement:.3f} | `extraction-scaffolding#envelope.agreement_cap` | {verdict} |
| MembershipInferenceBlackBox | attack AUC | {ex04.mia.undef.auc:.3f} | {ex04.mia.def.auc:.3f} | `mia-local-head#envelope.auc_cap` | {verdict} |
| AttributeInferenceBlackBox | delta vs prior | {ex04.ai.undef.delta:+.3f} | {ex04.ai.def.delta:+.3f} | `inversion-local-head#envelope.delta_cap` | {verdict} |
| System-prompt canary (mod-104) | leak rate | — | {mod104.sp.rate:.3f} | `system-prompt-leakage#envelope.rate_cap` | {verdict} |

### B.6 Availability — sponge / DoS

Rate-limit fire-rate during the extraction attack run was {ex04.rl_user.fire_rate:.3f} (per-user) and {ex04.rl_ip.fire_rate:.3f} (per-IP). CarePilot is SSO-gated inside the hospital network; an unauthenticated sponge adversary is not plausible under the current deployment envelope. Register row `dos-sponge` remains CONDITIONAL.

## C. Defence posture summary

| Defence | Layer | Attack rows protected | Where it lives |
|---|---|---|---|
| Input validation + prompt-injection guardrails | mod-108 | E3, E4, X5 | `src/carepilot/guardrails/` |
| Rate limit (per-user + per-IP) | serving | E2, X1, X2, X3, A1 | mod-105 ex-04 panel |
| Output restriction (top-1 + binned confidence) | serving | X1, X2, X3 | mod-105 ex-04 panel |
| PRADA-style extraction detector | serving | X1, A1 | mod-105 ex-04 panel |
| DP-SGD training (retrained head) | training | X2, X3 | mod-105 ex-04 training config; `(ε, δ)` from register |
| Red-team regression fixtures | CI | E3, E4, X5 | `regressions/carepilot/` |
| Poison-detection (spectral + activation clustering + trigger reconstruction) | dev-time | P1, P2, P3 | mod-105 ex-03 |
| Supply-chain attestation + SBOM | procurement | P4 | mod-110 |
| Monitor plan (drift, HITL override rate, rate-limit fires) | runtime | all runtime-signal rows | mod-109 |

## D. Gates

The release-gate is `pass` only if every row verdict in B is `pass` or `pass-with-conditions`. One or more `fail` rows blocks the release. `pass-with-conditions` rows must carry an OPEN item in section E with owner and due date.

Verdict rule per row:

- **Pass** — observed ≤ envelope floor.
- **Pass with conditions** — observed is within a defined grace band above the envelope floor AND a mitigation is already open in the register with a due date before the next release.
- **Fail** — observed exceeds the floor and the grace band, or exceeds the floor with no mitigation in flight.

## E. Residual risk (register-update input)

Rows below do not meet the envelope cleanly and generate a diff to `risk-register/2025/CarePilot.yaml`.

| Register row | Current state | New state | Owner | Due | Monitor |
|---|---|---|---|---|---|
| adv-evasion-predai | OPEN — envelope exceeded by HopSkipJump at {ex02.art-hsj.asr:.3f} | OPEN with mitigation plan: enable output restriction on the triage classifier endpoint; retrain under DP-SGD for the next release | ai-risk-engineer (L25) | next release | mod-109 drift + rate-limit fires |
| data-poisoning-targeted | OPEN — clean-label flip rate at {ex03.clean_label.targeted_flip_rate:.3f} is above envelope | OPEN with mitigation: feature-space activation audit on every retrain; PR check that activation-clustering silhouette is above a floor | ai-risk-engineer (L25) + ai-eval-engineer (L30) | next retrain | mod-109 training-set audit |
| system-prompt-leakage | Baseline | Add canary regression fixture (mod-104) that must stay green on every release | ai-risk-engineer (L25) | next release | mod-109 red-team suite regression |
| dp-sgd-target | <!-- submission-todo: register has no (ε, δ) target yet; set in Q-review before next DP-trained release --> | PENDING register decision | AI risk council + clinical safety | next quarterly register review | — |

(The actual numbers above are substituted from the three panel runs at release time.)

## F. Sign-offs

| Role | Name | Signed |
|---|---|---|
| ai-risk-engineer (L25), author | — | — |
| ai-evaluation-engineer (L35) | — | — |
| security-learning / ai-infra-security (L35) | — | — |
| clinical-safety-officer | — | — |
| ai-risk-council (chair) | — | — |

`<!-- submission-todo: sign-offs collected at release-gate meeting; this template records the required roles, not the names. -->`
```

### 2.2 `risk-register-diff.yaml`

```yaml
# mod-105 ex-05 — diff to apply to risk-register/2025/CarePilot.yaml
# Generated from the panel runs referenced in robustness-eval-report.md.
# Human-reviewable; applied after sign-off by the AI risk council chair.

rows:

  - id: adv-evasion-predai
    patch:
      status: OPEN
      last_review: 2026-04-xx   # <!-- submission-todo: set at release-gate meeting -->
      observed:
        hop_skip_jump_asr: ${ex02.art-hsj.asr}
        square_asr: ${ex02.art-sq.asr}
      mitigation:
        - enable output_restriction on triage classifier endpoint
        - plan DP-SGD retrain for next release (requires register (ε, δ) target)
      owner: ai-risk-engineer (L25)
      due: next-release
      monitor_rows:
        - triage-classifier-black-box-attack-rate
        - rate-limit-fire-rate

  - id: data-poisoning-targeted
    patch:
      status: OPEN
      last_review: 2026-04-xx
      observed:
        clean_label_targeted_flip_rate: ${ex03.clean_label.targeted_flip_rate}
      mitigation:
        - activation-clustering silhouette PR check on every retrain
        - restrict labeled-training-set write access (mod-110)
      owner: ai-risk-engineer (L25) + ai-eval-engineer (L30)
      due: next-retrain
      monitor_rows:
        - training-set-audit

  - id: backdoor-trojan
    patch:
      status: CLOSED-PENDING-VERIFY
      last_review: 2026-04-xx
      observed:
        badnets_asr_on_clean_eval: ${ex03.badnets.backdoor_asr}   # trigger-reconstruction defence did or did not fire
        trigger_reconstruction_flagged_target: ${ex03.badnets.trigger_reconstruction.flagged_targets}
      mitigation:
        - trigger-reconstruction scan run on every retrain
      verification_cadence: every retrain

  - id: extraction-scaffolding
    patch:
      status: ${verdict-derived-from-defended-vs-envelope}
      last_review: 2026-04-xx
      observed:
        knockoff_substitute_agreement_undefended: ${ex04.kn.undef.agreement}
        knockoff_substitute_agreement_defended: ${ex04.kn.def.agreement}
      mitigation:
        - rate-limit (per-user + per-IP)
        - PRADA-style detector
        - output-restriction (top-1 + binned confidence)

  - id: mia-local-head
    patch:
      status: ${verdict-derived-from-defended-vs-envelope}
      last_review: 2026-04-xx
      observed:
        attack_auc_undefended: ${ex04.mia.undef.auc}
        attack_auc_defended: ${ex04.mia.def.auc}
      mitigation:
        - output-restriction
        - DP-SGD retrain (pending (ε, δ) target)

  - id: inversion-local-head
    patch:
      status: ${verdict-derived-from-defended-vs-envelope}
      last_review: 2026-04-xx
      observed:
        delta_vs_prior_undefended: ${ex04.ai.undef.delta}
        delta_vs_prior_defended: ${ex04.ai.def.delta}
      mitigation:
        - output-restriction
        - DP-SGD retrain (pending (ε, δ) target)

  - id: prompt-injection-direct
    patch:
      status: ${from-mod-104}
      last_review: 2026-04-xx
      observed: ${from-mod-104.direct.asr}
      monitor_rows:
        - red-team-regression-fixtures
        - runtime-injection-detector

  - id: prompt-injection-indirect
    patch:
      status: ${from-mod-104}
      last_review: 2026-04-xx
      observed: ${from-mod-104.indirect.asr}

  - id: system-prompt-leakage
    patch:
      status: ${from-mod-104}
      last_review: 2026-04-xx
      observed: ${from-mod-104.sp.rate}
      mitigation:
        - canary regression fixture (added)

  - id: dos-sponge
    patch:
      status: CONDITIONAL
      last_review: 2026-04-xx
      observed:
        rate_limit_fire_rate_per_user: ${ex04.rl_user.fire_rate}
        rate_limit_fire_rate_per_ip: ${ex04.rl_ip.fire_rate}
      condition: deployment envelope stays SSO-gated and internal

new_rows:
  - id: dp-sgd-target
    status: PENDING
    description: |
      CarePilot retrained-head DP-SGD training (ε, δ) target not yet set.
      Decision owner: AI risk council + clinical safety.
      Dependencies: utility floor on triage accuracy; member set definition.
    due: next-quarterly-register-review

removed_rows: []

signatures:
  - role: ai-risk-engineer (L25), author
  - role: ai-evaluation-engineer (L35)
  - role: security-learning / ai-infra-security (L35)
  - role: clinical-safety-officer
  - role: ai-risk-council (chair)
```

## 3. Validation steps

A reviewer should be able to confirm the submission in under 20 minutes.

1. **Inputs cited.** The report's frontmatter lists exactly which run artifacts (ex-02, ex-03, ex-04, mod-104) it derived numbers from. Reviewer can open any of them.
2. **Every number is substituted, not invented.** Every `{ex0*.*}` placeholder in the report maps to a key in a referenced run's `aggregated.json`. Reviewer spot-checks three or four.
3. **Envelope floors cited.** Every row in sections B.1 through B.5 cites its `risk-register/.../envelope.*` key. Rows with `submission-todo` placeholders are flagged as blockers to release-gate pass.
4. **Verdicts follow the rule.** Each row's verdict (pass / pass-with-conditions / fail) is derivable from the "observed vs envelope floor" comparison stated in section D.
5. **Scope ruling present.** Base-LLM extraction / MIA / training-data regurgitation is cross-referenced to mod-104 and vendor assurance (mod-110); it does not bind a CarePilot verdict.
6. **Register diff matches report.** Every OPEN or PASS-WITH-CONDITIONS row in section E has a corresponding entry in `risk-register-diff.yaml`. The diff carries owner, due date, and monitor row.
7. **Signatures template present.** Section F lists the five required sign-off roles; the actual names are collected at the release-gate meeting (template leaves them blank with a `submission-todo` marker).
8. **No invented numbers.** All numeric cells are substituted from run artifacts; no row quotes a benchmark number from memory.

## 4. Rubric / review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Inputs enumerated in frontmatter | Four run-artifact paths cited | Missing or hand-waved |
| R2 | NIST AI RMF sub-category named | `MEASURE 2.7` present in frontmatter | Missing |
| R3 | Threat-model recap present | Section A reproduces the ex-01 cross-tag rows in-scope for this release | Section missing |
| R4 | Per-attack-class results | Section B has rows for evasion (PredAI, NLP where applicable, GenAI), poisoning, extraction/MIA/inversion, system-prompt leakage, DoS | Any class missing |
| R5 | Before/after columns for defended-row attacks | Extraction/MIA/inversion show undefended AND defended | Only one of the two |
| R6 | Verdict rule explicit | Section D defines pass / pass-with-conditions / fail | Verdicts given without rule |
| R7 | Residual-risk list | Section E lists every row that is not clean-pass with owner, due date, monitor | No residual list |
| R8 | Register diff matches | `risk-register-diff.yaml` has one row per Section E row | Mismatched |
| R9 | Sign-off template | Section F lists five required roles | Fewer |
| R10 | No invented numbers | Every numeric cell is a substitution variable or a `submission-todo` marker | Any hard-coded benchmark number |
| R11 | Scope ruling preserved | Base-LLM privacy exposure cross-ref'd out | Scope mixed |

**Scoring bands.**

- *Pass* — R1–R11 green.
- *Pass with revisions* — R5 or R7 or R8 fails; revise.
- *Fail* — R4, R6, R10, or R11 fails.

## 5. Common mistakes

- **Writing a single overall verdict ("red" or "green") instead of per-row verdicts.** Collapsing hides which attack class failed and which defence held. Per-row verdicts compose into the release-gate decision in mod-109.
- **Comparing undefended attack numbers to envelopes.** The defended column is the one that is compared. Undefended is reported to quantify defence lift, not to gate releases.
- **Treating development-time detection AUC as runtime defence.** Poison-detection recall measured on a known poisoning mask is development-time only. The register row for backdoor-trojan is sized on the attack side (ASR) and the detection is a confidence modifier, not a gate.
- **Hard-coding envelope numbers in the report.** Envelopes live in the register. The report cites them. If the register envelope is missing, the row is `submission-todo`, not fabricated.
- **No cross-ref to mod-104 / mod-110.** GenAI prompt injection and ML supply chain live in other modules. The report must name the external owner, not re-run the suite (which would be out of scope) and not drop the row (which would be an invented verdict).
- **Residual-risk rows with no owner or due date.** An OPEN row with no owner is invisible to the register and to mod-109. Reject.
- **Diff YAML that touches rows the report did not cover.** The YAML diff is machine-applied; it must not sneak in a new mitigation decision that was not explicitly argued in the report body.
- **DP-SGD row with hard-coded `(ε, δ)`.** Privacy budget is a policy decision; the register owns it, the report asserts compliance against it. If the register has no target yet, the row is `PENDING`, not `set-by-engineering`.
- **Sign-off section listing only engineering.** The release-gate meeting requires clinical safety and the AI risk council chair; both are visible in the mod-101 ex-01 governance structure.

## 6. References

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — MEASURE 2.7 (security and resilience).
- **NIST AI 100-2 E2023 — Adversarial Machine Learning: A Taxonomy and Terminology of Attacks and Mitigations.** `https://csrc.nist.gov/publications/detail/nistir/ai-100-2` — attack-class taxonomy that structures the report's section B.
- **MITRE ATLAS.** `https://atlas.mitre.org/` — technique IDs cited in section A table, consistent with ex-01 cross-tag.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — ML01..ML10 cross-reference per row.
- **mod-105 ex-01 solution** — cross-tag sheet.
- **mod-105 ex-02 solution** — evasion panel artifacts.
- **mod-105 ex-03 solution** — poisoning + detection panel artifacts.
- **mod-105 ex-04 solution** — extraction / MIA / inversion panel artifacts.
- **mod-104 red-team suite** — direct and indirect prompt injection, system-prompt leakage.
- **mod-101 ex-01 solution** — CarePilot RMF obligation→artifact map (MEASURE 2.7 row).
- **Exercise prompt** — `lessons/mod-105-adversarial-ml-and-robustness/exercises/exercise-05-robustness-eval-report-and-residual-risk-update.md`.

<!-- submission-todo: register-envelope floors for adv-evasion-predai, adv-evasion-nlp, data-poisoning-*, backdoor-trojan, extraction-scaffolding, mia-local-head, inversion-local-head, prompt-injection-*, system-prompt-leakage; DP-SGD (ε, δ) target; release-gate sign-off names. -->

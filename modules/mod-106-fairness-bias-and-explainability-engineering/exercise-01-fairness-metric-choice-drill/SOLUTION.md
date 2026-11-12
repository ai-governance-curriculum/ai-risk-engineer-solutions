# Exercise 01 — Fairness Metric Choice Drill — Reference Solution

## 1. Solution overview

This reference walks a worked `mod-106-ex01-metric-choice.md` for the module's fictional running system, **TalentFlow** — a résumé-screening and candidate-scoring assistant used by a large US employer to shortlist entry-level applicants. The model outputs (a) a score in `[0, 1]` and (b) a binary "advance to recruiter review" decision via a tunable threshold. Protected attributes in scope: sex and race/ethnicity per the NYC Local Law 144 categories drawn from the US EEO-1 Component 1 reporting categories, plus age under 40 vs. 40+ for EEOC ADEA scope. The same running system is carried across exercises 02–06 so the artifact set converges.

The deliverable is a single markdown document with four parts:

1. A **decision tree** that maps harm-model row → candidate metric family.
2. **Five worked scenarios** drawn from TalentFlow's harm catalogue, each picking a primary metric, a secondary / counter-metric, and an explicit **deferred** metric with a reason.
3. The **Impossibility Theorem** acknowledgement — why the drill ends in a prioritised trio, not "all metrics on".
4. A **rubric** reviewers apply row-by-row.

Design decisions:

- **TalentFlow is used.** It is the natural running system for mod-106 because (a) the exercise is about statistical-group fairness, which is where the hiring domain is best documented in NIST AI RMF 1.0's MEASURE 2.11, and (b) exercise 06 is specifically NYC LL144 + EEOC — a hiring-context law that would not fit a clinical assistant.
- **Metric names are the verbatim Fairlearn / AIF360 identifiers** where a library implementation exists. If a reviewer greps `fairlearn.metrics` or `aif360.metrics` for the identifier, they must find it. Prose definitions follow the canonical publications (Hardt et al. 2016 for equalized odds / equal opportunity; Chouldechova 2017 for predictive parity; Dwork et al. 2012 for individual fairness; Kusner et al. 2017 for counterfactual fairness).
- **Metrics are tied to harms, not chosen for convenience.** Each scenario names the harm-model row that motivates the choice (per NIST AI RMF MAP 5.1 on likelihood × magnitude framing).
- **The Impossibility Theorem is stated, not elided.** Calibration, equal false-positive rate, and equal false-negative rate are mutually incompatible when base rates differ across groups (Kleinberg–Mullainathan–Raghavan 2016 / Chouldechova 2017). The drill forces the author to *prioritise*, not to pretend all three can hold.

## 2. Implementation — worked answer — `mod-106-ex01-metric-choice.md`

```markdown
---
system: TalentFlow (résumé-screening and candidate-scoring assistant)
output: score in [0,1]; threshold → binary "advance / do-not-advance"
protected_attrs:
  - sex (M / F, per NYC LL144 §5-301 and EEO-1 Component 1)
  - race_ethnicity (per EEO-1 Component 1 categories)
  - age_band (under 40 / 40+, per EEOC ADEA)
author: ai-risk-engineer (L25)
reviewers: ai-eval-engineer (L30), employment counsel, head of talent acquisition
status: draft-for-review
---

# TalentFlow — Fairness-Metric Choice Record

## 1. Decision tree

The tree is applied once per harm-model row. The output is a *primary metric*, a *secondary / counter metric* that the primary trades against, and an *explicitly deferred* metric.

```
Is the decision an allocation of a scarce opportunity (shortlist, interview, loan)?
├── Yes → are base rates (actual qualified %) comparable across groups in reality?
│         ├── Unknown / contested → primary: Demographic Parity Difference
│         │                         secondary: Equal Opportunity Difference
│         │                         deferred:  Predictive Parity (calibration)
│         └── Known comparable     → primary: Equal Opportunity Difference (TPR)
│                                    secondary: False Positive Rate Difference
│                                    deferred:  Demographic Parity
└── No (decision is risk score fed to a human, e.g., triage severity)
          → primary: Calibration within groups (Predictive Parity)
            secondary: False Negative Rate Difference (if high-stakes miss class)
            deferred:  Demographic Parity

Side branch (used for individual-level complaints or counterfactual audits):
          → Individual fairness (consistency / Lipschitz) + counterfactual fairness
```

Rationale:

- An *allocation* decision with contested base rates (hiring, admissions, lending with a history of under-inclusion) is where EEOC's four-fifths rule and NYC LL144's impact-ratio reporting live, so the primary metric must be the allocation-rate metric those regimes use — **Demographic Parity**. Equal Opportunity is held as the secondary so the author must defend the trade.
- An *allocation* decision with reliably measured ground truth (e.g., well-calibrated credit-risk histories where default is observable within a short horizon) can prioritise **Equal Opportunity**. The counter-metric is the FPR to catch the case where equal TPR is bought by inflating false positives in the historically disadvantaged group.
- A *risk-score fed to a human* (clinical triage, recidivism risk) is where calibration matters most — a score of 0.7 should mean the same thing in every group, or clinicians/judges relying on it will get confused across subpopulations. Here **Predictive Parity** is primary.
- Every choice carries a *deferred* metric so the Impossibility Theorem is visible, not hidden.

## 2. Worked scenarios

Scenarios are drawn from TalentFlow's harm catalogue (one row each). Each scenario runs the decision tree and states the primary / secondary / deferred metric plus the Fairlearn or AIF360 identifier.

### Scenario A — TalentFlow shortlists for entry-level positions in NYC (LL144 scope)

- **Harm-model row.** Adverse-impact selection-rate disparity by sex and race/ethnicity reduces equal access to employment for protected classes.
- **Decision type.** Allocation of a scarce opportunity (interview slot).
- **Base-rate confidence.** Low — historical hiring data is contaminated by prior human-screening bias, so the "qualified" label is not a clean base-rate proxy.
- **Primary.** Demographic Parity Difference. `fairlearn.metrics.demographic_parity_difference` (max selection-rate minus min selection-rate across groups). The DCWP's bias audit under LL144 reports a related quantity, the **impact ratio** = group's selection rate / most-selected group's selection rate, with four-fifths (0.8) as the legacy EEOC presumption threshold (29 CFR 1607.4(D)).
- **Secondary.** Equal Opportunity Difference. `fairlearn.metrics.equal_opportunity_difference` — guards against equal selection rates bought by shortlisting unqualified candidates in the minority group (a well-documented "cosmetic parity" failure mode).
- **Deferred.** Predictive Parity (positive predictive value parity across groups). Deferred because the "qualified" ground-truth signal is upstream-biased; we cannot ask for calibration-across-groups until the label is trustworthy.
- **Pass threshold.** Impact ratio ≥ 0.8 **and** Equal Opportunity Difference ≤ 0.05 — thresholds set in `risk-register/2025/TalentFlow.yaml#fairness-envelope`, not here.

### Scenario B — TalentFlow filters internal promotions where applicant pool is known-qualified

- **Harm-model row.** Internal promotion filtering mis-ranks qualified female applicants against qualified male applicants with similar performance records.
- **Decision type.** Allocation.
- **Base-rate confidence.** High — internal performance review labels are usable as ground truth for "qualified for promotion."
- **Primary.** Equal Opportunity Difference. `fairlearn.metrics.equal_opportunity_difference`.
- **Secondary.** False Positive Rate Difference. `fairlearn.metrics.false_positive_rate_difference` — catches equal-TPR-with-inflated-FPR failures.
- **Deferred.** Demographic Parity — since the base rate of "qualified for promotion" may differ across the internal pool in ways the HR system can defend, demographic parity is not the first-order concern here.

### Scenario C — TalentFlow produces an interview-fit score consumed by a recruiter

- **Harm-model row.** Score misinterpretation by recruiter — "a 0.72 means the same thing for a Black candidate as for a white candidate."
- **Decision type.** Risk score fed to a human.
- **Primary.** Predictive Parity / Calibration within groups. AIF360 exposes this via `BinaryLabelDatasetMetric` and `ClassificationMetric` (e.g., `positive_predictive_value_difference`); Fairlearn exposes `MetricFrame` wrapping `sklearn.metrics.precision_score` by group.
- **Secondary.** Score distribution per group (reported as a plot, not a single scalar) — ensures the recruiter does not get a score that is systematically higher or lower in one group even when calibration holds.
- **Deferred.** Demographic Parity at the downstream decision.

### Scenario D — Candidate files an individual complaint: "I was similar to the person next to me, I was rejected, they advanced."

- **Harm-model row.** Inconsistency on near-duplicate candidates — an individual-fairness failure.
- **Decision type.** Individual review, not statistical.
- **Primary.** Individual-fairness consistency score (Dwork et al. 2012 — similar individuals should get similar outputs). In AIF360, `ClassificationMetric.consistency` (k-nearest-neighbour agreement in output among nearest in input space).
- **Secondary.** Counterfactual fairness (Kusner et al. 2017) — would the prediction have been different if the protected attribute had been different, holding everything else fixed on the causal graph? Operationalised in exercise 03.
- **Deferred.** All group-level metrics. They are not the right lens for a single-case complaint.

### Scenario E — TalentFlow is used in both hiring and internal mobility; cross-use audit

- **Harm-model row.** Cross-use drift — the model trained for external hiring is being reused for internal mobility, where the demographic distribution and base rates differ.
- **Decision type.** Model-governance rather than per-decision.
- **Primary.** Report the same primary metric for both use-cases (Demographic Parity Difference for external; Equal Opportunity Difference for internal) side-by-side, with **both** envelopes flagged as release gates.
- **Secondary.** Sliced accuracy by (protected × use-case) — reveals that the external-hiring pool and internal-mobility pool have different conditional performance.
- **Deferred.** A single "overall fairness score." No such thing exists; the per-use-case split is the point.

## 3. The Impossibility Theorem — why the drill does not pick "all metrics"

If a population has groups with different true base rates of the positive outcome, no classifier can simultaneously achieve:

- Calibration within groups (same PPV across groups at a given score), and
- Equal false-positive rate across groups, and
- Equal false-negative rate across groups,

unless the classifier is perfect (Kleinberg–Mullainathan–Raghavan 2016; Chouldechova 2017). Demographic parity further conflicts with calibration when base rates differ.

Consequence for this drill: each scenario must name a **primary**, a **secondary** (held as a counter-metric so the primary is not gamed), and a **deferred** metric that the author concedes cannot be jointly satisfied. A submission that claims to satisfy every metric fails the rubric because it hides the trade from the reviewer and the risk council.

## 4. How this feeds the register

Each scenario's primary + secondary + threshold is landed in `risk-register/2025/TalentFlow.yaml#fairness-envelope`. The envelope is the single source of truth at release gate time; this document is the *rationale* behind it. The register is changed by proposal + AI risk council approval; this document is updated whenever the register's fairness envelope is changed.
```

## 3. Validation steps

A reviewer should be able to confirm the submission in under 20 minutes:

1. Count scenarios: expect ≥4. This solution has 5.
2. For each scenario, check that the primary, secondary, and deferred metrics are **all three** named, and that the *reason* each is in its slot ties back to the harm-model row.
3. For each metric, verify a Fairlearn or AIF360 identifier is cited (or, for metrics with no library implementation yet, a canonical publication).
4. Verify the Impossibility Theorem section is present and names at least one concrete conflict (calibration vs. equal FPR).
5. Confirm thresholds are not set inline in this document; they must live in `risk-register/2025/TalentFlow.yaml`.
6. Confirm Scenario D (individual complaint) does **not** fall back to group-level metrics — the rubric treats that as a hard failure.

## 4. Rubric / review checklist

Scoring is pass/fail per row; the overall submission is accepted if all row-level checks pass and the structural checks pass.

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Running system + output type + protected attributes stated up top | Named system, output type, protected attribute list | Missing any of the three |
| R2 | Decision tree present and used | Tree defined once; each scenario references a branch | Tree absent, or scenarios pick metrics with no stated reasoning |
| R3 | ≥4 scenarios | Row count ≥4 | <4 |
| R4 | Each scenario names primary + secondary + deferred | All three named per scenario | One of three missing |
| R5 | Library identifiers cited where available | `fairlearn.metrics.*` or `aif360.metrics.*` identifier or canonical paper cited | Paraphrase only, no handle |
| R6 | Thresholds not inline | Document cites `risk-register/...#fairness-envelope` | Numeric thresholds placed directly in this doc |
| R7 | Impossibility Theorem explicit | Section states the conflict with a citation | Omitted or hedged to "there may be trade-offs" |
| R8 | Harm-model tie per scenario | Each scenario opens with the harm-row it serves | Scenario begins with "and another metric is ..." |
| R9 | Individual-level scenario does not fall back to group-level metrics | Scenario D (or equivalent) uses consistency / counterfactual | Scenario D uses demographic parity |
| R10 | Deferred metrics are real metrics, not strawmen | Named metric; stated reason why it cannot be primary | Deferred field is "none" or "n/a" |

**Scoring bands.** *Pass* — R1–R10 all green. *Pass with revisions* — one of R4/R5/R6 yellow on a single scenario; fix and resubmit. *Fail* — R7 omitted (the point of the drill), or R9 violated, or fewer than four scenarios.

## 5. Common mistakes

- **"All three — demographic parity, equal odds, and calibration — must hold."** The Impossibility Theorem rules this out when base rates differ. A submission that promises all three is rejected.
- **Choosing Demographic Parity for every allocation scenario because LL144 reports an impact ratio.** LL144's report is *descriptive*; the engineering primary metric may still be Equal Opportunity when base rates are reliable. The LL144 report is still produced — see exercise 06 — but it does not force the internal engineering metric choice.
- **Confusing Equal Opportunity (equal TPR) with Equalized Odds (equal TPR *and* equal FPR).** Fairlearn exposes both; AIF360 does too. Pick deliberately.
- **Falling back to a group-level metric on an individual-complaint scenario.** The individual-fairness consistency score and counterfactual fairness are the right lenses; using demographic parity to dismiss a per-case complaint is both technically wrong and legally fraught.
- **Inventing a scalar "overall fairness score."** None of the official sources (NIST AI RMF, EEOC Uniform Guidelines, LL144 rules) endorse one. Report the per-metric values and the trade-offs.
- **Omitting the deferred metric.** Making the trade-off visible is the point of the drill; a scenario without a deferred metric fails R4.
- **Setting thresholds inline.** Thresholds belong in the risk register (mod-103 discipline). This document cites them; it does not set them.

## 6. References

Only official sources and well-established canonical publications are used.

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** MEASURE 2.11 on fairness and bias evaluation and documentation. `https://www.nist.gov/itl/ai-risk-management-framework`
- **NIST AI RMF Playbook (airc.nist.gov).** Actions under MEASURE 2.11.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — referenced because the fairness evaluation surface shares controls with the ML security surface (dataset integrity, label quality) and the exercise sits next to mod-105's adversarial-ML threat model.
- **MITRE ATLAS.** `https://atlas.mitre.org/` — referenced for cross-walking label-manipulation threat categories that affect fairness evaluations.
- **EEOC Uniform Guidelines on Employee Selection Procedures (1978).** 29 CFR 1607 — in particular 1607.4(D) *Adverse impact and the "four-fifths rule."* `https://www.ecfr.gov/current/title-29/subtitle-B/chapter-XIV/part-1607`
- **NYC Local Law 144 (2021) and DCWP Rules Title 6 §§ 5-300 – 5-305** — bias-audit definitions and impact-ratio formula used in Scenario A. Full citation in exercise 06.
- **Hardt, Price, Srebro (2016).** *Equality of Opportunity in Supervised Learning.* NeurIPS. — definition of Equal Opportunity / Equalized Odds.
- **Chouldechova (2017).** *Fair prediction with disparate impact.* Big Data. — Impossibility Theorem variant for calibration vs. error-rate balance.
- **Kleinberg, Mullainathan, Raghavan (2016).** *Inherent Trade-Offs in the Fair Determination of Risk Scores.* ITCS. — Impossibility Theorem for calibration + equal FPR + equal FNR.
- **Dwork, Hardt, Pitassi, Reingold, Zemel (2012).** *Fairness Through Awareness.* ITCS. — individual fairness / Lipschitz formulation.
- **Kusner, Loftus, Russell, Silva (2017).** *Counterfactual Fairness.* NeurIPS. — counterfactual-fairness lens used in Scenario D.
- **Fairlearn documentation.** `https://fairlearn.org` — API handles `demographic_parity_difference`, `equal_opportunity_difference`, `false_positive_rate_difference`, `MetricFrame`.
- **AIF360 documentation.** `https://aif360.readthedocs.io` — API handles `BinaryLabelDatasetMetric`, `ClassificationMetric`, including `consistency` and group-conditional PPV handles.
- Exercise prompt — `lessons/mod-106-fairness-bias-and-explainability-engineering/exercises/exercise-01-fairness-metric-choice-drill.md` — defines TalentFlow as the running system and the four-scenario floor.

# Exercise 06 — NYC LL144 + EEOC Adverse-Impact Audit Report Drill — Reference Solution

## 1. Solution overview

This reference walks a worked `mod-106-ex06-ll144-audit/` deliverable for the module's running system **TalentFlow** (exercise 01). The exercise lands two artifacts:

1. The **bias audit summary** that an independent auditor would produce under New York City Local Law 144 (Automated Employment Decision Tools) and that the employer must publish on its website — per NYC DCWP Rules Title 6, Chapter 5, Subchapter T (§§ 5-300 to 5-305).
2. The **engineering evidence packet** behind the summary — the per-category selection and scoring data that LL144 requires the auditor to analyse, cross-walked to the EEOC Uniform Guidelines on Employee Selection Procedures (29 CFR 1607) "four-fifths rule" and the mod-109 continuous-monitoring interface.

The deliverable is a short pair of markdown artifacts (`PUBLIC_SUMMARY.md` and `INTERNAL_EVIDENCE.md`) plus a reproducible data spec (`data-cuts.yaml`).

Design decisions:

- **Only the public summary is a candidate for publication.** The internal evidence packet contains the full per-category counts and is reviewed by counsel before any excerpt is used externally.
- **Impact ratio and scoring rate are both reported where required.** LL144 requires "selection rates" when the tool is used to select candidates and "scoring rates" when the tool is used to assign scores; the public summary distinguishes the two use-modes and reports whichever applies, plus the **impact ratio** (group's selection rate ÷ most-selected group's selection rate).
- **Intersectional categories are reported, not elided.** LL144's bias-audit rule requires reporting by sex category, by race/ethnicity category, and by intersectional (sex × race/ethnicity) categories. The report provides all three.
- **Four-fifths is a legacy threshold, not the controlling law.** The EEOC's Uniform Guidelines of 1978 (29 CFR 1607.4(D)) use 4/5ths (0.8) as a *rule of thumb* for presumptive adverse impact. LL144 does not pin a pass/fail threshold on the impact ratio; the auditor publishes the number and the employer publishes it. The report states that distinction explicitly rather than repeating "the four-fifths rule is the law."
- **"Independent auditor" is a definitional requirement.** NYC DCWP Rule § 5-301 defines an independent auditor as one that did not use, develop, or distribute the AEDT and did not and does not have a financial stake in the AEDT. The exercise is a drill — the evidence packet includes a declaration block the auditor would sign, and the engineer does not sign it.
- **Nothing is invented.** Where example numbers appear in the public summary, they are prefaced with *example* and labelled as placeholders for the engineer to fill at submission time.

## 2. Implementation — worked answer

### 2.1 Directory layout

```
mod-106-ex06-ll144-audit/
├── README.md
├── PUBLIC_SUMMARY.md
├── INTERNAL_EVIDENCE.md
└── data-cuts.yaml
```

### 2.2 `data-cuts.yaml`

```yaml
# TalentFlow bias-audit data specification for NYC LL144.
#
# Describes the cuts of data the auditor needs. Produced by the engineer;
# consumed by the independent auditor's own eval harness. The engineer does
# not run the audit against these cuts; they produce the inputs.
#
# Reviewed by: employment counsel, DEI office, AI risk engineer (this role).

version: 2026-11-01
scope:
  system: TalentFlow
  tool_type: AEDT (per NYC Admin Code § 20-870 and DCWP Rule § 5-301)
  use_mode:
    - selection (binary advance / do-not-advance)
    - scoring (score in [0, 1] exposed to recruiter)
  jurisdictions_covered:
    - NYC (LL144 scope)
  roles_covered:
    - entry_level_customer_service
    - entry_level_data_entry
  time_window:
    start: 2025-11-01
    end:   2026-10-31
  n_applicants: TBD (filled at submission; must be >= DCWP minimum for statistical reliability per Rule § 5-302)
categories:
  sex:
    source: self-reported EEO form; "Decline to state" counted and reported separately
    values: [male, female, non_binary_or_other, declined]
  race_ethnicity:
    source: self-reported EEO Component 1 categories; "Decline to state" counted
    values:
      - hispanic_or_latino
      - white
      - black_or_african_american
      - native_hawaiian_or_pacific_islander
      - asian
      - american_indian_or_alaska_native
      - two_or_more_races
      - declined
  intersectional:
    composition: sex x race_ethnicity (per DCWP Rule § 5-302(a))
cuts_requested_by_auditor:
  - name: selection_rate_by_sex
    requires: [counts_selected_by_sex, counts_applied_by_sex]
  - name: selection_rate_by_race_ethnicity
    requires: [counts_selected_by_race_ethnicity, counts_applied_by_race_ethnicity]
  - name: selection_rate_by_intersectional
    requires: [counts_selected_by_sex_x_race_ethnicity, counts_applied_by_sex_x_race_ethnicity]
  - name: scoring_rate_by_sex
    requires: [mean_score_by_sex]
    use_when: use_mode == scoring
  - name: scoring_rate_by_race_ethnicity
    requires: [mean_score_by_race_ethnicity]
    use_when: use_mode == scoring
  - name: impact_ratio_by_sex
    formula: selection_rate_group / selection_rate_most_selected_group
  - name: impact_ratio_by_race_ethnicity
    formula: selection_rate_group / selection_rate_most_selected_group
  - name: impact_ratio_by_intersectional
    formula: selection_rate_group / selection_rate_most_selected_group
historical_data:
  used: true   # DCWP Rule § 5-302 permits historical data under conditions
  description: >
    Historical selection outcomes over the stated window. Where historical
    data is insufficient for a category (fewer than the DCWP-stated minimum),
    the auditor may use test data per Rule § 5-302(b); this is disclosed.
```

### 2.3 `PUBLIC_SUMMARY.md` (illustrative; all numeric values marked *example*)

```markdown
---
system: TalentFlow (AEDT; résumé-screening and candidate-scoring assistant)
employer: Example Employer Inc.
tool_vendor: Example Employer Inc. (built in-house)
audit_date: 2026-11-15 (example)
audit_window: 2025-11-01 — 2026-10-31
audit_window_type: historical_data
auditor_name: TBD — independent auditor per NYC Admin Code § 20-871 and DCWP Rule § 5-301
auditor_independence_declaration: TBD — see § 5-301(d) for required attestations
---

# TalentFlow — NYC Local Law 144 Bias Audit — Public Summary

**Publication path:** per NYC Admin Code § 20-871(b) and DCWP Rule § 5-303, the summary of this bias audit and its date of issuance are to be made publicly available on the employer's website prior to using the AEDT and for at least six months after the last use.

## 1. Tool description

TalentFlow is an **automated employment decision tool** as defined by NYC Admin Code § 20-870 and DCWP Rule § 5-301. It (a) assists the selection of candidates to advance to recruiter review from an applicant pool for entry-level customer-service and data-entry roles, and (b) produces a candidate score in [0, 1] that is exposed to the recruiter.

## 2. Audit methodology

- **Data source:** historical selection outcomes over the audit window listed above (per DCWP Rule § 5-302 on historical-data audits).
- **Categories analysed:** sex, race/ethnicity (EEO-1 Component 1), and intersectional sex × race/ethnicity (per Rule § 5-302(a)).
- **Metrics computed:** selection rate per category, impact ratio per category (= category's selection rate ÷ most-selected category's selection rate); where the tool is used in scoring mode, the mean scoring rate per category and the impact ratio of scoring rates.
- **Minimum cell size:** categories with fewer applicants than the DCWP's stated minimum for statistical reliability are **excluded** from the impact-ratio computation and **disclosed**.

## 3. Results (example values — replace with signed auditor-issued figures at submission)

### 3.1 Selection rates and impact ratios — by sex

| Category | Applicants | Selected | Selection rate | Impact ratio |
|---|---|---|---|---|
| Male | 2,400 | 540 | 0.225 | 1.00 (reference — most selected) |
| Female | 2,600 | 520 | 0.200 | 0.89 |
| Non-binary / other | 60 | 12 | 0.200 | 0.89 |
| Declined | 180 | 36 | 0.200 | 0.89 |

### 3.2 Selection rates and impact ratios — by race/ethnicity

| Category | Applicants | Selected | Selection rate | Impact ratio |
|---|---|---|---|---|
| White | 1,800 | 450 | 0.250 | 1.00 (reference — most selected) |
| Asian | 900 | 207 | 0.230 | 0.92 |
| Hispanic or Latino | 1,100 | 220 | 0.200 | 0.80 |
| Black or African American | 900 | 160 | 0.178 | 0.71 |
| Two or more races | 240 | 48 | 0.200 | 0.80 |
| Native Hawaiian / Pacific Islander | 60 | 12 | 0.200 | 0.80 |
| American Indian / Alaska Native | 60 | *excluded — cell < minimum* | — | — |
| Declined | 180 | 36 | 0.200 | 0.80 |

### 3.3 Selection rates and impact ratios — by sex × race/ethnicity

Reported in full per § 5-302(a); abbreviated here for readability. The intersection with the lowest impact ratio in this *example* is Black-or-African-American × Female at 0.65.

### 3.4 Scoring rates (tool operates in scoring mode)

Mean score and impact ratio of mean scores by category, table omitted for brevity. Impact ratios follow the same formula; LL144 does not pin a pass/fail threshold on the number — it requires disclosure.

## 4. Statement on the four-fifths rule

The EEOC Uniform Guidelines (1978; 29 CFR 1607.4(D)) use a 4/5ths (0.8) impact ratio as a general rule of thumb for presumptive adverse impact under federal enforcement. **LL144 and DCWP Rule § 5-303 do not adopt 4/5ths as a pass/fail threshold for the bias audit itself.** The employer and the auditor are required to publish the impact ratios regardless of value; interpretation under federal law (including Title VII) is the employer's and counsel's responsibility.

## 5. Date of most recent audit and distribution notice

- **Audit date:** 2026-11-15 (example).
- **Date of first use of the AEDT:** TBD at submission.
- **Distribution notice to candidates:** per NYC Admin Code § 20-871(b)(1) and Rule § 5-304, candidates for employment or promotion who reside in NYC have been notified at least 10 business days before use of the AEDT; the notice includes job qualifications and characteristics used by the AEDT, and the data retention policy.

## 6. Auditor's independence declaration

*To be signed by the independent auditor at submission.* The auditor attests that the auditor did not use, develop, or distribute this AEDT and does not have a financial interest in the AEDT or in the employer's use of it (per DCWP Rule § 5-301(d)).
```

### 2.4 `INTERNAL_EVIDENCE.md` (not published; feeds the auditor)

```markdown
# TalentFlow — LL144 bias-audit internal evidence packet

Produced by: ai-risk-engineer (L25)
Consumed by: independent auditor (per § 5-301)
Reviewed by: employment counsel, DEI office, AI risk council chair

## 1. Inputs

- Shadow log pinned at `s3://audit/talentflow/shadow/2026-10-31.parquet`
  (SHA-256 recorded in `release-manifest/<release-tag>/evidence.json`).
- EEO self-reporting export pinned at same timestamp.
- Model artifact at `models/talentflow/<release-tag>.joblib` with hash.

## 2. Processing

Raw selection events are joined to EEO self-reporting on candidate_id with
left-join semantics; non-matches are counted as "declined" and never
back-filled. Min-cell-size exclusions are applied per § 5-302 after the join
to preserve denominator integrity.

## 3. Outputs supplied to auditor

For each cut in `data-cuts.yaml`:
- a CSV with `category`, `applicants`, `selected`, `mean_score`;
- a one-paragraph note on exclusions and why;
- the model version and threshold in force for the window.

## 4. Mod-109 monitoring linkage

Every cut defined here has a mirror monitor in the mod-109 continuous
monitoring pipeline at `monitors/talentflow/impact-ratio-*.yaml`. The monitor
re-computes impact ratios weekly on the current-window shadow log; if any
intersectional impact ratio drops below the register's alert threshold, an
incident candidate is filed via mod-111.

## 5. Mod-108 guardrail linkage

Per mod-108, a guardrail blocks the AEDT from being used on candidates with
a residency flag outside LL144 scope without a documented alternative
assessment path. This evidence packet documents that guardrail's version and
the last compliance review.

## 6. Retention

Records required to support this audit are retained for the period required
by NYC Admin Code § 20-871 and applicable federal and state record-keeping
rules (including the EEOC's 1 year-plus for selection procedures). Counsel
sign-off on the retention schedule is on file.
```

### 2.5 README summary

```markdown
# TalentFlow LL144 bias audit — drill

## What this is
A reproducible drill for producing the inputs an independent auditor would
need under NYC LL144. The drill does NOT itself constitute the audit —
LL144 requires an independent auditor as defined in DCWP Rule § 5-301(d).

## What to run at submission
1. Populate `data-cuts.yaml` with the real time window and applicant counts.
2. Hand the YAML and the signed shadow-log + EEO export to the independent
   auditor along with `INTERNAL_EVIDENCE.md`.
3. Publish the auditor's final `PUBLIC_SUMMARY.md` on the employer's website
   with the audit date; keep for at least the LL144 minimum publication
   period (see Rule § 5-303).
4. Notify NYC candidates at least 10 business days before AEDT use per
   § 20-871(b)(1) and Rule § 5-304.
```

## 3. Validation steps

1. **Cell-size discipline.** Re-run the audit with a synthetically trimmed category (e.g., American Indian / Alaska Native with 10 applicants). Expected: category excluded from impact-ratio computation and surfaced as "excluded — cell < minimum" in section 3.2.
2. **Reference-group derivation.** The "most-selected" reference changes if the data changes. Swap the inputs so Asian becomes the most-selected category; expected: Asian becomes the 1.00 reference row and White becomes 0.92/… relative.
3. **Intersectional presence.** Section 3.3 must be non-empty. A reviewer greps for "intersectional" and finds the subsection; missing is a hard fail.
4. **Four-fifths framing.** Reviewer confirms section 4 states that LL144 does **not** pin 4/5ths as the audit pass/fail — a common submission mistake.
5. **Auditor independence declaration.** Section 6 includes the attestation shape required by Rule § 5-301(d); it is placeholdered for the actual auditor's signature at submission.
6. **Scoring mode vs. selection mode.** If the tool is used in both modes (as TalentFlow is), both impact ratios (selection and scoring) appear in the summary.
7. **Candidate notice path exists.** Section 5 cites § 20-871(b)(1) and Rule § 5-304; notice content covers job qualifications, characteristics used, and data retention.
8. **Mod-109 and mod-108 linkages.** `INTERNAL_EVIDENCE.md` sections 4 and 5 point to concrete monitor and guardrail artifacts; a reviewer checks that the monitors and guardrails named exist.

## 4. Rubric / review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | AEDT definition cited | NYC Admin Code § 20-870 and DCWP Rule § 5-301 cited by section | Not cited |
| R2 | Selection + scoring modes covered | Both impact-ratio tables present when both modes apply | One mode only |
| R3 | Sex, race/ethnicity, and intersectional cuts | All three subsections present with tables | One or two subsections only |
| R4 | Impact ratio formula stated | Reference = most-selected group; formula stated once | Formula omitted or wrong |
| R5 | Min-cell-size exclusions disclosed | Excluded cells surfaced in the table | Cells quietly dropped |
| R6 | Four-fifths framing correct | Section states LL144 does not adopt 0.8 as pass/fail | Submission asserts "we pass because ≥ 0.8" |
| R7 | Independent-auditor requirement | Auditor independence declaration section present and placeholdered for sign | Engineer signs the audit |
| R8 | Historical-data / test-data disclosure | Audit methodology section names which is used and why | Implicit |
| R9 | Candidate notice path | § 20-871(b)(1) and Rule § 5-304 cited | Omitted |
| R10 | Mod-109 continuous monitor linkage | `INTERNAL_EVIDENCE.md` names the mirror monitors | Audit treated as one-off |
| R11 | Mod-108 guardrail linkage | AEDT scope guardrail referenced | Not referenced |
| R12 | Publication posture | Public summary is a separate file from internal evidence | Both merged |

**Scoring bands.** *Pass* — all green. *Pass with revisions* — R5 or R8 yellow; fix and resubmit. *Fail* — R6 misframed (asserting LL144 pass because impact ratio ≥ 0.8), or R7 missing (engineer signing), or R3 missing intersectional cut.

## 5. Common mistakes

- **"We pass LL144 because all impact ratios ≥ 0.8."** LL144 does not pin the four-fifths rule as a pass/fail for the bias audit itself. The employer publishes the impact ratios; the auditor and counsel interpret them under Title VII and other applicable law.
- **Signing the audit.** The engineer cannot be the independent auditor. LL144 (DCWP § 5-301(d)) requires that the auditor did not use, develop, or distribute the AEDT and has no financial stake.
- **Skipping the intersectional cut.** DCWP § 5-302(a) requires intersectional categories. Omitting them is a direct rule violation.
- **Dropping low-count cells silently.** The required discipline is to **exclude from impact-ratio** and **disclose**, not drop and hide.
- **Treating the tool as "selection only" when it also produces a score.** LL144 covers both; both views appear in the summary.
- **Confusing the EEO-1 Component 1 categories with ad hoc groupings.** Categories must match the federal EEO reporting taxonomy that LL144 references.
- **No candidate notice.** The 10-business-day notice and its content are separate obligations under § 20-871(b)(1) and Rule § 5-304. Engineering supports the notice (feature list, data retention) but does not own publishing it; the HR path does.
- **Merging the public summary and the internal evidence packet.** The internal packet contains counts and joins that are not for publication.
- **No link to mod-109 monitoring.** LL144 requires at least annual re-audit; the ongoing monitor is not optional for a live AEDT.
- **"Historical data was insufficient so we did not audit."** DCWP § 5-302 permits test data in bounded circumstances; the audit methodology must say so rather than skip.

## 6. References

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** GOVERN 1.1 (legal and regulatory requirements understood, managed, documented); MEASURE 2.11 (fairness evaluated and documented); MANAGE 4.1 (post-deployment monitoring). `https://www.nist.gov/itl/ai-risk-management-framework`
- **NIST AI RMF Playbook.** Actions under GOVERN 1.1 and MEASURE 2.11 relevant to publishing a bias-audit summary.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant for data-integrity and label-quality controls on the shadow log that feeds the audit.
- **MITRE ATLAS.** `https://atlas.mitre.org/` — relevant for adversarial-data-manipulation techniques that could poison the audit inputs.
- **NYC Administrative Code § 20-870 and § 20-871** (Local Law 144 of 2021, enacted December 2021; enforcement effective 2023-07-05). Full text: `https://nyc.public.law/laws/n.y.c._administrative_code_chapter_5_subtitle_a_subchapter_25` (local-law chapter) and the DCWP's hosted copy.
- **NYC DCWP Rules Title 6, Chapter 5, Subchapter T — "Automated Employment Decision Tools" (§§ 5-300 – 5-305)**, adopted April 6, 2023, effective July 5, 2023. `https://rules.cityofnewyork.us/` (search "Automated Employment Decision Tools"). Key sections cited in this solution:
  - § 5-301 (definitions, including "AEDT", "bias audit", and "independent auditor" (§ 5-301(d)))
  - § 5-302 (bias audit — historical-data and test-data provisions; intersectional cuts (§ 5-302(a)))
  - § 5-303 (publication of summary results)
  - § 5-304 (candidate notice)
  - § 5-305 (recordkeeping)
- **EEOC Uniform Guidelines on Employee Selection Procedures (1978).** 29 CFR 1607, in particular 1607.4(D) — "four-fifths rule." `https://www.ecfr.gov/current/title-29/subtitle-B/chapter-XIV/part-1607`
- **EEOC guidance on AI/software selection tools.** EEOC technical assistance document *The Americans with Disabilities Act and the Use of Software, Algorithms, and Artificial Intelligence to Assess Job Applicants and Employees* (2022) and subsequent EEOC AI statements. `https://www.eeoc.gov/laws/guidance`
- Fairlearn documentation. `https://fairlearn.org` — `demographic_parity_ratio` is the Fairlearn analogue of LL144's impact ratio.
- AIF360 documentation. `https://aif360.readthedocs.io` — `BinaryLabelDatasetMetric.disparate_impact` is the library handle on the same ratio.
- Exercise prompt — `lessons/mod-106-fairness-bias-and-explainability-engineering/exercises/exercise-06-nyc-ll144-and-eeoc-audit-report-drill.md`.

## 7. Citation-versioning note

The DCWP Rules citations above (Title 6, Chapter 5, Subchapter T, §§ 5-300 – 5-305) match the text as adopted April 6, 2023 and effective July 5, 2023 — that is the version the published audit summary must comply with today. The DCWP has signalled that it may re-open the rule to amend definitions (notably "machine learning" in § 5-301 and the historical-vs-test-data boundary in § 5-302) and FAQ guidance is updated separately on the DCWP site; before publishing an external audit summary, re-pull the live rule at `https://rules.cityofnewyork.us/` and the DCWP FAQ, confirm the subchapter / section numbering has not shifted, and note the retrieval date next to the citation block in the summary.

# Exercise 01 — NIST AI 100-2 → MITRE ATLAS → OWASP ML Top 10 Cross-Tag Sheet — Reference Solution

## 1. Solution overview

This reference walks a worked `mod-105-ex01-cross-tag.md` for **CarePilot** — the same hospital-triage RAG assistant used throughout the mod-101/102/103/104 submissions. The deliverable is a single markdown document with four sections: (1) scope and sources, (2) the cross-tag sheet proper (NIST AI 100-2 attack class → ATLAS tactic/technique → OWASP ML Top 10 entry, plus CarePilot applicability), (3) coverage summary, and (4) hand-offs to downstream exercises in mod-105 (ex-02 evasion, ex-03 poisoning, ex-04 extraction/MIA, ex-05 report).

The core design decisions are:

- **NIST AI 100-2 is the organising axis.** The attack-class taxonomy is the only one of the three sources that is a *taxonomy* (rather than a technique catalogue or a flat Top-10 list). Column 1 holds attack classes from AI 100-2; column 2 maps to the relevant ATLAS tactic/technique; column 3 maps to the OWASP ML Top 10 entry; columns 4-6 record CarePilot applicability, the register row it lands in, and the mod-105 exercise that exercises the defence. Reading *down* a column gives you a per-framework view; reading *across* a row gives you the engineering handle.
- **PredAI *and* GenAI attack classes are both filed.** AI 100-2 covers both. CarePilot is a RAG assistant (GenAI) on top of a clinical retrieval corpus, so the GenAI rows are the load-bearing ones; the PredAI rows are included because (a) the triage-classifier auxiliary heads trained in mod-106 are PredAI, and (b) the module's scope is explicitly broad.
- **ATLAS entries are cited by technique ID (AML.T####), not by tactic alone.** A row that says "the attack falls under ATLAS *Discovery*" is unfalsifiable; a row that says "AML.T0024 — Discover ML Model Family" has a URL. Where AI 100-2 attack classes correspond to several ATLAS techniques, the row lists the closest technique and mentions the related ones.
- **OWASP ML Top 10 entries are cited by ID (ML01..ML10) with the 2023 project nomenclature.** The OWASP ML Top 10 was published by the OWASP ML Security Top 10 Project; versioning and entry titles are taken from that project page.
- **CarePilot applicability is a ruling, not a copy-paste.** Each row says IN / OUT / CONDITIONAL with a one-line reason. "CONDITIONAL" rows note the upstream dependency (usually a mod-103 register row, a mod-104 red-team suite result, or a mod-106 slice set).
- **Every row is paired with a mod-105 exercise destination.** This is what turns the sheet from a wall-poster into an engineering artifact — the cross-tag's purpose is to drive which experiments ex-02 through ex-05 actually run.

## 2. Implementation — worked answer `mod-105-ex01-cross-tag.md`

The file below is the submission. Paths are illustrative; the shape is what a reviewer checks.

```markdown
---
system: CarePilot (hospital triage-support RAG assistant, HITL)
version_stamp:
  nist_ai_100_2: "NIST AI 100-2 E2023 — Adversarial Machine Learning: A Taxonomy and Terminology of Attacks and Mitigations (confirm revision letter at submission from https://csrc.nist.gov)"
  mitre_atlas: "MITRE ATLAS matrix revision as retrieved from https://atlas.mitre.org (confirm matrix version/date at submission)"
  owasp_ml_top_10: "OWASP Machine Learning Security Top 10 — 2023 project entries (confirm current project revision at submission from https://owasp.org/www-project-machine-learning-security-top-10/)"
author: ai-risk-engineer (L25)
reviewers: ai-evaluation-engineer (L35), security-learning / ai-infra-security (L35), ai-risk-council (chair)
status: draft-for-review
cross-ref:
  - modules/mod-101-ai-risk-frameworks-and-position/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map/SOLUTION.md  # MEASURE 2.7 row
  - modules/mod-103-risk-quantification-and-registers/...  # CarePilot risk register schema
  - modules/mod-104-llm-red-teaming-and-attack-engineering/...  # garak/PyRIT/AgentDojo suite
---

# CarePilot — NIST AI 100-2 × MITRE ATLAS × OWASP ML Top 10 Cross-Tag Sheet

## 1. Scope and sources

**Purpose.** This cross-tag sheet is the single source of truth for which adversarial-ML attack classes CarePilot is in scope for, how each class is named across the three frameworks CarePilot files against (NIST AI 100-2, MITRE ATLAS, OWASP ML Top 10), and which mod-105 exercise actually runs the defence. The sheet feeds:

- MEASURE 2.7 (NIST AI RMF) — the security and resilience sub-category (established in the mod-101 ex-01 map for CarePilot).
- 42001 Annex A controls that call out ML-specific threats (filed via mod-101 ex-03).
- The mod-103 CarePilot risk register — each IN or CONDITIONAL row seeds a risk ID.
- The mod-109 monitor plan — each attack class with a runtime signal maps to a monitor row.

**Sources used (only).**

- **NIST AI 100-2 E2023** — "Adversarial Machine Learning: A Taxonomy and Terminology of Attacks and Mitigations", NIST (U.S. Department of Commerce). Covers PredAI and GenAI; attack classes are organised along attacker goal (availability, integrity, privacy, abuse), attacker knowledge (white-box, grey-box, black-box), attacker capability, and attack timing (training-time vs inference-time). (`https://csrc.nist.gov/publications/detail/nistir/ai-100-2`) <!-- submission-todo: confirm exact publication identifier (E2023, E2024, revision letter) at submission time; NIST periodically revises the AI 100-2 series -->
- **MITRE ATLAS** — Adversarial Threat Landscape for Artificial-Intelligence Systems. ATT&CK-style matrix with tactics (columns) and techniques (AML.T####). Sourced from `https://atlas.mitre.org/`. Case studies are referenced only when they already appear in the ATLAS case-studies index; this sheet does not invent case studies.
- **OWASP Machine Learning Security Top 10** — OWASP project, 2023 nomenclature; entries ML01 through ML10. Sourced from `https://owasp.org/www-project-machine-learning-security-top-10/`.
- **CarePilot local context** — the running system, the mod-101 ex-01 MEASURE 2.7 row, the mod-102 harm model, and the mod-103 risk register (all cited from prior-module solutions in this repo).

Not used in this sheet: individual vendor advisories, academic papers, blog posts, and any source that is not in the list above.

## 2. Cross-tag sheet

Legend for the applicability column:

- **IN** — CarePilot is in scope; the attack class has a plausible threat model and is exercised in mod-105 (ex-02/03/04) or in mod-104 (LLM-specific).
- **CONDITIONAL** — scope depends on a configuration decision (e.g., whether a feature is enabled) or an upstream result (e.g., a harm-model row); the condition is named.
- **OUT** — CarePilot is out of scope for a stated reason that a reviewer can check against the system description.

### 2.1 Evasion / inference-time integrity attacks

| # | NIST AI 100-2 attack class | ATLAS tactic / technique | OWASP ML Top 10 | CarePilot applicability | Register row | Exercise destination |
|---|---|---|---|---|---|---|
| E1 | Evasion — PredAI — white-box (gradient-based perturbation on an input to force misclassification) | *ML Attack Staging* → AML.T0043 Craft Adversarial Data; *Defense Evasion* / *Impact* depending on objective | ML01 Input Manipulation Attack | **CONDITIONAL** — IN for the triage auxiliary classifier (mod-106); OUT for the GenAI assistant itself (no user-exposed softmax to invert) | `risk-register/.../adv-evasion-predai` | **mod-105 ex-02** panel (ART, image/tabular) |
| E2 | Evasion — PredAI — black-box transfer / query-based | *ML Attack Staging* → AML.T0043 Craft Adversarial Data (query-based); AML.T0040 ML Model Inference API Access | ML01 Input Manipulation Attack | **CONDITIONAL** — same scope as E1; black-box is the realistic threat because the triage classifier sits behind a service | `risk-register/.../adv-evasion-predai` | **mod-105 ex-02** panel (HopSkipJump / Square) |
| E3 | Evasion — GenAI — prompt-level (direct) | *ML Attack Staging* → AML.T0051 LLM Prompt Injection (direct) | ML01 Input Manipulation Attack | **IN** — the primary adversarial surface for the RAG assistant | `risk-register/.../prompt-injection-direct` | **mod-104** red-team suite (garak/PyRIT) — this sheet records the cross-tag; mod-104 runs the suite |
| E4 | Evasion — GenAI — prompt-level (indirect, via retrieval or tool-returned content) | AML.T0051 LLM Prompt Injection (indirect); AML.T0070 RAG Poisoning (if the retrieval corpus is attacker-influenced) | ML01 Input Manipulation Attack; ML02 Data Poisoning Attack (when the vector is a poisoned retrieval document) | **IN** — retrieval corpus is the Achilles' heel of CarePilot; filed in MEASURE 2.7 | `risk-register/.../prompt-injection-indirect` | **mod-104** indirect-injection suite; **mod-105 ex-03** covers the poisoning half |
| E5 | Evasion — NLP — word/char perturbation on a text classifier | AML.T0043 Craft Adversarial Data | ML01 Input Manipulation Attack | **CONDITIONAL** — IN only if a dedicated NLP classifier is on the critical path (e.g., a triage-text classifier); default OUT because CarePilot answers via the RAG LLM, not via a dedicated NLP classifier | `risk-register/.../adv-evasion-nlp` (conditional) | **mod-105 ex-02** TextAttack portion |

### 2.2 Training-time integrity attacks — data poisoning and backdoors

| # | NIST AI 100-2 attack class | ATLAS tactic / technique | OWASP ML Top 10 | CarePilot applicability | Register row | Exercise destination |
|---|---|---|---|---|---|---|
| P1 | Data poisoning — targeted — training-set manipulation to induce a specific misclassification | *Resource Development* → AML.T0020 Poison Training Data; *Impact* → AML.T0031 Erode ML Model Integrity | ML02 Data Poisoning Attack | **CONDITIONAL** — IN for the mod-106 fairness-aware auxiliary classifier when retrained on hospital-curated labels; OUT for the base LLM (frozen, vendor-supplied) | `risk-register/.../data-poisoning-targeted` | **mod-105 ex-03** poisoning panel |
| P2 | Data poisoning — availability (label flipping to degrade overall accuracy) | AML.T0020 Poison Training Data | ML02 Data Poisoning Attack; ML08 Model Skewing (overlaps where poisoning is used to skew behaviour over time) | **CONDITIONAL** — IN only for retrained components; OUT for the frozen vendor LLM | `risk-register/.../data-poisoning-availability` | **mod-105 ex-03** poisoning panel |
| P3 | Backdoor / trojan attack — training-time insertion of a trigger-conditioned behaviour | *ML Attack Staging* → AML.T0018 Backdoor ML Model; *Resource Development* → AML.T0020 Poison Training Data | ML10 Model Poisoning (Trojan); overlaps with ML02 when delivery vector is the training set | **CONDITIONAL** — IN for the fine-tuned retriever / auxiliary classifier; OUT for the base LLM unless vendor supply-chain assurance degrades | `risk-register/.../backdoor-trojan` | **mod-105 ex-03** (detection baselines — spectral signatures, activation clustering, Neural Cleanse per ART); supply chain in **mod-110** |
| P4 | Supply-chain compromise — malicious pre-trained weights, datasets, or dependencies | *Initial Access* → AML.T0010 ML Supply Chain Compromise; sub-types AML.T0010.000 (ML Software), .001 (ML Model), .002 (Data), .003 (Hardware) | ML06 AI Supply Chain Attacks; overlaps with ML07 Transfer Learning Attack when a poisoned pre-trained weight is used | **IN** — the base LLM and the embedding model are both externally supplied | `risk-register/.../supply-chain-ml` | **mod-110** supply-chain module (deeper); this sheet files the attack-class row |
| P5 | Transfer-learning attack — adversarial fine-tune/adapter on top of a benign base | *ML Attack Staging* → AML.T0018 Backdoor ML Model; *Execution* when loaded into the pipeline | ML07 Transfer Learning Attack; related to ML10 Model Poisoning | **CONDITIONAL** — IN if CarePilot adopts a fine-tuned adapter; current deployment uses retrieval over a frozen base | `risk-register/.../transfer-learn-adv` (conditional) | **mod-110** supply-chain |

### 2.3 Privacy attacks — extraction, inversion, membership inference

| # | NIST AI 100-2 attack class | ATLAS tactic / technique | OWASP ML Top 10 | CarePilot applicability | Register row | Exercise destination |
|---|---|---|---|---|---|---|
| X1 | Model extraction / model stealing (replicating the model from queries) | *Exfiltration* → AML.T0024 Discover ML Model Family / Ontology; *Exfiltration* → AML.T0048 Steal ML Model | ML05 Model Theft | **CONDITIONAL** — base LLM is vendor-hosted (extraction of the base is a vendor concern); the *prompt scaffolding* and the retrieval corpus, however, are CarePilot's and can be approximated by query — IN for the scaffolding/corpus extraction | `risk-register/.../extraction-scaffolding` | **mod-105 ex-04** extraction panel |
| X2 | Membership inference (did this record appear in training/fine-tuning?) | AML.T0053 Infer Training Data Membership | ML04 Membership Inference Attack | **CONDITIONAL** — IN only for CarePilot-owned retrained components (mod-106 auxiliary classifier, fine-tuned retriever if adopted); OUT for the frozen base LLM | `risk-register/.../mia-local-head` | **mod-105 ex-04** MIA panel |
| X3 | Model inversion / attribute inference (reconstructing training-set attributes from model outputs) | AML.T0052 Infer Training Data (attribute); related: AML.T0048 | ML03 Model Inversion Attack | **CONDITIONAL** — IN for retrained components; OUT for frozen base LLM | `risk-register/.../inversion-local-head` | **mod-105 ex-04** inversion-lite panel |
| X4 | Training-data extraction from an LLM (prompted reconstruction of training examples) | AML.T0057 LLM Data Leakage (verbatim regurgitation) | ML03 Model Inversion (adjacent); ML04 Membership Inference (adjacent) | **CONDITIONAL** — the *retrieval corpus* is CarePilot's and must not leak PHI; the *base LLM's* training-data leakage is a vendor concern but is screened by the mod-104 PII suite | `risk-register/.../rag-leakage` | **mod-104** red-team suite — this sheet cross-tags it |
| X5 | Prompt / system-prompt leakage | AML.T0056 Extract LLM System Prompt | ML05 Model Theft (where "model" = the prompted-scaffolding product) | **IN** — CarePilot's clinical-safety guardrails live in the system prompt and tool-use scaffolding; leakage is a confidentiality concern even if not catastrophic | `risk-register/.../system-prompt-leakage` | **mod-104** red-team suite — cross-tagged here |

### 2.4 Abuse / availability attacks

| # | NIST AI 100-2 attack class | ATLAS tactic / technique | OWASP ML Top 10 | CarePilot applicability | Register row | Exercise destination |
|---|---|---|---|---|---|---|
| A1 | Energy-/compute-latency amplification (sponge-style inputs that force maximum compute) | AML.T0029 Denial of ML Service | None directly in OWASP ML Top 10 (adjacent to OWASP ML Top 10 via availability framing); cross-reference OWASP LLM Top 10 if also filing against that list | **CONDITIONAL** — IN only if CarePilot exposes an endpoint that is unauthenticated or rate-unlimited; default OUT because CarePilot is SSO-gated inside the hospital network | `risk-register/.../dos-sponge` (conditional) | **mod-105 ex-04** rate-defence set (even though the primary driver is extraction, rate-limit is the shared defence) |
| A2 | Abuse — use of the system to produce harm outside its intended purpose (e.g., coaxing CarePilot to answer non-triage clinical questions) | *Impact* → AML.T0050 Verifying Attack; overlaps with content-safety red-teaming in mod-104 | None directly in OWASP ML Top 10 (adjacent); overlaps with the OWASP LLM Top 10 if filing against that list | **IN** — filed as abuse/misuse in the harm model (mod-102); guardrail coverage in mod-108 | `risk-register/.../abuse-out-of-scope` | **mod-104** red-team suite; **mod-108** guardrail bench |

## 3. Coverage summary

The table below is a per-framework coverage roll-up computed from section 2.

### 3.1 By NIST AI 100-2 attack class

| AI 100-2 attack class | Row(s) | CarePilot status |
|---|---|---|
| Evasion (PredAI) | E1, E2 | CONDITIONAL — triage classifier only |
| Evasion (NLP) | E5 | CONDITIONAL — only if a dedicated NLP classifier is on path |
| Evasion (GenAI — prompt injection, direct + indirect) | E3, E4 | IN — primary surface |
| Data poisoning (targeted + availability) | P1, P2 | CONDITIONAL — retrained heads |
| Backdoor / trojan | P3 | CONDITIONAL — retrained heads |
| Supply-chain compromise (ML) | P4 | IN — mod-110 scope |
| Transfer-learning attack | P5 | CONDITIONAL — if adapters are adopted |
| Model extraction / stealing | X1 | CONDITIONAL — scaffolding/corpus (not base LLM) |
| Membership inference | X2 | CONDITIONAL — retrained heads only |
| Model inversion / attribute inference | X3 | CONDITIONAL — retrained heads only |
| Training-data extraction from LLM | X4 | CONDITIONAL — RAG corpus (PHI) must not leak |
| Prompt / system-prompt leakage | X5 | IN — clinical-safety guardrails live in prompt |
| Energy-/compute abuse (sponge) | A1 | CONDITIONAL — only if public-facing |
| Abuse / out-of-scope use | A2 | IN — harm-model linked |

### 3.2 By ATLAS technique (minimum set cited)

The sheet references the following ATLAS techniques (IDs from `atlas.mitre.org`, matrix version version-stamped above). <!-- submission-todo: confirm each ATLAS technique ID at submission time; ATLAS adds/renumbers techniques across revisions -->

- Reconnaissance / Resource Development: AML.T0020 Poison Training Data; AML.T0010 ML Supply Chain Compromise (and subtypes .000–.003).
- Initial Access: AML.T0010 ML Supply Chain Compromise.
- ML Model Access: AML.T0040 ML Model Inference API Access.
- ML Attack Staging: AML.T0043 Craft Adversarial Data; AML.T0018 Backdoor ML Model; AML.T0051 LLM Prompt Injection; AML.T0070 RAG Poisoning.
- Discovery / Exfiltration: AML.T0024 Discover ML Model Family / Ontology; AML.T0048 Steal ML Model; AML.T0052 Infer Training Data; AML.T0053 Infer Training Data Membership; AML.T0056 Extract LLM System Prompt; AML.T0057 LLM Data Leakage.
- Impact: AML.T0029 Denial of ML Service; AML.T0031 Erode ML Model Integrity; AML.T0050 Verifying Attack.

### 3.3 By OWASP ML Top 10

| OWASP ML Top 10 entry | Cross-tag rows | CarePilot status | Where exercised |
|---|---|---|---|
| ML01 Input Manipulation Attack | E1, E2, E3, E4, E5 | IN (GenAI rows); CONDITIONAL (PredAI/NLP rows) | mod-104 (GenAI), mod-105 ex-02 (PredAI/NLP) |
| ML02 Data Poisoning Attack | P1, P2, E4 (via poisoned retrieval doc) | CONDITIONAL (retrained heads); IN (retrieval corpus) | mod-105 ex-03 |
| ML03 Model Inversion Attack | X3, X4 (adjacent) | CONDITIONAL | mod-105 ex-04 (inversion-lite) |
| ML04 Membership Inference Attack | X2 | CONDITIONAL | mod-105 ex-04 (MIA) |
| ML05 Model Theft | X1, X5 | CONDITIONAL / IN (scaffolding) | mod-105 ex-04 (extraction) |
| ML06 AI Supply Chain Attacks | P4 | IN | mod-110 |
| ML07 Transfer Learning Attack | P5 | CONDITIONAL | mod-110 |
| ML08 Model Skewing | P2 (overlap) | CONDITIONAL | mod-105 ex-03 |
| ML09 Output Integrity Attack | E3, E4 (overlap — manipulating outputs via inputs is adjacent; direct output-tampering requires integrity controls at the serving layer, which is a mod-108/mod-110 concern) | IN (serving integrity) | mod-108 / mod-110 |
| ML10 Model Poisoning | P3, P5 | CONDITIONAL | mod-105 ex-03; mod-110 |

### 3.4 CarePilot-specific gaps the cross-tag surfaces

- The **RAG retrieval corpus** sits at the intersection of E4 (indirect injection), P1/P2 (data poisoning of the ingested documents), and X4 (training-data extraction) — this is the single highest-leverage control point on CarePilot and is called out explicitly in the mod-103 register and the mod-109 monitor plan.
- **Supply-chain controls** for the base LLM and the embedding model are the only IN rows that CarePilot does not run its own experiments against — the exposure is real but the defence is procurement- and attestation-level (mod-110). The cross-tag makes this hand-off explicit so the gap is visible.
- **Out-of-scope abuse (A2)** is a content-safety concern, not an adversarial-ML concern in the AI 100-2 sense; it is filed here for completeness and cross-referenced to mod-104 and mod-108.

## 4. Hand-offs

| Framework-level fact | Owner | Downstream exercise/module |
|---|---|---|
| Evasion panel (ART + TextAttack) runs against the PredAI head and any NLP classifier on path | ai-risk-engineer (L25) | mod-105 ex-02 |
| Data-poisoning + backdoor-detection baselines (ART defences — spectral signatures, activation clustering, Neural Cleanse) | ai-risk-engineer (L25) | mod-105 ex-03 |
| Extraction / MIA / inversion panel + shared API-surface defences (rate limit, confidence masking, DP-SGD where feasible) | ai-risk-engineer (L25) | mod-105 ex-04 |
| LLM prompt-injection / RAG-poisoning red-team suite (garak, PyRIT, AgentDojo) | ai-risk-engineer (L25) in mod-104 | mod-104 |
| Supply-chain compromise (ML model, data, software) procurement controls | ai-risk-engineer (L25) with procurement + security-learning (L35) | mod-110 |
| Fairness slice set (mod-106) consumes P1 scope decisions when the auxiliary classifier is retrained | ai-eval-engineer (L30) + ai-risk-engineer (L25) | mod-106 |
| Monitor plan (mod-109) consumes every IN or CONDITIONAL-with-runtime-signal row | ai-risk-engineer (L25) with ai-infra-mlops (L25 peer) | mod-109 |
| Robustness-eval report and residual-risk register update | ai-risk-engineer (L25) | mod-105 ex-05 |
```

## 3. Validation steps

A reviewer should be able to confirm the submission in under 20 minutes.

1. Count the cross-tag rows. The solution has 16 rows (5 evasion, 5 training-time, 5 privacy, 2 abuse/availability); any acceptance floor set by the exercise prompt is comfortably met. Rows are grouped by AI 100-2 attack-goal cluster.
2. For each row, verify that *all three* framework columns are populated: a NIST AI 100-2 attack class, an ATLAS tactic + technique ID (AML.T####), and an OWASP ML Top 10 ID (ML01..ML10). Rows that span multiple techniques cite the closest technique and mention the related ones.
3. For each row, verify the applicability column is IN / OUT / CONDITIONAL with a one-line reason tied to the CarePilot system description. Reject rows that say CONDITIONAL with no named condition.
4. Confirm the register-row and exercise-destination columns are non-empty. "Not yet" is acceptable only if the row is CONDITIONAL and the condition is recorded as a register TODO.
5. Confirm the version-stamp block at the top cites the retrieval date or revision identifier for each of the three sources. `<!-- submission-todo: ... -->` placeholders are acceptable where the revision identifier must be set at submission time (they block auto-merge and force the submitter to look up the current revision).
6. Cross-check the coverage roll-up tables in section 3 against the body — every OWASP ML Top 10 entry should have at least one cross-tag row, or an explicit OUT ruling with reason.
7. Confirm the hand-offs table enumerates which mod-105 exercise owns each experimental run, and which modules (104, 108, 109, 110) own the cross-cutting pieces.

## 4. Rubric / review checklist

Scoring is pass/fail at the row level; the submission is accepted if all row-level checks pass and the structural checks pass.

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Running system named + referenced across modules | CarePilot cited + cross-ref list to mod-101/103/104 submissions | Missing or ambiguous system scope |
| R2 | All three sources version-stamped at top | Each of AI 100-2, ATLAS, OWASP ML Top 10 has a revision/date line (a `<!-- submission-todo: ... -->` placeholder is acceptable) | Any source missing its version/date |
| R3 | All four AI 100-2 goal clusters covered | Rows exist for evasion, poisoning (incl. backdoor/supply-chain), privacy, and abuse/availability | A cluster is missing without stated reason |
| R4 | Every row has three framework columns populated | NIST class + ATLAS AML.T#### + OWASP ML## all present | Any row lists only one or two frameworks |
| R5 | ATLAS citations use technique IDs, not tactic names alone | At least one AML.T#### per row | Rows that cite only "ML Attack Staging" with no technique ID |
| R6 | CarePilot applicability ruling is reasoned | IN / OUT / CONDITIONAL with one-line reason; CONDITIONAL rows name the condition | Rulings without reasons or unconditional CONDITIONALs |
| R7 | Each row maps to a register row + exercise destination | Both columns populated or linked to a register TODO | Rows with no downstream handle |
| R8 | Coverage roll-up tables present | Section 3 tables match the body | Only inline rulings, no roll-up |
| R9 | Hand-offs section present | Section 4 enumerates mod-104/105/106/108/109/110 hand-offs by metric/artifact | Peer ownership hidden in L25 rows |
| R10 | No invented facts or metrics | Zero numeric performance claims, zero invented ATLAS case studies, zero invented OWASP entries | Any invented fact |

**Scoring bands.**

- *Pass* — R1–R10 all green.
- *Pass with revisions* — one row violates R4/R5/R6 but overall shape is right; revise and resubmit.
- *Fail* — R3 fails (a goal cluster is silently omitted), or R10 fails (an invented fact is present).

## 5. Common mistakes

- **Treating MITRE ATLAS as a flat list.** ATLAS is an ATT&CK-style matrix — tactics are columns, techniques are rows. A row that only cites a tactic is unreviewable. Use AML.T#### IDs.
- **Mixing OWASP ML Top 10 with OWASP LLM Top 10.** They are two separate OWASP projects. This exercise cross-tags against OWASP ML Top 10. If a row needs the OWASP LLM Top 10 (e.g., LLM01 Prompt Injection), cross-tag it there *in addition* and label the column clearly; do not relabel LLM Top 10 as ML Top 10.
- **Claiming CarePilot is in scope for everything.** The base LLM is vendor-supplied and frozen; the extraction/MIA/inversion attack class on the base LLM is a vendor concern. CarePilot's scope is its *retrained heads* (mod-106 auxiliary classifier, any fine-tuned retriever) plus its *retrieval corpus* plus its *prompt scaffolding*. The applicability ruling column makes this visible.
- **Filing prompt injection against only NIST AI 100-2 PredAI evasion.** AI 100-2 E2023 explicitly covers GenAI prompt injection as its own class. The cross-tag must use that class name, not the PredAI evasion class.
- **Inventing ATLAS case-study linkages.** The ATLAS case-studies index is the source of truth. If a CarePilot-adjacent case study is not in the index, do not invent one; cite the technique and leave the case-study column blank or `n/a`.
- **Collapsing supply-chain into a single row.** AI 100-2 and ATLAS both subdivide supply-chain compromise by vector (software, model, data, hardware). The sheet should either cite the parent AML.T0010 with a note on which subtype applies to CarePilot, or enumerate the subtypes.
- **Confusing ML08 Model Skewing with ML02 Data Poisoning.** OWASP's ML08 is about adversarial feedback loops that shift the model over time (e.g., gaming a recommender); ML02 is about training-set contamination. They can overlap but they are not synonyms — pick the one that fits the row.
- **Leaving the register-row column blank.** This cross-tag sheet exists to drive the risk register. A row with no register pointer (even a TODO) is a sheet entry with no engineering consequence.

## 6. References

Only the sources listed in the work packet are cited.

- **NIST AI 100-2 E2023 — Adversarial Machine Learning: A Taxonomy and Terminology of Attacks and Mitigations.** National Institute of Standards and Technology, U.S. Department of Commerce. `https://csrc.nist.gov/publications/detail/nistir/ai-100-2` (confirm exact revision identifier at submission). Attack-class taxonomy (evasion, poisoning, privacy, abuse), attacker-goal / attacker-knowledge / attacker-capability axes, PredAI and GenAI scope.
- **MITRE ATLAS — Adversarial Threat Landscape for Artificial-Intelligence Systems.** MITRE Corporation. `https://atlas.mitre.org/` — matrix of tactics and techniques (AML.T####), case-studies index.
- **OWASP Machine Learning Security Top 10.** The OWASP Foundation. `https://owasp.org/www-project-machine-learning-security-top-10/` — ML01..ML10 entries, 2023 project nomenclature.
- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — MEASURE 2.7 is the home sub-category this cross-tag files against.
- **mod-101 ex-01 solution** — `modules/mod-101-ai-risk-frameworks-and-position/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map/SOLUTION.md` — the CarePilot MEASURE 2.7 row this sheet extends.
- **Exercise prompt** — `lessons/mod-105-adversarial-ml-and-robustness/exercises/exercise-01-nist-ai-100-2-to-atlas-to-owasp-cross-tag.md` — including the CarePilot running system continuation and the required columns.

<!-- submission-todo: AI 100-2 revision letter/date, ATLAS matrix version, and OWASP ML Top 10 project revision date are set at submission time from the three source URLs; auto-merge is blocked until they are filled in. -->

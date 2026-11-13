# Exercise 01 — AI Incident Triage and Routing Drill — Reference Solution

## 1. Solution overview

This exercise is the module's **incident-response design + drill baseline**. The deliverable is a self-contained triage-and-routing package for **CarePilot** — the hospital-triage-support RAG assistant introduced in mod-101 — that an on-call engineer can pick up at 02:00 and use without a human in the loop for the first thirty minutes of an incident. Concretely the package contains (a) a four-tier **severity/impact rubric** keyed to AI-specific signals, (b) a **routing matrix** that maps each incident type to an owning on-call rotation with an explicit paging rule, (c) a **triage runbook** with exit criteria per step, and (d) a **worked drill** that walks three synthetic scenarios all the way to a declared and routed incident.

The reference solution:

- Defines severities **sev-1 … sev-4** on AI-specific blast-radius signals (jailbreak-rate spike, PHI/PII leakage, model-output abuse, guardrail-bypass volume, upstream model-vendor outage, drift-triggered safety regression, biased-decision complaint) instead of re-using the generic SRE sev ladder. We keep the numeric thresholds *modest* — they are calibrated for a clinical-adjacent system where a false negative is worse than a false positive.
- Treats the **routing matrix** as the source of truth for *who gets paged*. Every cell names (i) the primary rotation, (ii) the mandatory co-page, (iii) the paging rule (immediate page, business-hours ticket, or async channel). No row routes to an individual person — only to rotations — because incident rotations survive personnel changes and personal escalation doesn't.
- Models the **triage runbook** as a short ordered checklist (`detect → classify → sev-assign → page → open incident doc → form bridge → mitigate → comms → declare resolved`). Each step has an explicit **exit criterion** so the on-call can tell when they are allowed to move on. This is the main anti-pattern we are correcting: unbounded "investigation" steps that do not resolve.
- Walks three **synthetic drill scenarios** through the rubric, matrix, and runbook to declared-incident. These scenarios are explicitly labelled as drills on CarePilot; the numbers (jailbreak-rate deltas, token counts, vendor-outage windows) are invented for the drill.
- Produces two **machine-readable configs** — `configs/severity-rubric.yaml` and `configs/routing-matrix.yaml` — that a future on-call bot or Slack workflow can consume without re-parsing markdown.

Design decisions worth calling out:

- **Why AI-specific signals, not latency/error-rate.** Generic SRE sev-rubrics will mis-classify every AI incident in this module: a jailbreak-rate spike is not a 5xx, a biased-decision complaint is not a p99 regression, and a PHI echo is a single successful request. We keep the AI-signal axis orthogonal to SLO breaches because the two can occur independently and both need distinct pages.
- **Why sev-1 fires on single events for PHI.** CarePilot sits in a regulated clinical context. A *single* confirmed PHI disclosure crosses a regulator-trigger threshold (per the applicable health-privacy regime, confirmed with legal). The rubric therefore refuses to batch PHI events; one is sev-1. This is the most load-bearing calibration decision in the file.
- **Why rotations, not people.** On-call pages routed to a specific person have an average resolution time that is a function of that person's vacation calendar. Routing to a rotation makes the SLA property of the rotation, not the person. Every cell in §2.3 names a rotation and lists the minimum-skill composition for that rotation.
- **Why we declare "mitigate" before "root-cause."** The runbook explicitly separates *mitigation* (stop the bleeding) from *resolution* (understand it). In the AI context the mitigation lever is often "pin to the previous model build" or "turn off the affected tool in the agent config," and it is faster than a root-cause investigation. Post-mortems (exercise 02 of this module) absorb the RCA work.
- **Why the drill ships before the real incident.** Teams that have never run a triage drill will burn the first ten minutes of their first real incident deciding on a sev number. The three drills in §2.5 exist so that the first use of the rubric is at 11:00 on a Tuesday, not at 02:00 during a real event.

## 2. Worked answer or implementation

### 2.1 Repository layout

```
exercise-01-ai-incident-triage-and-routing-drill/
├── SOLUTION.md                     # this file
├── configs/
│   ├── severity-rubric.yaml        # machine-readable sev-1 … sev-4 rubric
│   └── routing-matrix.yaml         # incident-type → rotation + paging rule
├── runbook/
│   └── triage-runbook.md           # detect → … → declare resolved; one exit criterion per step
├── drills/
│   ├── drill-01-jailbreak-spike.md
│   ├── drill-02-phi-echo.md
│   └── drill-03-embedding-vendor-zeroday.md
└── templates/
    ├── incident-doc.md             # skeleton the on-call fills during an incident
    └── status-update.md            # outward-facing comms template
```

### 2.2 Severity and impact rubric

The rubric below is the human-readable view of `configs/severity-rubric.yaml`. Each tier lists the AI-specific signals that trigger it, the required response SLA, and the *regulator-trigger* flag that decides whether legal and comms must co-page.

| Sev | Headline | Trigger examples (any one is sufficient) | Response SLA | Regulator-trigger? |
|---|---|---|---|---|
| **sev-1** | User safety harm or regulator-reportable event is probable or confirmed | PHI/PII disclosure confirmed in a user-visible output; biased-decision complaint with a patient-harm allegation; model-output abuse causing a clinical-decision reversal; guardrail-bypass volume ≥ 100 unique users in 1h with safety-tagged payloads | Page primary + co-page legal within 5 min | Yes — legal and comms co-page mandatory |
| **sev-2** | Degraded safety or security posture; no confirmed harm yet | Jailbreak-rate spike ≥ 10x 7-day baseline sustained ≥ 15 min; upstream model-vendor outage affecting ≥ 50 % of inference traffic; drift-triggered safety regression on the weekly eval beyond the pinned regression threshold | Page primary within 15 min; co-page security | Not automatic; escalate to sev-1 if evidence of user harm appears |
| **sev-3** | Elevated risk signal; investigation required; no acute user impact | Jailbreak-rate spike 3x–10x baseline; guardrail-bypass volume between 10 and 100 unique users in 1h; single biased-decision complaint without harm allegation; non-safety drift alarm | Business-hours ticket to primary within 4h | No |
| **sev-4** | Low-signal AI-safety alarm; triage and document only | Single flagged prompt that tripped a guardrail but produced a safe refusal; one-off abusive user; non-safety drift warning below action threshold | Ticketed to the backlog; triaged in weekly review | No |

**Design decision — why these thresholds.** The sev-2 threshold of "10x 7-day baseline sustained ≥ 15 min" is deliberately a *rate ratio* over a *window*, not an absolute count. CarePilot serves variable traffic across the clinical shift; an absolute count would alarm every evening. The sev-1 jailbreak threshold ("≥ 100 unique users in 1h with safety-tagged payloads") is chosen because at that volume we are no longer looking at a probe — we are looking at campaign traffic, which is the point at which product and comms must be on the bridge.

**Machine-readable form.**

```yaml
# configs/severity-rubric.yaml
version: 1
system: CarePilot
owner_rotation: ml-platform-oncall
severities:
  sev-1:
    headline: "User safety harm or regulator-reportable event is probable or confirmed"
    sla_minutes: 5
    regulator_trigger: true
    triggers:
      - id: phi_pii_disclosure_confirmed
        description: "Confirmed PHI or PII disclosure in a user-visible model output (single event is sufficient)"
      - id: biased_decision_with_harm
        description: "Biased-decision complaint alleging patient harm"
      - id: model_output_abuse_clinical_reversal
        description: "Model output causally linked to a clinical-decision reversal"
      - id: guardrail_bypass_campaign
        description: "Guardrail-bypass volume ≥ 100 unique users in 1h with safety-tagged payloads"
  sev-2:
    headline: "Degraded safety or security posture; no confirmed harm yet"
    sla_minutes: 15
    regulator_trigger: false
    triggers:
      - id: jailbreak_rate_spike_10x
        description: "Jailbreak-rate spike ≥ 10x 7-day baseline sustained ≥ 15 min"
      - id: upstream_vendor_outage_major
        description: "Upstream model or embedding vendor outage affecting ≥ 50% of inference traffic"
      - id: drift_safety_regression
        description: "Weekly eval reports a safety-tagged metric beyond its pinned regression threshold"
  sev-3:
    headline: "Elevated risk signal; investigation required; no acute user impact"
    sla_minutes: 240
    regulator_trigger: false
    triggers:
      - id: jailbreak_rate_spike_3x
        description: "Jailbreak-rate spike 3x–10x baseline"
      - id: guardrail_bypass_mid
        description: "Guardrail-bypass volume between 10 and 100 unique users in 1h"
      - id: biased_decision_no_harm
        description: "Single biased-decision complaint with no harm allegation"
      - id: non_safety_drift_alarm
        description: "Drift alarm on a non-safety metric at action threshold"
  sev-4:
    headline: "Low-signal AI-safety alarm; triage and document only"
    sla_minutes: 10080   # 7d
    regulator_trigger: false
    triggers:
      - id: single_guardrail_safe_refusal
        description: "Single guardrail trip where the model produced a safe refusal"
      - id: one_off_abusive_user
        description: "Isolated abusive user without campaign signal"
      - id: non_safety_drift_warning
        description: "Drift warning below the action threshold"
```

### 2.3 Routing matrix

| Incident type | Primary rotation | Mandatory co-page | Paging rule |
|---|---|---|---|
| PHI/PII disclosure | LLM safety on-call | Security on-call + legal/comms | Immediate page, phone + Slack, 24/7 |
| Jailbreak-rate spike | LLM safety on-call | Security on-call | Immediate page if sev-1/2; async if sev-3 |
| Guardrail-bypass campaign | Security on-call | LLM safety on-call | Immediate page if sev-1/2; ticket if sev-3 |
| Upstream model/embedding vendor outage | ML platform on-call | LLM safety on-call | Immediate page; status-page update required |
| Drift-triggered safety regression | ML platform on-call | LLM safety on-call | Immediate page if sev-2; ticket if sev-3 |
| Biased-decision complaint | Product on-call | Legal/comms + LLM safety on-call | Immediate page if harm alleged; ticket otherwise |
| Model-output abuse (clinical) | Product on-call | LLM safety on-call + legal/comms | Immediate page; clinical safety lead co-paged |
| Non-safety drift alarm | ML platform on-call | — | Business-hours ticket |

**Design decision — why co-pages are mandatory, not suggestions.** Every incident type in this matrix has at least one rotation that will *not* think to join the bridge on its own. Legal does not read jailbreak dashboards; security does not watch drift panels. We make the co-page mandatory at the matrix level, not at the discretion of the on-call, so that the "I didn't think to page legal" failure mode cannot occur.

**Design decision — why product owns bias complaints.** A biased-decision complaint is a product-surface event first and a model event second. Product on-call owns the user relationship and the clinical-harm claim; LLM safety and legal join the bridge but do not own the response. This is deliberately different from the "ML owns all model incidents" default.

**Machine-readable form.**

```yaml
# configs/routing-matrix.yaml
version: 1
system: CarePilot
rotations:
  ml-platform-oncall:
    skills: [model-serving, inference-infra, vendor-liaison]
    page_channel: pagerduty:ml-platform
  llm-safety-oncall:
    skills: [prompt-eval, guardrails, red-team-response]
    page_channel: pagerduty:llm-safety
  security-oncall:
    skills: [appsec, abuse-response, campaign-analysis]
    page_channel: pagerduty:security
  product-oncall:
    skills: [clinical-ux, customer-escalation]
    page_channel: pagerduty:product
  legal-comms:
    skills: [regulatory-notification, external-comms]
    page_channel: email+phone
routes:
  - incident_type: phi_pii_disclosure
    primary: llm-safety-oncall
    co_page: [security-oncall, legal-comms]
    paging_rule: immediate_24_7
  - incident_type: jailbreak_rate_spike
    primary: llm-safety-oncall
    co_page: [security-oncall]
    paging_rule: immediate_if_sev1_or_sev2_else_async
  - incident_type: guardrail_bypass_campaign
    primary: security-oncall
    co_page: [llm-safety-oncall]
    paging_rule: immediate_if_sev1_or_sev2_else_ticket
  - incident_type: upstream_vendor_outage
    primary: ml-platform-oncall
    co_page: [llm-safety-oncall]
    paging_rule: immediate_with_status_page
  - incident_type: drift_safety_regression
    primary: ml-platform-oncall
    co_page: [llm-safety-oncall]
    paging_rule: immediate_if_sev2_else_ticket
  - incident_type: biased_decision_complaint
    primary: product-oncall
    co_page: [legal-comms, llm-safety-oncall]
    paging_rule: immediate_if_harm_alleged_else_ticket
  - incident_type: model_output_abuse_clinical
    primary: product-oncall
    co_page: [llm-safety-oncall, legal-comms]
    paging_rule: immediate_clinical_safety_lead_co_paged
  - incident_type: non_safety_drift_alarm
    primary: ml-platform-oncall
    co_page: []
    paging_rule: business_hours_ticket
```

### 2.4 Triage runbook

The runbook is the ordered checklist the on-call executes. Each step has one **exit criterion** — the condition that lets the on-call move to the next step. The runbook is intentionally short; anything longer than this goes into the incident doc, not the runbook.

```md
<!-- runbook/triage-runbook.md -->
# CarePilot AI Incident Triage Runbook

## 1. Detect
Trigger: automated alert fired, external report received, or on-call observation.
Exit: the on-call has one sentence describing the observation (what, where, when-first-seen).

## 2. Classify
Match the observation to an `incident_type` in `configs/routing-matrix.yaml`.
Exit: exactly one incident_type is written into the incident doc. If two match, pick the one with the higher SLA tier.

## 3. Sev-assign
Match the observation to the trigger list in `configs/severity-rubric.yaml`.
Exit: the sev tier is written into the incident doc with the trigger id that matched.

## 4. Page
Fire the primary page and all mandatory co-pages per the routing matrix cell.
Exit: a human has ack'd the primary page. If no ack in 5 min, escalate to the manager on-call.

## 5. Open incident doc
Instantiate `templates/incident-doc.md`. Fill the incident_type, sev, trigger id, and detection sentence.
Exit: the doc URL is posted to the incident Slack channel.

## 6. Form bridge
Open the voice bridge; co-pages join. Assign a scribe (not the IC).
Exit: IC, scribe, and all mandatory co-pages are on the bridge or have confirmed asynchronous participation.

## 7. Mitigate
Apply the mitigation lever (pin previous model build, disable affected tool, rate-limit affected route, roll routing to a backup vendor).
Exit: the signal that triggered the incident has returned below the sev's trigger threshold, OR the mitigation lever is confirmed deployed to production.

## 8. Comms
Issue an internal status update using `templates/status-update.md`. If regulator_trigger is true, legal owns the external comms decision.
Exit: at least one status update has been posted; cadence is set (default: every 30 min for sev-1, every 60 min for sev-2).

## 9. Declare resolved
IC declares resolved when the signal is below threshold for a full sev-specific hold window (sev-1: 60 min; sev-2: 30 min; sev-3: 15 min).
Exit: resolved-declaration posted; post-mortem owner assigned per mod-111 exercise 02.
```

**Design decision — why a scribe who is not the IC.** The IC is making decisions on the bridge and cannot simultaneously timestamp them. Post-mortems (exercise 02 of this module) depend on a reliable timeline; the scribe exists so that timeline survives. The scribe is a named role with its own exit criterion so the runbook cannot skip it.

### 2.5 Worked drill — three synthetic scenarios

Each drill below walks a synthetic observation through the rubric, the routing matrix, and the runbook. The numbers are invented for the drill and are not real CarePilot telemetry.

#### 2.5.1 Drill 01 — Jailbreak-rate spike on the triage endpoint

```md
<!-- drills/drill-01-jailbreak-spike.md -->
# Drill 01 — Jailbreak-rate spike on CarePilot triage endpoint

## Observation (synthetic)
At 14:03, the jailbreak-detector dashboard shows the jailbreak-rate on
/v1/triage rising from 0.2% (7-day baseline) to 4.1% over 15 minutes.
Safety-tagged payloads are 62% of the flagged requests. Unique-user count
in the window: 48.

## Walk-through
1. Detect: dashboard alarm fired at 14:03; on-call observes rate at 14:05.
   One-sentence observation: "Jailbreak-rate on /v1/triage is 20.5x baseline for 15 min with 48 unique users."
2. Classify: incident_type = jailbreak_rate_spike.
3. Sev-assign: ratio is >= 10x baseline sustained >= 15 min → trigger id jailbreak_rate_spike_10x → sev-2.
   Note: unique-user count (48) is below the sev-1 campaign threshold (100), so sev-2 is correct.
4. Page: primary = llm-safety-oncall; co-page = security-oncall. Immediate page per routing rule.
5. Open incident doc: filled with incident_type, sev-2, trigger id, 14:05 detection sentence.
6. Form bridge: IC = llm-safety primary; scribe = security co-page; bridge opened 14:11.
7. Mitigate: rate-limit the detected prompt-pattern at the edge; roll the latest system-prompt change back
   to the previous build; enable the campaign-tagging rule in the WAF. By 14:28 the rate is 0.9%.
8. Comms: internal status update posted at 14:31; cadence 60 min.
9. Declare resolved: rate below trigger for 30 min (sev-2 hold window) → resolved at 14:58.

## Post-incident hand-off
Post-mortem owner: llm-safety-oncall IC from this shift; exercise 02 drives the write-up.
```

#### 2.5.2 Drill 02 — PHI echo from a prior patient session

```md
<!-- drills/drill-02-phi-echo.md -->
# Drill 02 — PHI echo from a prior patient's session

## Observation (synthetic)
At 09:12 a hospital-admin user reports via the in-product feedback form:
"The triage assistant just quoted back a patient MRN and three lines of
clinical history that are not for the patient I'm currently looking at."
Support reproduces the issue once with a different MRN leak at 09:34.

## Walk-through
1. Detect: feedback form → support escalation at 09:14; reproduction at 09:34.
   One-sentence observation: "Two confirmed PHI disclosures of prior-session content to a different user, within 22 min."
2. Classify: incident_type = phi_pii_disclosure.
3. Sev-assign: trigger id phi_pii_disclosure_confirmed → sev-1. (Single confirmed event is sufficient; we have two.)
4. Page: primary = llm-safety-oncall; co-page = security-oncall + legal-comms. Immediate 24/7 page.
5. Open incident doc: fill incident_type, sev-1, trigger id, detection sentence. Flag regulator_trigger=true.
6. Form bridge: IC = llm-safety; scribe = security; legal joins by 09:21.
7. Mitigate: disable the session-context cache tool in the agent config for /v1/triage; force per-request
   context reinitialisation; invalidate all active session caches. By 09:46 the reproduction no longer fires.
8. Comms: internal status update at 09:49. Legal owns the external-notification decision per sev-1
   regulator_trigger; health-privacy counsel determines the notification obligation.
9. Declare resolved: 60-min sev-1 hold window with no new PHI reports → resolved at 10:46, pending legal sign-off on external comms.

## Post-incident hand-off
Post-mortem owner: llm-safety-oncall IC; ex 02 writes the RCA including the session-cache bug.
Register linkage: `risk-register/2025/CarePilot.yaml#risks[privacy-phi-echo]` opened.
```

#### 2.5.3 Drill 03 — Upstream embeddings vendor zero-day

```md
<!-- drills/drill-03-embedding-vendor-zeroday.md -->
# Drill 03 — Upstream embeddings vendor announces a zero-day

## Observation (synthetic)
At 23:40 the embeddings vendor publishes a security advisory: a zero-day in
the embedding model used by CarePilot for the RAG retriever allows a crafted
query to return deterministic chunks of other tenants' embedded content.
CarePilot uses this embedding model for ~100% of retrieval traffic.

## Walk-through
1. Detect: vendor advisory ingested by the vendor-watch feed at 23:42; on-call pages 23:45.
   One-sentence observation: "Upstream embeddings vendor discloses a cross-tenant leak zero-day affecting our retriever."
2. Classify: incident_type = upstream_vendor_outage (we treat vendor-side vulnerabilities as outage-class for routing purposes;
   the payload channel differs but the rotation we need is the same).
3. Sev-assign: >= 50% of inference traffic depends on the vulnerable embedding → trigger id upstream_vendor_outage_major → sev-2.
   Escalation watch: if any cross-tenant leak is reproduced against CarePilot, this becomes sev-1 (phi_pii_disclosure_confirmed).
4. Page: primary = ml-platform-oncall; co-page = llm-safety-oncall. Immediate page with status-page update.
5. Open incident doc: filled with incident_type, sev-2, trigger id, 23:45 detection sentence.
6. Form bridge: IC = ml-platform; scribe = llm-safety. Vendor-liaison role picked up by the ml-platform secondary.
7. Mitigate: fall back to the pinned alternate embedding model (slower, higher cost, lower recall) for the retriever;
   enable retrieval-time per-tenant filter to defence-in-depth against the described leak; disable cross-tenant chunk sharing in the cache.
   By 00:31 all production traffic is on the fallback embedding.
8. Comms: status page updated 00:33; internal update cadence 60 min per sev-2.
9. Declare resolved: fallback stable for 30 min (sev-2 hold window) → resolved-mitigation at 01:03.
   Note: "resolved" here means "mitigated to acceptable risk." The vendor-side fix is tracked as a follow-up work item.

## Post-incident hand-off
Post-mortem owner: ml-platform-oncall IC; ex 02 writes the RCA and the vendor-dependency review.
Register linkage: `risk-register/2025/CarePilot.yaml#risks[supply-chain-embedding]` updated with the drill findings.
```

### 2.6 Decision rationale — sev thresholds and register linkage

- **sev-1 fires on single PHI events.** CarePilot handles clinical-adjacent data; the applicable health-privacy regime treats confirmed disclosure as an individually reportable event. Batching PHI incidents to require "N events in 1h" would miss the regulator trigger. This ties to the CarePilot risk register row `risks[privacy-phi-echo]` and feeds the NIST AI RMF MANAGE function's response-plan requirements. <!-- verify-citation: confirm the specific NIST AI RMF MANAGE subcategory (e.g., MANAGE 2.x / 3.x) that pins the incident-response obligation for user-safety events and cite verbatim. -->
- **sev-2 jailbreak threshold is a rate ratio, not a count.** 10x the 7-day baseline sustained for 15 min is the smallest window that reliably distinguishes "attacker is probing" from "campaign in progress," based on the kind of signal the module 104 red-team exercise produces. This ties to register row `risks[prompt-injection-campaign]`.
- **sev-2 upstream-vendor threshold at >= 50% traffic.** Below 50%, the fallback-vendor path can absorb the outage transparently and the incident does not need an active bridge; above 50%, it does. This is also the threshold at which our status-page policy requires an external notice. Ties to register row `risks[supply-chain-embedding]`.
- **sev-3 as the "elevated signal, no user impact" tier.** We intentionally leave sev-3 a wide band so that signals which are "clearly something" but not acute land here and get a business-hours ticket rather than a page. The failure mode we are avoiding is alert fatigue burning the sev-2 page budget.
- **sev-4 as the documentation tier.** Every sev-4 event still gets a row in the triage log; the post-mortem process from exercise 02 periodically mines sev-4s for patterns (a cluster of sev-4 guardrail trips on the same prompt family is a sev-3 in disguise).

**Standards mapping.**

- **NIST AI Risk Management Framework — MANAGE function.** The rubric, matrix, and runbook are the artifacts the MANAGE function requires for a deployed AI system: documented response plans, assigned responsibilities, and post-incident improvement. The sev-rubric also informs the MEASURE function's monitoring thresholds (what signals are worth measuring and at what level of change action is taken). <!-- verify-citation: confirm the specific MANAGE and MEASURE subcategory numbers in NIST AI RMF 1.0 that map to incident-response plans and monitoring thresholds, and cite verbatim. -->
- **OWASP Machine Learning Security Top 10.** Jailbreak campaigns, guardrail bypass, and prompt-injection all live in the OWASP ML Top 10 risk family; the rubric's AI-signal triggers are the operational indicators corresponding to those risks. <!-- verify-citation: cite the specific OWASP ML Top 10 entry codes (and current revision date) that cover prompt-injection / jailbreak and output-based information disclosure. -->
- **MITRE ATLAS.** Model-output abuse and prompt-injection techniques are enumerated in the MITRE ATLAS matrix; the routing-matrix mapping of these to the security-oncall + llm-safety-oncall pair is the operational reflection of the ATLAS tactic categories. <!-- verify-citation: cite the exact ATLAS technique IDs (e.g., AML.T...) for prompt-injection / jailbreak and model-output-based data exfiltration. -->

### 2.7 Templates

```md
<!-- templates/incident-doc.md -->
# Incident: <one-line title>

- Detected: <timestamp UTC>
- Detector: <alert id | user report | manual>
- Incident type: <routing-matrix id>
- Severity: <sev-1 | sev-2 | sev-3 | sev-4>
- Trigger id: <severity-rubric trigger id>
- Regulator trigger: <true | false>
- IC: <rotation name>
- Scribe: <rotation name>
- Bridge: <url>
- Status page: <url | n/a>

## Timeline (scribe)
- HH:MM — ...

## Mitigation applied
- ...

## Resolution
- Resolved at: <timestamp>
- Hold window observed: <minutes>
- Post-mortem owner: <rotation+shift>
- Register row: <risk-register path>
```

```md
<!-- templates/status-update.md -->
Subject: [CarePilot][<sev>] <one-line title> — update <N>

Status: <investigating | mitigating | resolved>
Impact: <who is affected and how; if regulator_trigger, legal clears this line>
Current action: <what we are doing right now>
Next update: <HH:MM UTC>
```

## 3. Validation steps

1. **Rubric covers all seven AI-specific signal families.** `configs/severity-rubric.yaml` contains at least one trigger for each of: jailbreak-rate spike, PHI/PII leakage, model-output abuse, guardrail bypass, upstream model-vendor outage, drift-triggered safety regression, biased-decision complaint.
2. **Every severity has an SLA and a regulator-trigger flag.** For each of sev-1 … sev-4, `sla_minutes` is a positive integer and `regulator_trigger` is a boolean.
3. **Routing matrix has no individuals.** `configs/routing-matrix.yaml` references only rotation names; grep for `@` / email-looking fields returns nothing outside of the `legal-comms.page_channel` field.
4. **Every incident type in the matrix is reachable from at least one rubric trigger.** A reviewer can trace each `incident_type` in the routing matrix to a `trigger id` in the rubric; the drill walk-throughs do this explicitly.
5. **Runbook has one exit criterion per step.** `runbook/triage-runbook.md` contains exactly one `Exit:` line per numbered step.
6. **Three drills resolve.** `drills/drill-01…03.md` each walk the runbook to a resolved-declaration with a timestamped mitigation and a post-mortem owner assigned.
7. **PHI event is sev-1 on a single occurrence.** The rubric's `phi_pii_disclosure_confirmed` trigger explicitly says "single event is sufficient."
8. **Co-pages are declarative in the matrix, not left to the on-call.** Every row with a non-trivial response has at least one entry in `co_page`.
9. **No invented facts about real organizations, products, or vendors.** The three drills are labelled synthetic; the embedding-vendor advisory is a drill, not a real CVE.
10. **Framework-mapping claims are either verified or marked.** Any specific NIST AI RMF subcategory / OWASP ML Top 10 code / ATLAS technique ID carries a `<!-- verify-citation: ... -->` marker unless the exact reference has been checked against the three official sources.

## 4. Rubric or review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Repo layout matches the shape in §2.1 | `configs/`, `runbook/`, `drills/`, `templates/` directories all present with the named files | Any top-level directory missing or empty |
| R2 | Rubric covers the seven AI-specific signal families | All seven families appear as triggers in `configs/severity-rubric.yaml` | Any family missing or folded silently into a generic SRE sev |
| R3 | Severities have SLAs and regulator-trigger flags | sev-1 … sev-4 each define `sla_minutes` and `regulator_trigger` | Missing or type-wrong fields in any tier |
| R4 | PHI event is sev-1 on a single occurrence | `phi_pii_disclosure_confirmed` present in sev-1 triggers and the drill 02 walk-through enforces it | PHI classified at sev-2 or requires a count threshold |
| R5 | Routing matrix routes to rotations, not people | Only rotation names appear in `primary` and `co_page` fields | Any field names an individual |
| R6 | Co-pages are declarative, not discretionary | Every non-trivial route has a non-empty `co_page` with a documented reason in §2.3 | Co-pages left to the on-call's judgement |
| R7 | Runbook has one exit criterion per step | Every numbered step in `runbook/triage-runbook.md` has exactly one `Exit:` line | Any step has zero or two-plus exit lines |
| R8 | Three drills walk end-to-end | Each drill in `drills/` runs from detect to declare-resolved with a timestamped mitigation | Any drill stops before the mitigation lever is applied |
| R9 | Decision rationale ties sev thresholds to CarePilot risk register | §2.6 names the register rows each tier feeds | Thresholds justified only by analogy to generic SRE |
| R10 | Standards mapping is verified or marked | Any specific NIST / OWASP / ATLAS identifier is either verified against the official source or carries `<!-- verify-citation: ... -->` | A specific identifier is quoted without verification and without marker |
| R11 | Machine-readable configs parse | `configs/severity-rubric.yaml` and `configs/routing-matrix.yaml` parse as YAML and reference the same incident types | YAML parse error or mismatched identifiers between the two files |
| R12 | No invented real-world facts | Drills are labelled synthetic; no real CVEs, vendors, or incidents are quoted | Any real vendor name tied to an invented outage or advisory |

Scoring bands: *Pass* — R1–R9 and R11–R12 all green and R10 is honoured; *Pass with revisions* — one of R6/R8/R9 fails but the rubric and matrix (R2–R5) are sound; *Fail* — any of R4 / R5 / R12 fails.

## 5. Common mistakes

- **Reusing the generic SRE sev-rubric unchanged.** A 5xx-based rubric will never catch a PHI echo, because the request completed with a 200. The sev-rubric must be driven by AI-specific signals in parallel with the SLO rubric.
- **Requiring N events for PHI to reach sev-1.** The first confirmed PHI disclosure is the regulator-trigger event; waiting for a second to upgrade severity delays legal notification past the obligation window.
- **Routing to individuals.** Pages to a named person become undeliverable on PTO, turnover, or a team reorg. Route to rotations, with the rotation's minimum-skill composition documented alongside the config.
- **Leaving co-pages to on-call discretion.** "The on-call will decide whether to page legal" is the way legal doesn't get paged. Make co-pages declarative in the matrix.
- **Making the runbook a monolithic wall of prose.** A runbook without exit criteria becomes an unending checklist that the on-call cannot tell when to leave. One exit criterion per step, in the step.
- **Conflating "mitigated" with "root-caused."** Declaring resolved requires signal-below-threshold for the sev-specific hold window; root-cause lives in the post-mortem (exercise 02). Confusing the two will stretch sev-1 bridges for hours when the mitigation lever worked in minutes.
- **Static thresholds on variable-traffic signals.** "Jailbreak rate > 2%" will alarm every evening on CarePilot. Use ratios over rolling baselines.
- **Running the drill once and shelving it.** Rubric and routing matrix drift as new routes, new vendors, and new eval dimensions ship. The drill is a quarterly recurrence, not a one-time write-up.
- **Dropping the status-page step on sev-2.** External-facing systems need a status note even when the mitigation is invisible to end-users; without it, support traffic spikes and the bridge loses signal.
- **Fabricating a CVE or vendor advisory to make a drill feel real.** Drills must be clearly labelled synthetic; inventing a real-sounding advisory corrupts the drill library for everyone who reads it later.

## 6. References

Only the official sources listed in the work packet (and clearly-labelled module / local references) are used.

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — this exercise's artifacts (severity rubric, routing matrix, runbook, drill walk-throughs, templates) are the operational realisation of the NIST AI RMF **MANAGE** function's response-plan requirements for a deployed AI system, with monitoring thresholds feeding the **MEASURE** function. <!-- verify-citation: cite the specific MANAGE and MEASURE subcategory numbers that pin incident-response plans and monitoring-threshold documentation. -->
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — the AI-specific signal families in the severity rubric (prompt-injection / jailbreak, output-based data disclosure, model-output abuse) correspond to entries in the ML Top 10 catalogue. <!-- verify-citation: cite the exact OWASP ML Top 10 entry codes (and current revision date) that cover prompt-injection / jailbreak and output-based information disclosure. -->
- **MITRE ATLAS.** `https://atlas.mitre.org/` — the routing-matrix mapping from "model-output abuse" and "jailbreak-rate spike" to the security + LLM-safety rotation pair is the operational reflection of the ATLAS tactics for ML-targeted attacks. <!-- verify-citation: cite the exact ATLAS technique IDs for prompt-injection / jailbreak and model-output-driven data exfiltration. -->
- **Module running system.** CarePilot — hospital-triage-support RAG assistant, introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md` and reused across this module for continuity.
- **Module linkages.** Jailbreak-signal calibration draws on mod-104 (LLM red-teaming), guardrail-bypass detection draws on mod-108 (guardrails), drift-triggered safety regression draws on mod-109 (monitoring and drift), and post-mortem write-ups are produced by exercise 02 of this module.
- **Exercise prompt.** `lessons/mod-111-incident-response-and-postmortems/exercises/exercise-01-ai-incident-triage-and-routing-drill.md`.

<!-- Numeric thresholds in §2.2 and the three drills in §2.5 are synthetic by design; the solution does not quote real CarePilot telemetry or real external incidents. -->

# EU AI Act — Control Mappings

This document defines how AuditLayer evidence records map to the EU AI Act (Regulation (EU) 2024/1689). Mappings target Articles 12 (logging), 13 (transparency), and 14 (human oversight) — the operational triad for high-risk AI systems.

## Scope

Healthcare AI systems used to make or inform decisions about patients are classified as **high-risk** under Annex III. That makes Articles 12–14 mandatory operational obligations.

## Mapping rules

### Article 12 — Record-Keeping (Logs)

> "High-risk AI systems shall technically allow for the automatic recording of events ('logs') over the lifetime of the system."

**Triggers when:** any record (always-on).

**Satisfies:** `fully`

**Auditor narrative template:**
> Event automatically logged with model {agent.model} v{agent.model_version}, system prompt hash {agent.system_prompt_hash}, action {action.action_type} on {resource.resource_type} {resource.resource_id} at {timestamp}.

**Article 12(2) requirements covered:**
- (a) Recording of period of each use of the system → `timestamp` + `system.session_id`
- (b) Reference database used to check input data → `resource.system_of_record`
- (c) Input data for which the search led to a match → `action.parameters_hash`
- (d) Identification of the natural persons involved in verification → `outcome.human_in_the_loop.approver_id`

### Article 13 — Transparency and Provision of Information

> "High-risk AI systems shall be designed and developed in such a way as to ensure that their operation is sufficiently transparent to enable deployers to interpret a system's output and use it appropriately."

**Triggers when:** `intent.declared_purpose` is non-empty.

**Satisfies:** `fully`

**Auditor narrative template:**
> Declared purpose recorded per action: "{intent.declared_purpose}". Available to data subjects on request per Article 13(1).

### Article 14 — Human Oversight

> "High-risk AI systems shall be designed and developed in such a way... that they can be effectively overseen by natural persons during the period in which they are in use."

**Triggers when:** `outcome.human_in_the_loop.required == true` AND `outcome.human_in_the_loop.approver_id != null`.

**Satisfies:** `fully`

**Auditor narrative template:**
> Human oversight required and exercised: natural person {outcome.human_in_the_loop.approver_id} approved at {outcome.human_in_the_loop.approved_at}, {seconds_to_approval}s after the action was initiated.

**Article 14(4)(d) "stop button" coverage:** When `outcome.status == "blocked_by_policy"` or `human_override`, the human's ability to interrupt or override is demonstrated by record.

### Article 15 — Accuracy, Robustness and Cybersecurity (partial)

**Triggers when:** the consuming system observes a hash-chain break, an unexpected model version, or repeated failures.

**Satisfies:** `contributes_to`

**Auditor narrative template:**
> Continuous integrity monitoring of agent actions; chain breaks and model version drift surface as security events under Article 15(4).

## What this spec does NOT claim

- It does not certify your AI system as Article 6-compliant.
- It does not replace conformity assessment.
- It produces the operational logs and oversight evidence required by Articles 12–14. Conformity assessment is the program around it.

## Open questions for v1.1

- Mapping to Article 9 (risk management system) — likely requires periodic aggregation outputs, not per-event mapping.
- Mapping to Article 17 (quality management system) — out of scope for an evidence layer.
- General-purpose AI model obligations under Articles 51–55 — separate spec, separate mappings.

PRs welcome — especially from notified bodies and data protection authorities.

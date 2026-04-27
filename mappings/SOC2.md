# SOC 2 Trust Services Criteria — Control Mappings

This document defines how AuditLayer evidence records map to SOC 2 Trust Services Criteria (TSP Section 100, AICPA, 2017 with revisions through 2022). Mappings target the **Security** category (Common Criteria / CC).

## Why SOC 2 is the wedge

Mid-market SaaS companies have an annual SOC 2 cycle. Auditors are starting to issue specific testing requests on AI agent access in current Type II review windows (Schellman, A-LIGN, Prescient observed Q1–Q2 2026). Companies that cannot produce agent-attributable evidence will get qualified opinions.

## Mapping rules

### CC6.1 — Logical Access Controls

> "The entity implements logical access security software, infrastructure, and architectures over protected information assets to protect them from security events to meet the entity's objectives."

**Triggers when:** `actor.actor_type` in (`ai_agent`, `hybrid_human_supervised_agent`) AND `data_sensitivity.classification` in (`restricted`, `regulated`).

**Satisfies:** `contributes_to`

**Auditor narrative template:**
> Non-human principal ({agent.agent_id}) authenticated via session {system.session_id}; access tied to human principal {actor.human_principal_id} who authorized the agent.

### CC6.6 — External System Access

> "The entity implements logical access security measures to protect against threats from sources outside its system boundaries."

**Triggers when:** `system.environment == "production"` AND record involves an external API call OR data leaves the covered system.

**Satisfies:** `contributes_to`

**Auditor narrative template:**
> Agent action originated from host {system.host} (source IP {system.source_ip}); external boundary captured.

### CC7.2 — System Monitoring

> "The entity monitors system components and the operation of those components for anomalies that are indicative of malicious acts, natural disasters, and errors affecting the entity's ability to meet its objectives; anomalies are analyzed to determine whether they represent security events."

**Triggers when:** any record (always-on).

**Satisfies:** `fully` when paired with continuous AuditLayer monitoring.

**Auditor narrative template:**
> Agent action stream continuously monitored for tenant {tenant_id}; anomalies (policy blocks, human overrides, hash chain breaks) flagged in real time.

### CC7.3 — Incident Response (policy-blocked events)

> "The entity evaluates security events to determine whether they could or have resulted in a failure of the entity to meet its objectives (security incidents) and, if so, takes actions to prevent or address such failures."

**Triggers when:** `outcome.status == "blocked_by_policy"` OR `outcome.status == "human_override"`.

**Satisfies:** `fully`

**Auditor narrative template:**
> Agent action {evidence_id} was {outcome.status.replace('_',' ')} on {timestamp}. Error code: {outcome.error_code}. Triaged via incident response workflow.

### CC8.1 — Change Management (model version drift)

> "The entity authorizes, designs, develops or acquires, configures, documents, tests, approves, and implements changes to infrastructure, data, software, and procedures to meet its objectives."

**Triggers when:** the consuming system detects a change in `agent.model_version` or `agent.system_prompt_hash` between sessions for the same `agent.agent_id`.

**Satisfies:** `contributes_to`

**Auditor narrative template:**
> Agent model version {agent.model_version} and prompt hash {agent.system_prompt_hash} captured per action; version drift detection enabled.

## What this spec does NOT claim

- It does not constitute a SOC 2 Type II report.
- It does not replace the auditor's own testing.
- It produces structured evidence the auditor can sample from. The audit opinion remains the auditor's.

## Open questions for v1.1

- Mapping to CC6.3 (role-based access). Likely requires extension fields for role names.
- Privacy category mappings (P-series) for tenants under SOC 2 + Privacy.
- Availability category mappings (A-series) — probably out of scope for evidence.

PRs welcome.

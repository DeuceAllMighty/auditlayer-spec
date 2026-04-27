# HIPAA Security Rule — Control Mappings

This document defines how AuditLayer evidence records map to HIPAA Security Rule controls (45 CFR §164). Every mapping is a deterministic rule. The mapping engine is a pure function over the evidence record.

## Mapping rules

### 164.312(b) — Audit Controls

> "Implement hardware, software, and/or procedural mechanisms that record and examine activity in information systems that contain or use electronic protected health information."

**Triggers when:** `data_sensitivity.regulated_data_types` contains `"PHI"`.

**Satisfies:** `fully`

**Auditor narrative template:**
> Agent {agent.agent_name} ({agent.agent_id}) performed action {action.action_type} on PHI resource {resource.resource_id} ({resource.system_of_record}) at {timestamp}, tied to human principal {actor.human_principal_id}.

### 164.502(b) — Minimum Necessary Standard

> "A covered entity must make reasonable efforts to limit protected health information to the minimum necessary to accomplish the intended purpose of the use, disclosure, or request."

**Triggers when:** `data_sensitivity.regulated_data_types` contains `"PHI"` AND `intent.minimum_necessary_assertion == true`.

**Satisfies:** `fully` (with declared purpose + scoped fields), else `partially`.

**Auditor narrative template:**
> Declared purpose: {intent.declared_purpose}. Fields touched: {data_sensitivity.fields_touched}. Minimum-necessary assertion recorded at action time.

### 164.312(c)(1) — Integrity Controls

> "Implement policies and procedures to protect electronic protected health information from improper alteration or destruction."

**Triggers when:** `action.action_type` in (`write`, `update`, `delete`) AND `data_sensitivity.regulated_data_types` contains `"PHI"`.

**Satisfies:** `fully`

**Auditor narrative template:**
> Agent action that modified PHI was hash-chained for tamper evidence. Output hash: {outcome.output_hash}. Chain verified intact.

### 164.308(a)(1)(ii)(D) — Information System Activity Review

> "Implement procedures to regularly review records of information system activity, such as audit logs, access reports, and security incident tracking reports."

**Triggers when:** any record (always-on).

**Satisfies:** `contributes_to` per record; `fully` when paired with the AuditLayer weekly review export.

**Auditor narrative template:**
> Record included in continuous activity review stream for tenant {tenant_id}.

### 164.312(a)(2)(i) — Unique User Identification

> "Assign a unique name and/or number for identifying and tracking user identity."

**Triggers when:** `actor.human_principal_id != null`.

**Satisfies:** `fully` for the human principal; `contributes_to` for the agent (paired with `agent.agent_id`).

**Auditor narrative template:**
> Action attributed to human principal {actor.human_principal_id} via session {system.session_id}; non-human agent identified as {agent.agent_id}.

### 164.312(a)(2)(iii) — Automatic Logoff (inverse / human oversight)

When an agent action requires human approval and the approval is recorded, this contributes to evidence that long-running agent sessions remained under human control.

**Triggers when:** `outcome.human_in_the_loop.required == true` AND `outcome.human_in_the_loop.approver_id != null`.

**Auditor narrative template:**
> Human approval required and recorded: approver {outcome.human_in_the_loop.approver_id} approved at {outcome.human_in_the_loop.approved_at}.

## What this spec does NOT claim

- It does not assert your **organization** is HIPAA compliant.
- It does not replace your BAA, your risk assessment, or your written policies.
- It produces **evidence**. Compliance is the program around it.

## Open questions for v1.1

- Mapping for 164.308(a)(5) — security awareness and training. Probably out of scope for an evidence layer.
- Mapping for 164.310 — physical safeguards. Probably out of scope.
- How to handle 164.502(a)(1)(iii) "incidental" disclosures by an agent.

PRs welcome.

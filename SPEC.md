# AuditLayer Evidence Spec — v1.0

This document explains every field in the schema and the auditor question it exists to answer.

## Top-level structure

| Field | Auditor question it answers |
|---|---|
| `evidence_id` | "Show me the unique record for this event." |
| `tenant_id` | "Which customer environment did this happen in?" |
| `actor` | "Who ultimately authorized this action?" |
| `agent` | "Which AI system performed it, on what model and version?" |
| `timestamp` | "When did this happen, to the second, in UTC?" |
| `action` | "What did the agent actually do?" |
| `resource` | "What did it touch?" |
| `data_sensitivity` | "Was the data regulated, and if so under which regime?" |
| `intent` | "Why did the agent do this, and was it minimum necessary?" |
| `system` | "From what host, environment, and session?" |
| `outcome` | "Did it succeed, fail, get blocked, or require human approval?" |
| `control_mappings` | "Which framework controls does this satisfy, and what's the workpaper sentence?" |
| `hash_chain` | "Can you prove the record hasn't been altered?" |

## Object: `actor`

Every agent action ties to a human principal. There are no orphan actions in this spec.

| Field | Required | Notes |
|---|---|---|
| `actor_type` | yes | `ai_agent` \| `human` \| `hybrid_human_supervised_agent` |
| `human_principal_id` | required for ai_agent unless explicitly null | Account/user that authorized the agent's session |
| `on_behalf_of` | optional | Department, service account, or other organizational principal |

**Why this design:** HIPAA 164.312(a)(2)(i) requires unique user identification. SOC 2 CC6.1 requires logical access control over protected information. An autonomous agent without a tied human principal fails both.

## Object: `agent`

The agent itself — what it is, what version, what it can call.

| Field | Notes |
|---|---|
| `agent_id` | Stable internal identifier |
| `agent_name` | Human-readable, e.g. `PriorAuthCopilot` |
| `model` | `gpt-4.1`, `claude-sonnet-4.5`, `llama-3.3-70b`, etc. |
| `model_version` | Provider-versioned tag |
| `provider` | `OpenAI`, `Azure OpenAI`, `Anthropic`, `self-hosted`, etc. |
| `framework` | `LangGraph`, `CrewAI`, `MCP`, custom |
| `system_prompt_hash` | SHA-256 of the system prompt at execution time. Lets you prove the prompt didn't change without storing the prompt itself. |
| `tools_available` | Array of tool/function names the agent could have called in this session |

**Why this design:** EU AI Act Article 12 requires logs that record which version of a high-risk system was used. Model swaps are silent failures otherwise.

## Object: `action`

What the agent did at this moment.

| Field | Notes |
|---|---|
| `action_type` | `read` \| `write` \| `update` \| `delete` \| `query` \| `decision` \| `external_call` \| `code_execution` \| `file_access` \| `communication` |
| `tool_invoked` | Name of the MCP tool / function called |
| `parameters_hash` | SHA-256 of the parameters. Lets you prove what was passed without leaking the parameters into the audit log. |
| `reasoning_trace_id` | Pointer to a stored reasoning log if you keep one |

## Object: `resource`

What the agent touched.

| Field | Notes |
|---|---|
| `resource_type` | `phi_record` \| `pii_record` \| `financial_record` \| `file` \| `database_row` \| `api_endpoint` \| `email` \| `message` \| `ticket` \| `other` |
| `resource_id` | Stable identifier — MRN, account number, file path |
| `system_of_record` | `Epic Hyperspace`, `Salesforce`, `Snowflake`, `S3 bucket`, etc. |
| `resource_owner` | Department or user that owns the resource |

## Object: `data_sensitivity`

Classification of the data touched.

| Field | Notes |
|---|---|
| `classification` | `public` \| `internal` \| `confidential` \| `restricted` \| `regulated` |
| `regulated_data_types` | Array. Allowed values: `PHI` \| `PII` \| `PCI` \| `SPI` \| `FTI` \| `CUI` \| `EU_PERSONAL_DATA` \| `NONE` |
| `fields_touched` | The actual schema fields read/written. **Required for HIPAA minimum-necessary defense.** |

## Object: `intent`

Why the action happened. Often the most overlooked dimension and the one that wins HIPAA arguments.

| Field | Notes |
|---|---|
| `declared_purpose` | Human-readable purpose captured at agent initialization (e.g. "Assemble prior-auth packet for CPT 93458") |
| `business_justification` | Operational reason (e.g. "Payer Aetna requires clinical documentation") |
| `minimum_necessary_assertion` | Boolean. Did the agent assert at action time that this satisfied HIPAA 164.502(b)? |

**Why this design:** Without intent, every agent action looks like a security event. With intent, every agent action looks like a documented business decision.

## Object: `system`

Where it ran.

| Field | Notes |
|---|---|
| `environment` | `production` \| `staging` \| `development` |
| `host` | Hostname or container ID |
| `source_ip` | Source IP at time of action |
| `session_id` | Session identifier |
| `request_id` | Per-request identifier for tracing |

## Object: `outcome`

What happened.

| Field | Notes |
|---|---|
| `status` | `success` \| `failure` \| `blocked_by_policy` \| `partial` \| `human_override` |
| `records_affected` | Integer count |
| `error_code` | Optional, on failure or block |
| `output_hash` | SHA-256 of the output payload, for tamper-evidence without storing PHI |
| `human_in_the_loop` | Object: `{ required, approver_id, approved_at }` |

## Object: `control_mappings[]`

The framework controls this evidence record satisfies. Generated by the consuming system's mapping engine, not by the emitter.

| Field | Notes |
|---|---|
| `framework` | `HIPAA` \| `SOC2` \| `EU_AI_ACT` \| `NIST_AI_RMF` \| `FedRAMP` |
| `control_id` | E.g. `164.312(b)`, `CC6.1`, `Art. 12` |
| `control_language` | The literal control text from the framework |
| `satisfies` | `fully` \| `partially` \| `contributes_to` |
| `auditor_narrative` | One sentence the compliance officer copies into a workpaper |

## Object: `hash_chain`

Tamper evidence.

| Field | Notes |
|---|---|
| `prev_hash` | The `this_hash` of the prior record in the tenant's stream, or `genesis` for the first record |
| `this_hash` | SHA-256 over the canonicalized JSON of this record (excluding `hash_chain.this_hash`) plus `prev_hash` |

A consumer can verify the chain by walking the stream in order and recomputing. Any mutation, deletion, or reordering breaks the chain.

## Canonicalization

Records are canonicalized before hashing using:

- JSON, sorted keys
- UTF-8
- `null` for absent optional fields (do not omit)
- Datetimes in ISO 8601 UTC with `Z` suffix
- No trailing whitespace

## Versioning

Spec version is tracked in the `$id` URI of the schema. Breaking changes ship under v2 with a separate `$id`.

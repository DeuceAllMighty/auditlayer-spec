# AuditLayer Evidence Spec

**An open standard for capturing AI agent actions as audit-grade evidence — mapped to HIPAA, SOC 2, and the EU AI Act.**

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-green.svg)](LICENSE)
[![Spec Version](https://img.shields.io/badge/spec-v0.1.0-blue.svg)](SPEC.md)
[![Validate](https://github.com/DeuceAllMighty/auditlayer-spec/actions/workflows/validate.yml/badge.svg)](https://github.com/DeuceAllMighty/auditlayer-spec/actions/workflows/validate.yml)

> **Reference implementation:** A working FastAPI service that emits, hashes, and renders this schema as an auditor-ready PDF lives in a private repo. Read access available to design partners on request: [kevin@getauditlayer.com](mailto:kevin@getauditlayer.com).

## Why this exists

AI agents now read patient records, draft claims, execute trades, and ship code. HIPAA, SOC 2, and the EU AI Act were written for human actors. The result: **68% of organizations cannot distinguish AI agent actions from human actions in their audit logs** ([CSA, 2026](https://cloudsecurityalliance.org)).

Compliance teams are blocking AI deployments they cannot defend to auditors. Vendors disagree on what an "agent action" even *is*.

This repo defines a single, open evidence schema so:

- Agent frameworks (LangGraph, CrewAI, AutoGen, MCP servers) emit a consistent record
- Compliance platforms (Vanta, Drata, Secureframe, AuditLayer) ingest a consistent record
- Auditors (Schellman, A-LIGN, Prescient) review a consistent record

One schema. Many emitters. Many consumers. The compliance officer's vocabulary as the contract.

## What's in the box

| File | Purpose |
|---|---|
| [`schema/agent_action.schema.json`](schema/agent_action.schema.json) | The canonical JSON Schema (Draft 2020-12). |
| [`schema/example_record.json`](schema/example_record.json) | A real, filled record from a healthcare prior-authorization workflow. |
| [`SPEC.md`](SPEC.md) | Plain-English explanation of every field and why it exists. |
| [`mappings/HIPAA.md`](mappings/HIPAA.md) | Mapping rules from agent action patterns to HIPAA Security Rule controls. |
| [`mappings/SOC2.md`](mappings/SOC2.md) | Mapping rules to SOC 2 Trust Services Criteria. |
| [`mappings/EU_AI_Act.md`](mappings/EU_AI_Act.md) | Mapping rules to EU AI Act Articles 12–14. |

## The shape of an evidence record

```json
{
  "evidence_id": "uuid",
  "tenant_id": "string",
  "actor":            { "actor_type", "human_principal_id", "on_behalf_of" },
  "agent":            { "agent_id", "model", "model_version", "provider", "framework", "system_prompt_hash", "tools_available" },
  "timestamp":        "ISO 8601 UTC",
  "action":           { "action_type", "tool_invoked", "parameters_hash", "reasoning_trace_id" },
  "resource":         { "resource_type", "resource_id", "system_of_record", "resource_owner" },
  "data_sensitivity": { "classification", "regulated_data_types", "fields_touched" },
  "intent":           { "declared_purpose", "business_justification", "minimum_necessary_assertion" },
  "system":           { "environment", "host", "source_ip", "session_id", "request_id" },
  "outcome":          { "status", "records_affected", "human_in_the_loop" },
  "control_mappings": [{ "framework", "control_id", "satisfies", "auditor_narrative" }],
  "hash_chain":       { "prev_hash", "this_hash" }
}
```

Every field is designed to answer one specific question an auditor asks. See [SPEC.md](SPEC.md) for the question behind each field.

## Quick start — validate a record

```bash
pip install jsonschema
python - <<'PY'
import json, jsonschema
schema = json.load(open("schema/agent_action.schema.json"))
record = json.load(open("schema/example_record.json"))
jsonschema.validate(record, schema)
print("✓ valid")
PY
```

## Quick start — emit a record (any language)

```python
# Python pseudo-emitter
import json, hashlib, uuid, datetime as dt

def emit(actor_human, agent, tool, resource, fields):
    rec = {
      "evidence_id": str(uuid.uuid4()),
      "tenant_id": "your-tenant",
      "actor":   {"actor_type":"ai_agent", "human_principal_id": actor_human, "on_behalf_of": None},
      "agent":   agent,
      "timestamp": dt.datetime.utcnow().isoformat()+"Z",
      "action":  {"action_type":"read", "tool_invoked": tool, "parameters_hash": "sha256:..."},
      "resource": resource,
      "data_sensitivity": {"classification":"regulated", "regulated_data_types":["PHI"], "fields_touched": fields},
      "intent":  {"declared_purpose":"...", "business_justification":"...", "minimum_necessary_assertion": True},
      "system":  {"environment":"production", "host":"...", "source_ip":"..."},
      "outcome": {"status":"success", "records_affected":1, "human_in_the_loop":{"required":False}},
      "control_mappings": [],   # filled by the consumer's mapping engine
      "hash_chain": {"prev_hash":"genesis", "this_hash":"sha256:..."},
    }
    return rec
```

## Design principles

1. **One record per agent action.** Not per session, not per task. Auditors ask "what happened on April 18 at 14:22" — the schema must answer at that granularity.
2. **The auditor narrative is a first-class field.** Every control mapping carries a workpaper-ready sentence the compliance officer can paste directly. Pure data is not enough.
3. **Tamper evidence is built in.** Hash chains over canonicalized records. Mutate any record, the chain breaks.
4. **Identity is mandatory.** Every agent action is tied to a human principal, even for autonomous agents (the principal who authorized the deployment). No orphans.
5. **Intent is mandatory.** Declared purpose + business justification on every action. HIPAA's minimum-necessary standard is not optional.

## Status & versioning

- Spec version: **1.0** (April 2026)
- Stability: **draft**, accepting public feedback through Q3 2026
- Backward compatibility: any breaking change ships under v2

## Contribute

Open an issue with one of these labels:

- `field-proposal` — propose a new field with auditor justification
- `mapping` — propose a new framework or control mapping
- `emitter` — share a working emitter for a framework (LangGraph, CrewAI, MCP, etc.)
- `auditor-feedback` — feedback from an actual audit firm

PRs welcome. We move fast on auditor feedback.

## Reference implementation

A working ingestion API, dashboard, PDF auditor packet generator, and MCP wrapper that consumes this spec lives at [github.com/auditlayer/auditlayer](https://github.com/auditlayer/auditlayer). Currently private; available to design partners on request.

## License

Apache 2.0. Use it. Build on it. Ship products with it. Just don't trademark "AuditLayer."

---

*Built and maintained by AuditLayer. Find us at [auditlayer.io](https://auditlayer.io).*

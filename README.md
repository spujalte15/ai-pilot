# ai-pilot-eu

Customer-facing POC skills for **Treasure Work** and **Treasure AI Studio**.
Built for the AI POC framework — Data Ops skill build phase (Weeks 4–6).

## Skills

| Skill | JIRA | Description |
|---|---|---|
| `uc-data-lineage` | [ARCH-1335](https://treasure-data.atlassian.net/browse/ARCH-1335) | Full pipeline lineage from raw CDP landing to parent segment — interactive HTML dashboard |
| `uc-data-quality-monitor` | [ARCH-1325](https://treasure-data.atlassian.net/browse/ARCH-1325) | CDP data health monitoring — fill rate, row count trend, ID stitching coverage, dedup rate |
| `uc-parent-segment-overview` | [ARCH-1337](https://treasure-data.atlassian.net/browse/ARCH-1337) | Overview of all parent segments — health, activation summary, child segments, freshness dashboard |
| `skill-usage-tracker` | — | Skill usage tracking for Treasure Work (hooks) and Treasure AI Studio (inline Step 0 reference) |
| `cdp-architecture-map` | — | Evidence-backed mapping of parent segments, schemas, identifiers, joins, freshness, and glossary candidates |
| `business-glossary-build` | — | Progressive requirements interview for business terms, metrics, identity, governance, mappings, and approval |
| `studio-context-file` | — | Builds and validates portable organizational context and acceptance-test artifacts for Treasure Work and Treasure AI Studio |

## Organizational Context Workflow

```text
Live Treasure Data environment
          ↓
cdp-architecture-map
          ↓
[company]-architecture.md
          ↓
business-glossary-build
          ↓
[company]-glossary.md
          ↓
studio-context-file
          ↓
[company]-context.md + [company]-context-acceptance-tests.md
```

The workflow supports two execution modes:

- **Discovery mode:** fast orientation and draft artifacts. It cannot be labelled production-ready.
- **Production mode:** deeper identity/join validation, metric semantics, governance review, provenance, and approvals.

Execution mode is distinct from artifact readiness. `cdp-architecture-map` and `business-glossary-build` can produce the intermediate status `Pilot-ready input`; only `studio-context-file` can grant final `Pilot-ready` or `Production-ready` status after merge and runtime acceptance tests.

## Adding to Marketplace

This repository uses a Claude Code-compatible marketplace package layout for distribution. The skill instructions themselves are model-agnostic and define required capabilities rather than depending on Claude-specific behavior.

To load these skills in Treasure Work or Treasure AI Studio, add:

```json
{
  "source": "github:treasure-data/ai-pilot-eu",
  "plugin": "ai-pilot-eu"
}
```

Or clone and load locally:

```json
{
  "source": "./path/to/ai-pilot-eu",
  "plugin": "ai-pilot-eu"
}
```

## Model and Runtime Contract

The organizational-context skills perform a preflight before substantive work. They check runtime interfaces, TD access, artifact access, context capacity, multi-turn state, evidence tracking, conflict handling, and confirmation support.

They use four capability tiers:

| Tier | Typical capability | Context workflow use |
|---|---|---|
| 1 — Basic | Direct extraction and Markdown formatting | Approved draft formatting only |
| 2 — Standard | Reliable tools, multi-turn state, multiple moderate sources | Discovery mode |
| 3 — Advanced | Cross-source synthesis, data-model reasoning, provenance and conflict handling | Production mode |
| 4 — High-assurance | Long-horizon validation, identity/join/governance review | Sensitive or high-risk certification evidence |

Model selection is capability-first. When a concrete model is requested, ask the runtime for a currently available model that demonstrates the required tier, reliable tool use, and sufficient context. The workflow does not depend on a vendor or model family.

If the active model or runtime cannot complete a step safely, the skills stop with a structured `EXECUTION BLOCKED` record. They preserve valid work, identify the missing capability, recommend a minimum tier and safe fallback, and never fabricate missing facts.

## Dual-Runtime Compatibility

| Concern | Treasure Work | Treasure AI Studio | Portable behavior |
|---|---|---|---|
| TD authentication | Use connected account; on 401/403 open Settings (gear) and Add Account/Re-authenticate | Use Studio's connected credential/request flow | Never request API keys in chat |
| TD discovery | Available TD tools or local `tdx` | Available TD tools or authorized `tdx` execution | Read-only commands and bounded queries |
| Files/artifacts | Authorized workspace and file viewer | Studio artifact/file mechanism | Markdown is the interchange format |
| Knowledge-base publishing | Use an available connector/tool after approval, or provide the file | Use current Studio knowledge-base UI/tool after approval | Publication is separate from generation |
| Retrieval | Connector/runtime dependent | Relevance-based KB retrieval | Never guarantee every fact is injected into every answer |
| Missing capability | Structured exception and safe fallback | Structured exception and safe fallback | Downgrade readiness rather than guess |

Runtime-specific tool names are optional adapters, not requirements. A compatible runtime may substitute equivalent interfaces as long as it preserves evidence, authorization, and output semantics.

## Skill Design Principles

- **Model-agnostic instructions:** imperative steps, explicit inputs/outputs, capability checks, and stable exception codes.
- **Evidence before interpretation:** observed facts, user confirmations, inferences, conflicts, and unknowns remain distinct.
- **No silent conflict resolution:** business, data, and governance domains have explicit owners and precedence.
- **Read-only discovery by default:** schema exploration uses safe, bounded TD commands and avoids shell-pipeline assumptions.
- **Privacy-aware sampling:** avoid reproducing raw PII or sensitive values in artifacts.
- **Human approval:** production readiness and shared knowledge-base publication require explicit approval.
- **Tested context:** final output includes persona, metric, join, unknown, market, and governance acceptance tests.
- **Portable artifacts:** Markdown source artifacts work in both Treasure Work and Treasure AI Studio.

## Tracking

Where usage tracking is enabled and authorized, skill invocations log to `ai_usage.skills_usage_tracker` on eu01:

```sql
SELECT skill_name, user_id, td_time_string(time, 's!', 'UTC') AS invoked_at
FROM ai_usage.skills_usage_tracker
ORDER BY time DESC
LIMIT 20
```

Tracking availability is runtime-specific. Failure to write telemetry must not cause the skill to invent success or bypass the runtime's privacy and authorization controls.

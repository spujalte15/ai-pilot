---
name: studio-context-file
description: Use when the user asks to build, update, validate, publish, or load an organization-specific context or knowledge-base file for Treasure Work or Treasure AI Studio. Deterministically merges architecture and glossary artifacts, preserves provenance and conflicts, applies governance and readiness gates, and generates context plus acceptance-test artifacts. Requires Tier 2 capabilities for Draft/Pilot and Tier 3 for Production-ready; uses structured exceptions and never claims readiness or retrieval guarantees when inputs, approvals, tools, or model capability are insufficient.
---

# Organizational Context File

Build `[company]-context.md` for use in Treasure Work and Treasure AI Studio, plus `[company]-context-acceptance-tests.md`. The context helps a knowledge-enabled assistant retrieve organization-specific definitions and data guidance. Knowledge-base retrieval is relevance-based and runtime-dependent: loading a file does **not** guarantee that every fact is injected into every conversation or answer.

## 1. Select Build Path and Readiness Target

Use one of these paths:

| Path | Inputs | Maximum readiness |
|---|---|---|
| Draft shortcut | Inline answers only | Draft |
| Standard merge | Architecture + glossary | Pilot-ready if gates pass |
| Governed merge | Production-mode inputs + approvals + passed acceptance tests | Production-ready |

Never label an artifact production-ready merely because all template sections are filled.

## 2. Run the Execution Preflight

Record runtime, model if exposed, artifact access, output/write capability, knowledge-base publication capability, requested readiness, and available sources.

The active model must demonstrate:

- Accurate reading of multiple structured artifacts.
- Fact-level source and status preservation.
- Deterministic merging without silent conflict resolution.
- Governance-rule preservation.
- Long-context handling or safe chunking with reconciliation.
- Markdown generation and structural validation.
- Acceptance-test generation and result evaluation.

Use these capability tiers:

| Tier | Suitable use |
|---|---|
| 1 — Basic | Format an already reconciled draft |
| 2 — Standard | Small draft/pilot merge with no significant conflicts |
| 3 — Advanced | Production-scale merge, provenance, governance, acceptance tests |
| 4 — High-assurance | Sensitive or high-risk independent review and certification evidence |

Require Tier 2 for Draft/Pilot and Tier 3 for Production-ready. The runtime or user may supply a trusted model-tier classification. Otherwise, evaluate only capabilities demonstrated during preflight; do not claim hidden self-knowledge about the model. If the required capability cannot be demonstrated, stop with `MODEL_CAPABILITY_INSUFFICIENT`. If a concrete recommendation is needed, ask the runtime for an available model meeting the required tier; do not depend on a vendor or model family.

Use `READY`, `READY_WITH_LIMITATIONS`, or `BLOCKED`. For `READY_WITH_LIMITATIONS`, list affected sections/tests and reduce readiness; emit a blocking exception only when a required gate cannot be completed.

If a source exceeds context limits, use deterministic chunking by heading and carry forward source IDs. If no safe reconciliation pass is possible, emit `CONTEXT_LIMIT_EXCEEDED` instead of omitting sections.

## 3. Validate Source Artifacts

Preferred inputs:

- `[company]-architecture.md`
- `[company]-glossary.md`
- Optional governance/policy sources
- Optional prior context and acceptance-test results

Check each source for:

- Version/date and scope.
- Mode and readiness.
- Evidence/source register.
- Open conflicts and blockers.
- Owners and approval records.
- Compatibility of table/column names and IDs.
- Source runtime and any runtime-specific assumptions.
- Governance records: only `APPROVED_POLICY` or an approved policy source may become a normative production restriction; other governance statements remain proposed, contested, or TBD.

A missing architecture or glossary forces the Draft shortcut unless equivalent validated information is supplied. An unreadable or truncated source is not equivalent.

## 4. Merge Deterministically

Apply this precedence by **fact domain**, not by file recency alone:

1. Approved governance/policy source for permissions and restrictions.
2. Business-owner-approved glossary for business definitions.
3. Data-owner-approved architecture for schema, identity, joins, and freshness.
4. User-confirmed interview records within the participant's approved domain.
5. Observed but unapproved source facts.
6. Inferences, which remain explicitly non-normative.

Additional rules:

- A source cannot override a domain it does not own.
- Newer evidence does not silently replace an approved definition; create a change proposal.
- Preserve market, product, date, and persona scope.
- Keep `OBSERVED`, `USER_CONFIRMED`, `INFERRED`, `CONTESTED`, and `UNKNOWN` distinct.
- Keep evidence class, validation status, and confidence orthogonal. Approval never promotes an `INFERRED` fact to `OBSERVED` or `USER_CONFIRMED`; preserve the source evidence class.
- Use the shared validation enum: `PROPOSED`, `CONFIRMED_BY_BUSINESS`, `CONFIRMED_BY_DATA_OWNER`, `APPROVED_POLICY`, `CONTESTED`, or `TBD`.
- Never publish a contested claim as a rule.
- When sources disagree, create a Conflict Register entry. Qualify when a non-blocking uncertainty has a safe bounded answer; ask when the user/owner can choose the applicable scope; refuse or escalate when governance, safety, identity, or material decision correctness could be violated.

Record each normative context statement:

```text
CONTEXT_FACT_ID: CTX-###
STATEMENT: [one actionable statement]
DOMAIN: business | metric | schema | identity | market | governance | use-case
SCOPE: [market/product/persona/time]
SOURCE_IDS: [GLO/ARC/policy IDs]
SOURCE RUNTIME(S): [runtime list]
EVIDENCE CLASS: OBSERVED | USER_CONFIRMED | INFERRED | CONTESTED | UNKNOWN
VALIDATION STATUS: PROPOSED | CONFIRMED_BY_BUSINESS | CONFIRMED_BY_DATA_OWNER | APPROVED_POLICY | CONTESTED | TBD
OWNER: [name/team]
LAST VALIDATED: [date]
```

## 5. Handle Conflicts and Missing Facts

For every conflict, record:

- Conflicting statements and sources.
- Domain and affected use cases.
- Named resolution owner, fallback owner, and review date.
- Temporary assistant behavior.
- Resolution class: `BLOCKING`, `NON_BLOCKING`, or `NOT_APPLICABLE`.
- Whether it blocks Pilot or Production readiness.

Use `[TBD — owner: name/team — impact: description]` for unknowns. `TBD` is a validation status, not automatically a blocker. Classify it as `BLOCKING` only when it affects an in-scope definition, metric, identity/join, governance, safety, or required approval and no safe defer/qualify behavior exists. A blocking TBD is allowed only in Draft.

Contested items belong in the Conflict Register, not in normative Business Definitions, Metrics, Identity Rules, or Governance Rules. Those sections may link to the conflict and state temporary behavior.

Do not ask the model to decide policy, legal meaning, or business ownership.

## 6. Build `[company]-context.md`

Use this structure:

```markdown
# [Company] — Organizational Context for Treasure Work and Treasure AI Studio

- Version:
- Created/updated:
- Readiness: Draft | Pilot-ready | Production-ready
- Applicable runtime(s): Treasure Work | Treasure AI Studio | Both
- Scope:
- Business owner:
- Data owner:
- Governance owner:
- Required review date:
- Source artifacts and versions:

## How to Use This Context
- Use organization-specific definitions when retrieved and applicable to the stated scope.
- Cite or name the governing context fact when precision matters.
- Ask for clarification when a fact is TBD, contested, out of scope, or stale.
- Do not claim this file was retrieved unless the runtime confirms retrieval.

## Organization, Users, and Goals
[Company purpose, TD usage, personas, priority tasks, outputs, and success criteria]

## Scope and Supported Use Cases
[Included/excluded markets, products, decisions, and workflows]

## Market and Organizational Structure
| Code | Meaning | Scope/status | Source IDs |

## Business Definitions
### [Term]
- Definition:
- What it is not:
- Example / counterexample:
- Scope and variation:
- TD mapping:
- Validation/owner:
- Source IDs:

## Metrics and Scores
[Include numerator, denominator, grain, window, filters, attribution, currency,
timezone, late-data handling, variation, mapping, owner, and source IDs]

## Data Model and Freshness
| Object | Purpose | Key/grain | Freshness | Validation | Source IDs |

## Identity and Join Rules
[Canonical/source keys, entities, cardinality, match limits, deduplication,
temporal rules, prohibited joins, and source IDs]

## Governance and Permitted Use
[PII, consent, suppression, activation, residency, retention, role restrictions,
prohibited analysis/export, and escalation owner]

## What the Assistant Must NOT Assume
[Explicit negative rules, deprecated fields, unsafe joins, unsupported scopes]

## Unknown and Escalation Behavior
[When to ask, qualify, refuse, or route to a named owner]

## Conflict Register
| ID | Conflict | Affected use | Temporary behavior | Owner | Status |

## Active Use Cases and Persona Guidance
| Persona | Task | Expected output | Success criteria | Restrictions |

## Fact Register
[CTX records]

## Approval Record
| Domain | Approver | Decision | Date | Scope/conditions |

## Change Log
| Date | Change | Source/change request | Updated by |
```

Keep the retrieval artifact concise enough to retrieve effectively. Put detailed raw evidence in source artifacts, not in the context. Split very large contexts into stable topic files only if the target knowledge base supports multiple files; include a manifest and shared version.

## 7. Generate and Run Acceptance Tests

Create `[company]-context-acceptance-tests.md`:

```markdown
# [Company] — Context Acceptance Tests

- Context version:
- Runtime/model tested:
- Knowledge base/version:
- Test date:
- Evaluator:

| ID | Category | Applicable? + reason | Criticality | Persona | Prompt | Pass criteria | Prohibited elements | Evidence/source IDs | Result | Notes |
```

Assess every category and record either `Applicable: yes` or `Applicable: no` with a reason:

1. Definition retrieval.
2. Negative interpretation.
3. Schema/table selection.
4. Market variation.
5. Metric calculation.
6. Join safety.
7. Unknown handling.
8. Governance behavior.
9. Each priority persona/use case.

For each applicable test, mark `CRITICAL` when failure could make an in-scope decision incorrect, unsafe, non-compliant, or unusable; otherwise mark `INFORMATIONAL`. Each readiness target must have at least one critical test.

Define pass criteria before running the prompt. Use these minimum rubrics:

| Category | Minimum PASS criteria | Automatic FAIL |
|---|---|---|
| Definition | Matches the scoped approved definition and names/cites its source when supported | Generic or contradictory definition |
| Negative interpretation | Avoids the documented wrong assumption | Repeats the prohibited assumption |
| Schema selection | Selects only validated objects and respects scope/freshness | Invented or deprecated object |
| Market variation | Applies the correct market override or asks for market | Silently applies another market's rule |
| Metric | Correct numerator, denominator, grain, window, filters, currency/timezone where applicable | Material calculation semantic is missing/wrong |
| Join safety | Uses validated keys/cardinality or refuses/asks | Unsafe join or unqualified row multiplication |
| Unknown | States the unknown and asks/escalates per owner | Fabricates a value or policy |
| Governance | Applies approved restrictions without exposing restricted data | Leakage, prohibited action, or invented policy |
| Persona/use case | Meets documented expected elements and restrictions | Misses a critical success criterion |

Run tests in the target runtime when authorized and tooling permits. Otherwise leave them `NOT_RUN`, identify the designated tester, and limit readiness accordingly. The evaluator may be the user, domain owner, or an independent model/human reviewer, but must be named.

For each test:

- Evaluate semantic correctness, not exact wording.
- Require citations/source IDs when the runtime supports them.
- Verify the answer asks or escalates rather than inventing unknowns.
- Record `PASS`, `FAIL`, `NOT_RUN`, or `NOT_APPLICABLE`; never infer a pass.
- Treat governance leakage, unsafe joins, invented policy, or failure of a critical criterion as blocking.
- Preserve the failing response and remediation note; rerun after fixing the context rather than downgrading the criterion.

## 8. Apply Readiness Gates

### Draft

Require:

- Scope, intended users, and at least one use case.
- Sources and unknowns identified.
- No hidden conflict.

### Pilot-ready

Require Draft plus:

- Architecture and glossary inputs both carry the intermediate status `Pilot-ready input`.
- Definitions, metrics, identity/joins, and governance are complete for pilot scope.
- Business and data-owner approval is recorded for pilot scope.
- No unresolved item is classified `BLOCKING` for pilot scope.
- Every acceptance category has applicability and criticality recorded.
- All applicable critical pilot tests pass in the target runtime; informational failures are documented and explicitly accepted by the owner.

### Production-ready

Require Pilot-ready plus:

- Production-mode architecture validation.
- Required business, data, and governance approvals are recorded; normative restrictions are `APPROVED_POLICY`.
- No contested/TBD fact is classified `BLOCKING`.
- In each target runtime, every category is either `NOT_APPLICABLE` with a reason or has all critical tests passed; informational failures require an explicit owner waiver.
- Retrieval limitations are documented.
- Named owner, review cadence, and change process are present.

If gates fail, publish at the lower valid readiness and list unmet gates. Never change failed tests to warnings solely to reach a target label.

## 9. Publish by Runtime

### Treasure Work

- Save the artifacts in the authorized workspace or requested project location.
- Use the runtime's file viewer when available so the user can review them.
- If a knowledge-base connector is available, publish only after the requested approval. Otherwise provide the files and exact next step.
- On TD authentication errors, direct the user to **Settings** (gear icon) and **Add Account** or **Re-authenticate**.

### Treasure AI Studio

- Discover the currently available artifact and knowledge-base interfaces; do not assume a fixed tool name or UI path.
- Create or expose the Markdown artifacts using an available Studio artifact/file mechanism.
- Add the approved context file through an available knowledge-base interface only after publication approval.
- If programmatic publication is unavailable, provide the approved file and concise manual-upload instructions based on the current UI; record `PUBLICATION_UNAVAILABLE` only when neither path is possible.
- Use the runtime's connected-credential flow; never ask for secrets in chat.
- Confirm that the file appears and note its version. Do not claim retrieval success until tests demonstrate it.

### Other/unknown runtime

- Provide portable Markdown files.
- State which publication and retrieval tests could not be performed.
- Limit readiness to what the evidence supports.

Publishing to an external/shared knowledge base is an outward-facing action. Obtain explicit user approval before performing it unless that exact publication was already authorized.

## 10. Draft-Only Quick Capture

Use this shortcut only when the user cannot provide the architecture and glossary. Label the result `Draft — unvalidated quick capture`.

Ask progressively for:

1. Intended users, priority tasks, and success criteria.
2. Primary parent segment/database and whether the mapping is verified.
3. Core business term and counter-definition.
4. Primary metric with denominator, grain, and time window.
5. Customer/entity identifier and known join caveats.
6. Active markets and variations.
7. PII, consent, activation, or usage restrictions.
8. Known wrong assumptions and escalation owner.

Do not use quick capture to grant Pilot-ready or Production-ready status.

## 11. Final Validation

Before completion verify:

- Every normative statement has source IDs, scope, owner, and validation state.
- Every referenced `ARC-###`, `GLO-###`, or policy ID exists in a declared source artifact; missing references cause `VALIDATION_FAILED`.
- Source precedence was applied by domain.
- Contradictions are visible and have temporary behavior.
- No unsupported knowledge-base retrieval guarantee remains.
- Governance rules and negative assumptions survived the merge.
- Readiness equals the gates actually passed.
- Acceptance results identify runtime and model.
- Artifact versions and review date are present.

## Structured Exceptions

When blocked, preserve valid output and return:

```text
EXECUTION BLOCKED
Code: [stable code]
Step: [step]
Runtime: [runtime]
Required capability: [capability/access]
Observed limitation: [exact limitation]
Work completed: [valid artifacts/sections]
Data not collected: [missing work]
Safe fallback: [draft, chunking, stronger model, manual publication, or none]
Recommended model tier: [1–4]
Example model class: [optional, non-binding]
User action: [specific next action]
```

Use: `RUNTIME_UNSUPPORTED`, `MODEL_CAPABILITY_INSUFFICIENT`, `MODEL_UNAVAILABLE`, `SOURCE_ARTIFACT_MISSING`, `SOURCE_ARTIFACT_UNREADABLE`, `CONTEXT_LIMIT_EXCEEDED`, `CONFLICT_UNRESOLVED`, `APPROVAL_REQUIRED`, `VALIDATION_FAILED`, `OUTPUT_WRITE_UNAVAILABLE`, `PUBLICATION_UNAVAILABLE`, or `RETRIEVAL_TEST_FAILED`.

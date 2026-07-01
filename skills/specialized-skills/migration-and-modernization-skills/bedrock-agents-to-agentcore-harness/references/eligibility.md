# Eligibility rubric

Check each condition against the discovery manifest. A **hard-stop** means the agent has a feature with no validated AgentCore Harness path — stop the migration, name the failing condition, and suggest the manual alternative. Do not migrate the eligible parts of an ineligible agent: a half-migrated agent is worse than a clear "not yet."

State the result of *every* condition to the user, not just the first failure — they need the full picture to decide what to fix.

## Hard-stop conditions

### 1. Multimodal input
**Signal:** the agent processes images or audio (vision model for image input, or documented image/audio handling). The validated harness path is text-only.
**Alternative:** keep on Bedrock until a multimodal path is validated, or migrate accepting that image/audio is dropped.

### 2. Multi-agent collaboration
**Signal:** `agentCollaborationMode` is `SUPERVISOR` or `SUPERVISOR_ROUTER` — any collaboration, resolvable or not. Out of scope: do **not** flatten collaborators into the prompt, wire them as sub-agents, or migrate the supervisor alone.
**Alternative:** migrate a single non-collaborating agent, or handle the graph manually.

### 3. Unreachable knowledge base
**Signal:** an associated KB is in an account/region the credentials cannot reach — so `bedrock-agent-runtime:Retrieve` can't reproduce its retrieval.
**Alternative:** grant cross-account/region access, then re-run.
**Not the KB *type*:** every type (`VECTOR`/`MANAGED`/`KENDRA`/`SQL`) is reachable via `Retrieve` and eligible; type only picks the wiring (see [`mapping.md`](mapping.md)).

### 4. Custom orchestration
**Signal:** `orchestrationType` is `CUSTOM_ORCHESTRATION` (custom orchestration Lambda). The harness runs its own loop; that control flow has no equivalent and would be silently dropped.
**Alternative:** re-express the logic as harness tools/prompt as a fresh design, or keep on Bedrock.

## Eligible — do not mistake these for blockers

- **DRAFT-only agent** (no published version) — common; treat DRAFT as the source (see [`discovery.md`](discovery.md)).
- **Mixed action-group schema styles** (`functionSchema` and OpenAPI in one agent).
- **Managed KB with non-default retrieval config, code interpreter, guardrails, session memory** — all have harness targets (see [`mapping.md`](mapping.md)).

(Return-of-Control action groups are eligible for migration overall but that *capability* can't be reproduced — classified **cannot** in the assessment, see [`mapping.md`](mapping.md).)

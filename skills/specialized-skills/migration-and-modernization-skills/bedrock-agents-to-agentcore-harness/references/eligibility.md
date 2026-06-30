# Eligibility rubric

Check each condition against the discovery manifest. A **hard-stop** means the agent has a feature with no validated AgentCore Harness path — stop the migration, name the failing condition, and suggest the manual alternative. Do not migrate the eligible parts of an ineligible agent: a half-migrated agent is worse than a clear "not yet."

State the result of *every* condition to the user, not just the first failure — they need the full picture to decide what to fix.

## Hard-stop conditions

### 1. Multimodal input
**Signal:** the agent accepts or processes images or audio (vision-enabled foundation model used for image input, or documented image/audio handling in the instruction or action groups).
**Why stop:** the harness path validated by this skill is text-in/text-out. A multimodal harness flow has not been verified, so migrating one risks silent behavior loss.
**Alternative:** keep the agent on Bedrock until a multimodal harness path is validated, or migrate only if the user accepts dropping image/audio and re-tests manually.

### 2. Unresolvable multi-agent graph
**Signal:** `agentCollaborationMode` is `SUPERVISOR` or `SUPERVISOR_ROUTER` and one or more collaborators in `manifest.collaborators` cannot be resolved — owned by another team, in another account, or not discoverable.
**Why stop:** migrating a supervisor without its collaborators leaves dangling references; the result is a broken agent. You cannot migrate half a graph.

### 3. Unreachable knowledge base
**Signal:** an associated KB is in an account/region the migration's credentials cannot reach (cross-account/region with no access).
**Why stop:** retrieval is reproduced by calling `bedrock-agent-runtime:Retrieve` against the KB. What can't be reached can't be retrieved, so its behavior can't be migrated.
**Alternative:** grant cross-account/region access, then re-run.
**Not a blocker — KB *type*.** All Bedrock KB types (`VECTOR`, `MANAGED`, `KENDRA`, `SQL`) are reachable via `Retrieve` and therefore eligible. Type only decides the target wiring (MANAGED → native connector; others → KB shim) — see [`mapping.md`](mapping.md). A managed KB with non-default retrieval config (reranker/filter/hybrid/top-k) is eligible too; the shim preserves it.

### 4. Custom orchestration
**Signal:** `orchestrationType` is `CUSTOM_ORCHESTRATION`, or the agent uses a custom orchestration Lambda.
**Why stop:** the harness runs its own managed orchestration loop. A custom orchestration Lambda has no clean harness equivalent and its control flow would be silently dropped.
**Alternative:** re-express the custom logic as harness tools/prompt and migrate as a fresh design, or keep on Bedrock.

## Eligible (proceed) — do not mistake these for blockers

- **DRAFT-only agent** (no published version). Common; treat DRAFT as the source. See [`discovery.md`](discovery.md).
- **Managed KB with custom retrieval config.** The KB shim preserves reranker, metadata filter, hybrid override, and top-k.
- **Mixed action-group schema styles** (`functionSchema` and OpenAPI in one agent). Each maps to its own Gateway target.
- **Return-of-Control action groups, code interpreter, guardrails, session memory.** All have harness targets — see [`mapping.md`](mapping.md).

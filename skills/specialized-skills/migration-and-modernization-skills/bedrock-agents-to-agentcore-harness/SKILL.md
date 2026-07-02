---
name: bedrock-agents-to-agentcore-harness
description: Migrate a Bedrock Agent to an AgentCore Harness using the AgentCore CLI. Use when the user wants to migrate (or port/convert) a Bedrock Agent — given its id, name, or ARN — to an AgentCore Harness.
---

# Bedrock Agents to AgentCore Harness

Migrate a Bedrock Agent into an AgentCore **Harness** (the managed agent loop) using the **AgentCore CLI**. The skill drives the CLI to scaffold and deploy; it never hand-rolls boto3 infrastructure the CLI owns.

Two ideas run through every phase:

- **gate** — a hard checkpoint the migration must pass before continuing. Some gates are eligibility (a source feature with no validated harness path); others are user approval (account, region, cost). A failed gate stops the migration and is reported, never worked around silently.
- **mirror** — preserve user-visible *behavior*, not Bedrock-Agents *structure*. Shed the scaffolding the harness now owns (orchestration-loop mechanics, collaboration plumbing), but keep the *intent* inside it — a non-DEFAULT orchestration prompt can encode real tool-routing or input-classification logic worth folding into the system prompt (see [`references/mapping.md`](references/mapping.md)). Keep what the agent does; drop only what the harness does for you.

## Capture the request first

The triggering request *is* the input. Extract whatever the user gave — agent **id/name/ARN**, **region**, **profile/account** — and **confirm** it in the phases below rather than re-asking. Identity resolution (Phase 1) runs *after* preflight, since listing agents to disambiguate a name needs a confirmed account + region first.

**Fail fast.** If the request *already* names a hard-stop disqualifier (multi-agent, custom orchestration, multimodal — see [`references/eligibility.md`](references/eligibility.md)), say so and stop before preflight, giving that condition's specific alternative. Phase 3 still runs the full gate for everything that clears this.

## Phases

Finish each phase and summarize before the next.

### Phase 0 — Preflight (environment gates)

Establish the migration runs against the account, region, and CLI the user intends — before touching the source agent.

1. Confirm the **CLI** is present and current per [`references/cli.md`](references/cli.md). Stop if it is missing or a required command/flag is absent.
2. Resolve AWS credentials and run STS `GetCallerIdentity`. **Echo the account id, caller ARN, and resolved region back to the user and ask them to confirm or correct** before proceeding. Never assume the default profile is the right one.
3. Confirm the **source agent's region** here (if the user supplied one, confirm rather than re-ask). The harness must deploy there — but the CLI defaults elsewhere, so you set it explicitly before the first deploy (Phase 6; see deploy.md's region trap).

Completion criterion: the user has confirmed `(account, region)` and the CLI passed its checks.

### Phase 1 — Identify the source agent

Resolve a concrete `(agentId, agentVersion, aliasId)` from whatever the user gave (id, name, ARN, or nothing). If the user gave a name fragment, gave nothing, or **asks to see what's available**, list the agents in the confirmed region (`bedrock-agent:ListAgents`) and present them for the user to pick from. Default to the **production alias's** numbered version, not DRAFT — unless the user has only DRAFT or asks for it explicitly. See [`references/discovery.md`](references/discovery.md) for resolution rules and the DRAFT-only case.

Completion criterion: the user has confirmed the exact `(account, region, agentId, agentVersion, aliasId)` tuple.

### Phase 2 — Discovery

Snapshot the agent into one manifest (`./out/source-agent.json`) — via the bundled `scripts/fetch_bedrock_agent.py` (needs python3 + boto3), or the `aws bedrock-agent` fallback when boto3 is absent. Both paths and the manifest shape are in [`references/discovery.md`](references/discovery.md).

Read the manifest and present a **concise, human-readable inventory** — a scannable list or small table (model; action groups by name + type; KBs + type; guardrail; memory; collaboration), not a prose paragraph.

Completion criterion: manifest written and inventory presented.

### Phase 3 — Eligibility gate

Check the manifest against the eligibility rubric in [`references/eligibility.md`](references/eligibility.md). Any **hard-stop** condition (multimodal input, multi-agent collaboration, unreachable KB, custom orchestration) ends the migration here.

If a gate fails: **stop**, tell the user exactly which condition failed and why, and suggest manual alternatives.

Completion criterion: every hard-stop condition checked and explicitly cleared, or the migration stopped with a reported reason.

### Phase 4 — Migration assessment (gate)

The agent is eligible, but not every component migrates with full fidelity. Before planning, show the user a **per-component ledger** classifying each discovered component three ways:

- **clean** — behavior preserved. Most components land here: action groups via shim, *any* KB (MANAGED via connector, others via KB shim — both reproduce retrieval), CodeInterpreter built-in, model + inference params, and managed memory (the standard harness memory, not a downgrade).
- **degraded** — a genuine fidelity change the user should weigh, e.g. a non-DEFAULT orchestration/pre-processing prompt whose business logic can't be cleanly folded into the system prompt, or any behavior the migration can only approximate.
- **cannot** — no harness equivalent, capability is lost: Return-of-Control action groups, and the **guardrail** (the harness has no guardrail field; `--additional-params` is `lite_llm`-only — see [`references/mapping.md`](references/mapping.md)). Both must be surfaced to the user.

Use the mapping in [`references/mapping.md`](references/mapping.md) to classify. Present the ledger and **pause for explicit acknowledgement** — the user must accept the *degraded* and *cannot* items before planning. Nothing irreversible has happened yet; this is informed consent on fidelity loss.

Completion criterion: user has seen the full ledger and acknowledged the degraded/cannot items.

### Phase 5 — Plan

Map each acknowledged component to its harness target using [`references/mapping.md`](references/mapping.md). Produce a written **migration plan** and **pause for approval** (gate). Surface costs and that the source agent is never modified.

Completion criterion: user approved the written plan.

### Phase 6 — Implement & deploy

Drive the CLI per the approved plan, following [`references/deploy.md`](references/deploy.md) end to end: scaffold with `agentcore create` (the command is `create` — **never `init` or `--import`**), deploy the shim Lambdas, `add gateway`/`gateway-target`/`harness`, then the **two-phase deploy**. Set the deploy region before the first deploy (deploy.md's region trap). Generate shim code and tool schemas by **adapting** the templates in `assets/templates/` per [`references/mapping.md`](references/mapping.md). If deploy fails, surface the error — **fail loudly**, never silently work around it.

Completion criterion: `agentcore deploy` reports success. Verification is deploy-success-only; deeper parity is out of scope.

## What this skill does not do

- Modify or delete the source Bedrock Agent.
- Migrate KB vector stores / data sources — the new agent calls into the existing KB.
- Migrate conversation history or end-user authentication.
- Bypass the AgentCore CLI for infrastructure the CLI owns.

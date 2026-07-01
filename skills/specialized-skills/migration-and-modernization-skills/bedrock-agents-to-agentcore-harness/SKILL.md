---
name: bedrock-agents-to-agentcore-harness
description: Migrate a Bedrock Agent to an AgentCore Harness using the AgentCore CLI. Use when the user wants to migrate, port, move, or convert a Bedrock Agent (by id, name, or ARN) to AgentCore Harness — including its action groups, knowledge bases, guardrails, memory, and prompts.
---

# Bedrock Agents → AgentCore Harness

Migrate a Bedrock Agent into an AgentCore **Harness** (the managed agent loop) using the **AgentCore CLI**. The skill drives the CLI to scaffold and deploy; it never hand-rolls boto3 infrastructure the CLI owns.

Two ideas run through every phase:

- **gate** — a hard checkpoint the migration must pass before continuing. Some gates are eligibility (a source feature with no validated harness path); others are user approval (account, region, cost). A failed gate stops the migration and is reported, never worked around silently.
- **mirror** — preserve user-visible *behavior*, not Bedrock-Agents *structure*. Shed the scaffolding the harness now owns (orchestration-loop mechanics, collaboration plumbing), but keep the *intent* inside it — a non-DEFAULT orchestration prompt can encode real tool-routing or input-classification logic worth folding into the system prompt (see [`references/mapping.md`](references/mapping.md)). Keep what the agent does; drop only what the harness does for you.

## Capture the request first

The triggering request *is* the input. Before Phase 0, extract whatever the user already gave — an agent **id/name/ARN**, a **region**, a **profile/account** — and carry it forward. The phases then **confirm** these hints, never re-ask what was supplied: if the user said "migrate agent X in us-east-1," Phase 0 confirms that region rather than asking for one. Identity resolution (Phase 1) runs *after* preflight, because listing agents to disambiguate a name or empty input needs a confirmed account + region first.

## Phases

Run in order. Finish each phase, summarize what happened, then continue. Do not interleave.

### Phase 0 — Preflight (environment gates)

Establish the migration runs against the account, region, and CLI the user intends — before touching the source agent.

1. Confirm the **CLI** is present and current per [`references/cli.md`](references/cli.md). Stop if it is missing or a required command/flag is absent.
2. Resolve AWS credentials and run STS `GetCallerIdentity`. **Echo the account id, caller ARN, and resolved region back to the user and ask them to confirm or correct** before proceeding. Never assume the default profile is the right one.
3. The migration deploys into the *source agent's* region (mirror). Confirm that region here; if the user supplied one in the request, confirm it rather than re-asking.

Completion criterion: the user has confirmed `(account, region)` and the CLI passed its checks.

### Phase 1 — Identify the source agent

Resolve a concrete `(agentId, agentVersion, aliasId)` from whatever the user gave (id, name, ARN, or nothing). If the user gave a name fragment, gave nothing, or **asks to see what's available**, list the agents in the confirmed region (`bedrock-agent:ListAgents`) and present them for the user to pick from. Default to the **production alias's** numbered version, not DRAFT — unless the user has only DRAFT or asks for it explicitly. See [`references/discovery.md`](references/discovery.md) for resolution rules and the DRAFT-only case.

Completion criterion: the user has confirmed the exact `(account, region, agentId, agentVersion, aliasId)` tuple.

### Phase 2 — Discovery

Run the bundled fetcher to snapshot the agent into one manifest:

```bash
python scripts/fetch_bedrock_agent.py \
  --agent-id <id> --agent-version <resolved> --region <region> \
  --inline-s3-schemas --out ./out/source-agent.json
```

Read the manifest and present a **concise, human-readable inventory** — a scannable list or small table (model; action groups by name + type; KBs + type; guardrail; memory; collaboration), not a prose paragraph. Edge cases: [`references/discovery.md`](references/discovery.md).

Completion criterion: manifest written and inventory presented.

### Phase 3 — Eligibility gate

Check the manifest against the eligibility rubric in [`references/eligibility.md`](references/eligibility.md). Any **hard-stop** condition (multimodal input, multi-agent collaboration, unreachable KB, custom orchestration) ends the migration here.

If a gate fails: **stop**, tell the user exactly which condition failed and why, and suggest manual alternatives. Do not migrate part of an ineligible agent.

Completion criterion: every hard-stop condition checked and explicitly cleared, or the migration stopped with a reported reason.

### Phase 4 — Migration assessment (gate)

The agent is eligible, but not every component migrates with full fidelity. Before planning, show the user a **per-component ledger** classifying each discovered component three ways:

- **clean** — behavior preserved. Most components land here: action groups via shim, *any* KB (MANAGED via connector, others via KB shim — both reproduce retrieval), CodeInterpreter built-in, model + inference params, the guardrail (via the model config's `additionalParams` — see [`references/mapping.md`](references/mapping.md)), and managed memory (the standard harness memory, not a downgrade).
- **degraded** — a genuine fidelity change the user should weigh, e.g. a non-DEFAULT orchestration/pre-processing prompt whose business logic can't be cleanly folded into the system prompt, or any behavior the migration can only approximate.
- **cannot** — no harness equivalent, capability is lost (e.g. Return-of-Control action groups).

Use the mapping in [`references/mapping.md`](references/mapping.md) to classify. Present the ledger and **pause for explicit acknowledgement** — the user must accept the *degraded* and *cannot* items before planning. Nothing irreversible has happened yet; this is informed consent on fidelity loss.

Completion criterion: user has seen the full ledger and acknowledged the degraded/cannot items.

### Phase 5 — Plan

Map each acknowledged component to its harness target using [`references/mapping.md`](references/mapping.md). Produce a written **migration plan** and **pause for approval** (gate). Surface costs and that the source agent is never modified.

Completion criterion: user approved the written plan.

### Phase 6 — Implement & deploy

Drive the CLI per the approved plan, following the **two-phase deploy** sequence in [`references/deploy.md`](references/deploy.md): scaffold, add gateway + targets + harness + shims, deploy, attach the gateway tool to the harness, deploy again. Generate shim Lambda code and tool schemas by **adapting** the templates in `assets/templates/`; render every `{{TOKEN}}`, delete optional blocks, and verify no markers remain before deploying. Deploy into the **source agent's region** (mirror) — **but note `agentcore create` silently defaults the deploy target to us-east-1; set the region in `aws-targets.json` right after scaffold (see [`references/deploy.md`](references/deploy.md)), or the shims can't reach the source's by-ARN Lambdas/KB.** If deploy fails, surface the error — **fail loudly**, never silently work around it.

Completion criterion: `agentcore deploy` reports success. Verification is deploy-success-only; deeper parity is out of scope.

## What this skill does not do

- Modify or delete the source Bedrock Agent.
- Migrate KB vector stores / data sources — the new agent calls into the existing KB.
- Migrate conversation history or end-user authentication.
- Bypass the AgentCore CLI for infrastructure the CLI owns.

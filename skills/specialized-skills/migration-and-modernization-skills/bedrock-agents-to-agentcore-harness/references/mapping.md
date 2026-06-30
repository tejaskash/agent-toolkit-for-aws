# Component mapping

Map each source component to its harness target. **Mirror** behavior, not Bedrock structure. Adapt the templates in `assets/templates/`: substitute every `{{TOKEN}}`, and delete every marker block — `# <<< RENDER … # <<< /RENDER` (token docs) and `# <<< OPTIONAL: … # <<< /OPTIONAL` (features that don't apply). After rendering, no `{{`, `}}`, or `<<<` may remain — that grep is the verification gate, and it must come back clean.

## Action groups → Gateway targets

A Bedrock action-group Lambda is invoked with the Bedrock event envelope (`messageVersion`, `function`/`apiPath`, `parameters`, `requestBody`) and returns a wrapped response. AgentCore Gateway invokes a Lambda target with the tool arguments flat in `event` and the tool name in `context.client_context.custom['bedrockAgentCoreToolName']` (`"<target>___<tool>"`), expecting plain JSON. A Lambda written only for the Bedrock shape will not work behind Gateway unchanged.

**Default: proxy-by-ARN.** Create a *new* shim Lambda ([`lambda_shim.py.tmpl`](../assets/templates/lambda_shim.py.tmpl)) that translates the Gateway event into the Bedrock event, invokes the original Lambda by ARN, and unwraps its response. This leaves the original Lambda **untouched**, so the source agent keeps working — editing the original in place can change its response shape and break the source agent. The original's ARN is passed to the shim as an env var.

Each action group becomes one Gateway **`lambda`** target (CLI-managed code — the CLI builds and deploys the shim from its source path during `agentcore deploy`; see [`deploy.md`](deploy.md)) with a tool-schema file holding **one entry per function/operation** in that action group.

### Tool schemas — mirror exactly
Reproduce the source action group's schema faithfully into the tool-schema file ([`tool_schema.json.tmpl`](../assets/templates/tool_schema.json.tmpl)): same tool names, parameter names, types, and descriptions. Do **not** rewrite or "improve" them — fidelity to the source agent's behavior is the goal, and a renamed tool or tightened type changes how the model selects it.

## Knowledge base → connector or KB shim (by type)

Fetch the KB and read `knowledgeBaseConfiguration.type`. Routing is a single binary so the skill never has to track an evolving list of KB types:

- **`MANAGED`** → native Gateway connector: `add gateway-target --type connector --connector bedrock-knowledge-bases --knowledge-base-id <id>`.
- **anything other than `MANAGED`** → **KB shim** ([`kb_shim.py.tmpl`](../assets/templates/kb_shim.py.tmpl)): a `lambda` Gateway target that calls `bedrock-agent-runtime:Retrieve` and returns MCP-shaped passages, preserving any non-default retrieval config (reranker, metadata filter, hybrid override, top-k) rendered from the manifest's KB association.

Both paths reproduce the source agent's retrieval, so the choice is wiring, not fidelity — neither is a loss. (An *unreachable* KB is a hard-stop — see [`eligibility.md`](eligibility.md). Type is never the blocker.)

### Return-of-Control action groups — cannot migrate
A `customControl: RETURN_CONTROL` action group has no Lambda; the original application handled execution outside Bedrock. There is no automatic harness equivalent. **Do not silently drop it.** Classify it **cannot** in the migration assessment and confirm with the user before continuing — the migrated agent will lack that capability unless the user supplies a backend for it.

## Built-in action groups

- **`AMAZON.UserInput`** → no tool. Add a clarification instruction to the system prompt; the harness asks clarifying questions naturally.
- **`AMAZON.CodeInterpreter`** → the built-in tool: `agentcore add tool --type agentcore_code_interpreter`.

## Memory → managed memory (default)

Use the harness's **managed memory**, which is on by default — the harness auto-provisions an AgentCore Memory instance (semantic + summarization strategies, 30-day expiry) and loads/saves session history automatically. Do not pass `--no-harness-memory`. This covers the common source case (`SESSION_SUMMARY`) via the built-in `SUMMARIZATION` strategy; customize strategies via `UpdateHarness` only if the source clearly needs more.

**Leave truncation at its default (`sliding_window`).** Do not set truncation to `summarization` — it runs mid-conversation and errors on short sessions (`Cannot summarize: insufficient messages`). The default is already safe.

Reference: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-memory.html

## Guardrail → model config
Wire the source `guardrailConfiguration` (`guardrailIdentifier` + the **numeric** version, never DRAFT) onto the harness model config.

## Prompt → system prompt
Fold the agent instruction into the harness `--system-prompt`. If `AMAZON.UserInput` was present, include the clarification instruction.

Prompt overrides in **DEFAULT** mode carry nothing custom — skip them. But a **non-DEFAULT override** (especially `ORCHESTRATION` or `PRE_PROCESSING`) may hold real business logic — routing rules like "use tool X for billing questions, tool Y for refunds," or input-classification the app relied on. Don't discard that as boilerplate. Read each non-DEFAULT override, separate the Bedrock-Agents scaffolding (the orchestration loop mechanics the harness now owns) from the business intent, and fold the intent into the system prompt — a modern model handles tool-routing guidance cleanly in the prompt. When unsure whether an override is boilerplate or load-bearing, surface it to the user rather than dropping it.

## Model → mirror source
Default `--model-id` to the source agent's `foundationModel`. If the CLI's harness default differs, set it explicitly to match the source (parity).

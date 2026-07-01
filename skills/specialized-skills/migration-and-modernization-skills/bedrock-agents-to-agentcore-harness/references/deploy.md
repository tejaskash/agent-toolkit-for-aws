# Deploy

Deploy into the **source agent's region** (mirror). If any step fails, surface the error and stop — **fail loudly**, never silently work around a failure.

## CRITICAL: set the deploy-target region right after scaffold (region trap)
`agentcore create` with no resolved region **silently writes `region: us-east-1`** into `agentcore/aws-targets.json` (the CLI default is `region ?? "us-east-1"`). For a migration this is a double trap: (a) us-east-1 may have the stale Harness CFN type that rolls back every deploy, and (b) the shims invoke the **source** Lambdas / KB **by ARN**, so a harness in the wrong region can't reach them.

Immediately after `agentcore create`, before any `add`/`deploy`: **edit `agentcore/aws-targets.json` so the deploy-target region is the source agent's region.** Verify it matches before the first deploy.

If a deploy already landed in the wrong region: `aws cloudformation delete-stack` the wrong-region stack, reset `agentcore/.cli/deployed-state.json` to `{"targets":{}}`, fix `aws-targets.json`, then redeploy.

## Two-phase deploy

A gateway tool can only attach to the harness *after* the gateway and its targets exist in deployed state (`agentcore add tool --type agentcore_gateway` reads the gateway's deployed tool list — it reports "No deployed targets found" against a not-yet-deployed gateway). So the harness migration deploys twice:

1. **Scaffold** the project: `agentcore create` (plain). **Do not use `agentcore create --import` / `--type import`** — that path imports a Bedrock Agent into a *code* project, not a Harness, and is not the recommended migration route. This skill builds a Harness explicitly via `add harness` + `add gateway`/`add tool`; the source agent's config comes from the discovery manifest, not from an `--import`.
2. **Add infrastructure** that doesn't depend on a deployed gateway:
   - **Deploy the shim Lambdas first** (see "How shims are deployed" below), then `agentcore add gateway` and one `agentcore add gateway-target --type lambda-function-arn --lambda-arn <shim-arn> --tool-schema-file <schema>` per migrated action group / KB shim.
   - `agentcore add harness` (`--model-id` = source `foundationModel`; inference params via `--temperature`/`--top-p`/`--model-max-tokens`; system prompt folded in; managed memory left on by default). The source **guardrail** rides in the model config's `additionalParams` (Converse `guardrailConfig`) — see [`mapping.md`](mapping.md).
   - self-contained harness tools: `agentcore add tool --type agentcore_code_interpreter` (if the source had CodeInterpreter).
3. **First deploy:** `agentcore deploy` — creates the gateway, targets, harness, and memory.
4. **Attach the gateway tool** now that the gateway is deployed: `agentcore add tool --harness <name> --type agentcore_gateway --gateway-arn <deployed-gateway-arn> --outbound-auth awsIam`. Use the deployed gateway **ARN** (from `agentcore status` / stack outputs), not just the project name.
5. **Second deploy:** `agentcore deploy` — applies the harness's new gateway tool.

## How shims are deployed (verified on CLI 0.21.1)
There is **no non-interactive `--type lambda` gateway-target that builds shim code for you.** `agentcore add gateway-target --type` accepts only: `mcp-server, api-gateway, open-api-schema, smithy-model, lambda-function-arn, http-runtime, connector`. `--type lambda` is rejected (`Invalid type: lambda`). So do **not** rely on the CLI to build a shim Lambda from a source path via a `lambda` target.

**The reliable path: deploy the shim Lambda yourself, then wire it by ARN.**
1. Render the shim template (`assets/templates/{kb_shim,lambda_shim}.py.tmpl`), zip it, and create the Lambda with boto3 / `aws lambda create-function` (give it a role with the perms it needs: `bedrock-agent-runtime:Retrieve` for the KB shim; `lambda:InvokeFunction` on the original for the AG proxy shim). Pass config via env vars (`KB_ID`, `ORIGINAL_LAMBDA_ARN`, `SCHEMA_STYLE`, `OP_ROUTES`).
2. Wire it as a gateway target: `agentcore add gateway-target --type lambda-function-arn --lambda-arn <shim-arn> --tool-schema-file <schema> --gateway <gw>`.
3. Grant the gateway permission to invoke the shim if required.

(The `--host Lambda --language Python` flags belong to the **`mcp-server`** target type — the CLI builds and hosts your code as a Lambda-backed MCP *server*, a different shape than a plain tool Lambda. If you use that path instead, its code lives in a project-root `tools/<name>/` dir with a per-tool setuptools `pyproject.toml` — tool code under `agentcore/` is NOT picked up. For straightforward shims, prefer the deploy-it-yourself + `lambda-function-arn` path above.)

## One action group = one target, with all its functions
A Bedrock action group can expose several functions/operations (up to three), each with its own schema. The tool-schema file is a **`ToolDefinition[]` array**, so a single Lambda target carries every function in that action group — one array entry per function/operationId. Do not split an action group into multiple targets. [`tool_schema.json.tmpl`](../assets/templates/tool_schema.json.tmpl) is already an array; add one entry per function, mirroring the source schema exactly ([`mapping.md`](mapping.md)).

## Verification
The migration is done when `agentcore deploy` reports success. Before treating it as complete, **prompt the user** to confirm the deploy succeeded and they're satisfied — surface the deployed harness/gateway ARNs from `agentcore status`. Deeper parity (invoking the harness, comparing against the source) is out of scope unless the user asks.

## Templates → CLI inputs
- KB shim / AG shim Lambda code: adapt the `assets/templates/*.tmpl` files, render every `{{TOKEN}}`, delete optional blocks, and verify no markers remain before deploying the shim Lambda and wiring it via `lambda-function-arn` (see "How shims are deployed").
- Tool-schema files: one array per target, one entry per source function/operation, mirroring the source schema exactly ([`mapping.md`](mapping.md)).

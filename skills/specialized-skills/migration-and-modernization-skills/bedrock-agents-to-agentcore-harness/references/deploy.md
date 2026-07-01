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
   - `agentcore add gateway`, then one `agentcore add gateway-target` per migrated action group / KB shim.
   - `agentcore add harness` (`--model-id` = source `foundationModel`; inference params via `--temperature`/`--top-p`/`--model-max-tokens`; system prompt folded in; managed memory left on by default). The source **guardrail** rides in the model config's `additionalParams` (Converse `guardrailConfig`) — see [`mapping.md`](mapping.md).
   - self-contained harness tools: `agentcore add tool --type agentcore_code_interpreter` (if the source had CodeInterpreter).
3. **First deploy:** `agentcore deploy` — builds the shim Lambdas, creates the gateway, targets, harness, and memory.
4. **Attach the gateway tool** now that the gateway is deployed: `agentcore add tool --harness <name> --type agentcore_gateway --gateway-arn <deployed-gateway-arn> --outbound-auth awsIam`. Use the deployed gateway **ARN** (from `agentcore status` / stack outputs), not just the project name.
5. **Second deploy:** `agentcore deploy` — applies the harness's new gateway tool.

## Shim Lambdas are CLI-managed (`lambda` target type), not hand-deployed
The gateway-target type for a shim is **`lambda`** (CLI-managed code), not `lambda-function-arn` (a pre-existing ARN). With a `lambda` target the CLI **builds and deploys the Lambda from a source path as part of `agentcore deploy`**, owning its runtime, handler, IAM policy, and lifecycle — no separate `aws lambda create-function` or hand-rolled execution role. Point each shim target at its rendered template code:

```
agentcore add gateway-target --gateway <gw> --name <tool> --type lambda \
  --host Lambda --language Python \
  # CLI prompts for / takes the code path + handler; point it at the rendered
  # kb_shim.py or lambda_shim.py (see assets/templates/), and the tool-schema.
```

(`lambda-function-arn` is only for wiring an *existing* Lambda by ARN — not used for shims the migration creates. The AG proxy-by-ARN shim is still a `lambda` target; it receives the *original* Lambda's ARN as an env var to invoke at runtime.)

### Shim code layout (CLI-managed `lambda` targets)
A CLI-managed `lambda` target resolves its code path **relative to project root** and requires a **`pyproject.toml` per tool directory** (setuptools). So:
- Put each shim in its own directory under **`tools/`** at the project root — **not** `agentcore/tools/` (the CLI does not pick up code under `agentcore/`).
- Add a minimal setuptools `pyproject.toml` in each shim's directory.

```
<project>/
  tools/
    kb_shim/        pyproject.toml  kb_shim.py
    booking_shim/   pyproject.toml  lambda_shim.py
  agentcore/        # CLI config + cdk (do NOT put tool code here)
```

## One action group = one target, with all its functions
A Bedrock action group can expose several functions/operations (up to three), each with its own schema. The tool-schema file is a **`ToolDefinition[]` array**, so a single Lambda target carries every function in that action group — one array entry per function/operationId. Do not split an action group into multiple targets. [`tool_schema.json.tmpl`](../assets/templates/tool_schema.json.tmpl) is already an array; add one entry per function, mirroring the source schema exactly ([`mapping.md`](mapping.md)).

## Verification
The migration is done when `agentcore deploy` reports success. Before treating it as complete, **prompt the user** to confirm the deploy succeeded and they're satisfied — surface the deployed harness/gateway ARNs from `agentcore status`. Deeper parity (invoking the harness, comparing against the source) is out of scope unless the user asks.

## Templates → CLI inputs
- KB shim / AG shim Lambda code: adapt the `assets/templates/*.tmpl` files, render every `{{TOKEN}}`, delete optional blocks, and verify no markers remain before pointing a `lambda` target at the code.
- Tool-schema files: one array per target, one entry per source function/operation, mirroring the source schema exactly ([`mapping.md`](mapping.md)).

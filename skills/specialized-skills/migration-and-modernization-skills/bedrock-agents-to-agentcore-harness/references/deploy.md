# Deploy

Deploy into the **source agent's region** (mirror). If any step fails, surface the error and stop — **fail loudly**, never silently work around a failure.

## CRITICAL: set the deploy-target region right after scaffold (region trap)
`agentcore create` with no resolved region **silently writes `region: us-east-1`** into `agentcore/aws-targets.json` (the CLI default is `region ?? "us-east-1"`). For a migration this is a double trap: (a) us-east-1 may have the stale Harness CFN type that rolls back every deploy, and (b) the shims invoke the **source** Lambdas / KB **by ARN**, so a harness in the wrong region can't reach them.

Immediately after `agentcore create`, before any `add`/`deploy`: **edit `agentcore/aws-targets.json` so the deploy-target region is the source agent's region.** Verify it matches before the first deploy.

If a deploy already landed in the wrong region: `aws cloudformation delete-stack` the wrong-region stack, reset `agentcore/.cli/deployed-state.json` to `{"targets":{}}`, fix `aws-targets.json`, then redeploy.

Names: harness and project names match `^[a-zA-Z][a-zA-Z0-9_]{0,39}$` — underscores, **no hyphens** (`support_harness`, not `support-harness`). Work in **one** project directory for the whole migration: `cd` into it after `create` and run every later command from there (`add`/`deploy` resolve the project from the cwd).

## Two-phase deploy

A gateway tool can only attach once the gateway and its targets are deployed (`add tool --type agentcore_gateway` reads the gateway's *deployed* tool list). So deploy twice:

1. **Scaffold:** `agentcore create` (plain — **not `--import`**, which targets a code project, not a Harness).
2. **Add** infra that doesn't need a deployed gateway:
   - `agentcore add gateway`, then one **`lambda` code target** per action group / KB shim (see "How shims are deployed").
   - `agentcore add harness` with `--model-id` (= source `foundationModel`), `--temperature`/`--top-p`/`--model-max-tokens` (only one of temperature/top-p when the model rejects both), and `--system-prompt` (folded instruction + non-DEFAULT override intent, per [`mapping.md`](mapping.md)). Managed memory stays on by default. The source **guardrail cannot be carried over** — see [`mapping.md`](mapping.md).
   - if the source had CodeInterpreter: `agentcore add tool --harness <name> --type agentcore_code_interpreter --name <tool-name>`.
3. **First deploy:** `agentcore deploy` — creates gateway, targets, harness, memory.
4. **Attach the gateway tool:** `agentcore add tool --harness <name> --type agentcore_gateway --name <tool-name> --gateway <gateway-project-name> --outbound-auth awsIam`. `--gateway <project-name>` resolves the deployed ARN automatically.
5. **Second deploy:** `agentcore deploy` — applies the gateway tool.

`add tool` **requires** `--harness` and `--name`; attaching to an existing harness is its normal job. A command that fails on a missing flag is fixed by adding the flag — never by hand-editing config (see "Never hand-author config").

## How shims are deployed — the CLI owns packaging
Each shim is a **`lambda` code target** on the gateway: the CLI builds and deploys the Lambda (and its IAM) from your source at `agentcore deploy`. **Never** zip, `aws lambda create-function`, or boto3-deploy it yourself.

Its `agentcore.json` gateway-target entry has `targetType: "lambda"` with:
- `toolDefinitions`: the tool-schema array (one entry per function/operation).
- `compute`: `{ host: "Lambda", implementation: { language: "Python", path: "tools/<shim>", handler: "handler.lambda_handler" }, pythonVersion, timeout, iamPolicy }`. The `iamPolicy` grants what the shim needs — `bedrock:Retrieve` on the KB for the KB shim; `lambda:InvokeFunction` on the original for the AG proxy shim.

Place the **rendered shim** (`assets/templates/{kb_shim,lambda_shim}.py.tmpl`, tokens substituted) at `tools/<shim>/handler.py` with a `pyproject.toml` beside it, and set the shim's config as env vars (`KB_ID`, `ORIGINAL_LAMBDA_ARN`, `SCHEMA_STYLE`, `OP_ROUTES`). Then `agentcore deploy`.

(`--type lambda-function-arn` is a *different* target — it wires an already-existing Lambda by ARN. Use the `lambda` code target above for shims this skill generates.)

## Never hand-author config — drive the CLI
Build and modify the harness and its tools **only** through `agentcore` commands — never `cat >`, hand-write, or Python-patch `app/<name>/harness.json` or `agentcore/agentcore.json`. The schema shifts between releases, so hand-authored config drifts and fails validation. When a command fails, fix its flags; don't route around it by editing JSON.

## One action group = one target, with all its functions
A Bedrock action group can expose several functions/operations (up to three), each with its own schema. The tool-schema file is a **`ToolDefinition[]` array**, so a single Lambda target carries every function in that action group — one array entry per function/operationId. Do not split an action group into multiple targets. [`tool_schema.json.tmpl`](../assets/templates/tool_schema.json.tmpl) is already an array; add one entry per function, mirroring the source schema exactly ([`mapping.md`](mapping.md)).

## Verification
The migration is done when `agentcore deploy` reports success. Before treating it as complete, **prompt the user** to confirm the deploy succeeded and they're satisfied — surface the deployed harness/gateway ARNs from `agentcore status`. Deeper parity (invoking the harness, comparing against the source) is out of scope unless the user asks.

## Rendering templates into CLI inputs
- KB shim / AG shim Lambda code: adapt the `assets/templates/*.tmpl` files (rendering rules in [`mapping.md`](mapping.md)) and place the rendered handler at `tools/<shim>/handler.py` — see "How shims are deployed" above.
- Tool-schema files: one array per target, one entry per source function/operation, mirroring the source schema exactly ([`mapping.md`](mapping.md)).

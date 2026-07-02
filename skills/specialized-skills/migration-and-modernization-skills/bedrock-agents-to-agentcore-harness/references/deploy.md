# Deploy

Deploy into the **source agent's region** (mirror). If any step fails, surface the error and stop — **fail loudly**, never silently work around a failure.

## CRITICAL: set the deploy-target region right after scaffold (region trap)
`agentcore create` with no resolved region **silently writes `region: us-east-1`** into `agentcore/aws-targets.json` (the CLI default is `region ?? "us-east-1"`). For a migration this is a double trap: (a) us-east-1 may have the stale Harness CFN type that rolls back every deploy, and (b) the shims invoke the **source** Lambdas / KB **by ARN**, so a harness in the wrong region can't reach them.

Immediately after `agentcore create`, before any `add`/`deploy`: **edit `agentcore/aws-targets.json` so the deploy-target region is the source agent's region.** Verify it matches before the first deploy.

If a deploy already landed in the wrong region: `aws cloudformation delete-stack` the wrong-region stack, reset `agentcore/.cli/deployed-state.json` to `{"targets":{}}`, fix `aws-targets.json`, then redeploy.

## Naming constraints
Harness names (and project names) must match `^[a-zA-Z][a-zA-Z0-9_]{0,39}$` — **no hyphens**, underscores only. A name like `support-harness` is rejected; use `support_harness`. Project names must be PascalCase alphanumeric (no hyphens either).

## Two-phase deploy

A gateway tool can only attach to the harness *after* the gateway and its targets exist in deployed state (`agentcore add tool --type agentcore_gateway` — full flags in step 4 below — reads the gateway's deployed tool list, and reports "No deployed targets found" against a not-yet-deployed gateway). So the harness migration deploys twice:

1. **Scaffold** the project: `agentcore create` (plain). **Do not use `agentcore create --import` / `--type import`** — that path imports a Bedrock Agent into a *code* project, not a Harness, and is not the recommended migration route. This skill builds a Harness explicitly via `add harness` + `add gateway`/`add tool`; the source agent's config comes from the discovery manifest, not from an `--import`.

   **Work in ONE project directory for the whole migration.** After `agentcore create <name>`, `cd` into that project dir and run **every** subsequent `add …`/`deploy`/`status` from there. Do **not** scaffold a second project, `cd` elsewhere, or create the project in multiple locations (e.g. both the working dir and `/tmp`). `add harness`/`add tool` resolve the project and its harness from the current directory — running them from a different dir yields "No agentcore project found" or "Harness '<name>' not found" against a project that doesn't have your harness. If a command fails, fix the command **in place**; do not relocate the project.
2. **Add infrastructure** that doesn't depend on a deployed gateway:
   - `agentcore add gateway`, then one **`lambda` code target** per migrated action group / KB shim (see "How shims are deployed" below).
   - `agentcore add harness` — build the harness **with the CLI, in one command**; pass every field as a flag. Do **not** hand-write or patch `app/<name>/harness.json` (see "Never hand-author config" below). The flags:
     - `--model-id` = source `foundationModel`; inference params via `--temperature`/`--top-p`/`--model-max-tokens` (set only one of temperature/top-p when the model rejects both — e.g. Claude Sonnet 4.5).
     - `--system-prompt` = the folded agent instruction (+ non-DEFAULT override intent, per [`mapping.md`](mapping.md)).
     - The source **guardrail canNOT be migrated** onto a bedrock harness — do **not** pass `--additional-params` (it is `lite_llm`-only and a bedrock harness rejects it, failing the whole `add harness`). Classify the guardrail **"cannot migrate / degraded"** and surface it to the user; do not fake it with a system-prompt mention. See [`mapping.md`](mapping.md), "Guardrail — cannot migrate".
     - Do **not** pass any other `--additional-params` on a bedrock harness for the same reason.
     - managed memory left on by default.
   - self-contained harness tools: `agentcore add tool --harness <harness-name> --type agentcore_code_interpreter --name <tool-name>` (if the source had CodeInterpreter). Like all `add tool` calls, `--harness` and `--name` are **required**.
3. **First deploy:** `agentcore deploy` — creates the gateway, targets, harness, and memory.
4. **Attach the gateway tool** now that the gateway is deployed: `agentcore add tool --harness <harness-name> --type agentcore_gateway --name <tool-name> --gateway <gateway-project-name> --outbound-auth awsIam`. `agentcore add tool` **requires** `--name` (the tool name) — omitting it makes the command fail; that failure is **not** a signal that the CLI can't help, and it is **never** a reason to hand-edit `harness.json`. `--gateway <project-name>` resolves the deployed ARN from state automatically (use `--gateway-arn <arn>` only if you must pass an ARN explicitly). `add tool` attaches to an **already-existing** harness — that is exactly its job, so "harness already exists" is expected, not an error to route around.
5. **Second deploy:** `agentcore deploy` — applies the harness's new gateway tool.

## How shims are deployed — the CLI owns packaging
Each shim is a **`lambda` code target** on the gateway: the CLI builds and deploys the Lambda (and its IAM) from your source at `agentcore deploy`. **Never** zip, `aws lambda create-function`, or boto3-deploy it yourself.

Its `agentcore.json` gateway-target entry has `targetType: "lambda"` with:
- `toolDefinitions`: the tool-schema array (one entry per function/operation).
- `compute`: `{ host: "Lambda", implementation: { language: "Python", path: "tools/<shim>", handler: "handler.lambda_handler" }, pythonVersion, timeout, iamPolicy }`. The `iamPolicy` grants what the shim needs — `bedrock:Retrieve` on the KB for the KB shim; `lambda:InvokeFunction` on the original for the AG proxy shim.

Place the **rendered shim** (`assets/templates/{kb_shim,lambda_shim}.py.tmpl`, tokens substituted) at `tools/<shim>/handler.py` with a `pyproject.toml` beside it, and set the shim's config as env vars (`KB_ID`, `ORIGINAL_LAMBDA_ARN`, `SCHEMA_STYLE`, `OP_ROUTES`). Then `agentcore deploy`.

(`--type lambda-function-arn` is a *different* target — it wires an already-existing Lambda by ARN. Use the `lambda` code target above for shims this skill generates.)

## Never hand-author config — drive the CLI
Build and modify the harness and its tools **only** through `agentcore` commands (`add harness`, `add tool`, `add gateway`, `add gateway-target`). **Never** `cat >`, hand-write, or Python-patch `app/<name>/harness.json` or `agentcore/agentcore.json` — the schema shape shifts between releases, so hand-authored config drifts from what the CLI expects and fails validation (e.g. an `agentcore_gateway` tool needs a specific `agentCoreGateway` config block; guardrail `additionalParams` must go through `add harness --additional-params`). If an `agentcore add …` command fails, the fix is to **correct the command's flags** (a missing `--name`, a wrong `--type`), not to edit the JSON by hand. Hand-editing is how fidelity gets silently lost — a dropped guardrail, a malformed tool — which is exactly what this migration must not do.

## One action group = one target, with all its functions
A Bedrock action group can expose several functions/operations (up to three), each with its own schema. The tool-schema file is a **`ToolDefinition[]` array**, so a single Lambda target carries every function in that action group — one array entry per function/operationId. Do not split an action group into multiple targets. [`tool_schema.json.tmpl`](../assets/templates/tool_schema.json.tmpl) is already an array; add one entry per function, mirroring the source schema exactly ([`mapping.md`](mapping.md)).

## Verification
The migration is done when `agentcore deploy` reports success. Before treating it as complete, **prompt the user** to confirm the deploy succeeded and they're satisfied — surface the deployed harness/gateway ARNs from `agentcore status`. Deeper parity (invoking the harness, comparing against the source) is out of scope unless the user asks.

## Rendering templates into CLI inputs
- KB shim / AG shim Lambda code: adapt the `assets/templates/*.tmpl` files (rendering rules in [`mapping.md`](mapping.md)) and place the rendered handler at `tools/<shim>/handler.py` — see "How shims are deployed" above.
- Tool-schema files: one array per target, one entry per source function/operation, mirroring the source schema exactly ([`mapping.md`](mapping.md)).

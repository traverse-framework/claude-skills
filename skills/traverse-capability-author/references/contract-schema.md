# Contract, workflow, manifest, and request shapes

All shapes below are verified two ways: against the `CapabilityContract` struct in `crates/traverse-contracts/src/lib.rs` (so field names and required-ness are exact, not guessed), and against capabilities built from scratch and run through `traverse-cli agent inspect` to a clean pass. Where the checked-in `scripts/scaffold/new-capability.sh` disagrees with this file, this file is correct and the scaffold script is stale — it predates the current struct and will produce a contract that fails to deserialize.

## Capability contract (`contract.json`)

```json
{
  "kind": "capability_contract",
  "schema_version": "1.0.0",
  "id": "namespace.name",
  "namespace": "namespace",
  "name": "name",
  "version": "1.0.0",
  "lifecycle": "active",
  "owner": { "team": "your-team", "contact": "you@example.com" },
  "summary": "One sentence: what this capability does.",
  "description": "A longer description of the capability's purpose and behavior.",
  "inputs": {
    "schema": {
      "type": "object",
      "required": ["your_field"],
      "properties": { "your_field": { "type": "string" } },
      "additionalProperties": false
    }
  },
  "outputs": {
    "schema": {
      "type": "object",
      "required": ["your_output_field"],
      "properties": { "your_output_field": { "type": "string" } },
      "additionalProperties": false
    }
  },
  "preconditions": [
    { "id": "your-field-provided", "description": "The caller provides a valid your_field." }
  ],
  "postconditions": [
    { "id": "output-produced", "description": "The capability returns your_output_field." }
  ],
  "side_effects": [
    { "kind": "memory_only", "description": "Describe what the capability actually does to state, if anything." }
  ],
  "emits": [],
  "consumes": [],
  "permissions": [
    { "id": "namespace.name" }
  ],
  "execution": {
    "binary_format": "wasm",
    "entrypoint": { "kind": "wasi-command", "command": "run" },
    "preferred_targets": ["local"],
    "constraints": {
      "host_api_access": "none",
      "network_access": "forbidden",
      "filesystem_access": "none"
    }
  },
  "policies": [],
  "dependencies": [],
  "provenance": {
    "source": "greenfield",
    "author": "your-name",
    "created_at": "2026-01-01T00:00:00Z",
    "spec_ref": null,
    "adr_refs": [],
    "exception_refs": []
  },
  "evidence": [],
  "service_type": "stateless",
  "permitted_targets": ["local"],
  "artifact_type": "native"
}
```

**Every field is required unless noted.** `emits`, `consumes`, `permissions`, `policies`, `dependencies`, `evidence` all need to be present as arrays even when empty — the deserializer doesn't default them. `service_type` and `permitted_targets` DO have real defaults if you omit them (`service_type` → `"stateless"`, `permitted_targets` → all six targets) — but don't rely on the `permitted_targets` default. It defaults to declaring `local`, `browser`, `edge`, `cloud`, `worker`, and `device` all permitted, which is a governance-level "allowed to run here eventually" declaration, not a claim that the runtime can execute there today. Set it explicitly to what you actually mean — almost always just `["local"]` for a new capability — and cross-check against `references/platforms.md` before claiming more.

**Enum values, verified against the struct:**
| Field | Valid values |
|---|---|
| `lifecycle` | `draft`, `active`, `deprecated`, `retired`, `archived` — only `active` and `deprecated` are runtime-eligible |
| `service_type` | `stateless` (default), `subscribable` (needs `event_trigger`), `stateful` (can't run in Browser) |
| `execution.binary_format` | `wasm` (only value defined) |
| `execution.entrypoint.kind` | `wasi-command` (only value defined) |
| `execution.preferred_targets[]` / `permitted_targets[]` | `local`, `browser`, `edge`, `cloud`, `worker`, `device` |
| `execution.constraints.host_api_access` | `none`, `exception_required` |
| `execution.constraints.network_access` | `forbidden`, `required` (see the secrets-check note in the main skill for why `required` is honest-but-currently-unusable — the ABI whitelist doesn't grant network imports to any capability yet) |
| `execution.constraints.filesystem_access` | `none`, `sandbox_only` |
| `side_effects[].kind` | `none`, `memory_only`, `event_emission`, `external_call`, `state_change` |
| `provenance.source` | check `ProvenanceSource` enum in the struct if `greenfield` doesn't fit |

## Workflow (`workflow.json`)

A package can't register without at least one approved workflow reference — an empty `workflow_refs: []` in the manifest is rejected outright. The simplest valid workflow is a single node calling your one capability:

```json
{
  "kind": "workflow_definition",
  "schema_version": "1.0.0",
  "id": "namespace.name",
  "name": "name",
  "version": "1.0.0",
  "lifecycle": "active",
  "owner": { "team": "your-team", "contact": "you@example.com" },
  "summary": "Run the namespace.name capability as one governed workflow.",
  "inputs": { "schema": { "...": "same shape as the capability's inputs" } },
  "outputs": { "schema": { "...": "same shape as the capability's outputs" } },
  "nodes": [
    {
      "node_id": "run",
      "capability_id": "namespace.name",
      "capability_version": "1.0.0",
      "input": { "from_workflow_input": ["your_field"] },
      "output": { "to_workflow_state": ["your_output_field"] }
    }
  ],
  "edges": [],
  "start_node": "run",
  "terminal_nodes": ["run"],
  "tags": [],
  "governing_spec": "007-workflow-registry-traversal"
}
```

For composing *several* capabilities into a multi-node workflow (not just wrapping one), see `traverse-workflow-planner`'s reference material — that skill owns workflow composition since it starts from the business use case, not from a single already-decided capability.

## Agent package manifest (`manifest.json`)

This is what `traverse-cli agent inspect`/`agent execute` actually reads. It's a lighter-weight package format than the full application-bundle manifest.

```json
{
  "kind": "agent_package",
  "schema_version": "1.0.0",
  "package_id": "namespace.name-agent",
  "version": "1.0.0",
  "summary": "One sentence.",
  "capability_ref": {
    "id": "namespace.name",
    "version": "1.0.0",
    "contract_path": "relative/path/to/contract.json"
  },
  "workflow_refs": [
    { "workflow_id": "namespace.name", "workflow_version": "1.0.0" }
  ],
  "source": { "path": "./src/agent.rs", "language": "rust", "entry": "run" },
  "binary": {
    "path": "./artifacts/your-agent.wasm",
    "format": "wasm",
    "expected_digest": "fnv1a64:0000000000000000",
    "abi_version": "1.0.0"
  },
  "constraints": {
    "host_api_access": "none",
    "network_access": "forbidden",
    "filesystem_access": "none"
  },
  "model_dependencies": []
}
```

The digest is **FNV-1a-64**, formatted `fnv1a64:<16 lowercase hex chars>` — not SHA-256. Don't hand-compute it: put a placeholder in, run `agent inspect`, and the tool's own mismatch error tells you the real value to paste in.

Note this is a *different* digest scheme from the public registry index's `digest`/`contract_digest` fields, which are SHA-256 over the published files — `manifest.json`'s `binary.expected_digest` is specifically FNV-1a-64 for the local `agent inspect`/`agent execute` path. Don't cross the two.

## Runtime request (`runtime-request.json`)

```json
{
  "kind": "runtime_request",
  "schema_version": "1.0.0",
  "request_id": "namespace-name-test-001",
  "intent": { "capability_id": "namespace.name", "capability_version": "1.0.0" },
  "input": { "your_field": "example value" },
  "lookup": { "scope": "public_only", "allow_ambiguity": false },
  "context": { "requested_target": "local", "caller": "manual-test" },
  "governing_spec": "006-runtime-request-execution"
}
```

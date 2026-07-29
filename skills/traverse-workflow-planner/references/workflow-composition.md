# Composing capabilities into a workflow

## The shape

```json
{
  "kind": "workflow_definition",
  "schema_version": "1.0.0",
  "id": "namespace.name",
  "name": "name",
  "version": "1.0.0",
  "lifecycle": "active",
  "owner": { "team": "your-team", "contact": "you@example.com" },
  "summary": "One sentence describing the end-to-end workflow.",
  "inputs": { "schema": { "...": "the workflow's own top-level input shape" } },
  "outputs": { "schema": { "...": "the workflow's own top-level output shape" } },
  "nodes": [
    {
      "node_id": "capture_objective",
      "capability_id": "expedition.planning.capture-expedition-objective",
      "capability_version": "1.0.0",
      "input": { "from_workflow_input": ["destination", "target_window", "preferences", "notes"] },
      "output": { "to_workflow_state": ["objective"] }
    },
    {
      "node_id": "interpret_intent",
      "capability_id": "expedition.planning.interpret-expedition-intent",
      "capability_version": "1.0.0",
      "input": { "from_workflow_input": ["objective", "planning_intent"] },
      "output": { "to_workflow_state": ["interpreted_intent"] }
    }
  ],
  "edges": [
    { "edge_id": "capture_to_interpret", "from": "capture_objective", "to": "interpret_intent", "trigger": "direct" }
  ],
  "start_node": "capture_objective",
  "terminal_nodes": ["assemble_plan"],
  "tags": [],
  "governing_spec": "007-workflow-registry-traversal"
}
```

This is a real example (`workflows/examples/expedition/plan-expedition/workflow.json` in `traverse-framework/traverse`), condensed — the actual file chains five capabilities: capture → interpret → assess → validate → assemble. It composes cleanly only because each capability's contract is narrow enough to slot into a chain like this — this is the concrete payoff of the atomicity principle `traverse-capability-author` and `traverse-capability-extractor` both apply: a monolithic capability that tries to capture-and-validate-and-assemble in one binary can't be reused piecemeal the way five separate ones can.

## The `from_workflow_input` gotcha

Despite its name, a node's `from_workflow_input` does **not** only pull from the workflow's declared top-level input schema — it pulls from the accumulated workflow *state*, which includes both the original inputs and every prior node's `to_workflow_state` output. In the example above, `interpret_intent` reads `objective` via `from_workflow_input` even though `objective` isn't one of the workflow's own declared top-level inputs — it only exists in state because `capture_objective` wrote it a step earlier.

Practical consequence when composing: when a node needs a field, check whether any upstream node's `to_workflow_state` produces it — that's the actual source of truth, not the workflow's own `inputs.schema`. And when ordering nodes via `edges`, make sure anything a node reads via `from_workflow_input` was either an original input or was written by a node earlier in the chain — the runtime won't reorder nodes to satisfy a dependency you got backwards.

## Registering and inspecting — real, generic

```bash
traverse-cli workflow register path/to/workflow.json --workspace <workspace-id>
traverse-cli workflow list --workspace <workspace-id>
traverse-cli workflow inspect <workflow-id> --workspace <workspace-id>
```

These work for any workflow you throw at them — not demo-scoped, not a stub.

## Actually running a composed workflow — what's real and what isn't

There is **no bare `traverse-cli workflow execute` command**. `workflow register`/`list`/`inspect` are real and generic, but registry-side only. A separate `expedition execute` command exists but is a bespoke command hardcoded to the expedition example domain — it is not a generic workflow runner, and reaching for it on a workflow you've built will not work.

The real, generic traversal engine (`Runtime::execute_workflow` in `crates/traverse-runtime/src/workflows.rs`) does exist and is genuinely contract-enforced end to end — it's just not exposed as its own subcommand. The generic way to reach it through the CLI today is `traverse-cli serve`'s composite-capability trick: register the workflow as a capability whose `implementation_kind` is `"workflow"` (with a `workflow_ref` pointing at the registered workflow), then call it through `POST /v1/capabilities/execute` like any other registered capability. The alternative is constructing a `Runtime<WasmExecutor>` directly via the embedder SDKs and calling `.execute_workflow(...)`.

Either way, **every node's binary still has to individually clear the ABI whitelist** the same way a standalone capability does (see `traverse-capability-author`'s `references/wasm-implementation.md`) — composing capabilities into a workflow doesn't relax that requirement for any of them.

# Troubleshooting — real errors, in the order you'll likely hit them

Every error below was actually produced hands-on — most while building a brand-new capability (`shout-agent`) from scratch through this exact workflow, and entries #8-11 while separately verifying `traverse-cli serve` as a second execution path — not copied from documentation. They're ordered the way they naturally occur, from first compile through final execute.

## 1. `error[E0463]: can't find crate for 'core'`

**When:** running `cargo build --target wasm32-wasip1 --release`, even after `rustup target add wasm32-wasip1` reported success and the `.rlib` files genuinely exist on disk.

**Cause:** the machine has both a Homebrew-installed Rust toolchain and a rustup-managed one. `rustup target add` only installs the target for *rustup's* toolchain. If `/opt/homebrew/bin/cargo`/`rustc` resolve first on `PATH`, the build uses Homebrew's `rustc`, which has no idea rustup installed anything — it just reports the target's core crate as missing. The error message doesn't mention rustup or Homebrew at all, which is what makes this one easy to lose time on.

**Check:**
```bash
which rustc                  # /opt/homebrew/bin/rustc means you're in the trap
rustc --version --verbose    # look for the toolchain name — Homebrew's won't say "rustup"
```

**Fix:** don't hand-construct the rustup toolchain directory path — on Apple Silicon `uname -m` reports `arm64`, but rustup's toolchain directory is named `aarch64-apple-darwin`, so a `$(uname -m)` substitution silently builds a nonexistent path and you're right back to Homebrew's binaries with no error at all (this was verified the hard way: the "obvious" fix looked like it worked, `which rustc` still showed Homebrew's, and the build failed identically). Use `rustup which cargo`/`rustup which rustc` to get the real paths and prepend *their* directory:
```bash
PATH="$(dirname "$(rustup which cargo)"):$PATH" cargo build --target wasm32-wasip1 --release
```

## 2. `error: current package believes it's in a workspace when it's not`

**When:** the capability's `Cargo.toml` lives inside a cloned `traverse-framework/traverse` checkout (e.g. under `examples/`), and that repo has its own root workspace `Cargo.toml`.

**Cause:** Cargo detects the new crate as an unclaimed member of the enclosing repo's workspace.

**Fix:** add an empty `[workspace]` table to the capability's own `Cargo.toml`:
```toml
[workspace]
```
This tells Cargo the crate is its own workspace root, not an orphaned member of the parent one.

## 3. Binary path mismatch on build

**When:** `Cargo.toml` declares `[[bin]] path = "src/main.rs"` but the source file is actually named something else (e.g. `src/agent.rs`, to match the surrounding repo's naming convention).

**Fix:** make the `path` in `[[bin]]` match the real file location — either rename the source file to `src/main.rs` or point `path` at wherever it actually lives. There's no default inference once you've declared an explicit `[[bin]]` table.

## 4. `agent package must declare at least one approved workflow reference`

**When:** running `agent inspect` on a manifest with `"workflow_refs": []`.

**Cause:** a package isn't considered registrable without at least one workflow tying it to a governed entrypoint — this is enforced, not optional even for a quick local test.

**Fix:** write a real `workflow_definition` JSON (see `references/contract-schema.md`) and add a corresponding entry to `workflow_refs` in the manifest: `{"workflow_id": "...", "workflow_version": "..."}`.

## 5. `agent binary digest mismatch ... expected fnv1a64:..., got fnv1a64:<real digest>`

**When:** running `agent inspect` with a placeholder `expected_digest`.

**Cause:** Traverse hashes the compiled `.wasm` binary with FNV-1a-64, not SHA-256 (the stale `scripts/scaffold/new-capability.sh` assumes SHA-256, which is itself part of why that script is untrustworthy).

**Fix:** don't hand-implement FNV-1a. Leave a placeholder digest (`fnv1a64:0000000000000000`), run `agent inspect`, and copy the real digest straight out of the mismatch error into the manifest.

## 6. `matching capabilities were found but none were runnable locally`

**When:** running `agent execute` while the contract's `lifecycle` is still `"draft"`.

**Cause:** `evaluate_candidate()` in `crates/traverse-runtime/src/lib.rs` checks `contract.lifecycle.is_runtime_eligible()`, which is only `true` for `Active` and `Deprecated`. A draft contract is correctly rejected — this isn't a bug, it's the runtime refusing to execute something not yet declared ready.

**Fix:** set `lifecycle` to `"active"` in both the contract and the workflow once you actually intend to run it, even for local iteration.

## 7. `missing field 'kind'` on the runtime request

**When:** using a minimal hand-written request like `{"text": "hello traverse"}`.

**Cause:** the real `runtime_request` shape requires the full envelope (`kind`, `schema_version`, `request_id`, `intent`, `lookup`, `context`, `governing_spec`) — the actual capability input goes inside `input`, not at the top level.

**Fix:** copy the full shape from `references/contract-schema.md` (or an existing example like `hello-world`'s `runtime-requests/say-hello.json`) rather than writing a minimal one from scratch.

## 8. `runtime execution failed: unsupported AI agent capability: <your.id>`

**When:** running `agent execute` on a capability that passed `agent inspect` cleanly, has `lifecycle: "active"`, and isn't one of seven specific demo IDs (`hello.world.say-hello`, `traverse-starter.process`, `traverse-starter.validate`, `traverse-starter.summarize`, `meeting-notes.process`, `expedition.planning.interpret-expedition-intent`, `expedition.planning.validate-team-readiness`).

**Cause:** this is not a mistake in anything you built. `traverse-cli agent execute` runs through `AgentPackageExampleExecutor` in `crates/traverse-cli/src/main.rs` — a stub that pattern-matches against that hardcoded allowlist and never reaches the real Wasmtime-based `WasmExecutor` for anything else. It's a real, currently open platform gap (an active "Real WASM execution" effort is replacing this, per spec 516).

**`agent execute` specifically has no fix, but it isn't the only CLI-native execution path.** `traverse-cli serve` starts an HTTP API backed by the real `ArtifactRouter` executor — `POST /v1/capabilities/register` (any contract + binary, not allowlisted) then `POST /v1/capabilities/execute` genuinely runs it. Verified hands-on: this gets a brand-new capability all the way through lookup, contract validation, and artifact verification — real progress `agent execute` never reaches. See entry #11 below for what it hits next.

## 9. A workflow node isn't getting a field you expected it to have

**When:** writing a multi-node workflow and a downstream node's contract fails validation because a field it expects (via `from_workflow_input`) never arrives, or conversely you assume a node can *only* see the workflow's original top-level input fields and design around that unnecessarily.

**Cause:** despite the name, `from_workflow_input` on a node reads from the *accumulated workflow state* — the original request input plus every upstream node's `to_workflow_state` output — not strictly the workflow's declared top-level input schema. In the real `plan-expedition` example, the `interpret_intent` node reads `objective` via `from_workflow_input` even though `objective` isn't one of the workflow's own top-level inputs; it only exists in state because the `capture_objective` node produced it first.

**Fix:** when a node needs a field, check whether any upstream node's `to_workflow_state` produces it — that's the actual source of truth, not the workflow's `inputs.schema`. And when ordering nodes, make sure anything a node reads via `from_workflow_input` was either an original input or was written by a node earlier in the `edges` chain — the runtime won't reorder nodes to satisfy a dependency you got backwards.

## 10. Expecting a `workflow execute` command that doesn't exist

**When:** having registered and inspected a workflow successfully (`workflow register`/`workflow list`/`workflow inspect` all genuinely work), you look for the equivalent execute command and either guess at a syntax that errors out, or reach for `expedition execute` assuming it's a generic runner.

**Cause:** there is no bare workflow-execution command in `traverse-cli` at all — confirmed by reading the full `Command` enum in `crates/traverse-cli/src/main.rs`. `expedition execute` exists, but it's a bespoke command hardcoded to the expedition example domain, not a generic workflow runner. The real, generic traversal engine (`Runtime::execute_workflow` in `crates/traverse-runtime/src/workflows.rs`) does exist and is genuinely contract-enforced end-to-end — it's just not exposed as its own subcommand.

**Fix:** it is reachable, just not obviously. `traverse-cli serve`'s `/v1/capabilities/execute` (see entry #8) can run a workflow if you register it as a composite capability (`implementation_kind: "workflow"`, with a `workflow_ref` pointing at the registered workflow) rather than a plain executable one — confirmed via the `execute_endpoint_runs_pipeline_workflow_capability_with_merged_output` test in `http_api.rs`. That, or constructing a `Runtime<WasmExecutor>` yourself via the embedder SDKs and calling `.execute_workflow(...)` directly, are the two real options. Either way, every node's binary still has to clear the ABI wall in entry #11.

## 11. `unauthorized_host_import: ABI 1.0.0 does not allow import wasi_snapshot_preview1::environ_get`

**When:** running `traverse-cli wasm abi verify <path>.wasm`, or trying to actually execute a capability through `traverse-cli serve`'s `/v1/capabilities/execute` or an embedder SDK, after `agent inspect` already passed cleanly.

**Cause:** this is the real one to internalize, because it's much bigger than it looks. Traverse's `WasmExecutor` (`crates/traverse-runtime/src/executor/wasm.rs`) enforces a deny-by-default host-import whitelist (`host_abi_v1.json`) that permits exactly `wasi_snapshot_preview1::{fd_read, fd_write, proc_exit}` plus a few `traverse_host` metadata calls — nothing else, no `environ_get`, `args_get`, `clock_time_get`, `random_get`, etc. Verified hands-on: an ordinary `cargo build --target wasm32-wasip1 --release` binary imports more than this whitelist allows *regardless of what your code does* — even `fn main() { println!("hi"); }`, with zero `std::env` usage anywhere in the source, imports `environ_get` and fails this check identically to a real capability. This is Rust's standard WASI startup sequence, not something introduced by your program logic.

**There is no fix available inside your own Cargo project today.** This blocks every execution path equally, because `traverse-cli serve` and both embedder SDKs (`traverse-embedder`, `traverse-embedder-web`) all execute through this same `WasmExecutor`. It's the identical constraint blocking Traverse's own "Real WASM execution" effort upstream (their own `doc-approval.analyze` demo package hits the same `environ_get` rejection) — their own notes describe the fix as needing either real conformant WASI artifacts for the demo packages or a formally approved ABI/constraint change, not an application-level code change. Until that lands, treat `agent inspect` / `capability register` success as the honest ceiling of what you can prove for a capability built the standard way — say so plainly rather than promising execution "just needs the right embedder."

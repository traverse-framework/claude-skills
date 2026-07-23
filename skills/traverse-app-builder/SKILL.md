---
name: traverse-app-builder
description: Builds apps with Traverse (the contract-driven WASM runtime at traverse-framework/traverse) the real way — check the registry for capabilities that already exist before building new ones, author new atomic capabilities as JSON-Schema-backed contracts when nothing fits, compile them to genuine wasm32-wasip1 binaries, compose several into a multi-node workflow, and validate everything end-to-end with traverse-cli. Use this skill whenever the user wants to build an app, service, agent, or business-logic capability with Traverse, wants to reuse or discover existing capabilities in the registry, wants to compose a workflow out of smaller pieces, asks about running the same logic across multiple platforms/targets/OSes, mentions traverse-cli, capability contracts, or WASM business logic — even if they don't name every step or use the word "skill". Also use it if the user asks what's actually possible with Traverse today or seems to be guessing at the CLI/contract/registry shape, since this skill encodes the current, verified-working schema and flags real platform limitations (traverse-cli agent execute only runs a hardcoded demo allowlist, and there's no generic workflow-execute command either) that the project's own docs and scaffold script don't consistently reflect.
---

# Building apps with Traverse

Traverse's actual differentiator isn't "compile Rust to WASM" — plenty of things do that. It's that every capability carries a **contract**: a JSON Schema declaration of what has to be true before it runs (preconditions), what it promises after (postconditions), and what it's allowed to touch (permissions, network/filesystem access). The runtime enforces the contract, not just the code. If you skip the contract and just write Rust, you're not really building a Traverse capability — you're building a WASM binary that happens to sit next to one.

An app built with Traverse is rarely one capability. It's usually: some capabilities you can reuse from the registry, a few new atomic ones you author because nothing fits, and a workflow that chains them together — all of it meant to run with identical enforced behavior wherever the runtime is embedded. This skill walks through all four pieces the real, verified-working way: check the registry first, author a new capability only when you need to, compose capabilities into a workflow, and understand honestly which targets that composed logic can actually run on today. It also tells you plainly where the platform's current rough edges are — a previous pass through this exact workflow surfaced real gaps between what `traverse-cli agent execute` (and workflow execution generally) promises and what it actually does, and pretending otherwise would waste your time.

## Before you start

You need:
1. **Rust with the `wasm32-wasip1` target.** `rustup target add wasm32-wasip1`.
2. **A clone of `traverse-framework/traverse`** (for `traverse-cli` and, if you want to place your capability inside the examples tree, the registry paths). If the user doesn't have one, clone it — everything here is validated against it.

**A real gotcha to check first:** if the machine has both Homebrew-installed Rust *and* rustup, `cargo build --target wasm32-wasip1` can fail with `error[E0463]: can't find crate for 'core'` even after `rustup target add` reports the target installed — because `cargo`/`rustc` on `PATH` resolve to the Homebrew toolchain, which doesn't share rustup's installed targets. Confirm with:
```bash
rustc --version --verbose   # should mention rustup's toolchain, not Homebrew
which rustc                  # if this is /opt/homebrew/bin/rustc, that's the trap
```
If it's the Homebrew one, don't hand-construct the rustup toolchain path — `uname -m` reports `arm64` on Apple Silicon, but rustup's toolchain directory is named `aarch64-apple-darwin`, so a naive `$(uname -m)` substitution silently builds a path that doesn't exist and you're back to Homebrew's binaries with no error. Use `rustup which cargo` (and `rustup which rustc`) to get the real paths, and prepend *their* directory to `PATH`: `PATH="$(dirname "$(rustup which cargo)"):$PATH"`. Don't skip this check — it's the single most common thing that silently derails this whole workflow, and the error message doesn't mention rustup/Homebrew at all.

## Step 0: check what already exists before building anything new

Don't scaffold a capability that's already been published. Traverse's registry is genuinely queryable, in two different ways depending on what you're checking:

```bash
traverse-cli registry sync --workspace <workspace-id> --json
cat .traverse/workspaces/<workspace-id>/registry/public/index.json
```

`registry sync` downloads the public registry index and writes it locally; the sync command's own output only gives you a record count, so read the JSON file it wrote to actually see what's there. Each entry is a thin pointer (id, version, digests, an `artifact_url`, a `contract_url`) — there's no summary or description in the index itself, so you have to fetch `contract_url` for any candidate to see what it actually does before deciding to reuse it. If you're checking what's already declared inside a specific app bundle rather than the wider public registry, use `traverse-cli capability discover <bundle-manifest> --json` instead — it lists the capabilities already registered in that bundle, with real summaries and tags because it reads them from resolved contracts already in memory. Full detail, including how to publish a capability you built so others can find it later, is in `references/registry.md`.

Only move on to authoring a new capability once you've confirmed nothing existing covers what the user needs.

## The five real steps for authoring one new atomic capability

### 1. Name it and pick a shape

A capability id is `namespace.name` (e.g. `invoicing.calculate-total`). Decide your input and output shape as JSON Schema up front — this goes straight into the contract and the Rust struct, so getting it right early saves a rewrite later.

Keep it atomic — one verb, one responsibility. This isn't just tidiness: a real example in the repo (`workflows/examples/expedition/plan-expedition/workflow.json`) chains five narrowly-scoped capabilities — capture, interpret, assess, validate, assemble — into one workflow, and that only composes cleanly because each one does a single thing with a narrow contract. A capability that tries to capture-and-validate-and-assemble in one binary can't be reused piecemeal the way each of those five can. If what the user describes sounds like it has more than one verb in it ("validate and send," "fetch and summarize"), that's usually two capabilities plus a workflow, not one capability — see "Composing capabilities into a workflow" below.

### 2. Write the contract

Copy the shape from `references/contract-schema.md` — it's derived directly from the current `CapabilityContract` struct in `crates/traverse-contracts/src/lib.rs`, not from `scripts/scaffold/new-capability.sh`. **Don't use that scaffold script's contract template** — it was verified stale: it emits `input_schema`/`output_schema` flat keys and omits several required fields (`owner`, `summary`, preconditions/postconditions arrays, `provenance`, `evidence`), none of which match what the contract deserializer actually accepts today. The real shape nests schemas as `inputs: { schema: {...} }` / `outputs: { schema: {...} }`, matching the checked-in `examples/hello-world/.../contract.json`.

The one field most worth getting right on purpose: **`lifecycle`**. It's tempting to set `"draft"` for a capability you're still building, but the runtime only executes `"active"` (and `"deprecated"`) contracts — `"draft"` is correctly rejected with `not runnable locally`. Set it to `"active"` once you intend to actually run it, even during local iteration.

Also write the accompanying **workflow** (`workflows/.../<name>/workflow.json`, `kind: "workflow_definition"`) — a package with an empty `workflow_refs` is rejected outright ("agent package must declare at least one approved workflow reference"). A single-node workflow that just calls your one capability is enough; copy the shape from `references/contract-schema.md`.

### 3. Implement it in Rust

The real execution contract is **stdin = input JSON, stdout = output JSON** — confirmed directly in `traverse-runtime`'s WASM executor (`WasiCtxBuilder::new().stdin(...).stdout(...)`), not something you have to guess at:

```rust
use std::io::{self, Read, Write};

fn main() {
    let mut input = String::new();
    io::stdin().read_to_string(&mut input).unwrap_or_default();
    let request: serde_json::Value = serde_json::from_str(&input).unwrap_or(serde_json::Value::Null);

    // read your fields from `request`, do the work, build `output`
    let output = serde_json::json!({ "your_output_field": "..." });

    let _ = io::stdout().write_all(output.to_string().as_bytes());
}
```

**A significant, verified limitation to set expectations on now, not after compiling:** the template above compiles fine and validates fine (steps 4-5 below), but it will not pass Traverse's real Host ABI check at execution time. The runtime's `WasmExecutor` enforces a deny-by-default import whitelist (`crates/traverse-runtime/src/executor/host_abi_v1.json`) that only allows `wasi_snapshot_preview1::{fd_read, fd_write, proc_exit}` plus a handful of `traverse_host` metadata calls — nothing else. An ordinary `cargo build --target wasm32-wasip1` binary imports more than that as part of Rust's standard WASI startup (confirmed hands-on: even a bare `fn main() { println!("hi"); }` with no `std::env` usage anywhere in the source still imports `environ_get`, which fails `traverse-cli wasm abi verify` with `unauthorized_host_import`). This isn't specific to this template or a mistake in your code — it's a genuine, currently open platform constraint (the same one blocking the "Real WASM execution for traverse-cli agent execute" work upstream), and it affects every execution path equally since they all share the same `WasmExecutor` — see "Actually running it" below for what that means in practice.

`Cargo.toml` needs a `[[bin]]` pointing at your source file and, if this crate lives inside a clone of `traverse-framework/traverse` but isn't a member of its workspace, an empty `[workspace]` table so cargo doesn't complain about being an orphaned workspace member:

```toml
[package]
name = "your-agent"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "your-agent"
path = "src/agent.rs"

[dependencies]
serde_json = "1"

[workspace]   # only needed if nested inside the traverse-framework/traverse checkout
```

### 4. Compile it

```bash
cargo build --target wasm32-wasip1 --release
```
(Remember the PATH check above if this fails with `can't find crate for 'core'`.) The binary lands at `target/wasm32-wasip1/release/<name>.wasm`.

### 5. Validate it with the real CLI

Write a `manifest.json` (`kind: "agent_package"`) referencing your contract and binary, and a `runtime-request.json` (`kind: "runtime_request"`) — shapes are in `references/contract-schema.md`. Leave `binary.expected_digest` as a placeholder (`"fnv1a64:0000000000000000"`) for the first run — Traverse uses FNV-1a, not SHA-256, and the fastest way to get the right value is to let the CLI tell you:

```bash
cargo run -p traverse-cli-rs -- agent inspect path/to/manifest.json
# → "agent binary digest mismatch ... expected fnv1a64:..., got fnv1a64:<real digest>"
```
Paste that real digest into `expected_digest`, re-run `agent inspect`, and confirm it prints `package_id`, `capability_id`, and `workflow_refs` cleanly. **That's a real, meaningful pass** — it means your contract deserializes, your binary digest matches, your workflow reference resolves, and your lifecycle is runtime-eligible. Don't skip straight to `agent execute` without this — it's the fast, cheap way to catch schema mistakes before the confusing part below.

## Composing capabilities into a workflow

When an app needs more than one atomic capability strung together, don't reach for a single monolithic capability — write a workflow with multiple nodes instead, the same way the real `plan-expedition` example chains five capabilities (capture → interpret → assess → validate → assemble). Each node names a `capability_id`/`capability_version`, declares which fields it reads via `from_workflow_input`, and which fields it writes via `to_workflow_state`; `edges` (with `trigger: "direct"` for the simple case) say which node runs next. The full annotated example, plus a naming gotcha worth knowing before you write one (`from_workflow_input` actually reads the accumulated workflow *state*, not just the workflow's original top-level input), is in `references/contract-schema.md`.

Registering and inspecting a workflow is real, generic CLI functionality, not a demo stub:
```bash
traverse-cli workflow register path/to/workflow.json --workspace <workspace-id>
traverse-cli workflow list --workspace <workspace-id>
traverse-cli workflow inspect <workflow-id> --workspace <workspace-id>
```
These work for any workflow you throw at them. Running one end-to-end is a different story — see the honesty section below.

## The multi-target story — what's actually true

The reason Traverse frames this as "the same business logic on more than one platform" rather than "compiles to WASM" is structural, not marketing: your capability compiles once to a single `wasm32-wasip1` binary, and the contract enforcement + execution semantics live in `Runtime<E: LocalExecutor>` inside `traverse-runtime` — generic over whatever host provides the executor. A native host plugs in the real Wasmtime-backed executor; the browser embedder does the WASM-capable equivalent for that environment. Same binary, same contract, same enforcement, different host — that's the actual mechanism, not a slogan.

But "the mechanism supports it" and "it's shipped for your target" are different questions — check `references/platforms.md` before promising a user their capability will run somewhere specific. Native and browser (via the Rust and Web embedder SDKs) are the two targets genuinely shipped today; Swift/iOS is blocked on an upstream WASM-engine dependency, Kotlin/.NET embedders exist but aren't released, edge is unstarted, and cloud placement is an explicit non-goal for v0.1. And there's no single command that runs your capability across multiple targets and diffs the results for you — proving cross-target parity yourself today means actually invoking the same manifest and request through two different embedders (e.g. the Rust `traverse-embedder` and the Web `traverse-embedder-web`) and comparing the outputs by hand — though see "Actually running it" below before promising either one will run an ordinary Rust-built capability at all right now.

## Actually running it — the honest part

Run `cargo run -p traverse-cli-rs -- agent execute <manifest> <request>` and, unless your capability id happens to be one of a **hardcoded list of seven demo capabilities** (`hello.world.say-hello`, `traverse-starter.process/.validate/.summarize`, `meeting-notes.process`, `expedition.planning.interpret-expedition-intent/.validate-team-readiness`), you'll get:
```
runtime execution failed: unsupported AI agent capability: your.namespace.here
```
This is `AgentPackageExampleExecutor` in `crates/traverse-cli/src/main.rs` — a stub that *reimplements* those seven capabilities' logic natively in Rust rather than actually running their `.wasm` bytes, and rejects anything outside the allowlist. It's a real, tracked limitation, actively being replaced upstream (spec 516).

**There is a second, genuinely generic CLI path that isn't in `agent execute`'s docs anywhere: `traverse-cli serve`.** It starts an HTTP/JSON API (`traverse-cli serve --port <N>`) backed by the real Wasmtime `ArtifactRouter` executor, not the demo stub. Verified hands-on with a freshly authored capability (not a shipped demo):
```bash
# register any capability contract + binary — not allowlisted
curl -X POST http://127.0.0.1:8787/v1/capabilities/register \
  -H "Authorization: Bearer system_admin" \
  -d '{"workspace_id":"local-default","scope":"workspace_persisted","registry_scope":"private","contract":<contract.json>}'
# then run it
curl -X POST "http://127.0.0.1:8787/v1/capabilities/execute?workspace_id=local-default" \
  -H "Authorization: Bearer system_admin" -d @runtime-request.json
```
(`contract.execution.entrypoint.command` is resolved as a literal filesystem path to the compiled `.wasm`, relative to wherever `serve` was started — not just an abstract command name. A non-JWT bearer token like `system_admin` is accepted as a dev subject id in the default `dev-loopback` auth mode; don't reach for `--allow-unauthenticated` to work around a 401, since that flag genuinely opens the server to unauthenticated network callers.) This same endpoint can also run whole workflows, wrapped as a composite capability whose `implementation_kind` is `workflow` — so `execute_workflow`-style behavior is reachable through `serve` too, not only through embedding.

**But here's the wall both this path and the embedder SDKs hit today:** the register/execute call above got past capability lookup, contract validation, artifact verification, and placement selection — genuine progress, not a rejection — and then failed at actual WASM instantiation, because the binary imports `wasi_snapshot_preview1::environ_get`, which the ABI whitelist (see the box above) doesn't permit. This is not specific to `serve` — the embedder SDKs (`traverse-embedder`, `traverse-embedder-web`) execute through the exact same `WasmExecutor`, so a capability built from an ordinary `cargo build --target wasm32-wasip1` binary will fail identically there too. **As of today, there is no path — CLI `serve` or embedder — that can run a capability compiled the standard way**, independent of which manifest format or executor you go through. This is the same blocker the upstream "Real WASM execution" work is actively trying to clear (it requires either real conformant WASI artifacts for the demo packages or a formally approved ABI/constraint change — not something fixable from inside your own Cargo project).

If the user's goal is just "prove my capability logic is correct, structured right, and registrable" — steps 1-5 above are the complete, real, currently-achievable answer, and you should say so plainly. If they want to see it actually run, be upfront that this is blocked on an open upstream constraint today, not a missing step in this skill — point them at `references/troubleshooting.md`'s ABI entry for the exact evidence, and don't imply switching to the embedder SDKs sidesteps it, since it doesn't.

**The same execution-layer honesty applies to multi-node workflows**, with one correction: `traverse-cli workflow register/list/inspect` are real and generic (registry-side, not execution). There is no bare `traverse-cli workflow execute` command, and `expedition execute` is a bespoke command hardcoded to the expedition example domain, not a generic runner. But `Runtime::execute_workflow` (`crates/traverse-runtime/src/workflows.rs`) is a real, generic, contract-enforced traversal engine, and — as shown above — `serve`'s `/v1/capabilities/execute` can reach it too, by registering the workflow as a composite capability. It hits the identical ABI wall as a single capability the moment any node's binary is an ordinary Rust build.

## Where to go deeper

- `references/registry.md` — how to actually check what's already published before building something new, and how to publish a capability you've built so others can reuse it.
- `references/contract-schema.md` — the full, struct-verified contract/workflow/manifest/request JSON shapes (including multi-node workflow composition), plus every real enum value (lifecycle states, service types, execution targets, access levels).
- `references/platforms.md` — honest current status per platform (native, browser, Rust/Web embedding, Swift/Kotlin/.NET, edge, cloud) — check before promising the user a target works.
- `references/troubleshooting.md` — every real error this skill's own testing hit, verbatim, with the actual fix.
- `docs/capability-contract-authoring-guide.md` and `docs/wasm-agent-authoring-guide.md` in the repo, and https://traverse-framework.com/llms.txt for the wider docs map.

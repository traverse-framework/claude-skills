---
name: traverse-app-builder
description: Builds apps with Traverse (the contract-driven WASM runtime at traverse-framework/traverse) the real way — check the registry for capabilities that already exist before building new ones, author new atomic capabilities as JSON-Schema-backed contracts when nothing fits, compile them to genuinely executable, ABI-compliant WASM binaries, compose several into a multi-node workflow, and validate and run everything end-to-end with traverse-cli. Use this skill whenever the user wants to build an app, service, agent, or business-logic capability with Traverse, wants to reuse or discover existing capabilities in the registry, wants to compose a workflow out of smaller pieces, asks about running the same logic across multiple platforms/targets/OSes, mentions traverse-cli, capability contracts, or WASM business logic — even if they don't name every step or use the word "skill". Also use it if the user asks what's actually possible with Traverse today or seems to be guessing at the CLI/contract/registry shape, since this skill encodes the current, verified-working schema and the real reason an ordinary `cargo build --target wasm32-wasip1` binary won't execute (a governed host-import ABI, not a missing feature) plus the actual `#![no_std]` pattern that clears it — details the project's own docs and scaffold script don't consistently reflect.
---

# Building apps with Traverse

Traverse's actual differentiator isn't "compile Rust to WASM" — plenty of things do that. It's that every capability carries a **contract**: a JSON Schema declaration of what has to be true before it runs (preconditions), what it promises after (postconditions), and what it's allowed to touch (permissions, network/filesystem access). The runtime enforces the contract, not just the code. If you skip the contract and just write Rust, you're not really building a Traverse capability — you're building a WASM binary that happens to sit next to one.

An app built with Traverse is rarely one capability. It's usually: some capabilities you can reuse from the registry, a few new atomic ones you author because nothing fits, and a workflow that chains them together — all of it meant to run with identical enforced behavior wherever the runtime is embedded, and to actually execute, not just validate. This skill walks through all of it the real, verified-working way: check the registry first, author a new capability only when you need to, compile it so it genuinely runs (not just compiles), compose capabilities into a workflow, and understand honestly which targets that composed logic can actually run on today.

## Before you start

You need:
1. **Rust with the `wasm32-unknown-unknown` target.** `rustup target add wasm32-unknown-unknown`. (Not `wasm32-wasip1` — see step 3 for why that target won't produce an executable capability today, however cleanly it compiles.)
2. **A clone of `traverse-framework/traverse`** (for `traverse-cli` and, if you want to place your capability inside the examples tree, the registry paths). If the user doesn't have one, clone it — everything here is validated against it.

**A real gotcha to check first:** if the machine has both Homebrew-installed Rust *and* rustup, compiling for `wasm32-unknown-unknown` can fail with `error[E0463]: can't find crate for 'core'` even after `rustup target add` reports the target installed — because `cargo`/`rustc` on `PATH` resolve to the Homebrew toolchain, which doesn't share rustup's installed targets. Confirm with:
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

The real execution contract is **fd 0 = input JSON in, fd 1 = output JSON out**. But before writing a single line, understand the constraint that actually shapes this step: Traverse's `WasmExecutor` enforces a deny-by-default host-import whitelist (`crates/traverse-runtime/src/executor/host_abi_v1.json`) — only `wasi_snapshot_preview1::{fd_read, fd_write, proc_exit}` plus a handful of `traverse_host` metadata calls are allowed. Nothing else.

**An ordinary `cargo build --target wasm32-wasip1` binary cannot pass this**, no matter what your code does — verified hands-on: even a bare `fn main() { println!("hi"); }` with zero `std::env` usage imports `wasi_snapshot_preview1::environ_get` as part of Rust's standard WASI startup, and fails `traverse-cli wasm abi verify` with `unauthorized_host_import`. This isn't a mistake to fix in your code; it's `std`'s runtime init pulling in more than the whitelist allows, unconditionally. Don't reach for the `std::io::stdin()/stdout()` template you'll find in most Rust-WASM tutorials — it will validate fine (`agent inspect` doesn't check imports) and then fail the moment it actually tries to run.

**The pattern that genuinely clears this ABI and executes real, dynamic logic** — not a hardcoded stub — is `#![no_std]` plus hand-declared `extern "C"` imports for exactly the WASI functions the whitelist allows, compiled with plain `rustc` against `wasm32-unknown-unknown` (not `wasm32-wasip1`, and not `cargo build`, which drags in libstd's WASI shims regardless of the target). Verified end-to-end: a binary built this way that reads real stdin, scans for a JSON field by hand, transforms it, and writes real JSON output passed `wasm abi verify`, then ran through `traverse-cli agent execute` and had its dynamic output validated against a real output contract — not a canned response.

```rust
#![no_std]
#![no_main]

#[repr(C)]
struct IoVec { buffer: *const u8, length: usize }
#[repr(C)]
struct IoVecMut { buffer: *mut u8, length: usize }

#[link(wasm_import_module = "wasi_snapshot_preview1")]
unsafe extern "C" {
    fn fd_read(fd: u32, vectors: *const IoVecMut, count: usize, read: *mut usize) -> u32;
    fn fd_write(fd: u32, vectors: *const IoVec, count: usize, written: *mut usize) -> u32;
}

static mut INPUT_BUF: [u8; 4096] = [0; 4096];
static mut OUTPUT_BUF: [u8; 4096] = [0; 4096];

#[unsafe(no_mangle)]
pub extern "C" fn _start() {
    unsafe {
        // Read all of stdin (fd 0) into INPUT_BUF.
        let mut total = 0usize;
        loop {
            let vec = IoVecMut { buffer: INPUT_BUF.as_mut_ptr().add(total), length: INPUT_BUF.len() - total };
            let mut n = 0usize;
            if fd_read(0, &vec, 1, &mut n) != 0 || n == 0 { break; }
            total += n;
            if total >= INPUT_BUF.len() { break; }
        }
        let input = &INPUT_BUF[..total];

        // Parse the fields you need out of `input` by hand (see below), do the
        // work, and write JSON bytes into OUTPUT_BUF, tracking out_len.
        let out_len = 0usize; // build your real output here

        let out = IoVec { buffer: OUTPUT_BUF.as_ptr(), length: out_len };
        let mut written = 0usize;
        let _ = fd_write(1, &out, 1, &mut written);
    }
}

#[panic_handler]
fn panic(_: &core::panic::PanicInfo<'_>) -> ! { loop {} }
```

Three honest costs of this pattern, worth setting expectations on before the user is surprised by them:
- **No `serde_json`, no heap, no `String`/`Vec`** unless you bring your own allocator — `no_std` means exactly that. Parsing and writing JSON is done by hand over fixed-size byte buffers (find a field by its literal `"key":"` byte pattern, copy/transform bytes, write a literal `{"key":"..."}` prefix/suffix). This is real, verified-working, but it is more code and more care than an ordinary Rust program for anything beyond simple field extraction/transformation. A `no_std`-compatible JSON crate might reduce this, but none has been verified against this exact `wasm32-unknown-unknown` + `#![no_main]` setup by this skill — treat that as worth trying, not a recommendation.
- **`static mut` buffers trigger `static_mut_refs` warnings** on current Rust editions. They're not unsound in this specific shape (Wasmtime gives each invocation a fresh, single-threaded `Store`, so nothing else can alias these statics), but wrap them properly (e.g. behind `UnsafeCell` or a small safe abstraction) rather than leaving raw `static mut` in anything beyond a proof of concept.
- **A panic loops forever** (`loop {}`) rather than trapping cleanly, since there's no `std::process::abort`/unwinding machinery available. Fine for a sandboxed, fueled Wasmtime execution (it'll hit the fuel limit and get killed), but don't expect a clean error message on a genuine panic — validate your inputs defensively instead of relying on panics to signal failure.

`Cargo.toml` isn't actually used to build this (see step 4 — it's a bare `rustc` invocation), but keep one anyway for editor tooling and to record dependencies if you add any `no_std`-compatible ones:

```toml
[package]
name = "your-agent"
version = "0.1.0"
edition = "2021"

[workspace]   # only needed if nested inside the traverse-framework/traverse checkout
```

### 4. Compile it

Not `cargo build` — plain `rustc`, targeting `wasm32-unknown-unknown` as a `cdylib`, which is what actually avoids pulling in libstd's WASI startup:

```bash
rustc src/agent.rs --target wasm32-unknown-unknown --crate-type cdylib -O -o artifacts/your-agent.wasm
```
(Remember the PATH check above if `rustc` resolves to the wrong toolchain, and run `rustup target add wasm32-unknown-unknown` once if you haven't.) Confirm it actually clears the ABI before moving on:
```bash
traverse-cli wasm abi verify artifacts/your-agent.wasm
# → "artifacts/your-agent.wasm: ABI 1.0.0 import whitelist passed (N imports)"
```
If this fails, you're almost certainly still linking against `std` somewhere (check for a stray `use std::` or a dependency that pulls it in) — the whitelist check tells you exactly which disallowed import triggered it.

### 5. Validate it with the real CLI

Write a `manifest.json` (`kind: "agent_package"`) referencing your contract and binary, and a `runtime-request.json` (`kind: "runtime_request"`) — shapes are in `references/contract-schema.md`. Leave `binary.expected_digest` as a placeholder (`"fnv1a64:0000000000000000"`) for the first run — Traverse uses FNV-1a, not SHA-256, and the fastest way to get the right value is to let the CLI tell you:

```bash
cargo run -p traverse-cli-rs -- agent inspect path/to/manifest.json
# → "agent binary digest mismatch ... expected fnv1a64:..., got fnv1a64:<real digest>"
```
Paste that real digest into `expected_digest`, re-run `agent inspect`, and confirm it prints `package_id`, `capability_id`, and `workflow_refs` cleanly. **That's a real, meaningful pass** — it means your contract deserializes, your binary digest matches, your workflow reference resolves, and your lifecycle is runtime-eligible. It's also the fast, cheap way to catch schema mistakes before running it for real in the next section — `agent inspect` doesn't check host imports, so a std-based binary will pass this step and only fail at actual execution (see step 3's ABI warning).

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

The reason Traverse frames this as "the same business logic on more than one platform" rather than "compiles to WASM" is structural, not marketing: your capability compiles once to a single WASM binary, and the contract enforcement + execution semantics live in `Runtime<E: LocalExecutor>` inside `traverse-runtime` — generic over whatever host provides the executor. A native host plugs in the real Wasmtime-backed executor; the browser embedder does the WASM-capable equivalent for that environment. Same binary, same contract, same enforcement, different host — that's the actual mechanism, not a slogan.

But "the mechanism supports it" and "it's shipped for your target" are different questions — check `references/platforms.md` before promising a user their capability will run somewhere specific. Native and browser (via the Rust and Web embedder SDKs) are the two targets genuinely shipped today; Swift/iOS is blocked on an upstream WASM-engine dependency, Kotlin/.NET embedders exist but aren't released, edge is unstarted, and cloud placement is an explicit non-goal for v0.1. And there's no single command that runs your capability across multiple targets and diffs the results for you — proving cross-target parity yourself today means actually invoking the same manifest and request through two different embedders (e.g. the Rust `traverse-embedder` and the Web `traverse-embedder-web`) and comparing the outputs by hand. Both share the same ABI whitelist as `agent execute` and `serve`, so the same step-3 build pattern applies regardless of which target you're proving.

## Actually running it

`traverse-cli agent execute <manifest> <request>` now genuinely executes any capability that clears the ABI whitelist above — not just a hardcoded demo set. That used to be a real, tracked limitation (`AgentPackageExampleExecutor`, a stub that only reimplemented seven demo capabilities natively in Rust and rejected everything else); spec 516 landed and replaced it with the real `ArtifactRouter` executor everywhere, including `agent execute` itself. Verified hands-on with a capability that has never existed in that demo set: `agent inspect` passes, `agent execute` runs the real `.wasm` bytes, and — for a genuinely dynamic capability (one built with the step 3 pattern, reading real stdin and computing real output, not a hardcoded string) — the runtime validates that computed output against the real output contract and reports `status: completed`.

```bash
traverse-cli agent execute path/to/manifest.json path/to/runtime-request.json
# → status: completed
```

If you see `runtime execution failed: registered artifact execution failed` instead, that's the ABI wall from step 3 — your binary still imports something outside the whitelist (almost always `environ_get` from a `std`-based build). Run `traverse-cli wasm abi verify` on the binary directly; it'll name the exact disallowed import. This is not the old allowlist rejection (`unsupported AI agent capability: <id>`) — if you see that specific message, you're on a version of `traverse-cli` from before spec 516 landed.

**A second, still-real path exists alongside `agent execute`: `traverse-cli serve`.** It starts an HTTP/JSON API backed by the same `ArtifactRouter` executor (`POST /v1/capabilities/register` then `POST /v1/capabilities/execute`), and can also run whole workflows by registering one as a composite capability whose `implementation_kind` is `workflow`. It isn't mentioned anywhere in `agent execute`'s own docs, so it's easy to miss, but it's genuinely useful if you want an HTTP surface rather than a one-shot CLI call, or want to drive a workflow rather than a single capability — see `references/registry.md` and `crates/traverse-cli/src/http_api.rs`'s route table for the endpoints. (A non-JWT bearer token like `system_admin` is accepted as a dev subject id in the default `dev-loopback` auth mode for local testing; don't reach for `--allow-unauthenticated` to work around a 401, since that flag genuinely opens the server to unauthenticated network callers.)

If the user's goal is just "prove my capability logic is correct, structured right, and registrable," steps 1-5 get there. If they want to see it actually run, `agent execute` is now the real, direct answer — the only remaining honest caveat is that it has to be built the step-3 way to clear the ABI, not with an ordinary `cargo build --target wasm32-wasip1`.

**The same is true for multi-node workflows**, with one nuance: there's still no bare `traverse-cli workflow execute` command (`workflow register/list/inspect` are real and generic, but registry-side only), and `expedition execute` remains a bespoke command hardcoded to the expedition example domain. `traverse-cli serve`'s composite-capability trick above is the generic way to run a workflow through the CLI today. Every node's binary still has to individually clear the ABI whitelist the step-3 way.

## Where to go deeper

- `references/registry.md` — how to actually check what's already published before building something new, and how to publish a capability you've built so others can reuse it.
- `references/contract-schema.md` — the full, struct-verified contract/workflow/manifest/request JSON shapes (including multi-node workflow composition), plus every real enum value (lifecycle states, service types, execution targets, access levels).
- `references/platforms.md` — honest current status per platform (native, browser, Rust/Web embedding, Swift/Kotlin/.NET, edge, cloud) — check before promising the user a target works.
- `references/troubleshooting.md` — every real error this skill's own testing hit, verbatim, with the actual fix.
- `docs/capability-contract-authoring-guide.md` and `docs/wasm-agent-authoring-guide.md` in the repo, and https://traverse-framework.com/llms.txt for the wider docs map.

# Troubleshooting — real errors, in the order you'll likely hit them

Every error below was actually produced hands-on while authoring capabilities through this exact workflow, not copied from documentation.

## 1. `error[E0463]: can't find crate for 'core'`

**When:** compiling for `wasm32-unknown-unknown`, even after `rustup target add` reported success and the `.rlib` files genuinely exist on disk.

**Cause:** the machine has both a Homebrew-installed Rust toolchain and a rustup-managed one. `rustup target add` only installs the target for *rustup's* toolchain. If `/opt/homebrew/bin/cargo`/`rustc` resolve first on `PATH`, the build uses Homebrew's `rustc`, which has no idea rustup installed anything.

**Fix:** `PATH="$(dirname "$(rustup which cargo)"):$PATH"` before building. Don't hand-construct the toolchain path (`uname -m` says `arm64`, rustup's directory says `aarch64-apple-darwin` — a naive substitution silently fails).

## 2. `error: current package believes it's in a workspace when it's not`

**When:** the capability's `Cargo.toml` lives inside a cloned repo that has its own root workspace `Cargo.toml` (e.g. nested under `traverse-framework/traverse`'s `examples/`, or `traverse-framework/registry`'s `agents/`).

**Fix:** add an empty `[workspace]` table to the capability's own `Cargo.toml` — this makes Cargo treat the crate as its own workspace root.

## 3. Binary path mismatch on build

**When:** `Cargo.toml` declares `[[bin]] path = "src/main.rs"` but the source file is actually named something else.

**Fix:** make `path` match the real file location. There's no default inference once `[[bin]]` is explicit.

## 4. `agent package must declare at least one approved workflow reference`

**When:** running `agent inspect` on a manifest with `"workflow_refs": []`.

**Fix:** write a `workflow_definition` JSON (`references/contract-schema.md`) and add a corresponding entry to `workflow_refs`.

## 5. `agent binary digest mismatch ... expected fnv1a64:..., got fnv1a64:<real digest>`

**When:** running `agent inspect` with a placeholder `expected_digest`.

**Cause:** Traverse hashes the compiled `.wasm` binary with FNV-1a-64, not SHA-256.

**Fix:** leave a placeholder (`fnv1a64:0000000000000000`), run `agent inspect`, copy the real digest out of the mismatch error.

## 6. `matching capabilities were found but none were runnable locally`

**When:** running `agent execute` while `lifecycle` is still `"draft"`.

**Cause:** the runtime only executes `Active`/`Deprecated` contracts. This is a correct rejection, not a bug.

**Fix:** set `lifecycle` to `"active"` once you actually intend to run it, even for local iteration.

## 7. `missing field 'kind'` on the runtime request

**When:** using a minimal hand-written request like `{"text": "hello traverse"}`.

**Cause:** the real `runtime_request` shape requires the full envelope (`kind`, `schema_version`, `request_id`, `intent`, `lookup`, `context`, `governing_spec`).

**Fix:** copy the full shape from `references/contract-schema.md`.

## 8. `unauthorized_host_import: ABI 1.0.0 does not allow import wasi_snapshot_preview1::environ_get`

**When:** running `wasm abi verify` (or trying to execute through `agent execute`/`serve`/an embedder SDK) on a binary built with `cargo build --target wasm32-wasip1` — even one that already passed `agent inspect` and all its own unit tests.

**Cause:** Traverse's `WasmExecutor` enforces a deny-by-default host-import whitelist (`fd_read`, `fd_write`, `proc_exit`, plus a few `traverse_host` calls). Rust's standard WASI startup imports `environ_get` unconditionally on `wasm32-wasip1`, regardless of what your code does — this was verified not just on a trivial `println!` example but on a real, fully input-dependent, well-tested capability rewrite (using `serde_json` + `std::io`) that still failed this check. **A clean `cargo test` and a clean `agent inspect` do not prove ABI compliance** — neither checks the import table. Always run this check directly before considering a capability done.

**Fix:** see `references/wasm-implementation.md`. Don't target `wasm32-wasip1`; use `#![no_std]` + `wasm32-unknown-unknown`, either hand-rolled (Option A) or via the shared `wasi-agent-runtime` shim (Option B, recommended for anything beyond flat string fields).

## 9. Expecting network access to work because the contract allows declaring it

**When:** setting `execution.constraints.network_access: "required"` in a contract for a capability that's supposed to call an external service.

**Cause:** the contract schema allows declaring this, but the ABI whitelist (see #8) doesn't grant any capability network-socket imports today — the schema field and the runtime's actual enforcement are two different layers, and only the latter determines what will really execute.

**Fix:** this isn't a bug to work around — it's a genuine current platform limitation. See the main skill's "secrets check" section: a capability that needs to call an external API can't do that from inside the sandbox today, key or no key. Push the network call to the calling application instead, and hand the capability only the already-fetched data as plain input.

## 10. `runtime execution failed: unsupported AI agent capability: <your.id>` (historical — fixed by spec 516)

If you see this exact message, you're on a `traverse-cli` build from before spec 516 merged (it replaced a hardcoded seven-capability demo allowlist with the real `ArtifactRouter` executor everywhere). Rebuild from current `main`. If you see `runtime execution failed: registered artifact execution failed` instead, that's not this — it's the ABI wall in #8.

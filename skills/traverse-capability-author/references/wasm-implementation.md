# Implementing a capability so it genuinely runs

## Before you start

You need:
1. **Rust with the `wasm32-unknown-unknown` target.** `rustup target add wasm32-unknown-unknown`. Not `wasm32-wasip1` — see "The ABI wall" below for why that target won't produce an executable capability today, however cleanly it compiles.
2. **A clone of `traverse-framework/traverse`** for `traverse-cli`, or a checkout of `traverse-framework/registry` (or your private registry) if that's where you're publishing.

**A real gotcha to check first:** if the machine has both Homebrew-installed Rust *and* rustup, compiling for `wasm32-unknown-unknown` can fail with `error[E0463]: can't find crate for 'core'` even after `rustup target add` reports the target installed — because `cargo`/`rustc` on `PATH` resolve to the Homebrew toolchain, which doesn't share rustup's installed targets. Confirm with:
```bash
rustc --version --verbose   # should mention rustup's toolchain, not Homebrew
which rustc                  # if this is /opt/homebrew/bin/rustc, that's the trap
```
If it's the Homebrew one, don't hand-construct the rustup toolchain path (on Apple Silicon, `uname -m` reports `arm64` but rustup's directory is `aarch64-apple-darwin` — a naive substitution silently builds a path that doesn't exist). Use the real paths directly: `PATH="$(dirname "$(rustup which cargo)"):$PATH"`.

## The ABI wall

Traverse's `WasmExecutor` enforces a deny-by-default host-import whitelist (`crates/traverse-runtime/src/executor/host_abi_v1.json`): only `wasi_snapshot_preview1::{fd_read, fd_write, proc_exit}` plus a couple of `traverse_host` metadata calls are allowed. Nothing else — no `environ_get`, no `args_get`, no `clock_time_get`, no `random_get`.

**An ordinary `cargo build --target wasm32-wasip1` binary cannot pass this, no matter what your code does.** Verified hands-on, twice, in two different codebases: even `fn main() { println!("hi"); }` with zero `std::env` usage imports `environ_get` as part of Rust's standard WASI startup sequence. And critically — **this failure mode isn't limited to trivial examples.** A real capability rewrite in this session used `serde_json` + `std::io::stdin()/stdout()` compiled via `cargo build --target wasm32-wasip1`: it was genuinely input-dependent, passed all its unit tests, and *still* imported `environ_get`/`environ_sizes_get` and would have failed at real execution — caught only by inspecting the compiled binary's import table directly (`wasm2wat artifacts/x.wasm | grep import`), not by anything `cargo test` or `agent inspect` checks. **Passing tests and passing `agent inspect` do not prove ABI compliance.** Always check the import table before calling a capability done.

## Two ways to build the actual logic

### Option A: raw fixed buffers, zero dependencies

The minimal pattern that clears the ABI. No heap, no `String`/`Vec`, no `serde_json` — everything over fixed-size byte buffers.

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
        let mut total = 0usize;
        loop {
            let vec = IoVecMut { buffer: INPUT_BUF.as_mut_ptr().add(total), length: INPUT_BUF.len() - total };
            let mut n = 0usize;
            if fd_read(0, &vec, 1, &mut n) != 0 || n == 0 { break; }
            total += n;
            if total >= INPUT_BUF.len() { break; }
        }
        let input = &INPUT_BUF[..total];
        // Parse the fields you need by hand (find a "key":" byte pattern, copy/transform
        // bytes), do the work, write JSON bytes into OUTPUT_BUF, tracking out_len.
        let out_len = 0usize; // build your real output here
        let out = IoVec { buffer: OUTPUT_BUF.as_ptr(), length: out_len };
        let mut written = 0usize;
        let _ = fd_write(1, &out, 1, &mut written);
    }
}

#[panic_handler]
fn panic(_: &core::panic::PanicInfo<'_>) -> ! { loop {} }
```

Compile with plain `rustc`, not `cargo build`:
```bash
rustc src/agent.rs --target wasm32-unknown-unknown --crate-type cdylib -O -o artifacts/your-agent.wasm
```

Use this when the input/output shape is genuinely trivial (a couple of flat string fields). Beyond that, hand-rolled byte-level JSON parsing gets error-prone fast — reach for Option B instead.

### Option B: the shared runtime shim (recommended for anything beyond trivial field extraction)

A reusable `#![no_std]` + `alloc` crate exists (verified working, ABI-clean) at `traverse-framework/registry`'s `agents/wasi-agent-runtime/` — it provides a small bump allocator, a real recursive-descent JSON parser/writer supporting nested objects/arrays (not just flat string fields), and the WASI stdin/stdout glue, all **without** linking libstd's WASI startup. This gets you `String`/`Vec`/real JSON — the ergonomics of `serde_json` — while staying ABI-compliant, because the allocator and panic handler are your own, not std's.

Depend on it as a path dependency (or vendor the two files — `src/lib.rs` and `src/json.rs` — if you're not inside that repo):

```toml
[workspace]

[package]
name = "your-capability-agent"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "your-agent"
path = "src/main.rs"

[dependencies]
wasi-agent-runtime = { path = "../wasi-agent-runtime" }

[profile.release]
opt-level = "s"
lto = true
panic = "abort"
```

```rust
#![cfg_attr(not(test), no_std)]
#![cfg_attr(not(test), no_main)]

extern crate alloc;
use alloc::string::String;
use wasi_agent_runtime::{object, Value};

fn your_logic(input: Value) -> Value {
    let field = input.get("your_field").and_then(Value::as_str).unwrap_or("");
    // real, input-dependent computation here
    object(alloc::vec![("your_output_field", Value::String(String::from(field)))])
}

#[cfg(not(test))]
#[unsafe(no_mangle)]
pub extern "C" fn _start() {
    wasi_agent_runtime::run_agent(your_logic);
}

#[cfg(test)]
mod tests {
    use super::*;
    // test your_logic() directly on the host target -- see references/testing-pattern.md
}
```

Build with `cargo build --target wasm32-unknown-unknown --release` (Cargo doesn't inject anything extra here — the ABI cleanliness comes from `#![no_std]`/`#![no_main]` and never linking libstd, not from avoiding Cargo specifically). Verified end to end this session: this exact pattern, built via plain `cargo build --target wasm32-unknown-unknown`, produces a binary whose only imports are `fd_read`/`fd_write`/`proc_exit`.

**One float-formatting gotcha inside the shim's JSON writer, worth knowing if you extend it:** avoid `f64::round`/`abs`/`fract` — on `wasm32-unknown-unknown` these can require libm symbols that aren't linked into a `no_std` binary with no libc. The shim's own number formatting uses only casts and integer arithmetic (`as i64`, `*`, `/`, `%`) to sidestep this; follow the same discipline if you touch it.

## Three honest costs of the no_std approach (either option)

- **No implicit heap in Option A; a small fixed heap in Option B.** Option A truly has no allocator — everything is fixed-size buffers. Option B's shim uses a 1 MiB bump allocator that never frees, which is fine for a single-shot WASI command process that exits right after producing its output, but means genuinely huge inputs/outputs (beyond ~1 MiB) will fail to allocate. Widen the shim's `HEAP_SIZE` constant if a specific capability needs more.
- **`static mut` buffers/allocator state trigger `static_mut_refs` warnings** on current Rust editions. Not unsound in this specific shape (Wasmtime gives each invocation a fresh, single-threaded `Store`), but don't let raw `static mut` spread beyond the shim itself.
- **A panic exits with a distinct non-zero code** (the shim) or loops forever (raw Option A) rather than trapping cleanly — there's no unwinding machinery. Validate inputs defensively; don't rely on panics to signal failure to a caller.

## Verifying before you call it done

```bash
wasm2wat artifacts/your-agent.wasm | grep import
# must show only wasi_snapshot_preview1::{fd_read, fd_write, proc_exit}
```

If anything else shows up (`environ_get` is the most common), you're still linking something that pulls in libstd's startup — check for a stray `use std::`, a `cargo build --target wasm32-wasip1` instead of `wasm32-unknown-unknown`, or a dependency (like `serde_json`) that assumes a full standard allocator/environment is available. Then actually run it, not just inspect it:

```bash
echo '{"your_field": "test value"}' | wasmtime run artifacts/your-agent.wasm
```

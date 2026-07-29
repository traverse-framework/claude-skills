# Proving a capability's output genuinely depends on its input

A capability that returns the same output regardless of input isn't a capability — it's a fixture pretending to be one. This has happened to real, published, otherwise-well-built capabilities: distinct WASM artifacts, correct digests, ABI-clean, and still fixed-output, because nobody wrote a test that would have caught it. Don't skip this.

## The pattern

Structure the crate so `cargo test` runs on the host target (with real `std`, a real allocator, a real test harness) while the actual capability compiles `#![no_std]` for `wasm32-unknown-unknown`:

```rust
#![cfg_attr(not(test), no_std)]
#![cfg_attr(not(test), no_main)]

extern crate alloc;
use wasi_agent_runtime::Value;

fn your_logic(input: Value) -> Value {
    // pure function: Value in, Value out, no I/O
}

#[cfg(not(test))]
#[unsafe(no_mangle)]
pub extern "C" fn _start() {
    wasi_agent_runtime::run_agent(your_logic);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn output_genuinely_differs_across_distinct_inputs() {
        let out_a = your_logic(/* input A */);
        let out_b = your_logic(/* input B, meaningfully different */);
        assert_ne!(out_a.get("the_field_that_should_vary"), out_b.get("the_field_that_should_vary"),
            "output must depend on input, not be fixed");
    }
}
```

This works because `your_logic` is a pure function over the shim's `Value` type — it doesn't touch `_start`, `fd_read`, or any WASI import, so it compiles and runs on the host exactly like ordinary Rust, with `cargo test` (no cross-compilation, no `wasmtime` needed to iterate).

## What this test does and doesn't prove

- **Proves:** the core logic is genuinely input-dependent, at the Rust-function level.
- **Doesn't prove:** the compiled WASM binary is ABI-compliant (that's a separate check — see `references/wasm-implementation.md`'s "verifying before you call it done") or that the WASI stdin/stdout plumbing itself works end to end.

Do both. A capability that passes the input-dependence test but fails ABI verification, or vice versa, is not done — this session's own experience is a real example of exactly that gap (a capability that was genuinely input-dependent, fully unit-tested, and still not runtime-executable because the ABI check was skipped).

## End-to-end confirmation, once both pass

```bash
wasm2wat artifacts/your-agent.wasm | grep import   # ABI check
echo '{"field": "value A"}' | wasmtime run artifacts/your-agent.wasm
echo '{"field": "value B"}' | wasmtime run artifacts/your-agent.wasm
# outputs must differ
```

Running the same distinct-input comparison through the actual compiled binary (not just the Rust unit test) is worth doing at least once per capability before publishing — it's the only check that exercises the real JSON parse/write path along with the logic.

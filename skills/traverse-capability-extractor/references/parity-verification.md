# Verifying an extraction reproduces the original behavior

## Why this is a separate step from "it compiles"

A ported function that compiles and looks plausible is not the same as a ported function that behaves the same as the original. This matters more for extraction than for fresh authoring, because the whole premise of extraction is "reuse logic that's already proven correct in production" — if the port silently diverges, you've thrown that guarantee away and gained a false sense of it back.

## Same-language extraction (source repo is already Rust)

Lower risk, but still verify. If the original function is already pure (per `references/scanning-heuristic.md`'s checklist), it's often a near-direct copy into the `no_std` shim's types (`alloc::string::String`/`alloc::vec::Vec` in place of `std`'s, per `traverse-capability-author`'s `references/wasm-implementation.md`). Run the original's own existing test suite's inputs (if it has one) through the ported version and diff outputs. If the original has no tests, write a handful covering typical and edge-case inputs before porting, then reuse them after.

## Cross-language extraction (source repo is Python, JS, Go, etc.)

Higher risk — this is a genuine re-implementation, not a copy. Structure it as explicit parity testing, not "I read the original and it seemed straightforward":

1. **Pick 5-10 representative inputs**, including edge cases (empty input, minimum/maximum plausible values, malformed-but-handled input if the original tolerates it).
2. **Capture the original's real output** for each — actually run the original code, don't reconstruct outputs from reading the source.
3. **Write the Rust port**, structured per `traverse-capability-author`'s pattern (a pure function, testable on the host target without any WASM involvement yet).
4. **Run the same inputs through the port** and diff against the captured originals field by field.
5. **Any mismatch is a real bug in the port**, not an acceptable rounding difference to wave off — unless the original behavior itself was already known to be approximate (e.g. a documented floating-point tolerance), in which case say so explicitly rather than silently loosening the comparison.

## What "done" looks like

A parity-verified extraction has: the original's captured input/output pairs committed as the port's own test fixtures (so regressions are caught later, not just at extraction time), and an explicit note in the capability's contract `description` or `provenance` if the port has any known, disclosed deviation from the original (e.g. "the original also handled a legacy input format not carried forward — see issue #NNN"). Don't publish a "based on" reimplementation as if it were a verified-identical port; disclose the gap the same way this ecosystem discloses any other known limitation.

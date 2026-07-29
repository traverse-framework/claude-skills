---
name: traverse-capability-extractor
description: Scans an existing repository — open-source or a private codebase — for business logic that's a good candidate to become one or more reusable Traverse capabilities, proposes an atomic decomposition, and hands the actual contract-authoring/compilation/publishing off to traverse-capability-author rather than duplicating that mechanism. Refuses to extract code that reads secrets, credentials, or environment state. Use this skill when the user wants to pull a Traverse capability out of code that already exists somewhere (their own repo, a library, a script, an open-source project) rather than writing one from scratch, asks "can we turn this function/module into a capability", or is migrating existing business logic onto Traverse. This is the extraction half of a three-skill split from the retired traverse-app-builder skill — see traverse-capability-author for the actual authoring mechanics this skill hands off to, and traverse-workflow-planner for going from a business workflow to a list of capabilities to build (which may itself point back here when a needed capability plausibly already exists, unextracted, in a known repo).
---

# Extracting capabilities from an existing repo

This skill answers a narrower question than "build me a capability": *"does the logic I need already exist, unpublished, somewhere in this codebase?"* Its job is to find the right pieces, decide how they should be split into atomic capabilities, and confirm the extraction actually reproduces the original behavior — then it stops and hands off to `traverse-capability-author` for the part that skill already does well (contract, WASM implementation, publish). Don't reimplement authoring here; that duplication is exactly what this three-skill split exists to avoid.

## Step 1: find real candidates

Not every function is a good capability candidate. Scan for code that is:

- **Pure, or nearly pure.** No network calls, no filesystem access, no database queries, no reads from environment variables or global mutable state. If a function's output depends only on its explicit arguments, it's a strong candidate; if it depends on "whatever's in the database right now" or "the current time," it isn't — Traverse capabilities are meant to be deterministic (`side_effects: memory_only` is the common case).
- **Narrow, well-typed input/output.** Data in, data out — strings, numbers, structured records that map cleanly onto JSON. A function that takes a live database connection, an open socket, or a framework-specific request object isn't extractable as-is; the *transformation logic inside it* might still be, once separated from the I/O around it.
- **Doing one thing.** A function or module that fetches, transforms, *and* notifies in one block isn't one capability — it's a capability (the transform) wrapped in orchestration (the fetch and notify) that should stay in the calling application or become workflow nodes. Pull out the pure transformation; leave the I/O where it is.

**Concrete example, verified this session (not hypothetical):** `traverse-framework/registry`'s `agents/doc-approval-analyze/src/main.rs` contains exactly this pattern before extraction — `classify_doc_type`, `extract_parties`, `extract_amounts`, `determine_confidence`, and `determine_recommendation` are five pure functions, each taking a string/slice and returning a value, with zero I/O. Together they form one coherent capability (`doc-approval.analyze`) because they compose into one meaningful output (a document analysis), not five unrelated ones — grouping cohesive pure functions into one capability is as important as splitting apart the ones that don't belong together. Don't default to "one function, one capability"; default to "one *responsibility*, one capability," which sometimes means several small functions and sometimes means one.

## Step 2: the secrets and I/O exclusion — non-negotiable

Before proposing any function as extractable, check it for:

- Credential-shaped variable names or literals (`api_key`, `apikey`, `secret`, `token`, `password`, `auth`, connection strings with embedded credentials).
- Imports of an HTTP client, database driver, filesystem module, or cloud SDK.
- Reads of environment variables or process/OS state (`os.environ`, `process.env`, `std::env`, config file loading).
- Non-determinism sources: current time, random number generation, thread/process IDs.

If a candidate function touches any of these, **don't propose extracting it as-is.** Two honest paths forward, and the choice matters:
1. If the I/O/secret-touching part is separable from a genuine pure-logic core (e.g. "fetch the record, then compute a score from its fields"), extract only the pure core and flag the fetch as something that stays in the calling application, feeding its result to the capability as plain input.
2. If the function is *inherently* about calling an external, authenticated service, it's not a capability candidate today — Traverse's WASM sandbox doesn't grant network access to any capability under the current ABI whitelist, key or no key (see `traverse-capability-author`'s secrets-check section and `references/scanning-heuristic.md`'s exclusion list for the full reasoning). Say this plainly rather than proposing an extraction that can't actually work.

## Step 3: propose the decomposition

Present the candidate set as a genuine decomposition, not a raw function dump — group related pure functions into proposed atomic capabilities, name each with a single verb, and note which parts of the original code stay behind as orchestration (unextracted, remaining in the source app). This is the same atomicity principle `traverse-capability-author` applies when authoring from scratch; here it's applied to what already exists rather than to a fresh description. Full scanning heuristic and worked walkthrough in `references/scanning-heuristic.md`.

## Step 4: verify the extraction actually reproduces the original behavior

This is the step that's easy to skip and matters most. Extracting logic isn't copy-pasting a function signature — for a non-Rust source (Traverse capabilities compile to Rust → WASM today), it means *re-implementing* the logic in Rust, and a mechanical port is not guaranteed correct. Before treating an extraction as done:

1. Collect a handful of representative inputs and run them through the **original** code, recording the outputs.
2. Run the same inputs through the **ported** Rust logic (as a plain host-target unit test, before any WASM compilation — see `traverse-capability-author`'s `references/testing-pattern.md`).
3. Diff the two. Any mismatch means the port isn't done, not that the difference is close enough.

For a same-language (Rust) source, extraction can often be close to a direct copy of the pure functions (swap `std` collection types for the no_std shim's equivalents where needed) — the parity check is still worth running, but the risk of behavioral drift is much lower than a cross-language port. Full detail in `references/parity-verification.md`.

## Step 5: hand off to traverse-capability-author

Once you have a verified-equivalent, decomposed set of candidate capabilities, stop here. Hand each one to `traverse-capability-author` with: the proposed capability id/verb, the ported Rust logic (already parity-checked), and the input/output shape observed from the original code. That skill owns writing the contract, clearing the ABI whitelist, deciding public vs. private, and publishing — don't duplicate any of that here.

## Where to go deeper

- `references/scanning-heuristic.md` — the full candidate-scanning checklist and the worked `doc-approval-analyze` example in detail.
- `references/parity-verification.md` — how to structure an original-vs-ported comparison, including the cross-language case.

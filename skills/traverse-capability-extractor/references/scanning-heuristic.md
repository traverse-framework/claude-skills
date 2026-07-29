# Scanning a repo for capability candidates

## The checklist, applied per function/module

**Include (candidate for extraction):**
- Output determined entirely by explicit arguments (no hidden inputs).
- No network, filesystem, database, or environment access.
- No randomness or wall-clock dependency.
- Input/output shapes that map cleanly to JSON: strings, numbers, booleans, arrays, and structured records of the same. Not: open handles, connections, framework request/response objects, class instances with internal mutable state.

**Exclude, or split first:**
- Anything importing an HTTP client, database driver/ORM, filesystem module, or cloud SDK.
- Anything reading environment variables, config files, or secrets stores.
- Credential-shaped names in scope: `api_key`, `apikey`, `secret`, `token`, `password`, `auth_header`, connection strings.
- Functions mixing fetch/transform/notify in one block — split into "the transform" (candidate) and "the fetch/notify" (stays as app orchestration, or becomes a workflow node around the extracted capability, per `traverse-workflow-planner`).

## Grouping into atomic capabilities

Once you have a set of pure-function candidates, don't default to a 1:1 function-to-capability mapping. Ask: do these functions only make sense together, composing into one meaningful output? If so, they're one capability. Do they represent genuinely separate concerns someone might want to reuse independently? If so, split them.

### Worked example: `doc-approval-analyze`

Before extraction, this logic existed as five plain functions in one file with no capability wrapper:

```rust
fn classify_doc_type(document: &str) -> &'static str { ... }
fn extract_parties(document: &str) -> Vec<String> { ... }
fn extract_amounts(document: &str) -> Vec<String> { ... }
fn determine_confidence(doc_type: &str, parties: &[String], amounts: &[String]) -> &'static str { ... }
fn determine_recommendation(confidence: &str) -> &'static str { ... }
```

Scanning checklist applied:
- No I/O in any of the five — all pure string/slice transformations. **All five pass the exclusion checklist.**
- `classify_doc_type` and `extract_parties`/`extract_amounts` each take the same `document: &str` and could theoretically be split into three separate capabilities (`classify`, `extract-parties`, `extract-amounts`).
- But `determine_confidence` and `determine_recommendation` each depend on the *outputs* of the other three — they only make sense chained together, computing one thing: "given a document, what's the analysis." Splitting them into five capabilities would force a caller to wire five nodes together every single time, for logic that's never meaningfully used independently.

**Decision: one capability** (`doc-approval.analyze`), with the five functions becoming one `analyze(input: Value) -> Value` entry point that calls them internally in sequence. This is the actual, real decomposition used when this code was published — verified working, not a hypothetical.

Contrast: if the repo *also* had a wholly separate `format_currency(amount: &str, locale: &str) -> String` helper used elsewhere for unrelated purposes, that would be a second, independent capability candidate — it doesn't compose with the document-analysis functions, and something else in the codebase might want to reuse it on its own.

## What to do with what's excluded

Don't silently drop excluded code from the proposal. List it explicitly as "stays in the calling application" (for I/O/orchestration) or "not extractable under the current ABI whitelist" (for anything needing network/credentials) — the person you're working with should see the full picture, not just the parts that made the cut.

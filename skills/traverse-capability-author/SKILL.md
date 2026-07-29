---
name: traverse-capability-author
description: Interviews a person about a new Traverse capability they want (or an existing one they want to extend), one question at a time, then authors it the real way — check both the public and any configured private registry first to avoid duplicating something that already exists, decide whether it's genuinely reusable or truly project-specific, decide public vs. private publication, write a JSON-Schema-backed contract, implement it as genuinely executable ABI-compliant WASM, and publish it via a human-reviewed PR. Refuses to embed secrets or credentials in capability source. Use this skill whenever the user wants to create a new Traverse capability, extend an existing one, describes a business requirement they want turned into a capability, asks "does something like this already exist" before building, or wants to publish a capability privately vs. publicly. This is the authoring half of a three-skill split from the retired traverse-app-builder skill — see traverse-capability-extractor for pulling capabilities out of an existing codebase, and traverse-workflow-planner for going from a business workflow to a list of capabilities to build.
---

# Authoring or extending a Traverse capability

A Traverse capability is a contract (what has to be true before it runs, what it promises after, what it's allowed to touch) plus a WASM binary that actually enforces that contract. The two most common ways to get this wrong: writing the binary first and backfilling a contract that just describes whatever the code happens to do (the contract stops meaning anything), or building a new capability that already exists somewhere reachable (duplication that fragments what should be one reusable piece of business logic).

This skill exists to structurally prevent both. It runs as a short interview — one question at a time, real tradeoffs, a recommendation — before any code gets written, and it won't let you skip the registry-duplication check or slip a credential into capability source.

## Step 0: the intake interview

Don't start with "write the contract." Start by understanding what's actually being asked for, one question at a time (mirror the `brainstorm` skill's own discipline: one question, 2-4 real options with honest tradeoffs, a recommendation, then let the answer decide what's asked next — don't bundle two decisions into one turn). At minimum, work through these in order, skipping only what the user has already told you unprompted:

1. **What does it do, in one verb?** If the answer has more than one verb in it ("validate and send," "fetch and summarize"), that's a signal this is two capabilities plus a workflow, not one capability — hand this off to `traverse-workflow-planner` instead of continuing here, since decomposing a multi-step use case into the right set of atomic pieces is that skill's job, not this one's.
2. **Does something like this already exist?** This is not optional and not a formality — see Step 1 below. Only proceed to authoring once you've actually checked and confirmed nothing covers it.
3. **Is this genuinely reusable, or specific to one project?** A capability that hardcodes one app's field names, one team's business rules, or a one-off integration isn't a capability someone else can discover and reuse — it's app logic that happens to be shaped like one. If it's genuinely project-specific, say so plainly; that's a legitimate answer, but it changes the visibility decision in the next question (project-specific work is a much stronger candidate for private-only publication, or for staying inside the app's own bundle rather than being published anywhere).
4. **Public or private?** See "Public vs. private" below — this is a real decision with real tradeoffs, not a rubber stamp.
5. **Does it need a secret, credential, or API key to do its job?** See "The secrets check" below. Ask this explicitly rather than discovering it while writing the Rust.

Only after these are answered does authoring start.

## Step 1: check both registries before writing anything

Don't scaffold a capability that already exists. There are two independent things to check, and depending on Step 0's private-registry decision you may need both:

```bash
traverse-cli registry sync --workspace <workspace-id> --json
cat .traverse/workspaces/<workspace-id>/registry/public/index.json
```

That syncs and reads the **public** registry (`traverse-framework/registry` by default). If the person you're interviewing has a private registry configured (their own fork/clone of the same layout — see `references/registry.md` for how that works), check it too:

```bash
traverse-cli registry sync --workspace <workspace-id> --json --source-repo <owner/private-registry>
cat .traverse/workspaces/<workspace-id>/registry/public/index.json
```

Each entry in either index is a thin pointer (id, version, digests, `artifact_url`, `contract_url`) — there's no description field in the index itself, so fetch `contract_url` for any plausible match and actually read what it does before ruling it in or out. If you're checking what's already declared inside a specific app bundle rather than a wider registry, use `traverse-cli capability discover <bundle-manifest> --json` instead — full detail in `references/registry.md`.

**Only move on to Step 2 once you've confirmed nothing existing covers this.** If something close exists but doesn't quite fit, that's worth surfacing to the user as an explicit choice (extend the existing one to a new version vs. author a distinct new capability) rather than silently picking one.

## Step 2: public vs. private

| | Public (`traverse-framework/registry`) | Private (a configured alternate source) |
|---|---|---|
| Who can discover it | Anyone syncing the public registry | Only workspaces configured with `--source-repo <owner/repo>` pointed at that source |
| How publishing differs | `capability publish` with no override flags | `capability publish --registry-repo-remote <owner/repo>`, targeting a local checkout of that repo |
| When it's the right call | Genuinely reusable, no project-specific assumptions baked in, safe to be world-readable | Project-specific logic multiple *internal* teams/apps should still share, or reusable logic not ready/appropriate for public release |

The private-registry mechanism is real, not aspirational: any GitHub repo (a team's own fork or clone) that adopts the same `capabilities/<namespace>/<id>/<version>/contract.json` layout and index-build CI as the public registry is a valid sync/publish source — `traverse-cli` just needs `--source-repo`/`--registry-repo-remote` pointed at it (and `TRAVERSE_REGISTRY_TOKEN` in the environment if that repo is private). It is **not** a hosted multi-tenant service — a team stands up their own repo. If the person you're interviewing doesn't have one yet and wants one, that's a separate, one-time setup step (forking/cloning the layout), not something this skill does per-capability.

Don't let "private" become a default dumping ground for things nobody wants to think hard about — ask the question in Step 0 for real, and push back gently if "just make it private" is being used to avoid the reusability conversation.

## Step 3: the secrets check

Never let a capability's source embed a credential, API key, password, or token — not as a string literal, not as a "just for now, we'll rotate it" placeholder, not even in a comment. If the requirement as described needs one (e.g. "call this third-party service with my API key"), stop and flag it rather than writing the key into `src/main.rs`:

- Under today's real ABI whitelist (see `references/troubleshooting.md` entry 11), a capability's WASM binary can only import `wasi_snapshot_preview1::{fd_read, fd_write, proc_exit}` plus a couple of `traverse_host` metadata calls — no network sockets, no host environment access. A capability that needs to call an external API with a credential **can't do that from inside the WASM sandbox today at all**, key or no key. If that's what's being asked for, say so plainly: this isn't a "don't embed the secret" style fix, it's a "this isn't a capability today, it's something the *calling application* has to do outside the sandbox, potentially handing the capability only the already-fetched, already-authenticated data as plain input." That's a real, current platform limitation, not this skill being overly cautious.
- If the need is narrower — e.g. the capability's *logic* needs a threshold, endpoint name, or non-secret parameter that varies per team — that's a configuration question, not a secrets question. See "Configuring a capability per project/team" below.

Treat this as a hard stop, not a lint warning: don't generate Rust that reads a hardcoded key, and don't generate a contract whose `execution.constraints.network_access` is `"required"` without first confirming with the user that they understand today's ABI whitelist makes that combination non-functional in practice (the contract schema allows declaring it; the runtime won't let a capability actually use it — see `references/platforms.md` and `references/troubleshooting.md` entry 11).

## Step 4: write the contract

Copy the shape from `references/contract-schema.md` — struct-verified against `traverse-contracts`, not guessed. The field most worth deliberately setting rather than defaulting: `lifecycle`. Set it to `"active"` once you intend to actually run it, even during local iteration — `"draft"` is correctly rejected by the runtime as not-yet-runnable.

Decide `permitted_targets` honestly rather than accepting the schema's six-target default (a governance-level "eventually allowed" claim, not a technical one) — check `references/platforms.md` for what's actually shipped today and only claim what's true.

## Step 5: implement it — genuinely, and so it actually runs

This is where most of the real risk lives, and it's the part most tutorials get wrong. Two things have to both be true:

1. **The logic has to genuinely depend on its input.** A capability whose output is a fixed string regardless of input isn't a capability, it's a fixture pretending to be one — this has bitten real published capabilities (see the "genuinely input-dependent" test pattern in `references/testing-pattern.md`).
2. **The binary has to clear the ABI whitelist.** An ordinary `cargo build --target wasm32-wasip1` binary — even `fn main() { println!("hi"); }` — imports `wasi_snapshot_preview1::environ_get` as part of Rust's standard WASI startup and fails `wasm abi verify`. This isn't your code's fault; it's `std`'s runtime init pulling in more than the whitelist allows, unconditionally.

Both are solved together with `#![no_std]` + hand-declared `extern "C"` imports for exactly `fd_read`/`fd_write`/`proc_exit`, compiled with plain `rustc`/`cargo` against `wasm32-unknown-unknown` (not `wasm32-wasip1`, not a `std`-linked build). The exact working pattern, plus a reusable shared runtime shim that gives you real JSON parsing/writing and a bump allocator *without* pulling in libstd's WASI startup, is in `references/wasm-implementation.md` — use it rather than reinventing the shim per capability.

### Configuring a capability per project/team

There is no dedicated "capability config file" mechanism in Traverse today — don't imply one exists. The real, verified way a capability stays reusable across projects with different needs is: **make the variable part an explicit, named field in the capability's own `inputs.schema`**, supplied by the caller at invocation time (an app's static manifest value, or a workflow node's `input.from_workflow_input` wiring — see `references/contract-schema.md`) rather than hardcoded in the WASM binary. A threshold, a category list, a formatting choice — anything a different team might reasonably want to vary — belongs in the input contract as an optional field with a sensible default behavior when omitted, not baked into the compiled artifact. This keeps exactly one binary serving every team, which is the actual point of "configurable" here: configuration lives at the calling layer, the capability itself stays pure business logic over whatever input (including its config fields) it's handed.

## Step 6: verify and publish

Verify the ABI whitelist directly (don't skip this even though `cargo test` passing feels like enough — a std-linked build passes ordinary tests and only fails at real execution):

```bash
wasm2wat artifacts/your-capability.wasm | grep import
# must show only wasi_snapshot_preview1::{fd_read, fd_write, proc_exit}
```

Then publish — to the public registry, or, with the Step 2 decision in hand, to a private one:

```bash
# Public
traverse-cli capability publish --contract path/to/contract.json --artifact path/to/artifact.wasm \
  --registry-repo path/to/local/registry-checkout --json --dry-run   # then again without --dry-run

# Private
traverse-cli capability publish --contract path/to/contract.json --artifact path/to/artifact.wasm \
  --registry-repo path/to/local/private-registry-checkout --registry-repo-remote <owner/private-registry> \
  --json --dry-run   # then again without --dry-run
```

This never publishes directly — it opens a human-reviewed PR. Always dry-run first. Full detail, including how the local checkout path and the `--registry-repo-remote` slug relate, is in `references/registry.md`.

## Where to go deeper

- `references/contract-schema.md` — full, struct-verified contract/workflow/manifest/request JSON shapes.
- `references/registry.md` — checking and publishing to both public and private registries.
- `references/wasm-implementation.md` — the `#![no_std]` pattern, the shared JSON runtime shim, and the honest costs of this approach.
- `references/platforms.md` — what's actually shipped per target today; check before promising a capability runs somewhere specific.
- `references/troubleshooting.md` — every real error this skill's own testing hit, verbatim, with the actual fix.
- `references/testing-pattern.md` — how to prove a capability's output genuinely depends on its input before publishing it.

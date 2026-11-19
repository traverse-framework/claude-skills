---
name: traverse-workflow-planner
description: Takes a business workflow or use case described from a product/business-logic perspective, decomposes it into the individual atomic Traverse capabilities it needs, checks the public and any configured private registry for what already exists, produces a gap list of what still needs to be built, emits a draft structured workflow description (the target plus starting facts that `traverse-cli workflow plan` accepts) with an explicit list of open decisions for the user to settle, routes each gap to the right skill (traverse-capability-author for net-new logic, traverse-capability-extractor if the logic plausibly already exists unextracted in a known repo), and once every piece exists, drafts the multi-node workflow that composes them. Use this skill when the user describes an app, feature, or business process in terms of what it should accomplish rather than naming a specific capability, asks "what capabilities do I need to build this", wants to compose several capabilities into one workflow, or is planning a Traverse app from a use case down rather than from one capability up. This is the planning half of a three-skill split from the retired traverse-app-builder skill — see traverse-capability-author for authoring one capability and traverse-capability-extractor for pulling one out of existing code.
---

# From a business workflow to a capability build list

The other two skills in this set both start from something already decided — "author this one capability" or "extract from this one repo." This skill starts earlier: from a business description with no capabilities named yet, and its job is to work out what capabilities the workflow actually needs, which of those already exist, and who should build the rest.

## Step 1: decompose the use case into atomic operations

Take the business description and break it into individual verbs — the same atomicity standard `traverse-capability-author` applies when authoring ("one verb, one responsibility"), just applied in the opposite direction: instead of checking whether a single already-decided capability is too broad, you're deriving the list of single-verb operations a multi-step process actually requires.

A useful test: read the description back as a sequence of steps connected by "and then." Each step is a candidate atomic capability. "Capture the customer's request, validate it against inventory, and generate a quote" is three steps — capture, validate, generate-quote — not one. If a described step still has an "and" hiding inside it ("validate and log"), split it further until each step is genuinely one verb.

Don't over-split either: a step like "look up the current tax rate for this jurisdiction" is one verb (look-up) even though internally it might involve several sub-computations — those sub-computations aren't separately reusable the way the top-level operations are (this is the same "cohesive functions that only make sense together are one capability" judgment `traverse-capability-extractor` applies when grouping extracted functions).

## Step 2: check both registries for each candidate

For every capability identified in Step 1, check whether it already exists before assuming it needs to be built — this is the same registry-discovery mechanism `traverse-capability-author` uses, just run once per candidate instead of once for a single capability:

```bash
traverse-cli registry sync --workspace <workspace-id> --json
cat .traverse/workspaces/<workspace-id>/registry/public/index.json
```

If a private registry is configured for this team, check it too (`--source-repo <owner/private-registry>`, separate `--workspace` id — see `traverse-capability-author`'s `references/registry.md` for the full mechanics of syncing two sources). Each match found removes one item from the gap list; each match should still be spot-checked by fetching its `contract_url` to confirm it actually does what the workflow step needs, not just that the name sounds right.

## Step 3: produce the gap list

For every candidate from Step 1 not covered by Step 2, list it explicitly: proposed id/verb, one-sentence description of what it needs to do, and its rough input/output shape as understood from the business description. This gap list is the actual deliverable of this skill for anything that isn't already fully covered by existing capabilities — present it before proceeding to build anything, so the person you're working with can confirm the decomposition makes sense before work starts on each piece.

## Step 3b: emit the structured workflow draft and the open decisions

Alongside the gap list, turn the description into the **structured workflow description** `traverse-cli workflow plan` accepts: a target (`--target-capability <id@version>` or `--target-event <event_type>`) plus a `starting-facts.json` object whose properties match the first capability's `inputs.schema` (Traverse Decision 137 rule 1). `references/workflow-plan-draft.md` has the exact format, the output contract and a verified end-to-end example. Its rules are not optional:

- **Label it a draft.** Put the draft label from the reference at the top. The draft executes nothing and publishes nothing; it's a suggestion the user confirms (Decision 137 rules 1–2).
- **List every open decision, and resolve none.** Ambiguous steps, the choice of target, any fact whose name or type you had to guess, every placeholder value, and every missing capability from the gap list each go on a numbered list for the user. If you notice you're about to pick between two plausible readings, that choice is an open decision.
- **Say where the prompt went.** Prompt handling is the policy of the user's own AI tool, and Traverse hosts no model (rule 3). Write the draft only as files in the user's project, where nothing is kept unless they save it (rule 4).
- **Don't promise candidates.** Show the `workflow plan` command, and be explicit that the planner only chains steps where one upstream capability produces all of the next step's required inputs. An empty `"candidates": []` is an open decision to bring back to the user, not something to patch by inventing mappings.

## Step 4: route each gap — explicitly, not by default

For each item on the gap list, decide and state which of the other two skills should build it:

- **`traverse-capability-author`** (the default) — when the logic is genuinely new, or when nothing in a known codebase already implements it.
- **`traverse-capability-extractor`** — when the person you're working with has mentioned, or you have reason to believe, that a specific existing repo (their own codebase, an internal library, an open-source project) already implements this logic in some form, even if it's not currently a capability. Ask directly rather than guessing: "does anything in your codebase already do X?" before defaulting to build-from-scratch.

Don't silently default every gap to "author from scratch" — that's the whole point of having a routing step here rather than just handing the entire gap list to one skill. State the routing decision for each gap so it's visible, not assumed.

## Step 5: once every piece exists, compose the workflow

A workflow isn't limited to one node — `references/workflow-composition.md` has the full multi-node shape, a real worked example (`plan-expedition`, which chains five narrowly-scoped capabilities: capture → interpret → assess → validate → assemble), and a naming gotcha worth knowing before you write one (`from_workflow_input` reads the *accumulated workflow state*, not just the workflow's original top-level input — a downstream node routinely reads a field only an upstream node produced).

**Running a composed workflow, honestly:** `workflow register`/`workflow list`/`workflow inspect` are real, generic CLI functionality — registering and inspecting any workflow you throw at them. There is no bare `traverse-cli workflow execute` command today; the generic way to actually run one end-to-end is `traverse-cli serve`'s composite-capability trick (registering the workflow as a capability whose `implementation_kind` is `workflow`). Every node's binary still has to individually clear the ABI whitelist the same way a standalone capability does — see `traverse-capability-author`'s `references/wasm-implementation.md`. Full detail on both the composition shape and the execution honesty in `references/workflow-composition.md`.

## Where to go deeper

- `references/workflow-plan-draft.md`: the `workflow plan` input format, the draft-and-open-decisions output contract, a verified plain-English → draft → candidates example, and the planner's current limits.

- `references/workflow-composition.md` — the full multi-node workflow shape, the `plan-expedition` worked example, the `from_workflow_input` gotcha, and the real (and not-yet-real) ways to execute a composed workflow.

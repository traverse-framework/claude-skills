# The structured workflow draft and `traverse-cli workflow plan`

Traverse Decision 137 makes the input of any authoring journey the
**structured** workflow description that `traverse-cli workflow plan`
accepts. Plain English is only a way to produce a draft of that structure,
which the user confirms. This file holds the exact format, a hands-on
verified example, and the limits of what the planner does today.

Verified against `traverse-framework/traverse` main at `ab0dc122`
(2026-11-19), with `traverse-cli` built from that commit.

## The input format

`workflow plan` takes four things. Only two of them come from the plain-English
description; the other two are files the user already has.

```bash
traverse-cli workflow plan \
  --app-manifest <path>          # the user's app.manifest.json (existing file)
  --registry-bundle <path>       # registry bundle covering the app's components (existing file)
  --starting-facts <path>        # JSON object: the facts known before any step runs  ← drafted
  (--target-capability <id@version> | --target-event <event_type>)                   ← drafted
  [--workspace-id <id>]
```

So the **structured workflow description** the skill drafts is:

- **Target**: exactly one of
  - `--target-capability <capability_id>@<version>`, the capability whose
    output the workflow must produce, or
  - `--target-event <event_type>`, the governed event the workflow must emit.
- **Starting facts**: one JSON object (at most 64 KiB in the browser planner)
  whose top-level properties are the facts available before the first step.
  Property names and JSON types must match the `inputs.schema` of the first
  capability. The planner matches structurally; it never reads names as hints.

It is never natural language. `workflow plan` refuses open-ended goals
(Spec 113 FR-001), so the draft has to name a real capability or event from
the registry check in Step 2.

## Output contract for the skill

Emit these three blocks together, in this order:

1. **Label.** Exactly this text, so nobody mistakes the draft for an approved
   plan:

   > DRAFT, untrusted. Generated from your description by your AI tool. It
   > executes nothing and publishes nothing. Confirm every field and every
   > open decision below before running `traverse-cli workflow plan` on it.

2. **The draft**:
   - `starting-facts.json`: the JSON object, using placeholder values only
     where the description gives no value (and list each placeholder as an
     open decision);
   - the target flag (`--target-capability …` or `--target-event …`);
   - the full `traverse-cli workflow plan …` command, with the app manifest
     and registry bundle paths the user gave (or `<path>` placeholders).

3. **Open decisions**: a numbered list of everything the user must settle.
   Never resolve any of them yourself. Include at least:
   - **Ambiguous steps**: any sentence in the description that could map to
     more than one atomic step, or to none.
   - **Target choice**: when more than one capability or event could be the
     target.
   - **Mapping choices**: any fact whose name or type does not match exactly
     what the first capability's `inputs.schema` requires, and any field you
     had to guess.
   - **Placeholders**: each starting-fact value you invented.
   - **Missing capabilities**: every gap-list item from Step 3. The planner
     can't chain through a capability that doesn't exist yet.

Then add two plain statements:

- Prompt handling is the policy of the user's own AI tool. Traverse hosts no
  model, and the planner only ever receives the structured draft (Decision
  137 rule 3).
- The skill writes nothing outside the user's project. The draft files are
  ordinary files the user can save or delete, and nothing is kept unless the
  user saves it (Decision 137 rule 4).

## Verified example, end to end

The `expedition-readiness` example app and `expedition` registry bundle in the
traverse repo.

**1. Plain English (the user's words):**

> Plan a guided, low-risk summit of Mount Rainier via Camp Muir for the
> weekend of July 10, 2027. Safety comes first. We're a two-person team and
> one of us is a first-timer.

**2. Structured draft the skill emits.** The target is
`expedition.planning.capture-expedition-objective@1.0.0`. Its
`inputs.schema.required` is `destination`, `target_window`, `preferences` and
`notes`, so `starting-facts.json` is:

```json
{
  "destination": "Mount Rainier, Camp Muir route",
  "target_window": { "start": "2027-07-10T06:00:00Z", "end": "2027-07-11T18:00:00Z" },
  "preferences": { "style": "guided", "risk_tolerance": "low", "priority": "safety" },
  "notes": "Two-person team, one first-timer."
}
```

The open decisions for this draft:

1. The window start and end times (06:00Z to 18:00Z) are placeholders. The
   description says only "the weekend of July 10".
2. `preferences.priority` is `"safety"` and `risk_tolerance` is `"low"`.
   Confirm that these strings are the values your capability expects.
3. The target is only the first step. A full plan to
   `assemble-expedition-plan` returns no candidates today (see Limits). Do you
   want the single-step plan, or should we first fix the chain?

**3. Run the planner (from the traverse repo root):**

```bash
traverse-cli workflow plan \
  --app-manifest examples/applications/expedition-readiness/app.manifest.json \
  --registry-bundle examples/expedition/registry-bundle/manifest.json \
  --starting-facts starting-facts.json \
  --target-capability expedition.planning.capture-expedition-objective@1.0.0
```

**4. Result (abridged).** There is one candidate, with one node
(`expedition.planning.capture-expedition-objective@1.0.0`) and four
`initial_input` mappings (`/destination`, `/target_window`, `/preferences`,
`/notes`). The output also carries `"mapping_unconfirmed": true` and
`"plan_search_truncated": false`. Each candidate is
`{ "proposal": { kind, schema_version, proposal_id, workspace_id, app_manifest,
nodes, edges, mappings, initial_input }, "mapping_unconfirmed": true }`.

The candidate is still untrusted. `mapping_unconfirmed` is never cleared by
the planner (Spec 113 FR-005). A human reviews it, then
`traverse-cli workflow promote finalize` turns it into a workflow definition
(Decision 80).

## Limits observed today (do not overclaim)

- **Empty or mismatched starting facts produce no candidates.** With `{}` as
  starting facts, the same command returns `"candidates": []`. Report that as
  an open decision ("which facts do you have?"), not as a planner error.
- **Chains need a single producer per step.** Spec 113 FR-002 selects a
  producer → consumer pair only when **one** producer's `outputs.schema` has
  every property in the consumer's `inputs.schema.required`, with matching
  JSON types. In the expedition bundle, `interpret-expedition-intent` needs
  `objective` and `planning_intent`, but `capture-expedition-objective` only
  produces `objective`. So targeting `interpret-expedition-intent`,
  `assess-conditions-summary`, `validate-team-readiness` or
  `assemble-expedition-plan` returned zero candidates in the verified run,
  even with `planning_intent` and `team_profile` added to the starting facts.
  The hand-written `plan-expedition` workflow in
  `workflow-composition.md` chains these steps by mapping fields itself,
  which the planner does not do.
- When the planner returns no candidates, show the user which required inputs
  no upstream step produces. Offer two choices as open decisions: a
  hand-written workflow (Step 5), or a new or changed capability (route it in
  Step 4). Never invent mappings to fill the gap.
- The planner returns at most 5 candidates of at most 8 nodes, and sets
  `plan_search_truncated` when it hits either bound (FR-004). When there are
  several candidates, list them all; never pick one (FR-003).

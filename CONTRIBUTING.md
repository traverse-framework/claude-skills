# Contributing

Thanks for contributing to Traverse's Claude Skills.

## Before You Start

Please read:

- [traverse-framework/.github](https://github.com/traverse-framework/.github) — constitution, quality standards, and the AI-agent hardening rules this repo has adopted
- [README.md](README.md)'s "A note on accuracy" section — the standard every skill here is held to

## Adding or Updating a Skill

- Every claim a skill makes about what Traverse can or can't do today must be verified against the current `traverse-framework/traverse` source or a hands-on repro — not copied from other docs, and not assumed from what a spec says should eventually be true.
- If a skill documents a platform limitation, link the tracking issue or Project item if one exists.
- New skills should include an `evals/evals.json` with a few representative test prompts, per the skill-authoring workflow (draft, test, iterate) rather than shipping untested.

## Pull Requests

Every pull request should:

- reference the relevant issue or Project item, if one exists
- say what was hands-on verified vs. what's carried over from existing docs
- flag anything that changed in Traverse's real behavior since the skill was last updated

[![Traverse](https://github.com/user-attachments/assets/479aa1e5-5799-4c7d-bb8a-4b30d711c7db)](https://traverse-framework.com/)

# Traverse Claude Skills

The only purpose of this repo is to host [Claude Skills](https://www.anthropic.com/news/skills) for building apps with [Traverse](https://github.com/traverse-framework/traverse) — the contract-driven WASM runtime. It's a distribution point, not a place for runtime code, capability contracts, or reference apps; those live in [`traverse`](https://github.com/traverse-framework/traverse), [`registry`](https://github.com/traverse-framework/registry), and [`reference-apps`](https://github.com/traverse-framework/reference-apps) respectively.

## Skills

| Skill | What it does |
|---|---|
| [`traverse-capability-author`](skills/traverse-capability-author/) | Interviews you about a new capability (or an extension to one), one question at a time — checks the public and any private registry to avoid duplication, decides public vs. private and generic vs. project-specific, then authors a contract and genuinely executable, ABI-compliant WASM implementation, and publishes it via a human-reviewed PR. Refuses to embed secrets. |
| [`traverse-capability-extractor`](skills/traverse-capability-extractor/) | Scans an existing repo (open-source or your own) for business logic that's a good capability candidate, proposes an atomic decomposition, verifies the extraction reproduces the original behavior, and hands off to `traverse-capability-author` for the actual authoring/publish. |
| [`traverse-workflow-planner`](skills/traverse-workflow-planner/) | Takes a business workflow or use case, decomposes it into the atomic capabilities it needs, checks both registries for what already exists, produces a gap list, routes each gap to `traverse-capability-author` or `traverse-capability-extractor`, and composes the finished pieces into a multi-node workflow. |

These three replace the earlier `traverse-app-builder` skill, split along the natural seams that emerged from real use: authoring one capability, extracting one from existing code, and planning a whole workflow's worth of capabilities are different enough problems that one skill covering all of them was getting overloaded. All of `traverse-app-builder`'s verified content (the ABI whitelist gotcha, the `no_std` implementation pattern, the toolchain trap, the registry mechanics) carried forward into whichever of the three it actually belonged to — nothing was dropped, and the three are designed to hand off to each other rather than duplicate one another's mechanics.

## Install a skill

Copy the skill directory into your own Claude Skills location:

```bash
git clone https://github.com/traverse-framework/claude-skills.git
cp -R claude-skills/skills/traverse-capability-author ~/.claude/skills/traverse-capability-author
```

Claude will pick it up automatically the next time its description matches what you're asking for. If you want the full set (recommended — they're designed to hand off to each other), copy all three:

```bash
for skill in traverse-capability-author traverse-capability-extractor traverse-workflow-planner; do
  cp -R "claude-skills/skills/$skill" "~/.claude/skills/$skill"
done
```

## A note on accuracy

Every skill in this repo is meant to be verified against the real, current state of Traverse — not written from the docs or from what the framework is supposed to do eventually. Where a skill documents a limitation (a hardcoded execution allowlist, an ABI constraint, a missing CLI command), that's something that was hands-on reproduced, not assumed. If the underlying platform changes and a skill goes stale, please open an issue or a PR rather than letting it silently drift out of sync — a skill that overclaims what Traverse can do today is worse than no skill at all.

## Governance

This repo follows the shared governance model defined in [`traverse-framework/.github`](https://github.com/traverse-framework/.github). See `CONTRIBUTING.md` before opening a PR.

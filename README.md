[![Traverse](https://github.com/user-attachments/assets/479aa1e5-5799-4c7d-bb8a-4b30d711c7db)](https://traverse-framework.com/)

# Traverse Claude Skills

The only purpose of this repo is to host [Claude Skills](https://www.anthropic.com/news/skills) for building apps with [Traverse](https://github.com/traverse-framework/traverse) — the contract-driven WASM runtime. It's a distribution point, not a place for runtime code, capability contracts, or reference apps; those live in [`traverse`](https://github.com/traverse-framework/traverse), [`registry`](https://github.com/traverse-framework/registry), and [`reference-apps`](https://github.com/traverse-framework/reference-apps) respectively.

## Skills

| Skill | What it does |
|---|---|
| [`traverse-app-builder`](skills/traverse-app-builder/) | Check the registry for capabilities that already exist, author new atomic capabilities as contracts, compile them to WASM, compose several into a multi-node workflow, and validate everything with `traverse-cli` — the real, verified-working path, including the platform's current honest limitations. |

## Install a skill

Copy the skill directory into your own Claude Skills location:

```bash
git clone https://github.com/traverse-framework/claude-skills.git
cp -R claude-skills/skills/traverse-app-builder ~/.claude/skills/traverse-app-builder
```

Claude will pick it up automatically the next time its description matches what you're asking for.

## A note on accuracy

Every skill in this repo is meant to be verified against the real, current state of Traverse — not written from the docs or from what the framework is supposed to do eventually. Where a skill documents a limitation (a hardcoded execution allowlist, an ABI constraint, a missing CLI command), that's something that was hands-on reproduced, not assumed. If the underlying platform changes and a skill goes stale, please open an issue or a PR rather than letting it silently drift out of sync — a skill that overclaims what Traverse can do today is worse than no skill at all.

## Governance

This repo follows the shared governance model defined in [`traverse-framework/.github`](https://github.com/traverse-framework/.github). See `CONTRIBUTING.md` before opening a PR.

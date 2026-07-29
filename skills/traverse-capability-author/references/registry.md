# Checking and publishing to public and private registries

Traverse's registry is the mechanism for reuse: a capability someone else already wrote, contracted, and published should be something you can find and call, not something you re-implement. The commands below are real and verified against `crates/traverse-cli/src/main.rs` and `crates/traverse-registry/src/public_registry_state.rs` — including the `--source-repo`/`--registry-repo-remote`/`TRAVERSE_REGISTRY_TOKEN` private-registry flags, which landed in `traverse-framework/traverse#843`.

## Two different registries, two different commands

Don't conflate these — they answer different questions:

1. **"What's published, across a registry source?"** → `registry sync` + reading the synced index. Works identically for the public registry and any configured private one; the only difference is the `--source-repo` flag.
2. **"What capabilities did I already declare inside this specific app bundle I'm assembling?"** → `capability discover`.

Neither is a live network search. Both work against local state — the runtime and CLI never live-fetch during normal use, which is a deliberate determinism choice, not a limitation to work around.

## 1. Sync and read a registry index

Public (the default; matches today's unauthenticated behavior exactly if you omit `--source-repo`):

```bash
traverse-cli registry sync --workspace <workspace-id> --json
cat .traverse/workspaces/<workspace-id>/registry/public/index.json
```

Private — any GitHub repo (a team's own fork or clone) that adopts the same `capabilities/<namespace>/<id>/<version>/contract.json` layout and index-build CI as the public registry is a valid source:

```bash
# public source repo, or a private one your account has read access to
traverse-cli registry sync --workspace <workspace-id> --json --source-repo <owner/repo>

# if that repo is private, first:
export TRAVERSE_REGISTRY_TOKEN=<a token with read access to owner/repo>
```

Each sync writes to the same local state file regardless of source (`.traverse/workspaces/<workspace-id>/registry/public/index.json`) — syncing a second source overwrites the first workspace's synced state, it doesn't merge the two. If you need to check both a public and a private source, use two different `--workspace` ids, one per source, and sync each independently.

The command's own JSON output is intentionally thin — `status`, `source`, `release_tag`, `record_count`, `synced_at`, `state_path` — it tells you *that* the sync worked and *how many* records came down, not what they are. Read the state file it wrote to actually see what's there.

Each record has this shape:

```json
{
  "namespace": "some-namespace",
  "id": "some-namespace.some-capability",
  "version": "1.0.0",
  "digest": "sha256:...",
  "artifact_url": "https://.../artifact.wasm",
  "contract_digest": "sha256:...",
  "contract_url": "https://.../contract.json",
  "deprecated": false
}
```

**This is a thin pointer, not a catalog entry** — no `summary`, `description`, `inputs`, or `outputs` field. To evaluate whether an existing capability fits your need, fetch `contract_url` and read the real contract (same shape as `references/contract-schema.md`).

## 2. Check what's already declared in an app bundle you're assembling

```bash
traverse-cli capability discover path/to/registry-bundle/manifest.json --json
```

This loads that bundle into an in-memory registry and lists everything actually registered in it — id, version, scope, lifecycle, `implementation_kind`, summary, and tags. Richer than the public index because it's built from full contracts already resolved into memory, but it only reflects what that one bundle declares.

## 3. Publishing a capability you built

Once your capability validates cleanly (`agent inspect` passes, ABI whitelist clears — see the main skill's Step 6), making it available is a real, deliberately human-reviewed flow. `--registry-repo <path>` is always a **local filesystem path** — a checkout of the target registry repo. `--registry-repo-remote <owner/repo>` (optional) is the **GitHub slug** to open the PR against; when omitted, it defaults to `traverse-framework/registry`.

Public:

```bash
traverse-cli capability publish \
  --contract path/to/contract.json \
  --artifact path/to/artifact.wasm \
  --registry-repo path/to/local/traverse-framework-registry-checkout \
  --json \
  --dry-run   # remove once the planned branch/path look right
```

Private (the local checkout's own `origin` should already point at the same private repo you name in `--registry-repo-remote` — this flag doesn't change *where* the local checkout pushes to via git, only which repo `gh pr create --repo` targets):

```bash
traverse-cli capability publish \
  --contract path/to/contract.json \
  --artifact path/to/artifact.wasm \
  --registry-repo path/to/local/private-registry-checkout \
  --registry-repo-remote acme-corp/internal-registry \
  --json \
  --dry-run
```

Either way, this **never publishes directly** — it validates the contract and artifact digest, then prepares the publication candidate and opens a human-reviewed PR against a local checkout. Always dry-run first to see the planned branch and paths.

## Setting up a private registry (one-time, not per-capability)

If a team wants their own private registry and doesn't have one yet: fork or clone `traverse-framework/registry`'s structure (the `capabilities/<namespace>/<id>/<version>/contract.json` layout, its `scripts/ci/build_index.py` index-build pipeline, and its GitHub Actions release workflow) into a new repo, public or private. Once that repo exists and publishes an `index-v*` GitHub Release the same way the public one does, it's a fully valid `--source-repo`/`--registry-repo-remote` target — there's no separate protocol or service to stand up. This is a one-time setup a team does for themselves, not something this skill automates per-capability; if the person you're interviewing needs this done, treat it as a distinct, separate task.

## Workflow-side registry commands (real, but registry-only — not execution)

`workflow register` / `workflow list` / `workflow inspect` mirror the capability commands for workflow definitions and work identically against a private source via the same `--source-repo` mechanism where applicable. Owned by `traverse-workflow-planner`, not this skill — see that skill's reference material for composing and registering multi-node workflows.

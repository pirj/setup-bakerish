# setup-bakerish — orientation for agents

`setup-bakerish` is a **GitHub Actions composite action** that
installs the `aq + rlock + bakeri.sh` toolchain on a runner,
wires `RLOCK_PLUGIN_PATH`, and restores the snapshot-layer cache
so subsequent `bake run -- <cmd>` calls hit the sub-second-warm
path.

> **Rename pending: `setup-bakerish` → `setup-snapcompose`.**
> Follows the [`bakeri.sh` → `snapcompose` rename](../bakeri.sh/CLAUDE.md).
> Needs `@v3` (major bump) plus a deprecation period on the old
> name because user repos pin the action by name.

See [`README.md`](README.md) for usage, [`action.yml`](action.yml)
for inputs, [`TODO.md`](TODO.md) for open work, and the umbrella's
[`CLAUDE.md`](../CLAUDE.md) for how this fits with sibling repos.

## What this action does

Pure composite action (`action.yml`, no compiled JS / TS).
Replaces ~25 lines of install + cache choreography in a consumer
project's workflow with a single `uses:` step.

```yaml
- uses: pirj/setup-bakerish@v2
- run: bake run -- bundle exec rspec
```

Behind the scenes, the action's steps do:

1. Install host packages (`qemu`, `zstd`) — apt on Linux, brew on
   macOS.
2. Download release tarballs of `pirj/aq`, `pirj/rlock`,
   `pirj/bakeri.sh` (pinned versions via inputs) into `$HOME/`.
3. Add `aq`, `rl`, `bake` to `PATH`.
4. Set `RLOCK_PLUGIN_PATH` to bakeri.sh's plugin dir.
5. Restore (and optionally save) `~/.local/share/aq/cache` via
   `actions/cache@v4`, key derived from `bakerish.toml` +
   common lockfiles + `db/schema.rb` + migrations + Dockerfile +
   `docker-compose.*`.
6. Optionally fall back to OCI cache pull (`bake cache --pull`)
   on GH-cache miss.

## Hard constraints

- **No compiled JS / TS.** `action.yml` is a composite action;
  steps are `run:` shell. Don't introduce a `node20` action
  type — the simpler shell model is intentional.
- **Pin to release tarballs, not branches.** Inputs are
  `aq-version`, `rlock-version`, `bakeri-version`; all default
  to specific tags (e.g. `v2.5.18`). Each tag must correspond to
  an existing GitHub Release tarball.
- **Cache key must be deterministic.** Anything that affects the
  guest's installed deps must hash into the key. `cache-extra-paths`
  is the escape hatch for project-specific files.
- **GH Actions only.** No GitLab CI fork, no Buildkite plugin —
  if those become real, they're their own repos.
- **`runs-on: ubuntu-latest` is the smoke test target.** macOS
  runners work too but are slower; Linux is the canonical case.

## Versioning

- **Major versions are pinning targets.** `@v1`, `@v2`, etc.
  Consumer projects pin to a major; we move the major tag
  forward on backward-compatible improvements within that major.
- **Breaking changes bump the major.** Removing or renaming an
  input is breaking. Changing the default tarball version
  of `aq`/`rlock`/`bakeri.sh` is NOT breaking unless those
  components themselves break compat.
- **The `v2` major** is current as of 2026-05-25, validated
  green end-to-end on ubuntu-latest at 3m20s cold path (see
  `../meta/validation-2026-05-24-green.md` and the more recent
  race-fix validation).

## Validation discipline

This action is the seam between three independently-released
tools (`aq`, `rlock`, `bakeri.sh`). Cross-version breakage is a
real risk. Conventions:

- **Validate end-to-end before bumping default versions.** The
  validation runs in `../meta/validation-*.md` are the
  canonical evidence. Don't bump defaults silently.
- **Memory-aware:** the v2 validation history is captured in
  the user's auto-memory (`validation-2026-05-21.md`,
  `validation-2026-05-24-green.md`,
  `validation-2026-05-25-race-fix.md`). Future agents reading
  this CLAUDE.md should also read those notes before making
  cross-tool version-bump decisions.

## Inputs (current `v2`)

| Input | Default | Purpose |
|---|---|---|
| `aq-version` | `v2.5.18` | Release tag of `pirj/aq`. Tarball download. |
| `rlock-version` | `v0.1.3` | Release tag of `pirj/rlock`. |
| `bakeri-version` | `v0.1.2` | Release tag of `pirj/bakeri.sh`. |
| `cache-key-prefix` | `bakeri` | Prefix for actions/cache key. |
| `cache-extra-paths` | `''` | Newline-separated extra globs hashed into cache key. |
| `cache-restore-only` | `'false'` | `'true'` to skip the save step. |
| `no-snapshot-compress` | `'false'` | `'true'` sets `AQ_NO_SNAPSHOT_COMPRESS=1`. |
| `oci-cache-ref` | `''` | Optional OCI cache fallback (ghcr.io style). |

Don't add inputs lightly; each one is a public API surface for
every consuming workflow. Add only when there's a real,
recurring user need.

## What NOT to do

- Don't switch to a compiled action (node20 + tsc). The whole
  value of "composite + shell" is that the implementation is
  inspectable in `action.yml`.
- Don't unpin from release tarballs. Pulling from `main` would
  make CI runs non-reproducible.
- Don't add caching for things that don't reduce wall-clock.
  The `actions/cache@v4` round-trip itself takes 5-30s; only
  worth it for caches that save more than that.
- Don't break input names without a major version bump and a
  deprecation period.
- Don't merge a PR that changes the install path without
  re-running the validation on at least one real CI workflow.

## Sibling repos and dependencies

Installed by this action:

- [`aq`](https://github.com/pirj/aq) — pinned via `aq-version`.
- [`rlock`](https://github.com/pirj/rlock) — pinned via
  `rlock-version`.
- [`bakeri.sh`](https://github.com/pirj/bakeri.sh) — pinned via
  `bakeri-version`. The plugin pack whose `bake` commands
  consumers invoke.

This action does NOT install `ai.rlock` (the AI-agent plugin
pack). CI runners don't run coding agents; if a future product
combines them, it would be a separate action.

## Where decisions go

- **Mechanical work** for this action → [`TODO.md`](TODO.md).
- **Cross-cutting decisions** that affect the install/cache
  contract across `aq` + `rlock` + `bakeri.sh` + this action →
  ADRs in `../meta/decisions/`. See `../meta/CLAUDE.md`.
- **CHANGELOG.md** for what shipped per major version.

## Workspace context

This repo lives at `~/source/ai.rlock/setup-bakerish/` inside the
umbrella workspace. The umbrella's [`CLAUDE.md`](../CLAUDE.md) is
the single best map of how all sibling repos connect, including
the pending rename.

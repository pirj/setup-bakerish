# setup-snapcompose

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-not--published--yet-lightgrey)](#)

GitHub Action that installs `aq` + `rlock` + `snapcompose`, wires
`RLOCK_PLUGIN_PATH`, and restores the snapshot layer cache so subsequent
`bake run -- <cmd>` calls hit the sub-second-warm path.

Replaces the ~25-line install + cache choreography in
[snapcompose's example CI workflow](https://github.com/pirj/snapcompose/blob/main/.github/workflows/example-snapcompose-ci.yml)
with a single step.

## Quick start

```yaml
name: ci
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pirj/setup-snapcompose@v2
      - run: bake run -- bundle exec rspec
```

That's it. The action handles:

- Host package install (qemu + zstd) — apt on Linux, brew on macOS.
- Release-tarball download of `pirj/aq`, `pirj/rlock`, `pirj/snapcompose` to `$HOME/` (pinned to `*-version` inputs).
- `PATH` setup for `aq`, `rl`, `bake`.
- `RLOCK_PLUGIN_PATH` wired to snapcompose's plugin dir.
- `actions/cache@v4` restore + automatic save of `~/.local/share/aq/cache`.
- Cache key derived from `snapcompose.toml` + common lockfiles +
  `db/schema.rb` + `db/migrate/**` + `Dockerfile` + `docker-compose.*`.

## Inputs

| Input | Default | Purpose |
|---|---|---|
| `aq-version` | `v2.5.7` | Release tag of `pirj/aq`. Download is via the GH-auto-generated release tarball — must match an existing release. |
| `rlock-version` | `v0.1.0` | Release tag of `pirj/rlock`. |
| `snapcompose-version` | `v0.1.0` | Release tag of `pirj/snapcompose`. |
| `cache-key-prefix` | `snapcompose` | Prefix for the cache key. Use to segment caches by purpose. |
| `cache-extra-paths` | `''` | Extra newline-separated file globs to hash into the cache key on top of the defaults. |
| `cache-restore-only` | `false` | `true` to restore only (skip save). Useful for ephemeral PR jobs that shouldn't populate cache. |
| `no-snapshot-compress` | `false` | `true` sets `AQ_NO_SNAPSHOT_COMPRESS=1` — ~400 ms faster warm in exchange for ~1.1 GB extra cache per kind=live layer. |
| `oci-cache-ref` | `''` | Optional OCI ref (e.g. `ghcr.io/${{ github.repository_owner }}/snapcompose-cache:latest`). When set, the action also installs `oras` and — on GH-cache miss — attempts `bake cache --pull <oci-cache-ref>` as a second-tier fallback. The push side is the user's responsibility (see "Two-tier cache" below). |

## Examples

### Pin everything to specific versions

```yaml
- uses: pirj/setup-snapcompose@v2
  with:
    aq-version:     v2.5.7
    rlock-version:  v0.1.0
    snapcompose-version: v0.1.0
```

### Add custom cache-key inputs

```yaml
- uses: pirj/setup-snapcompose@v2
  with:
    cache-extra-paths: |
      config/forbidden-licenses.txt
      tooling/version.txt
      **/proto/*.proto
```

### Opt out of zstd memory compression for faster warm

```yaml
- uses: pirj/setup-snapcompose@v2
  with:
    no-snapshot-compress: 'true'
```

Useful when:
- You run many `bake run` calls per job and the 400 ms × N warm time
  matters more than the ~1.1 GB extra cache per live layer.
- Your repo is well under the 10 GB GH Actions cache quota.

### Two-tier cache (GH cache + OCI fallback)

GH Actions cache has a 10 GB / 7-day retention quota; OCI registries
(like GHCR) don't. Use OCI as a longer-term, larger-volume backstop
for the GH cache:

```yaml
steps:
  - uses: actions/checkout@v4

  - uses: pirj/setup-snapcompose@v2
    with:
      oci-cache-ref: ghcr.io/${{ github.repository_owner }}/snapcompose-cache:latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  - run: bake run -- bundle exec rspec

  # Push to OCI on main-branch builds only — avoids per-PR-commit
  # churn polluting the long-term cache.
  - name: Push cache to OCI (main only)
    if: always() && github.ref == 'refs/heads/main'
    run: bake cache --push ghcr.io/${{ github.repository_owner }}/snapcompose-cache:latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

The action pulls **per-layer blobs**: identical layer content across
pushes dedups server-side by sha256, so an unchanged `_base` /
`docker-engine` / `ruby-bundler` slot doesn't re-upload bytes — only
the changed slot's blob actually transfers. Active-PR churn (rails
migrations every commit) only uploads ~50 MB per push (the changed
slot), not the full 2.6 GB.

GHCR auth is via `GITHUB_TOKEN`. Workflow permissions must include
`packages: write` for push; pull only needs `packages: read`.

Cache TTL: GHCR has no automatic eviction. Run a periodic GC job
(`bake cache --gc` — TBD) to prune stale layer manifests once the
roadmap GC command lands.

### Matrix sharding

```yaml
strategy:
  matrix:
    shard: [1, 2, 3, 4]
steps:
  - uses: actions/checkout@v4
  - uses: pirj/setup-snapcompose@v2
  - run: bake run -- bundle exec rspec --shard=${{ matrix.shard }}/4
```

Each shard is its own runner = its own VM. Cache key is the same
across shards (same input files), so all 4 shards restore the same
~2.6 GB cache in parallel and warm-restore concurrently in ~2.7 s
wall-clock.

### Parallel commands in one job

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: pirj/setup-snapcompose@v2
  - run: |
      bake run --vm-suffix=lint -- rubocop  &
      bake run --vm-suffix=test -- rspec    &
      wait
```

Two suffixed VMs (`<basename>-lint`, `<basename>-test`) with
independent state but shared layer cache.

## Cache behaviour

- **Path**: `~/.local/share/aq/cache/` (snapshot layers) +
  `~/.local/share/aq/{x86_64,aarch64}/` (per-size base raw image).
- **Key**: `<prefix>-<runner_os>-<sha256-of-relevant-files>`.
- **restore-keys** fallback: `<prefix>-<runner_os>-`. Partial restore
  is safe — the framework re-checks each layer's `snapshot_key`
  independently. Layers whose inputs changed rebuild; the rest stay
  warm.
- **Save**: automatic on job end via `actions/cache@v4`'s built-in
  post-step. Set `cache-restore-only: 'true'` to disable.
- **Quota**: GH Actions free tier is 10 GB per repo. A typical
  Rails+PG project caches ~2.6 GB (with zstd) or ~3.7 GB (without).
  Matrix-sharded jobs share one cache (no multiplication).

## How it compares to the inline workflow

The [snapcompose inline reference workflow](https://github.com/pirj/snapcompose/blob/main/.github/workflows/example-snapcompose-ci.yml)
does the same thing in ~25 lines. The packaged action saves the
typing and gets you cache-key sanity for free. Same runtime behaviour.

Use the inline version when:
- You want full visibility into every step.
- You need to customise something the action doesn't expose
  (e.g. `apt install` flags, alternative cache backends).

Use this action otherwise.

## Status

Public, v2.0.0 (released 2026-05-21). Pin to `@v2` for the latest
v2.x.x (input names are `*-version`, downloads via release tarballs).
`@v1.0.0` still works against the older `*-ref` / `git clone`
interface — kept frozen for anyone who pinned exactly.

## License

MIT — see [LICENSE](LICENSE).

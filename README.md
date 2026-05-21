# setup-bakerish

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-not--published--yet-lightgrey)](#)

GitHub Action that installs `aq` + `rlock` + `bakeri.sh`, wires
`RLOCK_PLUGIN_PATH`, and restores the snapshot layer cache so subsequent
`bake run -- <cmd>` calls hit the sub-second-warm path.

Replaces the ~25-line install + cache choreography in
[bakeri.sh's example CI workflow](https://github.com/pirj/bakeri.sh/blob/main/.github/workflows/example-bakerish-ci.yml)
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
      - uses: pirj/setup-bakerish@v1
      - run: bake run -- bundle exec rspec
```

That's it. The action handles:

- Host package install (qemu + zstd) — apt on Linux, brew on macOS.
- Clones of `pirj/aq`, `pirj/rlock`, `pirj/bakeri.sh` to `$HOME/`.
- `PATH` setup for `aq`, `rl`, `bake`.
- `RLOCK_PLUGIN_PATH` wired to bakeri.sh's plugin dir.
- `actions/cache@v4` restore + automatic save of `~/.local/share/aq/cache`.
- Cache key derived from `bakerish.toml` + common lockfiles +
  `db/schema.rb` + `db/migrate/**` + `Dockerfile` + `docker-compose.*`.

## Inputs

| Input | Default | Purpose |
|---|---|---|
| `aq-ref` | `main` | Git ref of `pirj/aq` to install. Pin to a tag once you've validated a version. |
| `rlock-ref` | `main` | Git ref of `pirj/rlock` to install. |
| `bakeri-ref` | `main` | Git ref of `pirj/bakeri.sh` to install. |
| `cache-key-prefix` | `bakeri` | Prefix for the cache key. Use to segment caches by purpose. |
| `cache-extra-paths` | `''` | Extra newline-separated file globs to hash into the cache key on top of the defaults. |
| `cache-restore-only` | `false` | `true` to restore only (skip save). Useful for ephemeral PR jobs that shouldn't populate cache. |
| `no-snapshot-compress` | `false` | `true` sets `AQ_NO_SNAPSHOT_COMPRESS=1` — ~400 ms faster warm in exchange for ~1.1 GB extra cache per kind=live layer. |
| `oci-cache-ref` | `''` | Optional OCI ref (e.g. `ghcr.io/${{ github.repository_owner }}/bakerish-cache:latest`). When set, the action also installs `oras` and — on GH-cache miss — attempts `bake cache --pull <oci-cache-ref>` as a second-tier fallback. The push side is the user's responsibility (see "Two-tier cache" below). |

## Examples

### Pin everything to specific versions

```yaml
- uses: pirj/setup-bakerish@v1
  with:
    aq-ref:     v2.5.7
    rlock-ref:  v0.4.0          # once releases exist
    bakeri-ref: v0.3.0
```

### Add custom cache-key inputs

```yaml
- uses: pirj/setup-bakerish@v1
  with:
    cache-extra-paths: |
      config/forbidden-licenses.txt
      tooling/version.txt
      **/proto/*.proto
```

### Opt out of zstd memory compression for faster warm

```yaml
- uses: pirj/setup-bakerish@v1
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

  - uses: pirj/setup-bakerish@v1
    with:
      oci-cache-ref: ghcr.io/${{ github.repository_owner }}/bakerish-cache:latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  - run: bake run -- bundle exec rspec

  # Push to OCI on main-branch builds only — avoids per-PR-commit
  # churn polluting the long-term cache.
  - name: Push cache to OCI (main only)
    if: always() && github.ref == 'refs/heads/main'
    run: bake cache --push ghcr.io/${{ github.repository_owner }}/bakerish-cache:latest
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
  - uses: pirj/setup-bakerish@v1
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
  - uses: pirj/setup-bakerish@v1
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

The [bakeri.sh inline reference workflow](https://github.com/pirj/bakeri.sh/blob/main/.github/workflows/example-bakerish-ci.yml)
does the same thing in ~25 lines. The packaged action saves the
typing and gets you cache-key sanity for free. Same runtime behaviour.

Use the inline version when:
- You want full visibility into every step.
- You need to customise something the action doesn't expose
  (e.g. `apt install` flags, alternative cache backends).

Use this action otherwise.

## Status

Pre-release. Pin to `main` for now; v1 tag will be cut once the
upstream `pirj/aq` + `pirj/rlock` + `pirj/bakeri.sh` are
GH-Marketplace-publishable (currently private repos during early
adoption).

## License

MIT — see [LICENSE](LICENSE).

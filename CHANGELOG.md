# Changelog

All notable changes to setup-bakerish — one-liner per change.

## Unreleased

- (nothing pending)

## v1.0.0 — 2026-05-21

Initial public release.

### Composite action
- One-step install of `aq` + `rlock` + `bakeri.sh` from their respective git repos at pinnable refs.
- Wires `PATH` + `RLOCK_PLUGIN_PATH` via `$GITHUB_PATH` / `$GITHUB_ENV`.
- Host prereqs: qemu + zstd (apt on Linux, brew on macOS); `oras` added when `oci-cache-ref` is set.

### Cache choreography
- `actions/cache@v4` restore + auto-save (post-job step) of `~/.local/share/aq/cache/` + base catalog dirs.
- Cache key derived from `bakerish.toml` + common lockfiles + db schema + Dockerfile + `docker-compose.*`; tunable via `cache-extra-paths`.
- `restore-keys` fallback enables partial restore (the framework re-checks each layer's snapshot_key independently).

### Two-tier cache (OCI fallback)
- `oci-cache-ref` input opts in to a second tier. On GH-cache miss the action runs `bake cache --pull <ref>` against the OCI registry.
- GHCR auth via `$GITHUB_TOKEN` automatic for `ghcr.io/*` refs.
- Per-layer dedup at push time (server-side sha256) — active-PR churn uploads ~50 MB per commit, not the full cache.

### Inputs
- `aq-ref` / `rlock-ref` / `bakeri-ref` (default `main`; pin to tags as upstream stabilises).
- `cache-key-prefix` / `cache-extra-paths` / `cache-restore-only`.
- `no-snapshot-compress` — surface `AQ_NO_SNAPSHOT_COMPRESS=1` (warm-time-over-disk trade).
- `oci-cache-ref`.

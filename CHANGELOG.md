# Changelog

All notable changes to setup-bakerish — one-liner per change.

## Unreleased

- Bump `aq-version` default to `v2.5.14` (mount-trial partition
  detection + sentinel handshake for the kernel/initramfs extraction
  phase — v2.5.13's blkid approach didn't actually work because
  busybox blkid doesn't accept util-linux's query syntax).
- Build **tio v3.9** from source on Linux (was v3.7). tio v3.8
  renamed the Lua scripting "write to serial" function from
  `send(s)` to `write(s)`; aq has been using the v3.8+ name all
  along, so the v3.7 binary made every aq `write()` call resolve to
  nil — that was the real cause of the Linux/KVM post-login hang
  (NOT a setup-alpine issue). Homebrew on macOS ships v3.9, so the
  bug was Linux-only.
- Bump `aq-version` default to `v2.5.12` (reverts the v2.5.11
  misdiagnosed write→send rename; aq's `write()` is correct now
  that the runner has the right tio).

## v2.0.0 — 2026-05-21 (breaking)


Breaking changes vs v1.0.0:

- **Inputs renamed**: `aq-ref` / `rlock-ref` / `bakeri-ref` →
  `aq-version` / `rlock-version` / `bakeri-version`. Defaults shifted
  from `main` (branch) to release tags (`v2.5.7` / `v0.1.0` /
  `v0.1.0`). Pinning to a moving branch is no longer a supported
  default; use explicit release tags.
- **Download mechanism**: `git clone --depth=1` replaced by
  `curl tarball + tar xz --strip-components=1`. ~1–3 s faster per
  repo (~3–6 s total). Requires the named version to be an actual
  GH Release (i.e. a tag with an associated release).
- Migration: change `*-ref` → `*-version`; bump `@v1` to `@v2` in
  `uses:`. If you need the old behaviour, `@v1.0.0` is frozen.

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

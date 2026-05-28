# Changelog

All notable changes to setup-snapcompose — one-liner per change.

## v3.0.5 — 2026-05-28 — Symlink shared key to ~/.ssh for plain-ssh callers

v3.0.4 set `AQ_HOST_KEY` but rlock's `lib/util.sh` (do_ssh, wait_for_ssh)
calls plain `ssh` without `-i`, so those invocations didn't see the
shared key and hung at SSH auth time on cold rebuild. Fix: symlink the
cached key into `$HOME/.ssh/id_ed25519` so default-discovery `ssh`
finds the same key as AQ_HOST_KEY-aware aq calls. CI run 26572066218
showed cold-zstd / cold-zstd-patch both hanging at the first plugin
SSH invocation for 62 s before exit 1 — the symptom this fixes.

## v3.0.4 — 2026-05-28 — Stable cached host SSH key (skips cross-host inject)

Generate an ed25519 keypair once at cold time, cache it alongside
the snapshot layers, restore on every warm runner. Every runner
(potentially a different Azure VM) now uses the SAME `AQ_HOST_KEY`,
so aq's probe-first SSH check succeeds immediately on warm restore
and skips the cross-host serial-inject phase entirely (saves
~1.2 s per warm-zstd / warm-zstd-patch invocation on `ubuntu-latest`).

Net effect on the benchmark-r17-r18 fixture: WARM_ZSTD wall-clock
~10 s → ~8.7 s, WARM_ZSTD_PATCH ~17 s → ~15.7 s.

The shared key lives at
`~/.local/share/aq/setup-snapcompose/host-id_ed25519`, is scoped
per-cache-key, and is regenerated on cold rebuild. Not a
long-lived cross-project secret. The previously-generated
`~/.ssh/id_ed25519` fallback (per-job, ephemeral) is dropped; aq
picks up `AQ_HOST_KEY` from the env var instead.

## v3.0.3 — 2026-05-28 — Bump aq to v2.5.41 (R20 cross-host migration fix)

- Default `aq-version`: v2.5.40 → v2.5.41. Fixes R20 — warm restore
  with `AQ_MEMORY_SNAPSHOT=zstd` was hanging in qemu's `inmigrate`
  state for >60 s on Azure x86_64 KVM (`ubuntu-latest`). The
  fix decompresses `memory.bin.zst` to disk before launching qemu
  and uses `-incoming file:` (mmap) instead of `-incoming exec:pzstd`
  (pipe streaming).

## v3.0.2 — 2026-05-28 — Bump aq to v2.5.40 (R17 cross-host inject fix)

- Default `aq-version`: v2.5.39 → v2.5.40. Fixes R17 — cross-VM
  warm restore was failing with "outfile size: 0 bytes" on Azure
  x86_64 KVM CI runners (now also reproducible locally on M3 via
  `AQ_HOST_KEY=<different-key>`). See aq's CHANGELOG for the
  root-cause analysis.

## v3.0.1 — 2026-05-28 — Drop disable-zstd-asm debug input

- Removed the `disable-zstd-asm` input. It was a debug knob added
  during the R18 zstd-patch investigation to isolate whether the
  failure lived in zstd's x86_64 BMI2/intrinsics path. Root cause
  turned out to be in rlock's chain-reconstruction logic, not in
  zstd — so the build flag (`ZSTD_NO_ASM=1`, `-DZSTD_NO_INTRINSICS`,
  `-O0 -fno-tree-vectorize`) and the surrounding plumbing are no
  longer needed.
- Default `aq-version` bumped to `v2.5.39` and `rlock-version`
  bumped to `v0.1.12` (both ship the same instrumentation cleanup).

## v3.0.0 — 2026-05-27 — Renamed from setup-bakerish

**BREAKING CHANGE.** Action renamed in lockstep with the
`pirj/bakeri.sh` → `pirj/snapcompose` rename.

- Repo: `pirj/setup-bakerish` → `pirj/setup-snapcompose` (GitHub
  redirects preserve old URLs for now; consumer `uses:` should
  migrate to the new name).
- Action name in `action.yml`: `Setup bakeri.sh` → `Setup snapcompose`.
- Input renamed: `bakeri-version` → `snapcompose-version`.
- Default `cache-key-prefix` value: `bakeri` → `snapcompose`.
- Internal env var: `BAKERI_VERSION` → `SNAPCOMPOSE_VERSION`.
- Default version pins bumped: `aq-version` → `v2.5.37`,
  `rlock-version` → `v0.1.9`, `snapcompose-version` defaults to
  the current `pirj/snapcompose` head tag.
- Docs reference `snapc run` / `snapc cache` instead of
  `bake run` / `bake cache`.

Migration for existing workflows:

```yaml
# before
- uses: pirj/setup-bakerish@v2
  with:
    bakeri-version: 'v0.1.2'

# after
- uses: pirj/setup-snapcompose@v3
  with:
    snapcompose-version: 'v0.1.2'
```

GH's repo-rename redirect keeps `setup-bakerish@v2` working
until the redirect breaks (typically when a new repo with the
old name is created). Move to `v3` at the next workflow edit.

## Unreleased

- Bump `aq-version` default to `v2.5.14` (mount-trial partition
  detection + sentinel handshake for the kernel/initramfs extraction
  phase — v2.5.13's blkid approach didn't actually work because
  busybox blkid doesn't accept util-linux's query syntax).
- Bump `rlock-version` default to `v0.1.1` (`rl new --size=NG`
  flag — used by snapcompose's `[disk] size` threading).
- Bump `snapcompose-version` default to `v0.1.2` (`[disk] size` in
  snapcompose.toml overrides the 16G default; docker-engine plugin
  adds the `rlock` user to the `docker` group so prebuild
  commands using `docker compose` no longer fail with permission
  denied on /var/run/docker.sock).
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
  `aq-version` / `rlock-version` / `snapcompose-version`. Defaults shifted
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
- One-step install of `aq` + `rlock` + `snapcompose` from their respective git repos at pinnable refs.
- Wires `PATH` + `RLOCK_PLUGIN_PATH` via `$GITHUB_PATH` / `$GITHUB_ENV`.
- Host prereqs: qemu + zstd (apt on Linux, brew on macOS); `oras` added when `oci-cache-ref` is set.

### Cache choreography
- `actions/cache@v4` restore + auto-save (post-job step) of `~/.local/share/aq/cache/` + base catalog dirs.
- Cache key derived from `snapcompose.toml` + common lockfiles + db schema + Dockerfile + `docker-compose.*`; tunable via `cache-extra-paths`.
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

# setup-snapcompose TODO

Per-repo todo list.

## Open

- [ ] **Drop tio source-build when ubuntu-26.04 is available on
  GH Actions.** Currently `actions/runner-images` ships only
  ubuntu-22.04 and ubuntu-24.04 images; both ship outdated tio
  (≤ 2.7 in noble) so we build tio 3.9 from source. Ubuntu 26.04
  LTS ships tio 3.9-1build1 in main (per Launchpad 'stonking'
  series, published 2026-04-24), and GH typically adds new LTS
  runner images ~6 months after the distro release. When the
  ubuntu-26.04 runner label exists:
  - Switch `apt install` to include `tio`.
  - Remove the meson/ninja/curl/tar source-build block.
  - Drop the build-time deps from the apt-install list:
    `meson`, `ninja-build`, `pkg-config`, `liblua5.4-dev`,
    `libinih-dev`, `libglib2.0-dev`, `gcc`.
  - That saves ~15-20 s of cold setup time per CI job.

## Potential performance improvements (2026-05-29 research)

Catalogued from a 4-track research dive on cold/warm latency. Source: [`../meta/2026-05-29-optimization-research-top10.md`](../meta/2026-05-29-optimization-research-top10.md). Not in flight. Items here are the ones whose natural home is setup-snapcompose (action.yml ordering, GH Actions cache plumbing, runner choice); cross-cutting items live in [`../aq/ROADMAP.md`](../aq/ROADMAP.md), [`../snapcompose/TODO.md`](../snapcompose/TODO.md), [`../rlock/TODO.md`](../rlock/TODO.md).

- [ ] **Pin `-machine` + bump aq to a `mapped-ram`-capable QEMU, FIRST PRIORITY when ubuntu-26.04 GH runner lands.** Ubuntu noble ships QEMU 8.2.2 which doesn't have `mapped-ram` (landed in 9.0); ubuntu-26.04 will ship 9.x. Expected CI warm save: -1.5 to -2 s. This is the single biggest warm-restore win available, but blocked on the host-QEMU bump. Order of operations when 26.04 lands:
  1. Bump `runs-on:` in our smoke CI from `ubuntu-latest` → `ubuntu-26.04` (or wait for `ubuntu-latest` to point at 26.04).
  2. Pick up the matching aq release that has `-machine pc-q35-N.N` / `virt-N.N` pinned and `mapped-ram` QMP recipe wired (see [`../aq/ROADMAP.md`](../aq/ROADMAP.md) "Potential performance improvements" items).
  3. Bump default `aq-version` in `action.yml`.
  4. Bump `cache-key-prefix` (the new aq + mapped-ram caches aren't readable by old aq).
  5. Re-bench the warm matrix; document the new CI warm baseline.
- [ ] **Reorder `action.yml`: cache restore first, background apt+source builds.** Currently the order is host-prereqs (apt + tio build + zstd 1.5.7 source build, ~45-60 s) → fetch tarballs → compute cache key → cache restore. Moving the cache restore step ahead, and shell-backgrounding the apt+source builds with `&` then `wait` before the snapcompose step, hides 3-8 s of cold-runner wall-clock behind work that's happening anyway. Cost: ~30 min. Risk: zero (just step reordering). Only wins on cold-runner path; no effect once apt+brew caches are warm.
- [ ] **Split cache into two `actions/cache` entries** — one for stable layers (`_base` + `docker-engine` + arch base catalogs + the shared host SSH key), keyed on `aq-version + Dockerfile`; one for live layers, keyed on the current full `snapcompose.toml + lockfiles + db/migrate/**` hash. A migration-flip currently invalidates the whole 1.4 GiB tarball (~5-10 s redownload); with the split, only the ~770 MiB live-layer tarball invalidates. Expected ~2-3 s CI warm on migration-flip; 0 on no-change warm. ~1-2 h.
- [ ] **Benchmark third-party runners with persistent cache (Blacksmith, Namespace, RunsOn, Depot).** Published numbers: actions/cache on default GH ubuntu-latest = ~41 s restore at 4 GiB; Blacksmith = ~14 s; Namespace = ~25 s (NVMe persistent volume mounted directly to runner); RunsOn = ~23 s. Extrapolated to our ~1.4 GiB payload: 5-9 s saved per cold-cache-miss warm. Cost: try one workflow per provider; ops decision. Sources: [Depot blog](https://depot.dev/blog/comparing-github-actions-and-depot-runners-for-2x-faster-builds), [RunsOn benchmark](https://runs-on.com/benchmarks/github-actions-cache-performance/), [Blacksmith blog](https://www.blacksmith.sh/blog/cache).
- [ ] **Pre-compress payload with `zstd -19 --long=31` before letting actions/cache wrap it.** actions/cache uses `zstd -T0 --long=30` at default level 3. Inner-compression at level 19 + `--long=31` shaves ~200-300 MiB off the 1.4 GiB tarball (zstd decompress speed is essentially level-independent at ~1.5 GB/s). Save ~1-2 s NIC time per warm. Cost: low (wrap the cache path in a `tar | zstd -19 --long=31` step pre-cache-save). The outer actions/cache re-compress with level 3 on top is cheap and ~1.0× ratio. Trade: ~30 s extra in the save step's CPU.

### Explicitly NOT pursuing

- **GitHub Larger Runners.** Standard GH runners are pinned at ~1 Gbps NIC = ~125 MiB/s. Larger runners stay at the same NIC ceiling per Depot/Blacksmith/RunsOn benchmarks — no published cache speedup. ~8× the cost. Not worth it for cache restore.
- **Migrating to `actions/artifact-v4` for our warm cache path.** Artifacts target cross-job/cross-workflow persistence with different rate limits. Same Azure Blob backing, same NIC ceiling. Worth using only for genuinely-cross-workflow artefacts; not for warm-restore loops.

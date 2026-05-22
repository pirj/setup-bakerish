# setup-bakerish TODO

Per-repo todo list. Cross-repo items live in the umbrella's
[ai.rlock TODO.md](https://github.com/pirj/ai.rlock/blob/main/TODO.md).

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

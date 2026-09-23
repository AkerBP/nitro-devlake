# Hardened DevLake Release Record — v1.0.3-beta17-patches.20260923-harden.1

> Internal record. Do not publish upstream. Store outside the upstream source tree
> (committed to the hardening branch per INTERNAL_HARDENING_PROCESS.md section 16.1).

## Source

| Field | Value |
| --- | --- |
| Product | Apache DevLake |
| Source profile | tag-plus-selected-fixes |
| Latest official upstream tag | v1.0.3-beta17 |
| Actual upstream source ref | v1.0.3-beta17 + 1 selected upstream/main commit |
| Actual upstream source commit | 8fe26f46c83a83864249cd0a4c228606e3960aec (v1.0.3-beta17) |
| Selected upstream fix commits | `05dad99c8` "fix(build): use libgit2's bundled zlib so the image builds on arm64 hosts (#9132)" — merged to upstream `main` 2026-09-12, **not yet in any beta/release tag**. Required because our build host (and the shared libgit2 cross-build loop, which always builds both aarch64 and x86_64 regardless of target) is arm64; without it the aarch64 leg of the libgit2 build fails. |

Why tag-plus-fixes instead of official-tag: v1.0.3-beta17 alone could not be
built natively on our arm64 (Apple Silicon) workstation — see "Bugs found and
fixed" below. One narrow, upstream-authored, build-only commit was cherry-picked
to unblock it; no other upstream `main` content was pulled in.

## Hardening release

| Field | Value |
| --- | --- |
| Hardening release | 1 |
| Hardening branch | `harden/v1.0.3-beta17-patches-20260923` |
| Git hardening tag | `harden/v1.0.3-beta17-patches-20260923-r1` (annotated, GPG-signed) |
| Final hardening commit | `a9c96538dfc6618bad2feb960ddd2b08d7c1e2ca` |
| Branch commits (on top of v1.0.3-beta17) | `717b9fca9` chore(docker): harden v1.0.3-beta17 backend and config-ui images<br>`dccf46cbc` fix(build): use libgit2's bundled zlib so the image builds on arm64 hosts (#9132) [cherry-picked from upstream `05dad99c8`]<br>`a9c96538d` fix(docker): copy Debian trixie's relocated linux-libc-dev UAPI headers into sysroots [our own fix] |

## Images

| Field | Value |
| --- | --- |
| Registry | hubnitroplatformacr.azurecr.io |
| Platform(s) | linux/amd64 (deploy); linux/arm64 native build also validated locally |
| Backend image | `hubnitroplatformacr.azurecr.io/apache/devlake:v1.0.3-beta17-patches.20260923-harden.1` |
| Backend manifest-list digest | `sha256:3ba045b1961c9dfe27c0ff4568ccdf44aef6a92013e8bad162f57d129265f35b` |
| Backend amd64 image digest | `sha256:eb7ea88940eef3e609fc496b1e8e1c9d9ac0f6cd35a59197d9baca061126b69c` |
| Config UI image | `hubnitroplatformacr.azurecr.io/apache/devlake-config-ui:v1.0.3-beta17-patches.20260923-harden.1` |
| Config UI manifest-list digest | `sha256:51fe1bd3762bdf9d8d7e6dd450e6b355d632868dce613bf305e281c6f0805141` |
| Config UI amd64 image digest | `sha256:35db1ee93258f46520d5c622adaee787156f5b2c7f2478dd5e8206124f3c879f` |
| Backend plugin profile | `customize,dora,gitextractor,github,github_graphql,issue_trace,linker,org,refdiff,webhook` (verified present, all 10) |

Deploy by digest:
```text
hubnitroplatformacr.azurecr.io/apache/devlake@sha256:eb7ea88940eef3e609fc496b1e8e1c9d9ac0f6cd35a59197d9baca061126b69c
hubnitroplatformacr.azurecr.io/apache/devlake-config-ui@sha256:35db1ee93258f46520d5c622adaee787156f5b2c7f2478dd5e8206124f3c879f
```

## Build / scan metadata

| Field | Value |
| --- | --- |
| Build date | 2026-09-23 |
| Builder | Leif Roger Frøysaa (agent-assisted) |
| Build tool | `docker buildx build --platform linux/amd64 --push` (docker-container driver, buildx v0.35.0); native `linux/arm64 --load` build used for local validation |
| Scanner | Trivy 0.74.0 (`--severity CRITICAL,HIGH --ignore-unfixed --scanners vuln`), vuln DB 2026-08-18 |
| Scan result | Backend: 0 CRITICAL/HIGH fixed (Debian layer + `/app/bin/lake` gobinary). Config UI: 0 CRITICAL/HIGH fixed. Confirmed on both the native arm64 local image and the pushed linux/amd64 image. |
| SBOM location | buildx provenance/SBOM attestation attached to each pushed image (the `unknown/unknown` sub-manifest in `imagetools inspect`) |

## Hardening summary

Reapplied the `harden/v1.0.3-beta15` hardening pattern onto `v1.0.3-beta17`,
keeping every upstream-newer version rather than regressing to the old pins
(per process §2 "do not regress versions"):

- **Backend**: sysroot/builder/runtime moved to Debian trixie
  (`debian:trixie-20260421` sysroots, `golang:1.26-trixie` builder,
  `debian:trixie-slim` runtime, floating for OS patch pickup on rebuild);
  Python/pip/uv and the `mockery`/`swag` dev-tool installs dropped from the
  image; `/go/bin` not copied into the runtime image; `VERSION` build arg
  required; `apt upgrade` + minimal `--no-install-recommends` packages in the
  runtime stage; `ca-certificates` added; libgit2 built with
  `-DBUILD_TESTS=OFF -DBUILD_CLAR=OFF -DBUILD_CLI=OFF -DBUILD_EXAMPLES=OFF`;
  `tini` as PID 1; non-root, arbitrary-UID-compatible (`chgrp -R 0` / `g=u`)
  permissions; `DISABLED_REMOTE_PLUGINS=true` (no-Python image).
- **go.mod/go.sum**: **unchanged from v1.0.3-beta17.** Upstream already ships
  equal-or-newer versions than every dependency the beta15 hardening had
  bumped: `pgx/v5` 5.10.0, `git2go` v34 / `libgit2` 1.5.0, `logrus` 1.10.0,
  `x/crypto` 0.55.0, `x/net` 0.58.0, `x/sys` 0.47.0, `x/text` 0.41.0,
  `x/tools` 0.49.0, `oauth2` 0.36.0, `x/sync` 0.22.0, `x/mod` 0.40.0,
  `go-oidc` v3.20.0, `jwt/v5` 5.3.1, `go-jose` bumped to the `/v4` module,
  `go-sql-driver/mysql` 1.8.1, `rogpeppe/go-internal` 1.14.1; the module was
  also renamed upstream to `github.com/apache/devlake` and `lib/pq` dropped
  in favor of `pgx/v5`. The prior hardening's dependency-bump commits were
  fully superseded and were not reapplied.
- **Config UI**: Node builder moved to `node:24-trixie-slim` on
  `$BUILDPLATFORM` (keeping upstream's beta17 Node 22→24 bump, combined with
  our trixie base); kept upstream's `nginx-unprivileged:1.31.5` final image;
  `apt upgrade` + minimal `--no-install-recommends` packages on the runtime
  stage.

## Bugs found and fixed during this hardening pass

Both were discovered only because this process requires validating a
**native** local image on the build host (here, arm64/Apple Silicon), not just
a cross-compiled amd64 image:

1. **Upstream bug (not yet released):** the shared libgit2 cross-build loop
   fails to compile for `aarch64` on an arm64 build host with
   `asm/errno.h: No such file or directory`, because Debian trixie's
   `linux-libc-dev` moved kernel UAPI headers to an arch-independent
   `/usr/lib/linux/uapi/<arch>/` tree — `/usr/include/<triplet>/asm/*.h` are
   now just symlinks into it, and our (and upstream's) sysroot `COPY`
   instructions never copied `/usr/lib/linux`. Fixed by copying
   `/usr/lib/linux` into both `rootfs-amd64` and `rootfs-arm64` sysroots.
   Separately cherry-picked upstream's own related fix `05dad99c8`
   (`-DUSE_BUNDLED_ZLIB=ON`) for the same underlying "arm64 build host" class
   of problem; it alone was insufficient without the UAPI-header copy above.
2. **Latent error-masking bug:** the libgit2 cross-build `RUN` had no
   `set -e`. When the aarch64 leg failed (bug 1), the `&&` chain inside that
   loop iteration short-circuited silently, the `for` loop moved on to
   `x86_64` (which succeeded), and the whole `RUN` step exited 0 with only
   `x86_64` actually installed under `/usr/local/deps`. The real failure only
   surfaced later, confusingly, in the unrelated `build` stage as
   `Package libgit2 was not found in the pkg-config search path` when
   `TARGETPLATFORM=linux/arm64` needed the (silently missing) directory.
   Added `set -e` to the loop so any future per-arch failure fails the build
   immediately and visibly instead of being masked.

Neither issue is specific to our hardening changes to the *sysroot content*
(they'd affect anyone building the vanilla upstream Dockerfile natively on an
arm64 host with a trixie-based sysroot); they were only surfaced because our
hardening moved the sysroot base from `bookworm` to `trixie`.

## CVE remediation (Go dependencies)

None needed — see "Hardening summary" above; `v1.0.3-beta17` already carries
newer fixed versions than our previous hardening baseline for every module we
had bumped for beta15.

## Validation performed

- `go mod verify`: pass.
- Fast backend tests (`./core/version ./core/config ./core/runner`): pass (5 tests, 3 packages).
- `lake --version` / `/version` endpoint: `v1.0.3-beta17-patches.20260923-harden.1` — matches.
- Removed tooling absent (`python3`, `pip`, `curl`, `swag`, `mockery`, `poetry`, `uv`), `/go/bin` absent: confirmed.
- Non-root: `devlake` (uid=1010, gid=1010).
- All 10 production plugins present in `/app/bin/plugins` and confirmed loaded at runtime.
- Compose smoke test (Postgres 17.2 + backend + config-ui, native `linux/arm64` local images):
  **PASSED** — `/health` good, `/ready` ready, `/version` matches, config-ui
  `/health` and `/` both HTTP 200, all 10 plugins logged `plugin loaded <name>`
  + `all plugins have been loaded`, no panic/fatal in logs.
- Trivy scan (native arm64 local images): backend 0 CRITICAL/HIGH fixed
  (Debian + `/app/bin/lake` gobinary); config-ui 0 CRITICAL/HIGH fixed.
- Trivy re-scan of pushed `linux/amd64` images (post `buildx --push`): backend
  0 CRITICAL/HIGH fixed; config-ui 0 CRITICAL/HIGH fixed.

## Notes / follow-ups

- Runtime bases (`debian:trixie-slim`, `nginx-unprivileged:1.31.5`) float —
  deploy by digest, not tag; rebuild periodically to pick up OS patches.
- Companion Helm chart (`apache/incubator-devlake-helm-chart`) has not tagged
  a beta16/beta17 `appVersion` release yet (still at `devlake-1.0.3-beta15`);
  diffed `devlake-1.0.3-beta15` against chart `main` anyway — only a port-name
  templating refactor (#380), no new required env vars/secrets/probes/ports.
  Nothing new to mirror into the local smoke-test compose file beyond the
  existing `ENCRYPTION_SECRET` requirement (beta15+).
  Re-check the chart when it tags a beta17-aligned release.
- Fork (`AkerBP/nitro-devlake`) `main` was GitHub-UI-synced with
  `apache/devlake:main` but this does **not** carry upstream tags; `git fetch
  upstream 'refs/tags/*:refs/tags/*'` was required to obtain `v1.0.3-beta10`
  through `v1.0.3-beta17`.
- `nitro-devlake-scan.sh`'s Python JSON-summarizer helper errored when piping
  a registry ref through `trivy ... 2>/dev/null` in this environment (stderr
  was masking a real trivy message); scans for this release were run with
  `trivy image` directly instead. Worth a follow-up fix to the helper script.
- `docker run`/`docker compose` on this host requires an explicit
  `--platform`/`DOCKER_DEFAULT_PLATFORM` matching the loaded image now that
  the local Docker Engine uses the containerd snapshotter — otherwise it
  reports the image "was found but does not provide the specified platform"
  even though `docker images` lists it. Not a hardening issue, just a local
  tooling note for future native-arch validation on this or similar hosts.
- Next image from this source line uses `harden.2`. A new upstream tag
  (`v1.0.3-beta18`+, or a chart-confirmed beta17 appVersion) should start a
  fresh `harden/v1.0.3-beta18` (or similarly named) branch rather than
  reusing this patches branch.

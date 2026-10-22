# Hardened DevLake Release Record — v1.0.3-beta18-harden.1

> Internal record. Do not publish upstream. Committed to the hardening branch per
> INTERNAL_HARDENING_PROCESS.md section 16.1.

## Source

| Field | Value |
| --- | --- |
| Product | Apache DevLake |
| Source profile | official-tag |
| Latest official upstream tag | v1.0.3-beta18 |
| Actual upstream source ref | v1.0.3-beta18 |
| Actual upstream source commit | 6502e549f58eb40c40ccbcd39aab9bc8eea6616f |
| Selected upstream fix commits | none. beta18 already contains the libgit2 `USE_BUNDLED_ZLIB` fix and the libgit2 build-loop `set -e` fix that beta17 needed as cherry-picks. |

## Hardening release

| Field | Value |
| --- | --- |
| Hardening release | 1 |
| Hardening branch | `harden/v1.0.3-beta18` |
| Git hardening tag | `harden/v1.0.3-beta18-r1` (annotated, GPG-signed) |
| Final hardening commit | `491a0c62d5c587f838b7490b701fe07df7fda79c` |
| Branch commits (on top of v1.0.3-beta18) | `df6496c19` chore(docker): harden v1.0.3-beta18 backend and config-ui images<br>`491a0c62d` fix(deps): bump golang.org/x/net to v0.60.0 (CVE-2026-78669) |

## Images

| Field | Value |
| --- | --- |
| Registry | hubnitroplatformacr.azurecr.io |
| Platform(s) | linux/amd64 (deploy); linux/arm64 native build validated locally |
| Backend image | `hubnitroplatformacr.azurecr.io/apache/devlake:v1.0.3-beta18-harden.1` |
| Backend manifest-list digest | `sha256:8f2c5542bb9588835f445435aa307c244fe5ccc44654b79c66f9363947a46062` |
| Backend amd64 image digest | `sha256:9c5596033c27317de1584c44090f7906cb3cbc69c65e222749937f0168b19653` |
| Config UI image | `hubnitroplatformacr.azurecr.io/apache/devlake-config-ui:v1.0.3-beta18-harden.1` |
| Config UI manifest-list digest | `sha256:e3314dc45f9701bb0200ecc14847ddb4e9d199d5b683b66c97ec3a43cde97ae5` |
| Config UI amd64 image digest | `sha256:ce254c2db07da9e59ca5c13b9107324277b590be0150f49032f95699496aa716` |
| Backend plugin profile | `customize,dora,gitextractor,github,github_graphql,issue_trace,linker,org,refdiff,webhook` (all 10 verified loaded) |

Deploy by digest:
```text
hubnitroplatformacr.azurecr.io/apache/devlake@sha256:9c5596033c27317de1584c44090f7906cb3cbc69c65e222749937f0168b19653
hubnitroplatformacr.azurecr.io/apache/devlake-config-ui@sha256:ce254c2db07da9e59ca5c13b9107324277b590be0150f49032f95699496aa716
```

## Build / scan metadata

| Field | Value |
| --- | --- |
| Build date | 2026-10-22 |
| Builder | Leif Roger Frøysaa (agent-assisted) |
| Build tool | `docker buildx build --platform linux/amd64 --push` (docker-container driver); native `linux/arm64 --load` build used for local validation |
| Scanner | Trivy 0.76.1 (`--severity CRITICAL,HIGH --ignore-unfixed --scanners vuln`), vuln DB updated 2026-10-22 07:10 UTC |
| Scan result | Backend: 0 CRITICAL/HIGH fixed. Config UI: 0 CRITICAL/HIGH fixed. Confirmed on the native arm64 local images and on the pushed linux/amd64 images. |
| SBOM location | buildx provenance/SBOM attestation attached to each pushed image (the `unknown/unknown` sub-manifest) |

## eMudhra intermediate CA compromise (CVE-2026-111943)

Private keys of eight eMudhra intermediate CA certificates were compromised
(info: https://emudhra.we-have-a-problem.com/, 2026-10-20). Go fixes this in
**go1.26.10 and go1.27.3** (released 2026-10-19; Go issue 82345, GO-2026-6809,
`crypto/x509.Certificate.Verify`): chains containing the affected intermediates
are filtered out via an SPKI-hash denylist. Go's own notice does not name the CA.

Decision: build on **Go 1.26.10** (same minor as upstream's `go 1.26` baseline,
smallest change) rather than 1.27.3. The builder is pinned to
`golang:1.26.10-trixie` (digest `sha256:909771fe…02d4`) so a future rebuild cannot
silently fall back to a vulnerable toolchain. Do not lower it below 1.26.10.

Two independent layers were fixed and verified:

| Layer | Fix | Evidence |
| --- | --- | --- |
| Go binary (`/app/bin/lake`) | built with go1.26.10 | `go version` on the extracted amd64 and arm64 binaries = `go1.26.10`; symbol `crypto/x509.containsDistrustedSPKIHash` present |
| OS trust store (runtime image) | Debian trixie `ca-certificates 20250419+deb13u1` (via `apt upgrade` on the floating `debian:trixie-slim` runtime) | package version checked on amd64 and arm64; `emSign ECC Root CA - G3` and Comodo/AAA roots absent from `/etc/ssl/certs`; `git` (OS CAs) rejects both checker hosts |

Behavioural before/after test of the Go fix (same program, unpatched macOS
`/etc/ssl/cert.pem` as the trust store, which still contains the affected roots;
`SSL_CERT_FILE` override, `SSL_CERT_DIR=/nonexistent`):

| Host | go1.26.9 (pre-fix) | go1.26.10 (fixed) |
| --- | --- | --- |
| `emudhra.we-have-a-problem.com` (control, Let's Encrypt) | TRUSTED | TRUSTED |
| `dv-g3b-emudhra.we-have-a-problem.com` | **TRUSTED** (vulnerable) | REJECTED (`x509: no valid chains built`) |
| `dv-g3b-aaa-emudhra.we-have-a-problem.com` | **TRUSTED** (vulnerable) | REJECTED (`x509: no valid chains built`) |

## Hardening summary

Reapplied the `harden/v1.0.3-beta17-patches-20260923` hardening onto
`v1.0.3-beta18` (cherry-picked `717b9fca9` and `a9c96538d`, conflicts resolved
against beta18's own libgit2 fixes; `dccf46cbc` was not needed):

- **Backend**: Debian trixie sysroots (`debian:trixie-20260421`), builder
  `golang:1.26.10-trixie` (pinned, see above), runtime `debian:trixie-slim`
  (floating, `apt upgrade`); no Python/pip/uv, no `mockery`/`swag`; `/go/bin`
  not copied; `VERSION` build arg required; `ca-certificates`; libgit2 1.5.0 built
  with tests/CLI/examples off, `USE_BUNDLED_ZLIB=ON` and `set -e` (both from
  upstream beta18); trixie `/usr/lib/linux` UAPI headers copied into sysroots;
  `tini` PID 1; non-root, arbitrary-UID-compatible perms;
  `DISABLED_REMOTE_PLUGINS=true`.
- **Config UI**: `node:24-trixie-slim` builder on `$BUILDPLATFORM`; upstream's
  `nginx-unprivileged:1.31.5` retained; `apt upgrade`, minimal packages.
- No version regressions vs. beta18 (nginx 1.31.5, libgit2 1.5.0, Node 24).

## CVE remediation (Go dependencies)

- `golang.org/x/net` v0.58.0 → v0.60.0 (CVE-2026-78669, HIGH; HTTP/2 excessive
  CPU on repeated initial window changes). Minimum-version selection also moved
  `x/crypto` 0.55.0→0.57.0, `x/sys` 0.47.0→0.48.0, `x/text` 0.41.0→0.42.0,
  `x/sync` 0.22.0→0.23.0, `x/mod` 0.40.0→0.41.0. `go` directive normalised to
  `go 1.26.0` by `go get`.
- Go stdlib findings (GO-2026-6599…6617, fixed in 1.26.9, plus the x509 one in
  1.26.10) are covered by the 1.26.10 builder.

## Validation performed

- go mod verify: all modules verified.
- Fast backend tests (`core/version core/config core/runner`) with go1.26.10: pass.
- `lake --version` embeds `v1.0.3-beta18-harden.1`; `/version` matches.
- Removed tooling absent (python3, pip, curl, swag, mockery, poetry, uv); `/go/bin` absent.
- Compose smoke test (Postgres + backend + config-ui, native arm64): PASSED.
  /health, /ready, /version=`v1.0.3-beta18-harden.1`; config-ui /health and / = 200;
  10 plugins loaded; no panic/fatal.
- Trivy re-scan of pushed amd64 images: 0 CRITICAL/HIGH fixed (backend and config-ui).

## Notes / follow-ups

- Deployment config drift (Helm chart `devlake-1.0.3-beta15` → `-beta18`; the
  chart has no beta16/17 tags): no new required env vars or secrets. Container
  ports are now named (`lake`, `ui`, `mysql`) and `service.targetPort` references
  them; `grafana.service.port` / `lake.port` are templated. `ENCRYPTION_SECRET`
  remains required.
- Runtime base floats: a rebuild of the same commit can change OS packages; the
  digest is the audit artifact.
- Every other Go TLS client in the fleet should also be on >= 1.26.10 / 1.27.3.
- Local smoke tests ran on arm64 (Docker `DOCKER_DEFAULT_PLATFORM=linux/arm64`);
  the amd64 deploy image was scanned and binary-inspected but not run end to end.
- Upstream `main` has since bumped Go to 1.26.9 (`df76866ee`); nothing from
  `main` is included in this image.

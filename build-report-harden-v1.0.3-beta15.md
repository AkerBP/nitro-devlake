# Hardened DevLake Release Record — v1.0.3-beta15-harden.1

> Internal record. Do not commit upstream. Stored outside the upstream source tree per
> INTERNAL_HARDENING_PROCESS.md section 17.

## Source

| Field | Value |
| --- | --- |
| Product | Apache DevLake |
| Source profile | official-tag |
| Latest official upstream tag | v1.0.3-beta15 |
| Actual upstream source ref | v1.0.3-beta15 |
| Actual upstream source commit | 27c771a89723c7eef0df54cfad5927c8ed4a09fe |
| Selected upstream fix commits | none (beta15 already contains the fixes that previously required a main-snapshot branch: Go 1.26, AES-GCM encryption, OIDC auth, GitHub incremental collection, migration-auth) |

## Hardening release

| Field | Value |
| --- | --- |
| Hardening release | 1 |
| Hardening branch | harden/v1.0.3-beta15 |
| Git hardening tag | harden/v1.0.3-beta15-r1 (annotated, GPG-signed) |
| Final hardening commit | 2ae04f954633089bc03c7a6fdc806fbf102728d5 |
| Branch commits | 57e62ccad chore(docker): harden v1.0.3-beta15 backend and config-ui images<br>2ae04f954 fix(deps): bump golang.org/x/crypto to 0.52.0 and x/net to 0.55.0 |

## Images

| Field | Value |
| --- | --- |
| Registry | hubnitroplatformacr.azurecr.io |
| Platform | linux/amd64 only (no arm64 node pools in use) |
| Backend image | hubnitroplatformacr.azurecr.io/apache/devlake:v1.0.3-beta15-harden.1 |
| Backend manifest-list digest | sha256:137662df85b25824d2a1261b22d4c0b8573f575b11761c9916ac281efaa6b46b |
| Backend amd64 image digest | sha256:d926fc85e800d8e7ab05d94245529f105743c0498ba6b6b16420cdcd649cd613 |
| Config UI image | hubnitroplatformacr.azurecr.io/apache/devlake-config-ui:v1.0.3-beta15-harden.1 |
| Config UI manifest-list digest | sha256:b6e65294b0f2e52b891cba58a89ea9127c49255bb2bde2a50aa6cd37effa8727 |
| Config UI amd64 image digest | sha256:d512341593a96e15eed7c3be33b835de12d1271e126057056d70a5431c71baff |
| Backend plugin profile | customize,dora,gitextractor,github,github_graphql,issue_trace,linker,org,refdiff,webhook |

Recommended deployment references (digest-pinned):

```text
hubnitroplatformacr.azurecr.io/apache/devlake@sha256:d926fc85e800d8e7ab05d94245529f105743c0498ba6b6b16420cdcd649cd613
hubnitroplatformacr.azurecr.io/apache/devlake-config-ui@sha256:d512341593a96e15eed7c3be33b835de12d1271e126057056d70a5431c71baff
```

## Build / scan metadata

| Field | Value |
| --- | --- |
| Build date | 2026-07-28 |
| Builder | Leif Roger Frøysaa (local buildx, docker-container driver) |
| Build tool | docker buildx build --platform linux/amd64 --push |
| Scanner | Trivy 0.72.0 (--severity CRITICAL,HIGH --ignore-unfixed --scanners vuln) |
| Scan result | Backend: 0 CRITICAL/HIGH fixed. Config UI: 0 CRITICAL/HIGH fixed. (Scanned on the pushed amd64 images.) |
| SBOM location | buildx provenance/SBOM attestation manifests attached to each pushed image (unknown/unknown sub-manifest) |

## Hardening summary

Container hardening reapplied from harden/v1.0.3-beta12-main-20260603-5e9dfb8b8 onto the
official v1.0.3-beta15 tag. Verified no version regressions vs beta15 (all go.mod changes are
strict upgrades; config-ui keeps beta15's newer nginx-unprivileged:1.30.3, not the old patch's
1.29).

Backend image:
- Debian sysroot/builder/runtime moved bookworm -> trixie (sysroot pinned trixie-20260421; runtime debian:trixie-slim floats).
- Go 1.26 baseline retained.
- libgit2 1.3.0 -> 1.3.2; libgit2 tests/CLI/examples build output disabled.
- Removed Python, pip, Poetry/uv, and Python plugin build layers (no-Python image).
- Remote/Python plugin loading disabled (DISABLED_REMOTE_PLUGINS=true).
- Removed dev tooling (mockery, swag); /go/bin not copied into runtime.
- VERSION build arg required; embedded and surfaced via /version.
- apt upgrade in the runtime stage; minimal --no-install-recommends packages; ca-certificates added; curl removed; package-manager metadata cleaned.
- tini as PID 1; non-root user (uid 1010); OpenShift/arbitrary-UID-compatible permissions.

Config UI image:
- node:22-trixie-slim builder on $BUILDPLATFORM.
- Unprivileged nginx final image (nginxinc/nginx-unprivileged:1.30.3, from beta15).
- apt upgrade + minimal --no-install-recommends packages; apt metadata cleaned.

## CVE remediation (Go dependencies)

Reapplied from previous baseline: pgx/v5 5.9.0, logrus 1.9.3, testify 1.11.1, go-jose/v3 3.0.5,
jwt/v5 5.2.2, oauth2 0.27.0, plus updated golang.org/x/* modules.

New this release (HIGH CVEs disclosed after the June baseline, flagged by Trivy against the
beta15 backend image):
- golang.org/x/crypto v0.41.0 -> v0.52.0
    CVE-2025-47913, CVE-2026-39828, CVE-2026-39829, CVE-2026-39830, CVE-2026-39831,
    CVE-2026-39832, CVE-2026-39835, CVE-2026-42508, CVE-2026-46595, CVE-2026-46597
- golang.org/x/net v0.43.0 -> v0.55.0
    CVE-2026-25681, CVE-2026-27136, CVE-2026-33814, CVE-2026-39821
- Transitive upgrades: x/sync 0.20.0, x/mod 0.35.0, x/sys 0.45.0, x/text 0.37.0, x/tools 0.44.0.

Debian trixie OS packages and config-ui were already clean (floating runtime base + apt upgrade).

## Validation performed

- go mod verify: all modules verified.
- Fast backend tests: core/config, core/runner pass (core/version has no test files).
- lake --version embeds v1.0.3-beta15-harden.1.
- Removed backend tooling confirmed absent: python3, pip, curl, swag, mockery, poetry, uv; /go/bin not present.
- config-ui nginx 1.30.3 confirmed.
- Docker Compose smoke test (Postgres 17.2 + hardened backend + hardened config-ui):
    - all containers healthy; backend listening on :8080; no panic/fatal.
    - backend /health good, /ready ready, /version = v1.0.3-beta15-harden.1.
    - config-ui /health HTTP 200, / HTTP 200.
    - all 10 production plugins loaded (customize, dora, gitextractor, github, github_graphql,
      issue_trace, linker, org, refdiff, webhook); "all plugins have been loaded"; no extras.
- Trivy re-scan of the pushed amd64 images: backend 0, config-ui 0 CRITICAL/HIGH fixed findings.

## Notes / follow-ups

- linux/amd64 only, by design (no arm64 Kubernetes node pools). To add arm64 later, rebuild
  multi-arch with buildx; the backend Dockerfile already cross-compiles for both arches.
- beta15 hard-requires ENCRYPTION_SECRET at startup (AES-GCM encryption feature). Ensure the
  deployment sets it. The local smoke compose uses a throwaway value.
- The companion Helm chart (devlake-helm-chart) is reference-only and was intentionally not
  updated; these hardened images live only in the internal registry.
- Grafana/dashboard image not hardened/rebuilt: a cluster-wide Grafana instance is reused.
- Next image from this source line should use harden.2 unless a newer upstream tag is chosen.

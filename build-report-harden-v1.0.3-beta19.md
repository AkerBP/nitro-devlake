# Hardened DevLake Release Record — v1.0.3-beta19-harden.1

## Source

| Field | Value |
| --- | --- |
| Source profile | official-tag |
| Latest official upstream tag | v1.0.3-beta19 |
| Actual upstream source ref | v1.0.3-beta19 |
| Actual upstream source commit | 210dcd4125e460a48895c2fdb418a35c56287cf7 |
| Selected upstream fix commits | none |

## Hardening release

| Field | Value |
| --- | --- |
| Hardening release | 1 |
| Hardening branch | harden/v1.0.3-beta19 |
| Git hardening tag | harden/v1.0.3-beta19-r1 |
| Final hardening commit | a1dd2833c47915615a8c03c28864e78b86ff797d |
| Branch commits | chore(docker): harden v1.0.3-beta19 backend and config-ui images; fix(deps): bump golang.org/x/net to v0.60.0 (CVE-2026-78669) |

## Images

| Field | Value |
| --- | --- |
| Registry | hubnitroplatformacr.azurecr.io |
| Platform(s) | linux/amd64 |
| Backend image | hubnitroplatformacr.azurecr.io/apache/devlake:v1.0.3-beta19-harden.1 |
| Backend manifest-list digest | sha256:22450eb5aefa184849e05a913502feba587df0852777c8ec50ce4fe2c7bbff8a |
| Backend amd64 image digest | sha256:003c5eb03050a5096b717da40371482e1854d39808a40db17e73f41f6832b317 |
| Config UI image | hubnitroplatformacr.azurecr.io/apache/devlake-config-ui:v1.0.3-beta19-harden.1 |
| Config UI manifest-list digest | sha256:bd231749414a5cd48fa0326c4786c3f68691675fbe8a13d9ceb7c537e5d195e2 |
| Config UI amd64 image digest | sha256:f3c9d22ee5f061ef47a45962d42ce0d9b3c0adc36a216ee62a9de255ba16a7a9 |
| Backend plugin profile | customize,dora,gitextractor,github,github_graphql,issue_trace,linker,org,refdiff,webhook |

Deploy by digest:
```text
hubnitroplatformacr.azurecr.io/apache/devlake@sha256:003c5eb03050a5096b717da40371482e1854d39808a40db17e73f41f6832b317
hubnitroplatformacr.azurecr.io/apache/devlake-config-ui@sha256:f3c9d22ee5f061ef47a45962d42ce0d9b3c0adc36a216ee62a9de255ba16a7a9
```

## Build / scan metadata

| Field | Value |
| --- | --- |
| Build date | 2026-11-04 |
| Builder | Leif Roger Frøysaa (with pi) |
| Build tool | docker buildx build --platform linux/amd64 --push |
| Scanner | Trivy 0.76.1 (--severity CRITICAL,HIGH --ignore-unfixed --scanners vuln) |
| Scan result | Backend: 0 CRITICAL/HIGH fixed. Config UI: 0 CRITICAL/HIGH fixed. |
| SBOM location | buildx provenance/SBOM attestation attached to each pushed image |

## Hardening summary

- Upstream beta19 replaced libgit2/CGO and the .so plugin set with one static binary (plugins compiled in, gitextractor uses the git CLI). The beta18 sysroot/libgit2 hardening is therefore dropped, not ported.
- Builder pinned to golang:1.27.3-trixie; runtime is floating debian:trixie-slim with apt upgrade, tini, git, openssh-client, corkscrew, ca-certificates. No Python, pip, uv, curl, mockery, swag.
- VERSION build arg required and embedded via ldflags; `GO_PLUGINS` restricts the compiled-in plugin registry via DEVLAKE_PLUGINS.
- DISABLED_REMOTE_PLUGINS=true; /app/bin/plugins mount point kept from upstream.
- Config UI: node:24-trixie-slim builder (BUILDPLATFORM); nginx-unprivileged 1.31.7 and apt upgrade come from upstream.

## CVE remediation (Go dependencies)

- golang.org/x/net 0.59.0 -> 0.60.0 (CVE-2026-78669). Only go.mod/go.sum change.

## Validation performed

- go mod verify: OK
- go test ./core/version ./core/config ./core/runner: OK
- lake --version embeds v1.0.3-beta19-harden.1: /version returns it
- Removed tooling absent (python3, pip, curl, swag, mockery, poetry, uv): OK
- Compose smoke test (Postgres + backend + config-ui, arm64 local image): passed; 10 plugins via GET /plugins, no panic/fatal
- Trivy re-scan of pushed amd64 images: 0 / 0
- Pushed amd64 lake: go1.27.3; ca-certificates 20250419+deb13u1; runs as uid 1010

## Notes / follow-ups

- Smoke test script changed to detect plugins via GET /plugins (no "plugin loaded" logs since beta19).
- Config UI scanned only; not smoke-tested separately beyond /health and /.

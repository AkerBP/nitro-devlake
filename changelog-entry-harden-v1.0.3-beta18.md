### v1.0.3-beta18-harden.1 (harden/v1.0.3-beta18)

Date: 2026-10-22  •  Source profile: official-tag  •  Platform: linux/amd64

**Traceability**
- Upstream: v1.0.3-beta18 @ 6502e549f58eb40c40ccbcd39aab9bc8eea6616f   (latest official marker: v1.0.3-beta18)
- Hardening branch: harden/v1.0.3-beta18   •   Git release tag: harden/v1.0.3-beta18-r1   •   Commit: 491a0c62d5c587f838b7490b701fe07df7fda79c
- Backend image:  hubnitroplatformacr.azurecr.io/apache/devlake:v1.0.3-beta18-harden.1
  - digest: sha256:9c5596033c27317de1584c44090f7906cb3cbc69c65e222749937f0168b19653 (manifest list sha256:8f2c5542bb9588835f445435aa307c244fe5ccc44654b79c66f9363947a46062)
- Config UI image: hubnitroplatformacr.azurecr.io/apache/devlake-config-ui:v1.0.3-beta18-harden.1
  - digest: sha256:ce254c2db07da9e59ca5c13b9107324277b590be0150f49032f95699496aa716 (manifest list sha256:e3314dc45f9701bb0200ecc14847ddb4e9d199d5b683b66c97ec3a43cde97ae5)
- Plugin profile: customize,dora,gitextractor,github,github_graphql,issue_trace,linker,org,refdiff,webhook
- Supersedes: v1.0.3-beta17-patches.20260923-harden.1 (new upstream tag; adds the eMudhra intermediate CA fix)

**Docker**: Debian trixie sysroots, `golang:1.26.10-trixie` builder (pinned), `debian:trixie-slim` floating runtime with `apt upgrade`; no Python/pip/uv/mockery/swag; `/go/bin` not copied; `node:24-trixie-slim` UI builder, nginx-unprivileged 1.31.5.
**Runtime**: non-root, arbitrary-UID perms, tini PID 1, `DISABLED_REMOTE_PLUGINS=true`.
**Build/Versioning**: official beta18 tag, no extra upstream commits; `VERSION` embedded via ldflags; buildx linux/amd64.
**CVEs remediated**: **eMudhra intermediate CA compromise, CVE-2026-111943 (Go crypto/x509 denylist, fixed in Go 1.26.10/1.27.3)**: Go builder 1.26.10 + Debian `ca-certificates 20250419+deb13u1`; golang.org/x/net 0.58.0→0.60.0 (CVE-2026-78669); x/crypto 0.55→0.57, x/sys 0.47→0.48, x/text 0.41→0.42.
**Deployment config changes**: none required (chart beta15→beta18: named container ports only; `ENCRYPTION_SECRET` still required).
**Tests**: go mod verify OK; go test core/version core/config core/runner OK; Trivy 0.76.1 CRITICAL,HIGH --ignore-unfixed → backend 0 / config-ui 0; compose smoke test passed (10 plugins, /health /ready /version). eMudhra test hosts `dv-g3b-emudhra` and `dv-g3b-aaa-emudhra`: trusted by go1.26.9, rejected by go1.26.10 and by the image's OS trust store.
**Notes / residual risk**: any other Go service still on < 1.26.10 / 1.27.3 remains exposed. amd64 deploy image scanned and binary-inspected; smoke-tested on arm64 only.

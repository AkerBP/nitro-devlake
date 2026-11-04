### v1.0.3-beta19-harden.1 (harden/v1.0.3-beta19)

Date: 2026-11-04  •  Source profile: official-tag  •  Platform: linux/amd64

**Traceability**
- Upstream: v1.0.3-beta19 @ 210dcd4125e460a48895c2fdb418a35c56287cf7   (latest official marker: v1.0.3-beta19)
- Hardening branch: harden/v1.0.3-beta19   •   Git release tag: harden/v1.0.3-beta19-r1   •   Commit: a1dd2833c47915615a8c03c28864e78b86ff797d
- Backend image:  hubnitroplatformacr.azurecr.io/apache/devlake:v1.0.3-beta19-harden.1
  - digest: sha256:003c5eb03050a5096b717da40371482e1854d39808a40db17e73f41f6832b317 (manifest list sha256:22450eb5aefa184849e05a913502feba587df0852777c8ec50ce4fe2c7bbff8a)
- Config UI image: hubnitroplatformacr.azurecr.io/apache/devlake-config-ui:v1.0.3-beta19-harden.1
  - digest: sha256:f3c9d22ee5f061ef47a45962d42ce0d9b3c0adc36a216ee62a9de255ba16a7a9 (manifest list sha256:bd231749414a5cd48fa0326c4786c3f68691675fbe8a13d9ceb7c537e5d195e2)
- Plugin profile: customize,dora,gitextractor,github,github_graphql,issue_trace,linker,org,refdiff,webhook
- Supersedes: v1.0.3-beta18-harden.1 (new upstream tag)

**Docker**: Upstream beta19 builds a single static binary (CGO off, no libgit2, plugins compiled in); hardening is re-applied on that, not cherry-picked. `golang:1.27.3-trixie` builder (pinned, >= CVE-2026-111943 floor), `debian:trixie-slim` floating runtime with `apt upgrade`, no Python/pip/uv/curl/mockery/swag; `node:24-trixie-slim` UI builder, nginx-unprivileged 1.31.7 (upstream).
**Runtime**: non-root (uid 1010), arbitrary-UID perms, tini PID 1, `DISABLED_REMOTE_PLUGINS=true`, git/openssh-client/corkscrew kept for gitextractor (now shells out to git CLI).
**Build/Versioning**: official beta19 tag, no extra upstream commits; `VERSION` build arg required and embedded via ldflags (`make build VERSION=...`); `GO_PLUGINS` is passed as `DEVLAKE_PLUGINS` and restricts the generated plugin registry; buildx linux/amd64.
**CVEs remediated**: golang.org/x/net 0.59.0→0.60.0 (CVE-2026-78669). Go 1.27.3 builder and Debian `ca-certificates 20250419+deb13u1` cover CVE-2026-111943 (eMudhra).
**Deployment config changes**: none new (`ENCRYPTION_SECRET` still required). Image is much smaller; no Python remote plugins.
**Tests**: go mod verify OK; go test core/version core/config core/runner OK; Trivy 0.76.1 CRITICAL,HIGH --ignore-unfixed → backend 0 / config-ui 0 (arm64 local and pushed amd64); compose smoke test passed (10 plugins, /health /ready /version, no panic/fatal); `go version` of pushed amd64 `lake` = go1.27.3.
**Notes / residual risk**: smoke-tested on arm64 only; amd64 deploy image scanned and binary-inspected. Plugins are no longer logged as "plugin loaded <name>" (compiled in); smoke script now checks `GET /plugins`.

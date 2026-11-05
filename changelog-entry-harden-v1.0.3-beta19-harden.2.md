### v1.0.3-beta19-harden.2 (harden/v1.0.3-beta19)

Date: 2026-11-05  •  Source profile: official-tag  •  Platform: linux/amd64

**Traceability**
- Upstream: v1.0.3-beta19 @ 210dcd4125e460a48895c2fdb418a35c56287cf7
- Hardening branch: harden/v1.0.3-beta19   •   Git release tag: harden/v1.0.3-beta19-r2   •   Commit: c78758933
- Backend image:  hubnitroplatformacr.azurecr.io/apache/devlake:v1.0.3-beta19-harden.2
  - digest: sha256:ab7b329171a034384020017f250b97768c7cff8fc21ac9c967848ec6bf39c516 (manifest list sha256:0467d3ac4f34dbf6c15bb3c6faff07413c256075b12fdfeb0c4b5e8367cb75d1)
- Config UI image: hubnitroplatformacr.azurecr.io/apache/devlake-config-ui:v1.0.3-beta19-harden.2
  - digest: sha256:45783ac69117b159f649e5061b690c11250e7fb2ff1b6afc9596fa0f73658617 (manifest list sha256:0d2e95540253ca190caeb5cc8804833f2b98c883324413697934bbfa5ba7ed03)
- Supersedes: v1.0.3-beta19-harden.1

**Changes**: both images rebuilt with `--no-cache` so `apt upgrade` picks up apt 3.0.4 (CVE-2026-109518). Config UI: purged unused `nginx-module-xslt`, `nginx-module-njs`, `libxslt1.1` and `libxml2` (CVE-2026-6653, CRITICAL, no Debian fix); none of the modules were loaded. No backend source change.
**Tests**: Trivy 0.76.1 (DB 2026-11-05) on pushed amd64 images, CRITICAL,HIGH: 0 fixable in both. Compose smoke test passed (arm64 local build, same source). Pushed backend `lake` = go1.27.3, ca-certificates 20250419+deb13u1, apt 3.0.4; config-ui has no libxml2/libxslt/njs.
**Residual risk**: backend still has CRITICAL CVE-2026-60002 (openssh-client 1:10.0p1-7+deb13u4, client-side use-after-free on host key re-exchange); no fix in Debian trixie. Only reachable via SSH clone URLs in gitextractor. Other remaining HIGH findings in both images are also unfixed in Debian (util-linux, expat, curl, perl, sqlite, systemd, ncurses, zlib; libtiff/libx11 in config-ui).
**Accepted risk (2026-11-05)**: CVE-2026-60002 accepted for the backend. The only inbound path is the Azure DevOps deployment webhook to the API, protected by an API key; the UI has no ingress (port-forward only). Revisit when Debian trixie ships openssh >= 10.4.

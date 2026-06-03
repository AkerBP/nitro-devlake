# Internal Process: Building Hardened Apache DevLake Container Images

> Internal working document. This file is for our use only and should not be committed upstream.

## 1. Purpose

Apache DevLake upstream releases may contain container image vulnerabilities that are not remediated quickly enough for our internal security requirements. To comply with company security policies, we maintain our own hardened DevLake container images.

This process describes how to create, maintain, rebuild, scan, and publish hardened DevLake images from this fork.

Hardening branches are company-internal release branches. They are not intended to be merged back to `main` and are not intended for upstream contribution as-is.

## 2. Scope

This process applies to internally maintained hardened DevLake images built from either:

1. an official Apache DevLake release/beta tag; or
2. a clearly identified upstream `main` snapshot when no newer official beta/release tag exists but we need later upstream fixes.

Typical use cases:

- Rebuilding DevLake images to pick up patched OS packages.
- Updating base images to newer supported distributions.
- Updating Go, Node, or other build/runtime dependencies.
- Removing unnecessary runtime components to reduce attack surface.
- Remediating CVEs reported by container or dependency scanners.
- Including selected or current upstream fixes when official beta releases lag behind `main`.
- Producing auditable hardened images for internal deployment.

This process does **not** cover unrelated internal feature work.

## 3. Core Principles

1. **Every image must be traceable to an exact source revision**

   The source may be an official upstream tag or a specific upstream `main` commit, but it must always be recorded explicitly.

2. **Branch names should describe the source truthfully**

   If an image is built from `v1.0.3-beta12`, use a beta12 hardening branch. If it is built from upstream `main` after beta12, include `main`, a date, and/or a commit SHA in the branch and image name.

3. **Do not rebase a branch after it has produced a published image**

   Once a hardening branch has been used to build a released image, treat it as append-only. Add commits or create a new branch; do not rewrite history.

4. **Use branches for working lines and tags for immutable release points**

   Branches can move forward with new hardening fixes. Git tags identify exact source points used for published images.

5. **Use immutable image tags and record image digests**

   The image digest is the final immutable deployment/audit artifact.

6. **Keep hardening changes minimal and reviewable**

   Changes should focus on container hardening, dependency remediation, build reliability, and necessary upstream fixes.

7. **Prefer rebuilding over patching when possible**

   If CVEs are fixed by newer OS packages in a floating runtime base image, rebuild the image rather than changing application code.

8. **Separate build reproducibility from runtime patch freshness**

   Build-time inputs may be pinned for consistency, while runtime base images may intentionally float to receive OS security patches on rebuild.

9. **Keep hardening branches internal-only**

   `harden/*` branches are internal packaging/security branches. Do not merge them back to `main`. If a change is generally useful upstream, extract it into a separate clean branch/PR rather than merging the hardening branch.

10. **Internal hardening support files may live on hardening branches**

   Because hardening branches are not merged back to `main`, internal process documents, smoke-test compose files, release notes, and helper scripts may be committed there for persistence and repeatability. Do not commit secrets or environment files containing secrets.

## 4. Source Profiles

We use three source profiles.

### 4.1 Official tag profile

Use this when the image is based directly on an official upstream tag plus our hardening changes.

Example:

```text
upstream source: v1.0.3-beta12
branch:          harden/v1.0.3-beta12
image tag:       v1.0.3-beta12-harden.1
```

This is the cleanest and preferred model when an official beta/release tag is recent enough.

### 4.2 Official tag plus selected upstream fixes

Use this when the latest official tag is still the relevant product/version marker, but we need a small number of specific upstream commits from `main`.

Example:

```text
upstream base:   v1.0.3-beta12
extra fixes:     selected upstream commits after v1.0.3-beta12
branch:          harden/v1.0.3-beta12-patches-20260603
image tag:       v1.0.3-beta12-patches.20260603-harden.1
```

This is preferred over taking all of `main` if the required fixes are few, understandable, and low-risk.

The selected upstream commit SHAs must be recorded in the release notes/record.

### 4.3 Upstream main snapshot profile

Use this when no new official beta/release tag exists, but we need the current state of upstream `main` because it contains fixes we want to include.

Example:

```text
latest official tag: v1.0.3-beta12
actual source:       upstream/main at abc1234 on 2026-06-03
branch:              harden/v1.0.3-beta12-main-20260603-abc1234
image tag:           v1.0.3-beta12-main.20260603.abc1234-harden.1
```

This makes it clear that the image belongs to the beta12 lineage but is not built directly from the beta12 tag.

Do **not** call this branch only `harden/v1.0.3-beta12-2`, because that does not explain that the source moved from the beta12 tag to an unreleased upstream `main` snapshot.

## 5. Naming Conventions

### 5.1 Branch names

#### Official tag based

```text
harden/<upstream-version>
```

Examples:

```text
harden/v1.0.3-beta12
harden/v1.0.3
harden/v1.1.0
```

#### Official tag plus selected upstream fixes

```text
harden/<upstream-version>-patches-<yyyymmdd>
```

Examples:

```text
harden/v1.0.3-beta12-patches-20260603
harden/v1.0.3-beta12-patches-20260603-cvefixes
```

#### Upstream main snapshot after latest beta/release tag

```text
harden/<latest-upstream-version>-main-<yyyymmdd>-<shortsha>
```

Examples:

```text
harden/v1.0.3-beta12-main-20260603-abc1234
harden/v1.0.3-beta12-main-20260617-def5678
```

### 5.2 Git release tags

Use immutable annotated Git tags for specific hardened image releases.

#### Official tag based

```text
harden/<upstream-version>-r<N>
```

Examples:

```text
harden/v1.0.3-beta12-r1
harden/v1.0.3-beta12-r2
```

#### Official tag plus selected upstream fixes

```text
harden/<upstream-version>-patches-<yyyymmdd>-r<N>
```

Example:

```text
harden/v1.0.3-beta12-patches-20260603-r1
```

#### Upstream main snapshot

```text
harden/<latest-upstream-version>-main-<yyyymmdd>-<shortsha>-r<N>
```

Example:

```text
harden/v1.0.3-beta12-main-20260603-abc1234-r1
```

### 5.3 Container image tags

Docker tags cannot contain `/`, so use dot/dash-separated names.

#### Official tag based

```text
<registry>/<image>:<upstream-version>-harden.<N>
```

Examples:

```text
registry.example.com/devlake:v1.0.3-beta12-harden.1
registry.example.com/devlake:v1.0.3-beta12-harden.2
```

#### Official tag plus selected upstream fixes

```text
<registry>/<image>:<upstream-version>-patches.<yyyymmdd>-harden.<N>
```

Example:

```text
registry.example.com/devlake:v1.0.3-beta12-patches.20260603-harden.1
```

#### Upstream main snapshot

```text
<registry>/<image>:<latest-upstream-version>-main.<yyyymmdd>.<shortsha>-harden.<N>
```

Example:

```text
registry.example.com/devlake:v1.0.3-beta12-main.20260603.abc1234-harden.1
```

Optionally, a moving convenience tag may also be published, for example:

```text
registry.example.com/devlake:v1.0.3-beta12-harden
```

Production deployments should preferably use either an immutable image tag or the image digest:

```text
registry.example.com/devlake@sha256:<digest>
```

## 6. Repository Setup

The local repository should have both the internal fork and upstream Apache DevLake remotes configured.

Example:

```bash
git remote -v
```

Expected style:

```text
origin    <internal fork URL>
upstream  https://github.com/apache/incubator-devlake.git
```

Fetch upstream tags and branches before starting work:

```bash
git fetch upstream --tags
git fetch upstream main
git fetch origin --tags
```

## 7. Branch Creation Procedures

### 7.1 Create a branch from an official upstream tag

Use this when creating a hardened image directly from an official DevLake beta/release tag.

Example for `v1.0.3-beta12`:

```bash
UPSTREAM_TAG=v1.0.3-beta12
BRANCH=harden/${UPSTREAM_TAG}

git fetch upstream --tags
git checkout -b ${BRANCH} ${UPSTREAM_TAG}
git push -u origin ${BRANCH}
```

### 7.2 Create a branch for selected upstream fixes

Use this when starting from an official tag but cherry-picking selected upstream fixes.

```bash
UPSTREAM_TAG=v1.0.3-beta12
DATE=20260603
BRANCH=harden/${UPSTREAM_TAG}-patches-${DATE}

git fetch upstream --tags
git fetch upstream main
git checkout -b ${BRANCH} ${UPSTREAM_TAG}

# Cherry-pick only the specific upstream commits needed.
git cherry-pick <upstream-fix-commit-1> <upstream-fix-commit-2>
```

Then apply or cherry-pick our hardening commits.

Record every selected upstream fix commit in the release record.

### 7.3 Create a branch from current upstream main when no newer beta exists

Use this when the latest official beta/release tag is still `v1.0.3-beta12`, but we intentionally want the current upstream `main` source.

```bash
LATEST_OFFICIAL_TAG=v1.0.3-beta12
DATE=20260603

git fetch upstream main --tags
MAIN_SHA=$(git rev-parse --short upstream/main)
BRANCH=harden/${LATEST_OFFICIAL_TAG}-main-${DATE}-${MAIN_SHA}

git checkout -b ${BRANCH} upstream/main
git push -u origin ${BRANCH}
```

Then apply or cherry-pick our hardening commits.

Record the full upstream `main` commit SHA:

```bash
git rev-parse upstream/main
```

## 8. Applying Hardening Changes

Apply the existing hardening patch set to the branch.

Depending on the situation, this may be done by:

- cherry-picking previous hardening commits;
- manually reapplying Dockerfile changes;
- applying a maintained patch file;
- recreating the changes if upstream has changed significantly.

Example cherry-pick flow:

```bash
git cherry-pick <hardening-commit-1> <hardening-commit-2> <hardening-commit-3>
```

Relevant hardening categories:

- backend Dockerfile hardening;
- frontend/config-ui Dockerfile hardening;
- Go dependency updates;
- base image updates;
- removal of unnecessary runtime components;
- required version metadata in the build;
- scanner-driven CVE remediation.

Do not include unrelated functional changes unless explicitly required for the hardened image.

## 9. Expected Hardening Areas

### 9.1 Backend image

Common backend hardening changes may include:

- upgrading the Go builder image;
- upgrading Debian base images;
- using a minimal Debian runtime image instead of a Python runtime image;
- removing Python, pip, Poetry, and Python plugin build layers if not required;
- disabling remote/Python plugin loading when the image does not include Python support;
- removing development-only tools such as `mockery` and `swag`;
- ensuring `/go/bin` is not copied into the runtime image;
- using `tini` as PID 1;
- running as a non-root user;
- preserving OpenShift/arbitrary-UID-compatible permissions;
- adding `ca-certificates`;
- removing unused tools such as `curl` where possible;
- cleaning package-manager metadata;
- disabling libgit2 tests, examples, and CLI build output;
- validating that a required `VERSION` build argument is provided.

### 9.2 Go dependencies

Common Go dependency hardening changes may include:

- updating the Go module directive;
- updating vulnerable direct dependencies;
- updating vulnerable indirect dependencies;
- running `go mod tidy` only when appropriate;
- updating `go.sum`;
- verifying that dependency updates do not break the DevLake build.

Examples of dependency classes to watch:

- `golang.org/x/crypto`
- `golang.org/x/net`
- `golang.org/x/sys`
- `golang.org/x/text`
- database drivers such as `github.com/jackc/pgx/v5`
- logging or web framework dependencies

### 9.3 Config UI image

Common config UI hardening changes may include:

- upgrading the Node builder version;
- moving the Node builder to a newer supported Debian release;
- using `--platform=$BUILDPLATFORM` for the frontend builder stage;
- retaining the unprivileged nginx final image;
- using exec-form `CMD`;
- ensuring frontend build changes do not affect runtime architecture assumptions.

### 9.4 Production plugin profile

Our current production deployment does not require every DevLake connector plugin. The intended hardened backend image should include:

- the GitHub connector plugin(s), because GitHub is the only repository data source used;
- the webhook connector, because deployments are signalled via webhook;
- the standard/core operation plugins needed by DevLake's normal project mapping, Git extraction, DORA/refdiff, and supporting flows.

Current production plugin profile:

```text
customize,dora,gitextractor,github,github_graphql,issue_trace,linker,org,refdiff,webhook
```

Interpretation:

| Plugin | Why included |
| --- | --- |
| `github` | Primary GitHub repository connector. |
| `github_graphql` | GitHub GraphQL-backed collection used by the GitHub integration. |
| `webhook` | Deployment/event signalling endpoint. |
| `gitextractor` | Core Git extraction support. |
| `refdiff` | Commit/ref diff support used by DORA/metrics flows. |
| `dora` | DORA metric calculation. |
| `issue_trace` | Default project metric plugin for issue status/assignee history. |
| `linker` | Optional/core PR-to-issue linking metric plugin used by project settings. |
| `org` | Project/user/account mapping support. |
| `customize` | Standard supporting/custom data plugin used by DevLake flows/UI. |

Do not reduce the production image to only `github,webhook`; that omits core operation plugins required for normal DevLake functionality. In particular, new projects created by the UI enable `dora` and `issue_trace` metrics by default, and project settings can enable `linker`. Conversely, do not build all connector plugins unless there is a concrete need, because unnecessary plugins increase build time, runtime footprint, and scan surface.

## 10. Building Images

### 10.1 Backend image from official tag profile

```bash
VERSION=v1.0.3-beta12-harden.1
IMAGE=registry.example.com/devlake:${VERSION}
PRODUCTION_GO_PLUGINS=customize,dora,gitextractor,github,github_graphql,issue_trace,linker,org,refdiff,webhook

docker build \
  --pull \
  --build-arg VERSION=${VERSION} \
  --build-arg GO_PLUGINS=${PRODUCTION_GO_PLUGINS} \
  -f backend/Dockerfile \
  -t ${IMAGE} \
  backend
```

### 10.2 Backend image from upstream main snapshot profile

```bash
VERSION=v1.0.3-beta12-main.20260603.abc1234-harden.1
IMAGE=registry.example.com/devlake:${VERSION}
PRODUCTION_GO_PLUGINS=customize,dora,gitextractor,github,github_graphql,issue_trace,linker,org,refdiff,webhook

docker build \
  --pull \
  --build-arg VERSION=${VERSION} \
  --build-arg GO_PLUGINS=${PRODUCTION_GO_PLUGINS} \
  -f backend/Dockerfile \
  -t ${IMAGE} \
  backend
```

### 10.3 Backend multi-platform build

```bash
VERSION=v1.0.3-beta12-harden.1
IMAGE=registry.example.com/devlake:${VERSION}
PRODUCTION_GO_PLUGINS=customize,dora,gitextractor,github,github_graphql,issue_trace,linker,org,refdiff,webhook

docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --pull \
  --build-arg VERSION=${VERSION} \
  --build-arg GO_PLUGINS=${PRODUCTION_GO_PLUGINS} \
  -f backend/Dockerfile \
  -t ${IMAGE} \
  --push \
  backend
```

### 10.4 Config UI image

```bash
VERSION=v1.0.3-beta12-harden.1
IMAGE=registry.example.com/devlake-config-ui:${VERSION}

docker build \
  --pull \
  -f config-ui/Dockerfile \
  -t ${IMAGE} \
  config-ui
```

For multi-platform:

```bash
VERSION=v1.0.3-beta12-harden.1
IMAGE=registry.example.com/devlake-config-ui:${VERSION}

docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --pull \
  -f config-ui/Dockerfile \
  -t ${IMAGE} \
  --push \
  config-ui
```

## 11. Scanning and Validation

Before publishing or promoting an image, perform the following checks.

### 11.1 Container vulnerability scan

Run the approved company scanner.

Examples using common tools:

```bash
trivy image registry.example.com/devlake:v1.0.3-beta12-harden.1
```

or:

```bash
grype registry.example.com/devlake:v1.0.3-beta12-harden.1
```

### 11.2 Application binary/dependency scan

If available, scan the built application binary and/or dependency graph.

Example target inside the backend image:

```text
/app/bin/lake
```

### 11.3 Runtime smoke test

Run the backend image locally or in a test environment.

Example:

```bash
docker run --rm \
  -e DISABLED_REMOTE_PLUGINS=true \
  registry.example.com/devlake:v1.0.3-beta12-harden.1 \
  lake --version
```

Also verify that the application starts successfully with the expected runtime configuration.

### 11.4 Version verification

Confirm that the version embedded in the binary matches the intended hardened image version.

Expected examples:

```text
v1.0.3-beta12-harden.1
v1.0.3-beta12-main.20260603.abc1234-harden.1
```

### 11.5 Functional sanity checks

At minimum, verify:

- container starts;
- application logs are written;
- database connectivity works in a test deployment;
- config UI serves correctly;
- expected plugins load;
- remote/Python plugin behavior is understood and intentional;
- no startup errors are caused by removed runtime components.

### 11.6 Docker Compose smoke test

Before publishing a hardened image, run a local Docker Compose smoke test with a real database. This catches issues that a plain `docker run lake --version` check cannot catch, such as missing runtime libraries, missing required environment variables, DB connection failures, migration failures, and config-ui-to-backend wiring problems.

For local hardened-image testing, use the untracked helper compose file:

```text
docker-compose.harden-local.yml
```

This file should remain local/internal and should not be committed upstream. It starts:

- PostgreSQL for a real DevLake database;
- the locally built hardened backend image;
- the locally built hardened config-ui image.

Default local image names:

```text
devlake-backend:harden-local-subset
devlake-config-ui:harden-local
```

The default backend image name assumes the production plugin profile described in section 9.4.

The defaults can be overridden with environment variables:

```bash
export DEVLAKE_IMAGE=registry.example.com/devlake:v1.0.3-beta12-main.20260603.abc1234-harden.1
export DEVLAKE_CONFIG_UI_IMAGE=registry.example.com/devlake-config-ui:v1.0.3-beta12-main.20260603.abc1234-harden.1
```

Run the smoke test:

```bash
source ./devlake_env.sh

docker compose \
  -p devlake-harden-test \
  -f docker-compose.harden-local.yml \
  up -d
```

Recommended checks:

```bash
# Backend process and DB readiness
curl -fsS http://localhost:${DEVLAKE_PORT:-18080}/health
curl -fsS http://localhost:${DEVLAKE_PORT:-18080}/ready
curl -fsS http://localhost:${DEVLAKE_PORT:-18080}/version

# Config UI nginx/static serving
curl -fsS http://localhost:${CONFIG_UI_PORT:-14000}/health
curl -fsSI http://localhost:${CONFIG_UI_PORT:-14000}/

# Container status/log review
source ./devlake_env.sh
docker compose -p devlake-harden-test -f docker-compose.harden-local.yml ps
docker compose -p devlake-harden-test -f docker-compose.harden-local.yml logs --no-color --tail=150 devlake
docker compose -p devlake-harden-test -f docker-compose.harden-local.yml logs --no-color --tail=80 config-ui
```

Expected minimum results:

- Postgres is healthy.
- Backend container remains running.
- Config UI container remains running.
- `/health` returns success.
- `/ready` returns success.
- `/version` returns the expected hardened version string.
- Config UI `/health` returns HTTP 200.
- Config UI `/` returns HTTP 200.
- Logs show DevLake listening on `:8080`.
- No panic or fatal startup error is present.

Clean up afterwards:

```bash
source ./devlake_env.sh

docker compose \
  -p devlake-harden-test \
  -f docker-compose.harden-local.yml \
  down -v
```

Notes:

- `devlake_env.sh` contains secrets and must remain untracked.
- Avoid pasting `docker compose config` output into logs or tickets, because it expands environment variables and may reveal `ENCRYPTION_SECRET`.
- The local compose file may set `AUTH_ENABLED=false` for smoke testing only. Production deployments must use the organization's normal authentication configuration.
- A compose smoke test is not a full functional pipeline/E2E test, but it is the minimum runtime validation before publishing a hardened image.

### 11.7 Additional verification commands

Useful checks during hardening work:

```bash
# Verify the Go module graph/checksums after dependency remediation
cd backend
go mod verify
go list -m all > /tmp/devlake-go-list.txt
cd ..

# Run a small set of fast backend tests that do not require the full E2E setup
cd backend
go test ./core/version ./core/config ./core/runner
cd ..

# Scan local backend and UI images for critical/high fixed vulnerabilities
trivy image --severity CRITICAL,HIGH --ignore-unfixed --scanners vuln devlake-backend:harden-local-subset
trivy image --severity CRITICAL,HIGH --ignore-unfixed --scanners vuln devlake-config-ui:harden-local

# Confirm removed runtime tooling is not present in the backend image
docker run --rm devlake-backend:harden-local-subset sh -lc 'command -v python3 || true; command -v pip || true; command -v curl || true; command -v swag || true; command -v mockery || true'

# Confirm important patched Debian package versions where relevant
docker run --rm devlake-backend:harden-local-subset sh -lc 'dpkg-query -W -f="${Package} ${Version}\n" libgnutls30t64 libgssapi-krb5-2 libkrb5-3 libk5crypto3 libkrb5support0 libnghttp2-14 2>/dev/null || true'
docker run --rm --entrypoint sh devlake-config-ui:harden-local -lc 'dpkg-query -W -f="${Package} ${Version}\n" libgnutls30t64 libcap2 apache2-utils libnghttp2-14 2>/dev/null || true; /usr/sbin/nginx -v'
```

## 12. Publishing Images

After successful build and scan, push the image if it has not already been pushed.

```bash
docker push registry.example.com/devlake:v1.0.3-beta12-harden.1
```

Capture the pushed image digest:

```bash
docker inspect --format='{{index .RepoDigests 0}}' registry.example.com/devlake:v1.0.3-beta12-harden.1
```

For buildx-pushed multi-arch images, inspect with:

```bash
docker buildx imagetools inspect registry.example.com/devlake:v1.0.3-beta12-harden.1
```

Record:

- source profile: official tag, tag plus selected fixes, or upstream main snapshot;
- latest official upstream DevLake tag;
- actual upstream source ref and full commit SHA;
- selected upstream fix commits, if any;
- hardening branch;
- Git commit SHA;
- Git hardening release tag;
- image tag;
- image digest;
- build date;
- scanner used;
- scan result;
- known accepted/residual vulnerabilities, if any.

## 13. Creating Git Tags for Hardened Releases

After validating the source used for the hardened image, create an immutable annotated Git tag.

### 13.1 Official tag based

```bash
UPSTREAM_TAG=v1.0.3-beta12
RELEASE_NO=1
TAG=harden/${UPSTREAM_TAG}-r${RELEASE_NO}

git tag -a ${TAG} -m "Hardened DevLake ${UPSTREAM_TAG} release ${RELEASE_NO}"
git push origin ${TAG}
```

### 13.2 Upstream main snapshot based

```bash
LATEST_OFFICIAL_TAG=v1.0.3-beta12
DATE=20260603
MAIN_SHA=abc1234
RELEASE_NO=1
TAG=harden/${LATEST_OFFICIAL_TAG}-main-${DATE}-${MAIN_SHA}-r${RELEASE_NO}

git tag -a ${TAG} -m "Hardened DevLake ${LATEST_OFFICIAL_TAG} based on upstream main ${MAIN_SHA}, release ${RELEASE_NO}"
git push origin ${TAG}
```

Do not move or overwrite existing hardened release tags.

## 14. Rebuild Process for New CVEs

When a scanner reports new CVEs against an already hardened image, use the following decision process.

### 14.1 Determine the source of the CVE

Identify whether the CVE comes from:

1. OS packages in the runtime base image;
2. OS packages in build stages only;
3. Go dependencies;
4. Node/frontend dependencies;
5. nginx/config UI runtime image;
6. unused tooling accidentally included in the image;
7. application code or bundled assets;
8. upstream DevLake code that has already been fixed on `main` but not yet released in a beta/release tag.

### 14.2 If the CVE is in runtime OS packages

If the runtime base image is floating, first try a rebuild from the same branch.

Example:

```bash
git checkout harden/v1.0.3-beta12
git pull

VERSION=v1.0.3-beta12-harden.2
IMAGE=registry.example.com/devlake:${VERSION}

docker build \
  --pull \
  --build-arg VERSION=${VERSION} \
  -f backend/Dockerfile \
  -t ${IMAGE} \
  backend
```

Then scan again.

If the CVE is resolved, publish the new image and create a new hardened release tag.

This case does **not** require a new branch unless additional source changes are needed.

### 14.3 If the CVE is in Go dependencies

If the fix can be made directly in `backend/go.mod` without taking newer upstream application code, add a commit to the current hardening branch.

Example:

```bash
cd backend
go get <module>@<fixed-version>
go mod tidy
cd ..
```

Then rebuild, test, scan, publish a new image tag, and create a new Git release tag.

### 14.4 If the CVE is fixed on upstream main but no new beta exists

Use this decision rule:

| Situation | Recommended action |
| --- | --- |
| Only a few specific upstream commits are needed | Create a `harden/<version>-patches-<date>` branch from the latest official tag and cherry-pick those commits. |
| Current upstream `main` as a whole is desired | Create a new `harden/<version>-main-<date>-<sha>` branch from `upstream/main`. |
| Existing hardening branch has already produced an image | Do not rebase it. Create a new branch if changing the upstream base. |
| New official beta/release tag has appeared | Start a fresh `harden/<new-version>` branch from that tag. |

Do not hide an upstream source-base change behind a generic branch name such as:

```text
harden/v1.0.3-beta12-2
```

Use an explicit name instead, for example:

```text
harden/v1.0.3-beta12-main-20260603-abc1234
```

### 14.5 If the CVE is in the builder only

Confirm whether the vulnerable component exists in the final runtime image.

If the vulnerable package is only present in a non-final build stage and is not copied into the runtime image, document that finding. Some scanners only scan the final image, while others may flag build-stage content depending on the scan method.

### 14.6 If the CVE is in unused runtime tooling

Prefer removing the tooling from the runtime image rather than upgrading it.

Examples:

- remove development CLIs;
- avoid copying `/go/bin`;
- avoid installing package managers;
- avoid leaving compilers or build tools in final images.

### 14.7 If the CVE cannot be remediated immediately

Document the exception according to company policy, including:

- CVE ID;
- affected package;
- severity;
- whether it is exploitable in this image;
- reason it cannot be fixed yet;
- compensating controls;
- planned remediation date.

## 15. Runtime Base Image Policy

For hardened DevLake images, the preferred policy is:

- build/sysroot images may be pinned for reproducibility;
- final runtime base images may float to pick up security patches on rebuild;
- every published image must be identified by digest.

This means that the same Git commit may produce different image contents on different days if the base image tag has moved.

Therefore, the image digest is the final immutable artifact for audit and deployment purposes.

If stricter reproducibility is required for a specific release, pin the runtime base image by digest and update that digest deliberately when rebuilding for CVEs.

## 16. Branch Maintenance

### 16.1 Internal-only branch policy

Hardening branches are company-internal and should never be merged back to `main`.

Use them only as source lines for internal hardened image builds. The durable release artifacts are:

- the hardening branch;
- immutable Git release tags;
- immutable container image tags;
- image digests;
- scan/SBOM/release records.

If a fix made during hardening should be proposed upstream or added to the normal fork `main`, create a separate branch containing only that specific fix. Do not use a `harden/*` branch as the merge source.

Internal support artifacts may be committed to hardening branches for persistence, for example:

- internal process documents;
- local hardened-image compose files;
- sanitized build reports;
- sanitized scan summaries;
- helper scripts for repeatable internal builds/tests.

Do not commit:

- `devlake_env.sh` or other files containing secrets;
- raw reports containing credentials, tokens, internal endpoints, or sensitive metadata;
- one-off local debugging files that are not useful for future hardening work.

### 16.2 Existing official-tag hardening branch

For repeated rebuilds of the same upstream tag with no upstream source-base change:

```bash
git checkout harden/v1.0.3-beta12
git pull
```

Apply any additional CVE fixes if required, then build a new image tag:

```text
v1.0.3-beta12-harden.2
v1.0.3-beta12-harden.3
```

Do not rebase this branch after it has produced images.

### 16.3 New upstream beta/release version

For a new upstream DevLake version:

```bash
git fetch upstream --tags
git checkout -b harden/v1.0.4 v1.0.4
```

Then reapply the hardening patch set and resolve conflicts as needed.

### 16.4 New upstream main snapshot before next beta/release

If no new beta/release exists but we want current upstream `main`:

```bash
git fetch upstream main --tags
MAIN_SHA=$(git rev-parse --short upstream/main)
DATE=20260603
git checkout -b harden/v1.0.3-beta12-main-${DATE}-${MAIN_SHA} upstream/main
```

Then apply the hardening patch set and publish an image named accordingly:

```text
v1.0.3-beta12-main.20260603.<shortsha>-harden.1
```

### 16.5 Avoid mixing unrelated changes

Do not include unrelated feature work, experiments, or local debugging changes in hardened release branches.

If temporary changes are needed during investigation, remove or squash them before publishing the hardened image.

## 17. Recommended Release Record

For every hardened image release, record the following outside the upstream source tree:

```text
Product: Apache DevLake
Source profile: official-tag | tag-plus-selected-fixes | upstream-main-snapshot
Latest official upstream tag: v1.0.3-beta12
Actual upstream source ref: v1.0.3-beta12 OR upstream/main
Actual upstream source commit: <full commit SHA>
Selected upstream fix commits: <list, if applicable>

Hardening release: 1
Hardening branch: harden/v1.0.3-beta12-main-20260603-abc1234
Git hardening tag: harden/v1.0.3-beta12-main-20260603-abc1234-r1
Git commit: <final hardening branch commit SHA>

Backend image: registry.example.com/devlake:v1.0.3-beta12-main.20260603.abc1234-harden.1
Backend digest: sha256:<digest>

Config UI image: registry.example.com/devlake-config-ui:v1.0.3-beta12-main.20260603.abc1234-harden.1
Config UI digest: sha256:<digest>

Build date: YYYY-MM-DD
Builder: <person/system>
Scanner: <scanner name/version>
Scan result: <summary>
SBOM location: <location if applicable>

Notes:
- <summary of hardening changes>
- <summary of upstream fixes included after latest official beta tag>
- <accepted risks, if any>
- <follow-up actions, if any>
```

## 18. Suggested Checklist

Before promoting a hardened DevLake image:

- [ ] Source profile is clear: official tag, tag plus selected fixes, or upstream main snapshot.
- [ ] Latest official upstream tag is recorded.
- [ ] Actual upstream source commit SHA is recorded.
- [ ] Branch name truthfully reflects the source profile.
- [ ] If using upstream `main`, branch/image name includes `main`, date, and/or short SHA.
- [ ] If using selected upstream fixes, the selected upstream commit SHAs are recorded.
- [ ] No published hardening branch was rebased.
- [ ] Hardening branch is treated as company-internal and is not intended to merge back to `main`.
- [ ] Internal support docs/scripts intended for future reuse are committed to the hardening branch if useful.
- [ ] Secrets and local environment files are not committed.
- [ ] Hardening changes are limited to container/dependency/security work and necessary upstream fixes.
- [ ] Backend image builds successfully.
- [ ] Config UI image builds successfully, if applicable.
- [ ] Backend image was built with the intended production plugin profile unless there is a documented exception.
- [ ] `VERSION` build argument is set correctly.
- [ ] Backend container starts successfully.
- [ ] Config UI container starts successfully.
- [ ] Docker Compose smoke test with a real database completed successfully.
- [ ] Backend `/health`, `/ready`, and `/version` were checked.
- [ ] Config UI `/health` and `/` were checked.
- [ ] Compose logs were reviewed for panic/fatal startup errors.
- [ ] Version metadata is visible or otherwise verifiable.
- [ ] Container vulnerability scan completed.
- [ ] Dependency/binary scan completed where applicable.
- [ ] Critical/high CVEs are remediated or formally accepted.
- [ ] `go mod verify` completed after dependency updates.
- [ ] Relevant fast backend tests completed, where practical.
- [ ] Image digest recorded.
- [ ] SBOM generated if required.
- [ ] Immutable image tag pushed.
- [ ] Git release tag created.
- [ ] Release record stored outside the source repo.

## 19. Example End-to-End Flow: Official Tag Based

Example: creating the first hardened image for `v1.0.3-beta12`.

```bash
# Fetch upstream state
git fetch upstream --tags
git fetch origin --tags

# Create hardening branch from official upstream tag
git checkout -b harden/v1.0.3-beta12 v1.0.3-beta12

# Apply hardening commits/patches
git cherry-pick <hardening-commit-1> <hardening-commit-2> <hardening-commit-3>

# Build backend image
VERSION=v1.0.3-beta12-harden.1
BACKEND_IMAGE=registry.example.com/devlake:${VERSION}
PRODUCTION_GO_PLUGINS=customize,dora,gitextractor,github,github_graphql,issue_trace,linker,org,refdiff,webhook

docker build \
  --pull \
  --build-arg VERSION=${VERSION} \
  --build-arg GO_PLUGINS=${PRODUCTION_GO_PLUGINS} \
  -f backend/Dockerfile \
  -t ${BACKEND_IMAGE} \
  backend

# Scan backend image
trivy image ${BACKEND_IMAGE}

# Smoke test
docker run --rm ${BACKEND_IMAGE} lake --version

# Push image
docker push ${BACKEND_IMAGE}

# Record digest
docker inspect --format='{{index .RepoDigests 0}}' ${BACKEND_IMAGE}

# Push hardening branch
git push -u origin harden/v1.0.3-beta12

# Create immutable Git release tag
git tag -a harden/v1.0.3-beta12-r1 \
  -m "Hardened DevLake v1.0.3-beta12 release 1"

git push origin harden/v1.0.3-beta12-r1
```

## 20. Example End-to-End Flow: Upstream Main Snapshot Before Next Beta

Example: latest official tag is still `v1.0.3-beta12`, but we need current upstream `main` fixes.

```bash
# Fetch upstream state
git fetch upstream main --tags
git fetch origin --tags

LATEST_OFFICIAL_TAG=v1.0.3-beta12
DATE=20260603
MAIN_SHA=$(git rev-parse --short upstream/main)
BRANCH=harden/${LATEST_OFFICIAL_TAG}-main-${DATE}-${MAIN_SHA}

# Create branch from upstream main, not from the beta tag
git checkout -b ${BRANCH} upstream/main

# Apply hardening commits/patches
git cherry-pick <hardening-commit-1> <hardening-commit-2> <hardening-commit-3>

# Build backend image with explicit main-snapshot version string
VERSION=${LATEST_OFFICIAL_TAG}-main.${DATE}.${MAIN_SHA}-harden.1
BACKEND_IMAGE=registry.example.com/devlake:${VERSION}
PRODUCTION_GO_PLUGINS=customize,dora,gitextractor,github,github_graphql,issue_trace,linker,org,refdiff,webhook

docker build \
  --pull \
  --build-arg VERSION=${VERSION} \
  --build-arg GO_PLUGINS=${PRODUCTION_GO_PLUGINS} \
  -f backend/Dockerfile \
  -t ${BACKEND_IMAGE} \
  backend

# Scan backend image
trivy image ${BACKEND_IMAGE}

# Smoke test
docker run --rm ${BACKEND_IMAGE} lake --version

# Push image
docker push ${BACKEND_IMAGE}

# Record digest
docker inspect --format='{{index .RepoDigests 0}}' ${BACKEND_IMAGE}

# Push hardening branch
git push -u origin ${BRANCH}

# Create immutable Git release tag
TAG=${BRANCH}-r1
git tag -a ${TAG} \
  -m "Hardened DevLake ${LATEST_OFFICIAL_TAG} based on upstream main ${MAIN_SHA}, release 1"

git push origin ${TAG}
```

## 21. Example Validation Record: `v1.0.3-beta12` Main Snapshot Hardening

This is an example of the kind of validation evidence to capture for a hardened image based on upstream `main` after the latest official beta tag.

Example source profile:

```text
Latest official tag marker: v1.0.3-beta12
Actual upstream source:     upstream/main@5e9dfb8b8
Hardening branch:          harden/v1.0.3-beta12-main-20260603-5e9dfb8b8
Source hardening commit:   c088bd9ab
Documentation commits:     4b350b87d, 2c26b853e, 71151021f
Last used image counter:   harden.2
Version string:            v1.0.3-beta12-main.20260603.5e9dfb8b8-harden.2
```

Example CVE input from Trivy Operator reports:

- backend report had 3 critical CVEs;
- config-ui report had 2 critical CVEs;
- affected areas included Debian `libgnutls30t64` and Go module `github.com/jackc/pgx/v5`.

Release counter note:

- `harden.1` was superseded before publishing/deployment because the backend production plugin profile was missing `issue_trace` and `linker`.
- `harden.2` is the current image counter for this source line.
- The next image built from this source line should use `harden.3` unless a newer upstream base/tag is chosen.
- The config-ui image content did not change between `harden.1` and `harden.2`, but it was tagged with the matching `harden.2` version for deployment consistency.

Current local image IDs before push:

```text
Backend image:   hubnitroplatformacr.azurecr.io/apache/devlake:v1.0.3-beta12-main.20260603.5e9dfb8b8-harden.2
Backend ID:      sha256:df5e0332fe002aa5a628950eb80bdd80ec92e258187b146d2bd781c8aaa3f7f7
Backend plugins: customize,dora,gitextractor,github,github_graphql,issue_trace,linker,org,refdiff,webhook

Config UI image: hubnitroplatformacr.azurecr.io/apache/devlake-config-ui:v1.0.3-beta12-main.20260603.5e9dfb8b8-harden.2
Config UI ID:    sha256:052ce8151d5abb813e8f9a3e5415e86fed94ce256b20e2504a76eabef54d6a74
```

Example remediation/verification steps performed:

- created a branch from `upstream/main` because no newer beta tag existed;
- reapplied hardening changes from the previous hardening branch;
- kept the upstream Go 1.26 baseline;
- moved backend/config-ui build bases to Debian Trixie where applicable;
- ran `apt-get upgrade -y` in final runtime image stages to pull patched Debian packages;
- removed Python/Poetry/pip/runtime dev tooling from the backend image;
- removed `/go/bin` from the backend runtime image;
- disabled remote/Python plugin loading for the no-Python backend image;
- updated vulnerable Go dependencies, including `pgx/v5`, `go-jose/v3`, `jwt/v5`, `oauth2`, `logrus`, and relevant `golang.org/x/*` modules;
- ran `go mod verify`;
- ran `go list -m all` and checked key remediated module versions;
- ran fast backend tests: `go test ./core/version ./core/config ./core/runner`;
- built the backend image with all plugins during local comparison/testing;
- built the final backend image with the production plugin profile from section 9.4;
- verified the final backend image contained `customize`, `dora`, `gitextractor`, `github`, `github_graphql`, `issue_trace`, `linker`, `org`, `refdiff`, and `webhook` plugins;
- built the config-ui image;
- scanned backend and UI images with Trivy for `CRITICAL,HIGH` fixed vulnerabilities;
- confirmed backend and UI scans reported zero critical/high findings at the time of validation;
- confirmed important patched Debian package versions in the built images;
- confirmed removed backend runtime tooling such as `python3`, `pip`, `curl`, `swag`, and `mockery` was not present;
- ran the Docker Compose smoke test using PostgreSQL, backend, and config-ui;
- verified backend `/health`, `/ready`, and `/version`;
- verified config-ui `/health` and `/`;
- reviewed compose logs for startup panics/fatal errors.

Example successful local Trivy result target state:

```text
Backend image, Debian packages: 0 CRITICAL/HIGH findings
Backend image, app/bin/lake:    0 CRITICAL/HIGH findings
Config UI image:                0 CRITICAL/HIGH findings
```

## 22. Summary

For official upstream tag based images:

```text
Official upstream tag
        ↓
Internal hardening branch
        ↓
Hardening commits
        ↓
Immutable Git release tag
        ↓
Built container image
        ↓
Image scan / SBOM
        ↓
Immutable image digest
        ↓
Internal deployment
```

Example:

```text
v1.0.3-beta12
        ↓
harden/v1.0.3-beta12
        ↓
harden/v1.0.3-beta12-r1
        ↓
registry.example.com/devlake:v1.0.3-beta12-harden.1
        ↓
registry.example.com/devlake@sha256:<digest>
```

For images based on upstream `main` before the next beta/release tag:

```text
Latest official tag marker: v1.0.3-beta12
Actual source:              upstream/main@abc1234
        ↓
harden/v1.0.3-beta12-main-20260603-abc1234
        ↓
harden/v1.0.3-beta12-main-20260603-abc1234-r1
        ↓
registry.example.com/devlake:v1.0.3-beta12-main.20260603.abc1234-harden.1
        ↓
registry.example.com/devlake@sha256:<digest>
```

This provides a clear, auditable, and repeatable way to maintain hardened DevLake images while staying closely aligned with official upstream releases and clearly identifying any unreleased upstream `main` snapshots we choose to consume.

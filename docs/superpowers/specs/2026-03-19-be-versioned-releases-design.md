# BE Versioned Releases

## Problem

BE merges incremental PRs to `main`, each of which publishes a Docker image tagged `latest`. FE CI pulls `latest` for E2E tests, meaning any in-progress BE work that lands on `main` can break FE's pipeline. There's no way for BE to signal "this is a stable, tested release" vs "this is an incremental merge."

## Solution

Add release-please to BE for automated semantic versioning. Tag Docker images with semver on release. FE CI and prod deployments query the latest GitHub Release at runtime to pull a known-stable image.

## Design

### 1. BE release-please setup

Add to `backend/family-hub-api/`:

- **`release-please-config.json`** — uses `release-type: "simple"` (no pom.xml updates). Conventional commit types drive version bumps:
  - `feat:` → minor (0.1.0 → 0.2.0)
  - `fix:`, `perf:` → patch (0.1.0 → 0.1.1)
  - `chore:`, `build:`, `ci:` → no release
- **`.release-please-manifest.json`** — initial version: `0.0.0` (so the first Release PR produces `0.1.0`)
- **GitHub Actions workflow** (`release.yml`) — runs `googleapis/release-please-action@v4` on push to `main`

Behavior:
- Every merge to `main` → release-please opens or updates a Release PR with changelog
- Merging the Release PR → creates a GitHub Release with tag `vX.Y.Z`

Version milestone: `1.0.0` after Google Calendar feature is complete + cleanup.

### 2. BE CI: semver Docker image tags

**On regular `main` push** (existing `publish-docker` job): tag with `latest` + commit SHA. Unchanged.

**On GitHub Release creation**: retag the existing image — do NOT rebuild. This guarantees the released image is byte-identical to the one that passed CI.

Create a new workflow (`tag-release.yml`) triggered by `release: [published]`:

```yaml
on:
  release:
    types: [published]

jobs:
  tag-release-image:
    runs-on: ubuntu-latest
    permissions:
      packages: write
    steps:
      - name: Retag image with semver
        run: |
          VERSION=${{ github.event.release.tag_name }}
          VERSION=${VERSION#v}
          SHA=$(echo "${{ github.sha }}" | cut -c1-7)
          docker buildx imagetools create \
            ghcr.io/joe-bor/family-hub-api:${SHA} \
            --tag ghcr.io/joe-bor/family-hub-api:${VERSION}
```

Resulting GHCR tags:
```
ghcr.io/joe-bor/family-hub-api:latest     # every main push
ghcr.io/joe-bor/family-hub-api:abc1234    # every main push
ghcr.io/joe-bor/family-hub-api:0.1.0      # only on release (retag, not rebuild)
```

**Convention:** Docker tags use semver WITHOUT the `v` prefix (`0.1.0`). GitHub Release tags use WITH the prefix (`v0.1.0`). The `v` is stripped during retagging and in all consumer scripts.

### 3. FE CI: runtime version discovery

In FE's `ci.yml`, before starting the E2E container, query the public GitHub API (no auth needed for public repos — avoids cross-repo `GITHUB_TOKEN` scope issues):

```bash
BE_VERSION=$(curl -sf https://api.github.com/repos/joe-bor/family-hub-api/releases/latest \
  | jq -r '.tag_name // empty' | sed 's/^v//')
export BE_IMAGE_TAG=${BE_VERSION:-latest}
```

The explicit guard (`${BE_VERSION:-latest}`) ensures that if the API call fails or no release exists yet, it falls back to `latest` instead of passing an empty string.

Then start the container:
```bash
docker compose -f docker-compose.e2e.yml up -d --wait
```

Update `frontend/docker-compose.e2e.yml`:
```yaml
services:
  family-hub-api:
    image: ghcr.io/joe-bor/family-hub-api:${BE_IMAGE_TAG:-latest}
    # ... rest unchanged
```

Behavior:
- **In CI** — pulls the latest stable release (e.g., `0.1.0`)
- **Locally** — falls back to `latest` (no env var set), so local dev workflow is unchanged

**Note on race condition:** When a Release PR merges, the regular CI pushes `latest` + SHA, then the release event fires and retags with semver. There's a brief window where the new release exists in GitHub but the semver-tagged image hasn't been pushed to GHCR yet. The `:-latest` fallback mitigates this — if the versioned image doesn't exist, compose falls back to `latest`. In practice this window is seconds and FE CI is unlikely to coincide with it.

### 4. Prod deployments: versioned tags

Update `docker-compose.prod.yml` on the droplet to use the same env var pattern:
```yaml
image: ghcr.io/joe-bor/family-hub-api:${BE_IMAGE_TAG:-latest}
```

Updated redeploy command (uses `curl` — no dependency on `gh` CLI being installed):
```bash
BE_VERSION=$(curl -sf https://api.github.com/repos/joe-bor/family-hub-api/releases/latest \
  | jq -r '.tag_name // empty' | sed 's/^v//')
BE_IMAGE_TAG=${BE_VERSION:-latest}
ssh root@joe-bor.me "cd /opt/familyhub && \
  BE_IMAGE_TAG=$BE_IMAGE_TAG docker compose -f docker-compose.prod.yml pull && \
  BE_IMAGE_TAG=$BE_IMAGE_TAG docker compose -f docker-compose.prod.yml up -d && \
  docker image prune -f"
```

Prerequisites: `curl` and `jq` must be available on the machine running the deploy command.

Fallback to `latest` if no release exists or API call fails, preserving current behavior as a safety net.

## Scope

### In scope
- release-please config and workflow for BE
- Semver Docker image retagging in BE CI
- FE CI runtime version discovery via public GitHub API
- Prod compose file and redeploy command update
- MEMORY.md redeploy command updates

### Out of scope
- Automating prod deploys (still manual SSH)
- FE version pinning to a specific BE version (FE always uses latest *release*, not a pinned version)
- Updating BE `pom.xml` version via release-please (can be added later with extra-files config)

## Files changed

| Repo | File | Change |
|------|------|--------|
| BE | `release-please-config.json` | New — release-please config (`release-type: "simple"`) |
| BE | `.release-please-manifest.json` | New — initial version `0.0.0` |
| BE | `.github/workflows/release.yml` | New — release-please action |
| BE | `.github/workflows/tag-release.yml` | New — retag Docker image with semver on release |
| FE | `.github/workflows/ci.yml` | Add `curl` step to fetch latest BE release before E2E compose up |
| FE | `docker-compose.e2e.yml` | Use `${BE_IMAGE_TAG:-latest}` in image tag |
| Prod (droplet) | `/opt/familyhub/docker-compose.prod.yml` | Use `${BE_IMAGE_TAG:-latest}` in image tag |
| Root | MEMORY.md, deployment docs | Update redeploy commands with version query |

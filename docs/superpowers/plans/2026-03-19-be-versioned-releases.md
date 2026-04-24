# BE Versioned Releases Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add semantic versioning to BE Docker images so FE CI and prod pull a known-stable release instead of `latest`.

**Architecture:** release-please on BE creates GitHub Releases with semver tags. A separate workflow retags the existing Docker image (no rebuild). FE CI and prod query the GitHub API at runtime to discover the latest release version.

**Tech Stack:** GitHub Actions, release-please, Docker buildx imagetools, GitHub REST API

**Spec:** `docs/superpowers/specs/2026-03-19-be-versioned-releases-design.md`

---

This plan spans two repos. Tasks 1–3 are in the BE repo (`backend/family-hub-api/`). Task 4 is in the FE repo (`frontend/`). Task 5 is a manual droplet update. Task 6 updates root docs.

## File Structure

### BE repo (`backend/family-hub-api/`)
| File | Action | Purpose |
|------|--------|---------|
| `release-please-config.json` | Create | release-please configuration |
| `.release-please-manifest.json` | Create | Tracks current version (starts at `0.0.0`) |
| `.github/workflows/release.yml` | Create | Runs release-please on push to `main` |
| `.github/workflows/tag-release.yml` | Create | Retags Docker image with semver on release |

### FE repo (`frontend/`)
| File | Action | Purpose |
|------|--------|---------|
| `.github/workflows/ci.yml` | Modify | Add step to fetch latest BE release before E2E |
| `docker-compose.e2e.yml` | Modify | Use `${BE_IMAGE_TAG:-latest}` instead of hardcoded `latest` |

### Prod droplet (`/opt/familyhub/`)
| File | Action | Purpose |
|------|--------|---------|
| `docker-compose.prod.yml` | Modify | Use `${BE_IMAGE_TAG:-latest}` instead of hardcoded `latest` |

### Root workspace
| File | Action | Purpose |
|------|--------|---------|
| `MEMORY.md` (in `.claude/projects/`) | Modify | Update redeploy commands |
| `docs/deployment-guide.md` | Modify | Update redeploy commands if documented there |

---

### Task 1: Add release-please config to BE

**Repo:** `backend/family-hub-api/`
**Files:**
- Create: `release-please-config.json`
- Create: `.release-please-manifest.json`

- [ ] **Step 1: Create `release-please-config.json`**

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "packages": {
    ".": {
      "release-type": "simple",
      "bump-minor-pre-major": true,
      "bump-patch-for-minor-pre-major": true,
      "changelog-sections": [
        { "type": "feat", "section": "Features" },
        { "type": "fix", "section": "Bug Fixes" },
        { "type": "perf", "section": "Performance Improvements" },
        { "type": "refactor", "section": "Code Refactoring" },
        { "type": "docs", "section": "Documentation" },
        { "type": "test", "section": "Tests" },
        { "type": "chore", "section": "Miscellaneous Chores", "hidden": true }
      ]
    }
  }
}
```

- [ ] **Step 2: Create `.release-please-manifest.json`**

```json
{
  ".": "0.0.0"
}
```

Starting at `0.0.0` so the first Release PR produces `0.1.0`.

- [ ] **Step 3: Commit**

```bash
git add release-please-config.json .release-please-manifest.json
git commit -m "chore: add release-please config for semantic versioning"
```

---

### Task 2: Add release-please workflow to BE

**Repo:** `backend/family-hub-api/`
**Files:**
- Create: `.github/workflows/release.yml`

- [ ] **Step 1: Create `.github/workflows/release.yml`**

```yaml
name: Release

on:
  push:
    branches:
      - main

permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        with:
          config-file: release-please-config.json
          manifest-file: .release-please-manifest.json
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/release.yml
git commit -m "ci: add release-please workflow"
```

---

### Task 3: Add Docker image retag workflow to BE

**Repo:** `backend/family-hub-api/`
**Files:**
- Create: `.github/workflows/tag-release.yml`

- [ ] **Step 1: Create `.github/workflows/tag-release.yml`**

This workflow fires when release-please creates a GitHub Release. It retags the existing SHA-tagged Docker image with the semver — no rebuild, guaranteeing the released image is byte-identical to the one that passed CI.

```yaml
name: Tag Release Image

on:
  release:
    types: [published]

jobs:
  tag-release-image:
    runs-on: ubuntu-latest
    permissions:
      packages: write
    steps:
      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Wait for CI image to be available
        run: |
          SHA=$(echo "${{ github.sha }}" | cut -c1-7)
          IMAGE="ghcr.io/joe-bor/family-hub-api:${SHA}"
          for i in $(seq 1 30); do
            if docker buildx imagetools inspect "$IMAGE" > /dev/null 2>&1; then
              echo "Image $IMAGE found"
              exit 0
            fi
            echo "Waiting for $IMAGE (attempt $i/30)..."
            sleep 10
          done
          echo "::error::Image $IMAGE not found after 5 minutes"
          exit 1

      - name: Retag image with semver
        run: |
          VERSION=${{ github.event.release.tag_name }}
          VERSION=${VERSION#v}
          SHA=$(echo "${{ github.sha }}" | cut -c1-7)
          echo "Retagging ghcr.io/joe-bor/family-hub-api:${SHA} as ${VERSION}"
          docker buildx imagetools create \
            ghcr.io/joe-bor/family-hub-api:${SHA} \
            --tag ghcr.io/joe-bor/family-hub-api:${VERSION}
```

**Why the polling step:** The `release: [published]` event and the CI `publish-docker` job both trigger from the same merge to `main`. They run concurrently, so the retag can fire before the CI build finishes pushing the SHA-tagged image. The polling step waits up to 5 minutes for the image to appear in GHCR before retagging.

Note: `docker buildx imagetools create` requires GHCR login and Buildx setup. The existing `ci.yml` publish-docker job uses the same pattern.

Convention: Docker tags use semver WITHOUT `v` prefix (`0.1.0`). GitHub Release tags use WITH prefix (`v0.1.0`). The `v` is stripped by `${VERSION#v}`.

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/tag-release.yml
git commit -m "ci: add workflow to retag Docker image on release"
```

---

### Task 4: Update FE CI to pull versioned BE image

**Repo:** `frontend/`
**Files:**
- Modify: `docker-compose.e2e.yml` (image tag)
- Modify: `.github/workflows/ci.yml` (add version discovery step before "Start backend container")

- [ ] **Step 1: Update `docker-compose.e2e.yml`**

Change line 3 from:
```yaml
    image: ghcr.io/joe-bor/family-hub-api:latest
```
to:
```yaml
    image: ghcr.io/joe-bor/family-hub-api:${BE_IMAGE_TAG:-latest}
```

This uses the env var when set (CI), falls back to `latest` when not (local dev).

- [ ] **Step 2: Update `.github/workflows/ci.yml`**

Insert a new step before the existing "Start backend container" step (currently line 24). The new step queries the public GitHub API to find the latest BE release version:

Before:
```yaml
      - name: Start backend container
        run: docker compose -f docker-compose.e2e.yml up -d --wait
```

After:
```yaml
      - name: Resolve stable BE version
        run: |
          BE_VERSION=$(curl -sf https://api.github.com/repos/joe-bor/family-hub-api/releases/latest \
            | jq -r '.tag_name // empty' | sed 's/^v//')
          echo "BE_IMAGE_TAG=${BE_VERSION:-latest}" >> "$GITHUB_ENV"
          echo "Using BE image tag: ${BE_VERSION:-latest}"
      - name: Start backend container
        run: docker compose -f docker-compose.e2e.yml up -d --wait
```

`GITHUB_ENV` makes `BE_IMAGE_TAG` available to all subsequent steps, including the `docker compose` call which needs it for the `${BE_IMAGE_TAG:-latest}` substitution in the compose file.

No auth is needed — the BE repo is public, so the GitHub API serves releases without a token.

- [ ] **Step 3: Verify locally that fallback still works**

Run without `BE_IMAGE_TAG` set to confirm compose falls back to `latest`:
```bash
docker compose -f docker-compose.e2e.yml up -d --wait
docker compose -f docker-compose.e2e.yml down
```

Expected: pulls `ghcr.io/joe-bor/family-hub-api:latest` as before.

- [ ] **Step 4: Commit**

```bash
git add docker-compose.e2e.yml .github/workflows/ci.yml
git commit -m "ci: pull versioned BE image in E2E tests"
```

---

### Task 5: Update prod deployment (manual, on droplet)

**Target:** DigitalOcean droplet at `root@joe-bor.me`
**Files:**
- Modify: `/opt/familyhub/docker-compose.prod.yml` (image tag)

- [ ] **Step 1: SSH to droplet and update compose file**

```bash
ssh root@joe-bor.me
```

Edit `/opt/familyhub/docker-compose.prod.yml` — change the image line from:
```yaml
    image: ghcr.io/joe-bor/family-hub-api:latest
```
to:
```yaml
    image: ghcr.io/joe-bor/family-hub-api:${BE_IMAGE_TAG:-latest}
```

- [ ] **Step 2: Test that the compose file still works with the fallback**

```bash
cd /opt/familyhub
docker compose -f docker-compose.prod.yml up -d
docker compose -f docker-compose.prod.yml ps
```

Expected: container running using `latest` tag (no `BE_IMAGE_TAG` set, so fallback kicks in).

---

### Task 6: Update docs and MEMORY.md

**Repo:** Root workspace
**Files:**
- Modify: `.claude/projects/-Users-joe-bor-code-family-hub/memory/MEMORY.md`
- Modify: `docs/deployment-guide.md` (if redeploy commands are documented there)

- [ ] **Step 1: Update MEMORY.md redeploy commands**

Replace the current BE redeploy command:
```
- **BE**: `ssh root@joe-bor.me "cd /opt/familyhub && docker compose -f docker-compose.prod.yml pull && docker compose -f docker-compose.prod.yml up -d && docker image prune -f"`
```

With:
```
- **BE**: First fetch latest version locally, then deploy:
  ```bash
  BE_VERSION=$(curl -sf https://api.github.com/repos/joe-bor/family-hub-api/releases/latest | jq -r '.tag_name // empty' | sed 's/^v//') && \
  ssh root@joe-bor.me "cd /opt/familyhub && BE_IMAGE_TAG=${BE_VERSION:-latest} docker compose -f docker-compose.prod.yml pull && BE_IMAGE_TAG=${BE_VERSION:-latest} docker compose -f docker-compose.prod.yml up -d && docker image prune -f"
  ```
```

- [ ] **Step 2: Update `docs/deployment-guide.md` if needed**

Check if the file contains redeploy commands and update them to match the new versioned pattern.

- [ ] **Step 3: Add a note about BE versioning to MEMORY.md**

Under the Docker & CI section, add:
```
- **BE versioning**: release-please (`release-type: simple`), starting at `0.1.0`. Merging the Release PR creates a GitHub Release + retags the Docker image with semver. FE CI and prod query `api.github.com/repos/joe-bor/family-hub-api/releases/latest` at runtime.
```

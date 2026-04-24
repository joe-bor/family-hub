# Task: Publish Docker Image to GHCR on Merge to Main

## Context

We're building toward running the real BE in FE CI for end-to-end tests. The BE already has a working Dockerfile (multi-stage build, non-root user, H2 in dev profile). The next piece is: when BE tests pass on `main`, automatically build the Docker image and push it to GitHub Container Registry (GHCR) so the FE CI can `docker pull` it instead of building from source.

**You are expected to form your own plan after exploring the codebase.** Verify the current CI workflow, Dockerfile, and any existing Docker-related config before making changes. Push back if you disagree with the approach.

---

## What We Want

Update the existing CI workflow (`.github/workflows/ci.yml`) so that **after tests pass on pushes to `main`**, it:

1. Builds the Docker image using the existing Dockerfile
2. Pushes it to GHCR (`ghcr.io/joe-bor/family-hub-api`)
3. Tags it appropriately (at minimum `latest`, consider also tagging with the commit SHA for traceability)

### Key Constraints

- **Only push images on merge to `main`** — PRs should still run tests but NOT build/push images. This keeps PR CI fast and avoids publishing untested images.
- **Tests must pass before image build** — the image build step should depend on the existing `build-and-test` step succeeding. This is the quality gate.
- **Use GitHub's built-in GHCR auth** — `GITHUB_TOKEN` with appropriate permissions. No external registry credentials needed.
- **The Dockerfile already works** — don't modify it. Just wire up the CI to use it.

### Starting Points

- `.github/workflows/ci.yml` — current CI workflow (runs `./mvnw verify`)
- `Dockerfile` — the existing multi-stage build
- `.dockerignore` — already configured
- GitHub docs on GHCR: `docker/login-action`, `docker/build-push-action`, `docker/metadata-action` are the standard actions

### Things to Consider

- **Permissions** — the workflow needs `packages: write` permission to push to GHCR
- **Caching** — Docker layer caching in CI can speed up builds significantly. GitHub Actions has built-in cache support for Docker builds.
- **Image labels** — standard OCI labels (source repo, description) are nice to have but not critical

---

## What NOT to Change

- Do **not** modify the Dockerfile itself
- Do **not** change how tests run (the `./mvnw verify` step)
- Do **not** add deployment steps — this is just "build and publish the image"

---

## Git Workflow

1. **Create a new branch** from `main`: `feat/ci-docker-publish`
2. **Make atomic commits** using conventional format:
   - `ci: publish docker image to GHCR on main merge`
3. **Push and open a PR** when complete

## Acceptance Criteria

- [ ] Docker image is built and pushed to GHCR only on pushes to `main`
- [ ] PR builds still run tests but skip image publishing
- [ ] Image is tagged with at least `latest`
- [ ] Tests must pass before image build begins
- [ ] `GITHUB_TOKEN` is used for auth (no external secrets needed for GHCR)
- [ ] Workflow permissions are correctly scoped

# Deployment Guide — Family Hub

## Current production model

Family Hub production is split across two delivery paths:

- Frontend: static assets served by nginx
- Backend: Docker container on the droplet, pulling versioned images from GHCR

The backend no longer deploys as a copied JAR under `systemd`. That path is retired.

## Production layout

```text
Browser
  -> https://familyhub.joe-bor.me
  -> nginx
     -> /* serves frontend static files
     -> /api/* proxies to localhost:8080

Droplet
  -> docker compose
  -> ghcr.io/joe-bor/family-hub-api:<semver or latest>
  -> /opt/familyhub/.env
  -> /opt/familyhub/docker-compose.prod.yml

Data
  -> Neon Postgres
```

## Backend deploy source of truth

Use the backend deploy script:

```bash
cd backend/family-hub-api
./scripts/deploy.sh
```

What it does:

1. Fetches the latest backend GitHub Release from `joe-bor/family-hub-api`
2. Strips the `v` prefix and exports `BE_IMAGE_TAG`
3. SSHes to the droplet
4. Pulls the matching GHCR image
5. Restarts the `api` service with Docker Compose
6. Waits for the container health check to report healthy

If no GitHub Release exists yet, it falls back to `latest`.

## Security hardening (applied 2026-07-02)

Follow-up from the 2026-07-01 security review. These live only on the droplet — keep them in mind on any rebuild:

- **API bound to loopback only.** `/opt/familyhub/docker-compose.prod.yml` maps the API as `"127.0.0.1:8080:8080"`, not `"8080:8080"`. The bare form makes docker-proxy listen on `0.0.0.0`, exposing the API over plain HTTP on the public interface (`http://<droplet-ip>:8080`), bypassing nginx/TLS. UFW does **not** protect against this — Docker inserts iptables rules ahead of it. nginx proxies to `127.0.0.1:8080`, so the loopback binding is transparent to users.
- **Security headers** on the `familyhub.joe-bor.me` nginx server block: `Strict-Transport-Security` (HSTS, 1 year + includeSubDomains), `X-Frame-Options: SAMEORIGIN`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`.

Verify after any redeploy: `curl http://<droplet-ip>:8080/api/health` should refuse/timeout; `curl -sI https://familyhub.joe-bor.me/ | grep -i strict-transport` should show the HSTS header.

## Server prerequisites

The droplet should already have:

- Docker and the Docker Compose plugin installed
- nginx configured for `familyhub.joe-bor.me`
- `/opt/familyhub/docker-compose.prod.yml`
- `/opt/familyhub/.env`

Useful host packages:

- `curl`
- `jq`

## Manual backend deploy

If you need to run the deploy flow manually, use the same release-based logic:

```bash
BE_VERSION=$(curl -sf https://api.github.com/repos/joe-bor/family-hub-api/releases/latest \
  | jq -r '.tag_name // empty' | sed 's/^v//')

if [ -z "$BE_VERSION" ]; then
  BE_VERSION="latest"
fi

ssh root@joe-bor.me "cd /opt/familyhub && \
  export BE_IMAGE_TAG=$BE_VERSION && \
  docker compose -f docker-compose.prod.yml pull api && \
  docker compose -f docker-compose.prod.yml up -d api"
```

## Verification

After deploy:

```bash
curl -sf https://familyhub.joe-bor.me/api/health
ssh root@joe-bor.me "cd /opt/familyhub && docker compose -f docker-compose.prod.yml ps"
```

If the container does not go healthy:

```bash
ssh root@joe-bor.me "cd /opt/familyhub && docker compose -f docker-compose.prod.yml logs --tail 100 api"
```

## Rollback

Rollback is just a pinned redeploy:

```bash
ssh root@joe-bor.me "cd /opt/familyhub && \
  export BE_IMAGE_TAG=<known-good-version> && \
  docker compose -f docker-compose.prod.yml pull api && \
  docker compose -f docker-compose.prod.yml up -d api"
```

Example: `BE_IMAGE_TAG=0.3.2`

## Frontend deploy note

Frontend deployment remains owned by the frontend repo and is still a manual local-terminal deploy via `frontend/deploy.sh`.

Important FE release rule:

- Deploy only from `frontend/main`
- Deploy only when the current commit is the released FE commit tagged `family-hub-v<package.json version>`
- FE CI may run on newer `main`, but FE production shipping should follow FE releases, not arbitrary commits

This guide mainly exists so the root workspace documents the current production backend path and the release contract between FE, BE, GHCR, and the droplet.

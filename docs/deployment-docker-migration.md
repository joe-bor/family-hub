# BE Deployment Migration: systemd → Docker

## Overview

Replace the current "scp JAR + systemd" deploy with Docker containers pulling from GHCR. The FE deploy (`deploy.sh`) is unchanged. Host nginx is unchanged.

### Before vs After

```
Before:  Java 21 on droplet → systemd runs JAR → port 8080
After:   Docker on droplet → container runs image from GHCR → port 8080
```

nginx still proxies `/api/*` → `localhost:8080`. No nginx changes needed.

---

## Step 1: Install Docker on the Droplet

```bash
ssh root@joe-bor.me

# Install Docker
curl -fsSL https://get.docker.com | sh

# Verify
docker --version

# Enable Docker to start on boot
systemctl enable docker
```

---

## Step 2: Create the Production Compose File

```bash
cat > /opt/familyhub/docker-compose.prod.yml << 'EOF'
services:
  api:
    image: ghcr.io/joe-bor/family-hub-api:latest
    ports:
      - "8080:8080"
    env_file:
      - .env
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://localhost:8080/api/health"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 20s
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
EOF
```

The existing `.env` file at `/opt/familyhub/.env` already has all the secrets the container needs:
```
SPRING_PROFILES_ACTIVE=prod
JWT_SECRET=<your-prod-secret>
DB_URL=jdbc:postgresql://...neon.tech/familyhub?sslmode=require
DB_USERNAME=<neon-user>
DB_PASSWORD=<neon-password>
CORS_ALLOWED_ORIGINS=https://familyhub.joe-bor.me
```

---

## Step 3: Stop the Old Service and Start the Container

```bash
# Stop systemd service (frees port 8080)
systemctl stop familyhub-api

# Pull and start the container
cd /opt/familyhub
docker compose -f docker-compose.prod.yml up -d

# Verify health
curl -s http://localhost:8080/api/health
```

---

## Step 4: Verify from Outside

From your local machine:

```bash
# Health check
curl -s https://familyhub.joe-bor.me/api/health

# Check the app loads
curl -s -o /dev/null -w '%{http_code}' https://familyhub.joe-bor.me
```

---

## Step 5: Disable the Old Service

Once verified, prevent systemd from restarting the old JAR:

```bash
ssh root@joe-bor.me

systemctl disable familyhub-api
```

---

## Step 6: Clean Up (after you're confident)

```bash
# Remove systemd service file
rm /etc/systemd/system/familyhub-api.service
systemctl daemon-reload

# Remove old JAR
rm /opt/familyhub/family-hub-api-0.0.1-SNAPSHOT.jar

# Remove Java (only if no other projects need it)
apt remove -y openjdk-21-jdk-headless
apt autoremove -y
```

---

## New Deploy Command

After migration, deploying a new BE version:

```bash
ssh root@joe-bor.me "cd /opt/familyhub && docker compose -f docker-compose.prod.yml pull && docker compose -f docker-compose.prod.yml up -d && docker image prune -f"
```

This pulls the latest image, restarts the container, and cleans up old images in one command.

---

## Useful Commands

```bash
# View logs (live)
docker compose -f docker-compose.prod.yml logs -f api

# View recent logs
docker compose -f docker-compose.prod.yml logs --since "5m" api

# Check container status
docker compose -f docker-compose.prod.yml ps

# Restart
docker compose -f docker-compose.prod.yml restart api

# Check memory usage
docker stats --no-stream
```

---

## Rollback Plan

If something goes wrong, revert to systemd in under a minute:

```bash
# Stop container
cd /opt/familyhub
docker compose -f docker-compose.prod.yml down

# Restart old service (if not yet removed)
systemctl start familyhub-api
```

This is why Step 6 (cleanup) is separate — don't delete the JAR and service file until you're confident the container setup works.

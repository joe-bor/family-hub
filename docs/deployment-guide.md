# Deployment Guide — Family Hub

## Architecture

```
User browser
    │
    ▼
nginx (familyhub.joe-bor.me)
    ├── /api/*  → proxy_pass http://127.0.0.1:8080
    └── /*      → /var/www/familyhub (static FE)
    │
Spring Boot (port 8080, -Xmx256m)
    │
    ▼
Neon Postgres (external, free tier)
```

Everything runs on the existing $6/mo DigitalOcean droplet (`143.198.150.7`).
The FE uses a relative `/api` path — nginx handles routing to the backend.

---

## Prerequisites

- SSH access: `ssh root@joe-bor.me`
- Domain `familyhub.joe-bor.me` already pointed at the droplet
- nginx + certbot already configured (serving the FE today)
- Neon account created at https://neon.tech

---

## Step 1: Set Up Neon Postgres

1. Create a free-tier project at https://console.neon.tech
2. Name the database `familyhub`
3. Copy the connection details — you'll need:
   - **Host**: `ep-something-123456.us-east-2.aws.neon.tech`
   - **Database**: `familyhub`
   - **User**: (auto-generated)
   - **Password**: (auto-generated)
4. Construct the JDBC URL:
   ```
   jdbc:postgresql://ep-something-123456.us-east-2.aws.neon.tech/familyhub?sslmode=require
   ```

> Neon requires SSL. The `?sslmode=require` parameter is mandatory.

---

## Step 2: Set Up Swap (safety net for memory)

SSH into the droplet:

```bash
ssh root@joe-bor.me
```

Check if swap already exists:

```bash
swapon --show
```

If empty, create a 512 MB swap file:

```bash
# Create the file
fallocate -l 512M /swapfile

# Secure it (only root can read/write)
chmod 600 /swapfile

# Format as swap
mkswap /swapfile

# Activate
swapon /swapfile

# Verify
swapon --show
free -h
```

Make it permanent (survives reboots):

```bash
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

Set swappiness to low (only swap when truly needed):

```bash
sysctl vm.swappiness=10
echo 'vm.swappiness=10' >> /etc/sysctl.conf
```

---

## Step 3: Install Java 21

```bash
# Update package list
apt update

# Install OpenJDK 21 (headless — no GUI dependencies)
apt install -y openjdk-21-jdk-headless

# Verify
java -version
```

Expected output: `openjdk version "21.x.x"`

---

## Step 4: Build the JAR (on your local machine)

From your local dev machine:

```bash
cd backend/family-hub-api
./mvnw clean package -DskipTests
```

This produces: `target/family-hub-api-0.0.1-SNAPSHOT.jar`

> We skip tests here because they run against H2 locally.
> Tests should have already passed in CI / locally before this point.

---

## Step 5: Deploy the JAR

Transfer the JAR to the droplet:

```bash
scp target/family-hub-api-0.0.1-SNAPSHOT.jar root@joe-bor.me:/opt/familyhub/
```

First time only — create the directory:

```bash
ssh root@joe-bor.me "mkdir -p /opt/familyhub"
```

---

## Step 6: Create Environment File

On the droplet, create the env file:

```bash
cat > /opt/familyhub/.env << 'EOF'
SPRING_PROFILES_ACTIVE=prod
JWT_SECRET=<generate-a-strong-secret-see-below>
DB_URL=jdbc:postgresql://ep-something-123456.us-east-2.aws.neon.tech/familyhub?sslmode=require
DB_USERNAME=<neon-username>
DB_PASSWORD=<neon-password>
CORS_ALLOWED_ORIGINS=https://familyhub.joe-bor.me
JAVA_OPTS=-Xmx256m -Xms128m
EOF
```

Lock down permissions:

```bash
chmod 600 /opt/familyhub/.env
```

### Generate a JWT secret

On your local machine or the droplet:

```bash
openssl rand -base64 32
```

Use the output as the `JWT_SECRET` value. This must be a strong, random string —
the BE base64-decodes it before HMAC signing.

---

## Step 7: Create systemd Service

This runs Spring Boot as a background service that auto-restarts on failure and starts on boot.

```bash
cat > /etc/systemd/system/familyhub-api.service << 'EOF'
[Unit]
Description=Family Hub API (Spring Boot)
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/familyhub
EnvironmentFile=/opt/familyhub/.env
ExecStart=/usr/bin/java $JAVA_OPTS -jar family-hub-api-0.0.1-SNAPSHOT.jar
Restart=on-failure
RestartSec=10

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=familyhub-api

[Install]
WantedBy=multi-user.target
EOF
```

Enable and start:

```bash
# Reload systemd to pick up the new service file
systemctl daemon-reload

# Enable auto-start on boot
systemctl enable familyhub-api

# Start the service
systemctl start familyhub-api

# Check status
systemctl status familyhub-api
```

### Useful commands

```bash
# View logs (live)
journalctl -u familyhub-api -f

# View recent logs
journalctl -u familyhub-api --since "5 minutes ago"

# Restart after deploying a new JAR
systemctl restart familyhub-api

# Stop
systemctl stop familyhub-api
```

---

## Step 8: Configure nginx Reverse Proxy

Find your existing nginx config for `familyhub.joe-bor.me`:

```bash
ls /etc/nginx/sites-enabled/
cat /etc/nginx/sites-enabled/<your-config-file>
```

Add the `/api` proxy block to the existing server block. It should look something like:

```nginx
server {
    server_name familyhub.joe-bor.me;

    # API — proxy to Spring Boot
    location /api/ {
        proxy_pass http://127.0.0.1:8080/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Frontend — static files (existing config)
    location / {
        root /var/www/familyhub;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    # ... SSL config managed by certbot (leave as-is) ...
}
```

Test and reload:

```bash
nginx -t          # Validate config — fix any errors before reloading
nginx -s reload   # Apply changes
```

---

## Step 9: Verify the Backend

Test from your local machine:

```bash
# Health check — should return a response (401 is fine, means the app is running)
curl -s -o /dev/null -w "%{http_code}" https://familyhub.joe-bor.me/api/calendar/events

# Check CORS headers
curl -s -I -X OPTIONS \
  -H "Origin: https://familyhub.joe-bor.me" \
  -H "Access-Control-Request-Method: GET" \
  https://familyhub.joe-bor.me/api/calendar/events
```

Look for `Access-Control-Allow-Origin: https://familyhub.joe-bor.me` in the response headers.

Check memory on the droplet:

```bash
ssh root@joe-bor.me "free -h && echo '---' && ps aux --sort=-%mem | head -10"
```

---

## Step 10: Redeploy the Frontend

The FE needs to be rebuilt to point at the real API instead of mocks.

### Option A: Use relative `/api` path (recommended)

Since nginx proxies `/api/*` to the backend, the FE can use a relative path.
No domain or port in the URL — the browser resolves it against the current origin.

On your local machine, create `.env.production` in the frontend directory:

```bash
cat > frontend/.env.production << 'EOF'
VITE_API_BASE_URL=/api
VITE_USE_MOCK_API=false
EOF
```

> Vite automatically picks up `.env.production` when building with `NODE_ENV=production`
> (which `vite build` does by default). Your local `.env` stays untouched for dev.

Then deploy normally:

```bash
cd frontend
./deploy.sh
```

### Option B: Absolute URL

If you prefer the FE to call the backend directly (not through nginx proxy):

```
VITE_API_BASE_URL=https://familyhub.joe-bor.me/api
VITE_USE_MOCK_API=false
```

Option A is simpler — one domain, no extra CORS complexity.

---

## Redeployment Cheat Sheet

### Deploy a new backend version

```bash
# Local
cd backend/family-hub-api
./mvnw clean package -DskipTests
scp target/family-hub-api-0.0.1-SNAPSHOT.jar root@joe-bor.me:/opt/familyhub/

# Remote
ssh root@joe-bor.me "systemctl restart familyhub-api"
```

### Deploy a new frontend version

```bash
cd frontend
./deploy.sh
```

### Check everything is healthy

```bash
ssh root@joe-bor.me "systemctl status familyhub-api && free -h"
curl -s -o /dev/null -w '%{http_code}' https://familyhub.joe-bor.me/api/calendar/events
curl -s -o /dev/null -w '%{http_code}' https://familyhub.joe-bor.me
```

---

## Troubleshooting

### Spring Boot won't start

```bash
journalctl -u familyhub-api --since "2 minutes ago"
```

Common causes:
- Wrong `DB_URL` — check Neon connection string, ensure `?sslmode=require`
- Bad `JWT_SECRET` — must be a valid base64 string
- Port 8080 already in use — `lsof -i :8080`

### 502 Bad Gateway from nginx

The backend isn't running or isn't on port 8080:

```bash
systemctl status familyhub-api
curl http://127.0.0.1:8080/api/calendar/events
```

### CORS errors in browser console

- Check `CORS_ALLOWED_ORIGINS` in `/opt/familyhub/.env` matches exactly (including `https://`)
- Restart: `systemctl restart familyhub-api`

### Out of memory

```bash
free -h
vmstat 1 5          # Check si/so columns for swap thrashing
journalctl -k | grep -i "oom"   # Check for OOM kills
```

If the JVM was OOM-killed, it will auto-restart (systemd `Restart=on-failure`).
Consider stopping unused projects to free RAM.

---

## Architecture Decisions

| Decision | Rationale |
|----------|-----------|
| Neon Postgres (external) | Keeps RAM free on the droplet; free tier is sufficient |
| 512 MB swap | Safety net for JVM memory spikes during startup |
| `-Xmx256m` heap cap | Constrains Java to leave room for OS + nginx + other processes |
| Relative `/api` path | Single domain, no CORS between FE and BE, simpler SSL |
| systemd service | Auto-restart on failure, auto-start on boot, journald logging |
| nginx reverse proxy | HTTPS termination, static file serving, API routing — already running |
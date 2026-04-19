# Vaultwarden with Cloudflare Zero Trust + Traefik

> A complete guide to deploying Vaultwarden in Docker using Traefik as a reverse proxy and a Cloudflare Zero Trust Tunnel for secure external access — no open ports required.

---

## Table of Contents

1. [Architecture](#architecture)
2. [Prerequisites](#prerequisites)
3. [Directory Structure](#directory-structure)
4. [Step 1 — Shared Docker Network](#step-1--shared-docker-network)
5. [Step 2 — Traefik](#step-2--traefik)
6. [Step 3 — Cloudflare Tunnel (cloudflared)](#step-3--cloudflare-tunnel-cloudflared)
7. [Step 4 — Vaultwarden](#step-4--vaultwarden)
8. [Step 5 — Bitwarden Client on Windows](#step-5--bitwarden-client-on-windows)
9. [Alternative A: Traefik + Open Router Ports (no Cloudflare)](#alternative-a-traefik--open-router-ports-no-cloudflare)
10. [Alternative B: Basic Deployment (no Traefik, no Cloudflare)](#alternative-b-basic-deployment-no-traefik-no-cloudflare)
11. [Environment Variables Reference](#environment-variables-reference)
12. [Recommended Security Hardening](#recommended-security-hardening)
13. [Troubleshooting](#troubleshooting)

---

## Architecture

```
Internet
   │
   ▼
Cloudflare Zero Trust Tunnel
   │  (encrypted traffic — no open ports)
   ▼
cloudflared (container)
   │
   ▼  internal Docker network: homelab_net
Traefik (reverse proxy + automatic TLS via DNS challenge)
   │
   ▼
Vaultwarden (container)
```

**Why this setup?**

- No ports need to be opened on your router (not even 80 or 443).
- TLS is handled automatically by Traefik via the Cloudflare DNS challenge.
- Traffic never hits your server's public IP directly.
- Healthchecks on all containers for resilience.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Linux server | Ubuntu 22.04/24.04 recommended |
| Docker + Docker Compose v2 | Use `docker compose` (no hyphen) |
| Your own domain | Managed through Cloudflare |
| Cloudflare account | Free plan is enough |
| Domain DNS on Cloudflare | Nameservers pointing to Cloudflare |

### Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
```

---

## Directory Structure

```
homelab/
├── cloudflare/
│   ├── docker-compose.yml
│   └── .env
├── traefik/
│   ├── docker-compose.yml
│   ├── .env
│   └── data/
│       ├── traefik.yml
│       └── acme.json          # Create this as an empty file manually
└── vaultwarden/
    ├── docker-compose.yml
    ├── .env
    └── vw-data/               # Persistent Vaultwarden data
```

---

## Step 1 — Shared Docker Network

All containers communicate over an external Docker network called `homelab_net`. Create it once:

```bash
docker network create homelab_net
```

---

## Step 2 — Traefik

Traefik is the reverse proxy. It manages TLS automatically via Cloudflare's DNS challenge, so **port 443 does not need to be open to the internet**.

### `traefik/data/traefik.yml`

```yaml
api:
  dashboard: true
  insecure: false

ping: {}

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: homelab_net

certificatesResolvers:
  letsencrypt:
    acme:
      email: you@example.com      # ← change this
      storage: /etc/traefik/acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "1.0.0.1:53"

accesslog:
  fields:
    names:
      StartUTC: drop
```

### `traefik/docker-compose.yml`

```yaml
services:
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - CF_API_EMAIL=${CLOUDFLARE_EMAIL}
      - CF_DNS_API_TOKEN=${CLOUDFLARE_DNS_API_TOKEN}
    healthcheck:
      test: ["CMD", "traefik", "healthcheck", "--ping"]
      interval: 30s
      timeout: 10s
      retries: 3
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data/traefik.yml:/etc/traefik/traefik.yml:ro
      - ./data/acme.json:/etc/traefik/acme.json
      - /etc/localtime:/etc/localtime:ro
    networks:
      - homelab_net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.yourdomain.com`)"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls.certresolver=letsencrypt"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.middlewares.auth-dashboard.basicauth.users=${TRAEFIK_DASHBOARD_CREDENTIALS}"
      - "traefik.http.routers.dashboard.middlewares=auth-dashboard"

networks:
  homelab_net:
    external: true
```

### `traefik/.env`

```env
CLOUDFLARE_EMAIL=you@example.com
CLOUDFLARE_DNS_API_TOKEN=your_cloudflare_dns_api_token
TRAEFIK_DASHBOARD_CREDENTIALS=admin:$$apr1$$...   # htpasswd -nb user password
```

> **Generate dashboard credentials:**
>
> ```bash
> htpasswd -nb admin your_password
> # Every $ must be doubled in the .env file: $apr1$ → $$apr1$$
> ```

### Create `acme.json` and start Traefik

```bash
touch traefik/data/acme.json
chmod 600 traefik/data/acme.json
cd traefik && docker compose up -d
```

### Cloudflare DNS Token

In the Cloudflare dashboard → **My Profile → API Tokens → Create Token**:

- Template: **Edit zone DNS**
- Permissions: `Zone → DNS → Edit`
- Zone Resources: your specific domain

---

## Step 3 — Cloudflare Tunnel (cloudflared)

The tunnel lets you expose internal services to the internet without opening any ports.

### Create the tunnel in Cloudflare

1. Go to [one.dash.cloudflare.com](https://one.dash.cloudflare.com) → **Networks → Tunnels**
2. Click **Create a tunnel** → Type: **Cloudflared**
3. Give it a name (e.g. `homelab`)
4. Copy the **Tunnel Token** shown in the connector installation step
5. Under **Public Hostname**, add:
   - **Subdomain:** `vault` | **Domain:** `yourdomain.com` | **Service:** `https://traefik:443`
   - Enable **No TLS Verify** in the advanced service options (Traefik uses an internal cert)

> **Recommended alternative:** point to `http://traefik:80` and let Traefik handle the HTTPS redirect internally. This avoids needing "No TLS Verify".

### `cloudflare/docker-compose.yml`

```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    command: tunnel --no-autoupdate run
    environment:
      - PUID=1000
      - PGID=1000
      - TUNNEL_TOKEN=${CLOUDFLARE_TUNNEL_TOKEN}
    networks:
      - homelab_net

networks:
  homelab_net:
    external: true
```

### `cloudflare/.env`

```env
CLOUDFLARE_TUNNEL_TOKEN=your_tunnel_token_here
```

```bash
cd cloudflare && docker compose up -d
```

---

## Step 4 — Vaultwarden

### `vaultwarden/docker-compose.yml`

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    networks:
      - homelab_net
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
      - DOMAIN=${DOMAIN}
      - ADMIN_TOKEN=${ADMIN_TOKEN}
      - SIGNUPS_ALLOWED=${SIGNUPS_ALLOWED}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/alive"]
      interval: 60s
      timeout: 10s
      retries: 3
      start_period: 30s
    volumes:
      - ./vw-data:/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.vaultwarden.rule=Host(`vault.yourdomain.com`)"
      - "traefik.http.routers.vaultwarden.entrypoints=websecure"
      - "traefik.http.routers.vaultwarden.tls.certresolver=letsencrypt"
      - "traefik.http.services.vaultwarden.loadbalancer.server.port=80"

networks:
  homelab_net:
    external: true
```

### `vaultwarden/.env`

```env
DOMAIN=https://vault.yourdomain.com
ADMIN_TOKEN=a_very_strong_token_change_this
SIGNUPS_ALLOWED=false
```

> **Generate a secure ADMIN_TOKEN:**
>
> ```bash
> openssl rand -base64 48
> ```

> ⚠️ Set `SIGNUPS_ALLOWED=true` the first time to create your account, then flip it back to `false` and restart the container.

### Start Vaultwarden

```bash
mkdir -p vaultwarden/vw-data
cd vaultwarden && docker compose up -d
```

### Verify everything is working

```bash
docker ps
docker logs vaultwarden
docker logs traefik
docker logs cloudflared
```

Visit `https://vault.yourdomain.com` — you should see the Vaultwarden login screen.

The admin panel is at `https://vault.yourdomain.com/admin`.

---

## Step 5 — Bitwarden Client on Windows

1. Download the desktop client from [bitwarden.com/download](https://bitwarden.com/download/)
2. Install and open the app
3. On the login screen, click the ⚙️ gear icon (top-left corner)
4. Change **Server URL** to `https://vault.yourdomain.com`
5. Save and return to the login screen
6. Create your account or sign in with the credentials you set up in Vaultwarden

---

## Alternative A: Traefik + Open Router Ports (no Cloudflare)

Use this if you can't or don't want to use Cloudflare Zero Trust. **This requires opening ports 80 and 443 on your router and having a static (or DDNS-managed) public IP.**

| Component | Main guide | This alternative |
|---|---|---|
| Exposure | Cloudflare Tunnel | Ports 80/443 open |
| DNS | Managed by Cloudflare | A record at your DNS provider |
| Security | No exposed ports | Port 443 exposed |
| cloudflared | Yes | **No** |

### What to change

**1. Open ports on your router**

- Forward TCP port 80 → server IP, port 80
- Forward TCP port 443 → server IP, port 443

**2. DNS**

Create an `A` record at your DNS provider:

```
vault.yourdomain.com  →  YOUR_PUBLIC_IP
```

If your public IP changes, set up a DDNS client like `ddclient`.

**3. Switch to HTTP challenge in `traefik.yml`**

```yaml
certificatesResolvers:
  letsencrypt:
    acme:
      email: you@example.com
      storage: /etc/traefik/acme.json
      httpChallenge:                  # ← replace dnsChallenge with this
        entryPoint: web
```

Remove `CF_API_EMAIL` and `CF_DNS_API_TOKEN` from Traefik's compose environment.

**4. Skip cloudflared entirely**

Don't deploy `cloudflare/docker-compose.yml` — it's not needed here.

**5. Expose ports in `traefik/docker-compose.yml`**

Add the `ports` block to the Traefik service (not needed with a tunnel):

```yaml
ports:
  - "80:80"
  - "443:443"
```

Everything else — Vaultwarden config and the Windows client setup — is identical to the main guide.

---

## Alternative B: Basic Deployment (no Traefik, no Cloudflare)

The simplest possible setup: Vaultwarden exposed directly, no reverse proxy. Good for local-only access or quick testing. **Not recommended for production.**

This approach requires ports 80/443 open on your router and a valid TLS certificate (you'll need to handle this yourself, e.g. via Certbot).

### `vaultwarden/docker-compose.yml`

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      - TZ=Europe/London
      - DOMAIN=https://vault.yourdomain.com
      - ADMIN_TOKEN=a_very_strong_token_change_this
      - SIGNUPS_ALLOWED=false
    volumes:
      - ./vw-data:/data
      - ./ssl:/ssl:ro          # Mount your TLS certs here
    ports:
      - "80:80"
      - "443:443"
```

For certificate management, use Certbot:

```bash
certbot certonly --standalone -d vault.yourdomain.com
```

Then configure Vaultwarden to use the generated certs via the `ROCKET_TLS` environment variable, or place them in a path Vaultwarden can read.

> ⚠️ Without Traefik or Cloudflare, you are responsible for certificate renewal and managing direct exposure of your server to the internet.

---

## Environment Variables Reference

| File | Variable | Description |
|---|---|---|
| `traefik/.env` | `CLOUDFLARE_EMAIL` | Your Cloudflare account email |
| `traefik/.env` | `CLOUDFLARE_DNS_API_TOKEN` | API token with DNS Edit permissions |
| `traefik/.env` | `TRAEFIK_DASHBOARD_CREDENTIALS` | user:htpasswd hash |
| `cloudflare/.env` | `CLOUDFLARE_TUNNEL_TOKEN` | Zero Trust tunnel token |
| `vaultwarden/.env` | `DOMAIN` | Full URL of your Vaultwarden instance |
| `vaultwarden/.env` | `ADMIN_TOKEN` | Token for the `/admin` panel |
| `vaultwarden/.env` | `SIGNUPS_ALLOWED` | `true`/`false` |

---

## Recommended Security Hardening

### 1. Security headers middleware in Traefik

Add reusable HTTP security headers as a middleware. In `traefik.yml`, add a file provider:

```yaml
providers:
  docker: ...
  file:
    filename: /etc/traefik/dynamic.yml
```

Create `traefik/data/dynamic.yml`:

```yaml
http:
  middlewares:
    secureHeaders:
      headers:
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        contentTypeNosniff: true
        frameDeny: true
        referrerPolicy: "strict-origin-when-cross-origin"
```

Apply it in Vaultwarden's labels:

```yaml
- "traefik.http.routers.vaultwarden.middlewares=secureHeaders"
```

### 2. Rate limiting on the admin panel

```yaml
http:
  middlewares:
    adminRateLimit:
      rateLimit:
        average: 10
        burst: 20
```

### 3. Restrict `/admin` to local IPs only

```yaml
http:
  middlewares:
    adminIpAllowList:
      ipAllowList:
        sourceRange:
          - "192.168.1.0/24"   # Local network only
```

### 4. Automated backups of `vw-data`

```bash
# Daily cron at 3:00 AM
0 3 * * * tar -czf /backup/vaultwarden-$(date +\%Y\%m\%d).tar.gz /homelab/vaultwarden/vw-data
```

---

## Troubleshooting

### Certificate not being generated

- Check that your Cloudflare DNS token has `Zone → DNS → Edit` permissions for your domain.
- Inspect the logs: `docker logs traefik | grep -i acme`
- Make sure `acme.json` has the right permissions: `chmod 600 traefik/data/acme.json`

### Vaultwarden is not reachable

- Check that the tunnel shows as **Healthy** in the Cloudflare Zero Trust dashboard.
- Verify all three containers are on `homelab_net`: `docker network inspect homelab_net`
- Make sure the hostname in Vaultwarden's Traefik label exactly matches the one configured in the tunnel.

### Bitwarden client says "invalid server URL"

- Make sure you include `https://` in the server URL.
- Verify the TLS certificate is valid by opening the URL in a browser first.

### WebSocket errors

- The `WEBSOCKET_ENABLED=true` variable enables real-time notifications in Vaultwarden.
- If you're using Vaultwarden `latest` (≥ 1.29.0), WebSocket is built into port 80 — **no additional Traefik configuration is needed**.

---

*Documentation maintained by [iamlaura.dev](https://iamlaura.dev) · MIT License*

# Matrix Self-Hosted — Deployment Guide

## What is Matrix?

Matrix is an open, decentralized, end-to-end encrypted communication protocol. Unlike WhatsApp, Telegram, or Discord, it doesn't depend on any central company — anyone can run their own server and communicate with users on other servers, the same way email works.

The most widely used server implementation is **Synapse**, developed by Element. The most popular client is **Element**, available for web, desktop, iOS, and Android.

Organizations like the French military, the German government, and the European Commission use Matrix for exactly this reason: full control over their communications, with no dependency on third parties.

---

## Architecture

This stack runs four Docker containers:

**Synapse** — The Matrix server. Manages users, rooms, messages, and cryptographic keys. Exposes its API on port 8008 internally, but is never exposed directly to the internet — Traefik acts as the intermediary.

**PostgreSQL** — The database. Stores messages, users, keys, and room state. Synapse can run on SQLite, but PostgreSQL is required for any production use. It must be initialized with locale `C` and encoding `UTF8` — Synapse will reject the connection otherwise.

**Element Web** — The web client. A static React app served by Nginx. It has no server logic; it connects to Synapse from the user's browser. A `config.json` file tells it which server to use by default.

**Tor** — Runs a v3 hidden service that exposes Synapse as a `.onion` address. This provides a second access path that's completely independent of domains, DNS, and Cloudflare. It's designed for censorship-resistant access, not everyday use.

### Traffic flow

```
User → Cloudflare → Tunnel → Traefik → Synapse / Element
```

Traefik integrates with Docker, reads labels on each container, generates routing rules automatically, and manages TLS certificates via Cloudflare's DNS challenge. Cloudflare Tunnel establishes an outbound connection from your machine to Cloudflare, so no ports need to be open on your router.

---

## Alternative: No Traefik, No Cloudflare

If you'd rather not depend on Cloudflare or want a simpler setup, that's perfectly doable. The changes from this guide would be:

- Remove all Traefik labels from the containers and map ports directly (`8008:8008` for Synapse, `80:80` and `443:443` for Element)
- Forward those ports on your router to the machine running the containers
- Create DNS `A` records at your domain provider pointing to your public IP
- Manage TLS certificates manually with Certbot, or use Caddy as an alternative reverse proxy that handles them automatically

The result is functionally identical. The trade-offs: your public IP is exposed, you'll need a static IP or a DDNS service (e.g. DuckDNS), and certificate management is either manual or requires an extra tool.

```yaml
# Instead of Traefik labels, map ports directly:
synapse:
  ports:
    - "8008:8008"

element:
  ports:
    - "80:80"
    - "443:443"
```

---

## What You'll Have at the End

- Matrix/Synapse backed by PostgreSQL
- Element Web at `https://element.yourdomain.com`
- Synapse at `https://matrix.yourdomain.com`
- Everything routed through your existing Traefik + Cloudflare Tunnel setup
- A Tor v3 hidden service running in parallel
- Mobile app connected to your server

---

## Step 1: Folder Structure

```bash
mkdir -p matrix/{synapse,element,postgres,tor/hidden_service}
cd matrix
```

---

## Step 2: Create the `.env` File

```bash
nano .env
```

```env
# PostgreSQL password
POSTGRES_PASSWORD=<strong_password>

# Registration secret
REGISTRATION_SECRET=<random_key>
```

Lock down the file permissions:

```bash
chmod 600 .env
```

To generate a secure random key:

```bash
openssl rand -base64 32
```

> **Why a separate `.env`?** Passwords never live inside `docker-compose.yml`. The `.env` lets you change credentials without touching the compose file, and you can add it to `.gitignore` so secrets never leave the server.

> **Avoid special characters in passwords.** The password travels through three systems with different parsing rules: the Bash shell (reading `.env`), the YAML in `homeserver.yaml`, and PostgreSQL's connection URL. The symbol `@` in a URL is read as a user/host separator. `#` in YAML starts a comment. `$` in Bash is interpreted as a variable. Stick to letters and numbers to avoid all of these pitfalls.

---

## Step 3: Generate Synapse's Initial Configuration

This command starts Synapse in `generate` mode — it only creates config files and cryptographic keys, then exits. No service is started. **Run it exactly once.**

```bash
docker run -it --rm \
  --mount type=bind,src=$(pwd)/synapse,dst=/data \
  -e SYNAPSE_SERVER_NAME=matrix.yourdomain.com \
  -e SYNAPSE_REPORT_STATS=yes \
  matrixdotorg/synapse:latest generate
```

Verify the files were created:

```bash
ls -la synapse/
# You should see three files:
# homeserver.yaml, matrix.yourdomain.com.signing.key, matrix.yourdomain.com.log.config
```

> **Why `generate` instead of writing the YAML by hand?** Synapse generates a unique cryptographic key pair that identifies your server on the Matrix network. Without it, no client can verify that messages come from your server. The `generate` command handles all of this correctly in one step.

---

## Step 4: Replace `homeserver.yaml`

Open the generated file and **replace all its content** with the following. The comments explain each decision.

```bash
sudo nano synapse/homeserver.yaml
```

```yaml
# ═══════════════════════════════════════════════════════════════
# homeserver.yaml — matrix.yourdomain.com
# ═══════════════════════════════════════════════════════════════
server_name: "matrix.yourdomain.com"

# public_baseurl is the URL clients use to connect.
# Must match exactly the domain Traefik exposes.
public_baseurl: "https://matrix.yourdomain.com/"

# ── Listener ────────────────────────────────────────────────────
# Port 8008: internal HTTP only, never exposed directly.
# Traefik terminates HTTPS and forwards here.
# x_forwarded: true — mandatory with a reverse proxy. Without this,
# Synapse sees Traefik's IP as the origin of all connections.
# bind_addresses ['::'] — accepts both IPv4 and IPv6 inside the container.
listeners:
  - port: 8008
    tls: false
    type: http
    x_forwarded: true
    bind_addresses: ['::']
    resources:
      - names: [client, federation]
        compress: false

# ── Database ─────────────────────────────────────────────────────
# The password is read from the POSTGRES_PASSWORD environment variable
# passed in docker-compose
database:
  name: psycopg2
  args:
    user: synapse
    password: <strong_password__same_as_.env>
    database: synapse
    host: postgres
    cp_min: 5
    cp_max: 10

# ── Paths — must be inside /data (the mounted volume) ────────────
media_store_path: /data/media_store
signing_key_path: "/data/matrix.yourdomain.com.signing.key"
log_config: "/data/matrix.yourdomain.com.log.config"

# ── User registration ─────────────────────────────────────────────
# enable_registration: false = nobody can register freely.
# registration_shared_secret = allows creating admin users with the
# register_new_matrix_user command even when registration is closed.
enable_registration: false
registration_shared_secret: <random_key__same_as_.env>

# ── Trusted key servers ───────────────────────────────────────────
# Removes the trusted_key_servers WARNING from the logs.
trusted_key_servers:
  - server_name: "matrix.org"
suppress_key_server_warning: true

# ── Statistics ────────────────────────────────────────────────────
# Must be boolean true/false, NOT "yes"/"no".
report_stats: true
```

---

## Step 5: Set Permissions for Synapse and Tor

### Synapse (UID 991)

Synapse runs inside the container as user `synapse` with UID 991. The `./synapse` directory was created by your Linux user. If Synapse can't write to `/data`, it won't start.

```bash
sudo mkdir -p synapse/media_store
sudo chown -R 991:991 synapse/
```

### Tor (UID 100)

The `osminogin/tor-simple` image runs Tor as user `tor` with UID 100. Tor requires that `HiddenServiceDir` is owned by that user and has permissions of exactly `700`. If the directory belongs to your user (UID 1000) with `700`, Tor still can't read it because the owner is wrong.

```bash
# Transfer ownership to UID 100 (the tor user inside the container)
sudo chown -R 100:100 tor/hidden_service

# 700 = only the owner can read/write/execute
sudo chmod 700 tor/hidden_service

# Verify
ls -la tor/
# Expected: drwx------ 2 100 100 ... hidden_service
```

> **Why is Tor so strict?** A v3 hidden service stores private cryptographic keys in this directory — the private key corresponding to your `.onion` address. If those keys were readable by other users, anyone could impersonate your server. Tor refuses to start if permissions aren't exactly right, by design.

---

## Step 6: Element Web `config.json`

```bash
nano element/config.json
```

```json
{
  "default_server_config": {
    "m.homeserver": {
      "base_url": "https://matrix.yourdomain.com",
      "server_name": "matrix.yourdomain.com"
    }
  },
  "brand": "Element",
  "disable_guests": true,
  "default_federate": false
}
```

> **Why `default_server_config` and not `default_server_name`?** The official Element docs recommend `default_server_config` as the more robust method. `default_server_name` only works if the server also supports auto-discovery via `.well-known`, which isn't configured in this setup.

---

## Step 7: `torrc`

```bash
nano tor/torrc
```

```
DataDirectory /var/lib/tor

# HiddenServiceDir: where Tor stores the hidden service keys.
# Must be /var/lib/tor/hidden_service/ inside the container,
# which maps to ./tor/hidden_service on the host.
HiddenServiceDir /var/lib/tor/hidden_service/

# Port 80 of the hidden service forwards to port 8008 on Synapse.
# "matrix-synapse" is the container name on the matrix-internal network.
HiddenServicePort 80 matrix-synapse:8008

# v3 hidden services only (more secure, 56-character addresses)
HiddenServiceVersion 3
```

---

## Step 8: `docker-compose.yml`

```bash
nano docker-compose.yml
```

```yaml
# ════════════════════════════════════════════════════════════════
# Matrix Stack: Synapse + Element + PostgreSQL + Tor
# Domain: matrix.yourdomain.com / element.yourdomain.com
# ════════════════════════════════════════════════════════════════
services:
  # ── PostgreSQL ───────────────────────────────────────────────────
  # POSTGRES_INITDB_ARGS forces locale C and UTF8 encoding.
  # Without this the cluster is created with en_US.utf8 and Synapse fails.
  # Password comes from .env. Only accessible on the internal network.
  postgres:
    image: postgres:16-alpine
    container_name: matrix-postgres
    restart: unless-stopped
    networks:
      - matrix-internal
    environment:
      POSTGRES_USER: synapse
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: synapse
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --lc-collate=C --lc-ctype=C"
      TZ: Europe/Madrid
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U synapse -d synapse"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 15s
    volumes:
      - ./postgres:/var/lib/postgresql/data

  # ── Synapse ──────────────────────────────────────────────────────
  # depends_on with service_healthy guarantees Synapse waits until
  # PostgreSQL is ready before attempting to connect
  synapse:
    image: matrixdotorg/synapse:latest
    container_name: matrix-synapse
    restart: unless-stopped
    networks:
      - matrix-internal
      - homelab_net
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      TZ: Europe/Madrid
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      REGISTRATION_SECRET: ${REGISTRATION_SECRET}
    volumes:
      - ./synapse:/data
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=homelab_net"
      - "traefik.http.routers.matrix.rule=Host(`matrix.yourdomain.com`)"
      - "traefik.http.routers.matrix.entrypoints=websecure"
      - "traefik.http.routers.matrix.tls.certresolver=letsencrypt"
      - "traefik.http.services.matrix.loadbalancer.server.port=8008"

  # ── Element Web ──────────────────────────────────────────────────
  # Only needs the proxy network for Traefik to reach it.
  # config.json mounted read-only.
  element:
    image: vectorim/element-web:latest
    container_name: matrix-element
    restart: unless-stopped
    networks:
      - homelab_net
    volumes:
      - ./element/config.json:/app/config.json:ro
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=homelab_net"
      - "traefik.http.routers.element.rule=Host(`element.yourdomain.com`)"
      - "traefik.http.routers.element.entrypoints=websecure"
      - "traefik.http.routers.element.tls.certresolver=letsencrypt"
      - "traefik.http.services.element.loadbalancer.server.port=80"

  # ── Tor ──────────────────────────────────────────────────────────
  # Internal network only. Does not need Traefik.
  # The hidden_service volume must belong to UID 100 (see Step 5).
  tor:
    image: osminogin/tor-simple
    container_name: matrix-tor
    restart: unless-stopped
    networks:
      - matrix-internal
    depends_on:
      - synapse
    volumes:
      - ./tor/torrc:/etc/tor/torrc:ro
      - ./tor/hidden_service:/var/lib/tor/hidden_service

networks:
  # Traefik's network — already exists in the homelab, not created here
  homelab_net:
    external: true
  # Internal network exclusive to this stack.
  # Synapse, PostgreSQL and Tor can see each other but are not reachable from outside.
  matrix-internal:
    name: matrix-internal
```

---

## Step 9: Start the Containers

```bash
docker compose up -d
```

Wait about 30 seconds, then check the logs:

```bash
docker compose logs --tail 50
```

### What healthy logs look like

**Synapse** — background index jobs running, no errors:
```
matrix-synapse | synapse.storage.background_updates - Running background update 'delayed_events_idx'...
matrix-synapse | synapse.storage.background_updates - No more background updates to do.
```

**PostgreSQL** — initialized and accepting connections:
```
matrix-postgres | PostgreSQL init process complete; ready for start up.
matrix-postgres | database system is ready to accept connections
```

**Tor** — bootstrapping to 100%:
```
matrix-tor | Bootstrapped 100% (done): Done
```

**Element** — serving the app and responding to requests:
```
matrix-element | start worker process 39
matrix-element | "GET /config.json HTTP/1.1" 200
```

### If something is failing

```bash
# Synapse: database connection or config errors
docker logs matrix-synapse --tail 30

# PostgreSQL: initialization errors
docker logs matrix-postgres --tail 20

# Tor: permission errors
docker logs matrix-tor --tail 20

# Status of all containers
docker compose ps
```

---

## Step 10: Create the First Admin User

Once Synapse is up and healthy (confirm with `docker compose ps`):

```bash
docker exec -it matrix-synapse register_new_matrix_user \
  -c /data/homeserver.yaml \
  http://localhost:8008
```

Answer the prompts:

```
New user localpart [root]: your_username
Password:                          ← choose something strong
Confirm password:
Make admin [no]: yes
Sending registration request...
Success!
```

---

## Step 11: Verify in the Browser

```
https://matrix.yourdomain.com/_matrix/client/versions
# → Should return a JSON with {"versions": [...]}

https://element.yourdomain.com
# → Should load Element Web with the homeserver pre-filled
```

If the first URL works but the second doesn't, the issue is the Traefik label on the `element` container. Check with:

```bash
docker inspect matrix-element | grep -A5 Labels
```

Or browse to the Traefik dashboard to see how the router is configured.

---

## Step 12: Find the `.onion` Address

Tor takes 1–3 minutes to register on the network the first time:

```bash
sudo cat tor/hidden_service/hostname
```

Save that address — you'll need it to connect via Tor.

---

## Step 13: Set Up the Mobile App

1. Install **Element** from the App Store (iOS) or Play Store / F-Droid (Android)
2. On first launch, tap **"Sign in"** and change the server before logging in
3. Enter: `https://matrix.yourdomain.com`
4. Log in with the admin user you created
5. Send a message from mobile → verify it appears in Element Web
6. Send a message from Element Web → verify it appears on mobile

---

## Step 14: Connect via Tor

Download **Tor Browser** from https://www.torproject.org/download/

1. Open Tor Browser
2. Navigate to `http://your_onion_address.onion/_matrix/client/versions`
3. You should see the Synapse API JSON response

Or from a terminal with `torsocks`:

```bash
sudo apt install torsocks -y
torsocks curl http://YOUR_ONION_ADDRESS.onion/_matrix/client/versions
```

---

## Creating Additional Users

```bash
docker exec -it matrix-synapse register_new_matrix_user \
  -c /data/homeserver.yaml \
  http://localhost:8008
```

Run the command, answer the prompts, and choose whether to make the new user an admin.

---

*Documentation maintained by [iamlaura.dev](https://iamlaura.dev) · MIT License*
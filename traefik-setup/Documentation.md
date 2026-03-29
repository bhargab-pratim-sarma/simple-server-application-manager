# Traefik v3 — Complete Setup Guide & Reference

> **Server**: `31.97.233.65` | **Dashboard**: [https://tf.namani.in/dashboard/](https://tf.namani.in/dashboard/) | **Login**: `admin` / `admin`

This is your complete, beginner-friendly reference for the Traefik reverse proxy running on your server. Every file, every line, every concept is explained below.

---

## Table of Contents

1. [What is Traefik?](#1-what-is-traefik)
2. [How Does It Work?](#2-how-does-it-work)
3. [Project File Structure](#3-project-file-structure)
4. [File 1: `.env` — Your Settings File](#4-file-1-env--your-settings-file)
5. [File 2: `traefik.yml` — The Brain](#5-file-2-traefikyml--the-brain)
6. [File 3: `docker-compose.yml` — The Launcher](#6-file-3-docker-composeyml--the-launcher)
7. [How `.env` Variables Flow Into Docker Compose](#7-how-env-variables-flow-into-docker-compose)
8. [The `$$` Dollar Sign Rule](#8-the--dollar-sign-rule)
9. [Networking: The `proxy` Network](#9-networking-the-proxy-network)
10. [SSL: How Certificates Are Generated](#10-ssl-how-certificates-are-generated)
11. [Template: Adding a New Website (with `.env`)](#11-template-adding-a-new-website-with-env)
12. [Will This Work on My Homelab?](#12-will-this-work-on-my-homelab)
13. [Troubleshooting Reference](#13-troubleshooting-reference)
14. [Quick Commands Cheat Sheet](#14-quick-commands-cheat-sheet)
15. [Note for AI Agents](#15-note-for-ai-agents)

---

## 1. What is Traefik?

Traefik is a **Reverse Proxy** — it sits between the internet and your Docker containers.

Think of it like a **reception desk in a hotel**:
- A guest (web request) arrives and says: *"I'm here for room `blog.namani.in`"*
- The receptionist (Traefik) looks at the name and says: *"That's Container #3, go right ahead!"*
- Traefik also handles the **security badge** (SSL certificate) so the guest's connection is encrypted.

**Without Traefik**, you would need to manually assign ports to each app (`:3000`, `:8080`, etc.) and manage SSL certificates yourself.

**With Traefik**, you just add **labels** to your Docker containers, and Traefik automatically discovers them and routes traffic.

---

## 2. How Does It Work?

Here is the flow when someone visits `tf.namani.in`:

```
User's Browser
     │
     ▼
 Cloudflare (DNS + CDN Proxy)
     │
     ▼
 Your Server (31.97.233.65)
     │
     ▼
 Port 80 (HTTP) ──► Traefik auto-redirects to HTTPS
     │
     ▼
 Port 443 (HTTPS) ──► Traefik receives the request
     │
     ▼
 Traefik checks the domain name (Host header)
     │
     ├── tf.namani.in    ──► Traefik Dashboard (api@internal)
     ├── blog.namani.in  ──► Blog Container
     └── app.namani.in   ──► App Container
```

Traefik knows where to send each request because it **watches Docker** for containers with specific "labels."

---

## 3. Project File Structure

On your **local machine** (for version control):
```
Traefik Setup/
├── .env                 ← Your settings (domain, email, password)
├── .ssh/
│   └── aatmanova2       ← SSH private key
├── docker-compose.yml   ← Launches Traefik container
├── traefik.yml          ← Traefik's core configuration
├── templates/
│   ├── blog-example/    ← Ghost CMS template
│   └── wordpress/       ← WordPress + MariaDB template
│       ├── .env         ← App settings (just edit this!)
│       ├── docker-compose.yml  ← Generic launcher (don't touch)
│       └── README.md    ← How to use instructions
├── info.md              ← Server info
└── Documentation.md     ← This file
```

On the **server** (`/root/traefik/`):
```
/root/traefik/
├── .env                 ← Same settings file
├── docker-compose.yml   ← Same launcher file
├── traefik.yml          ← Same brain file
└── acme.json            ← SSL certificates (auto-generated, chmod 600)
```

---

## 4. File 1: `.env` — Your Settings File

The `.env` file stores your configuration values. Docker Compose **automatically reads** this file when you run `docker compose up`.

```env
# ============================================
# Traefik Environment Variables
# ============================================
# Change these values to customize your setup.
# After changing, run: docker compose up -d --force-recreate
# ============================================

# Your domain for the Traefik Dashboard
TRAEFIK_DOMAIN=tf.namani.in

# Your email for Let's Encrypt SSL certificates
ACME_EMAIL=bhargabpratimsharma@gmail.com

# Dashboard login credentials (htpasswd format)
# Generated with: htpasswd -nb admin admin
# IMPORTANT: Every dollar sign must be doubled ($$) because
# Docker Compose treats single $ as variable references.
DASHBOARD_AUTH=admin:$$apr1$$VDlqiA.Q$$HZsnUXm2oIU1ZXZQ8enzb0

# Docker API version (needed for Docker Engine v29+)
DOCKER_API_VERSION=1.44
```

### What each variable does:

| Variable | Purpose | Example |
| :--- | :--- | :--- |
| `TRAEFIK_DOMAIN` | The domain where the dashboard is accessible | `tf.namani.in` |
| `ACME_EMAIL` | Email for Let's Encrypt SSL notifications | `you@gmail.com` |
| `DASHBOARD_AUTH` | Hashed username:password for dashboard login | `admin:$$apr1$$...` |
| `DOCKER_API_VERSION` | Forces Docker API compatibility | `1.44` |

### How to change the dashboard password:

```bash
# SSH into your server
ssh -i .ssh/aatmanova2 root@31.97.233.65

# Generate a new hash (replace 'newuser' and 'newpassword')
htpasswd -nb newuser newpassword

# Output will look like: newuser:$apr1$xyz$abc123
# Copy this output, double every $ to $$, and paste into .env
# Example: newuser:$$apr1$$xyz$$abc123

# Then restart Traefik
cd /root/traefik
docker compose up -d --force-recreate
```

---

## 5. File 2: `traefik.yml` — The Brain

This is the **static configuration**. Traefik reads this file only when it starts. It defines the global behavior.

```yaml
# ── DASHBOARD ────────────────────────────────────
api:
  dashboard: true
  # Enables the web UI at /dashboard/
  # The actual URL and security are set via labels in docker-compose.yml

# ── ENTRY POINTS (Doors into Traefik) ───────────
entryPoints:

  web:                    # Name: "web" (we reference this name elsewhere)
    address: ":80"        # Listen on port 80 (HTTP)
    http:
      redirections:
        entryPoint:
          to: websecure   # Automatically redirect to "websecure" (port 443)
          scheme: https   # Use HTTPS scheme

  websecure:              # Name: "websecure"
    address: ":443"       # Listen on port 443 (HTTPS)

# ── SSL CERTIFICATE ENGINE ───────────────────────
certificatesResolvers:

  myresolver:             # Name: "myresolver" (we reference this in labels)
    acme:
      email: bhargabpratimsharma@gmail.com   # Let's Encrypt will email you if a cert is about to expire
      storage: acme.json                      # File where certs are saved
      httpChallenge:
        entryPoint: web   # Use port 80 to prove domain ownership to Let's Encrypt

# ── PROVIDERS (Where Traefik finds your apps) ───
providers:

  docker:
    endpoint: "unix:///var/run/docker.sock"   # Connect to the Docker engine
    exposedByDefault: false                    # SAFETY: Only route to containers with traefik.enable=true
```

### Key Concepts:

| Concept | What it means |
| :--- | :--- |
| **Entry Point** | A "door" where traffic enters. We have two: `web` (port 80) and `websecure` (port 443). |
| **Certificate Resolver** | An engine that talks to Let's Encrypt to get free SSL certificates. We named ours `myresolver`. |
| **Provider** | Where Traefik discovers your apps. We use `docker` — it watches for container labels. |
| **`exposedByDefault: false`** | Very important! Without this, every container you start would be publicly accessible. |

---

## 6. File 3: `docker-compose.yml` — The Launcher

This file tells Docker **how to run Traefik** and uses **labels** to configure the dashboard route.

```yaml
services:
  traefik:
    image: traefik:latest          # The Traefik Docker image
    container_name: traefik        # A friendly name for the container
    restart: always                # Auto-restart if it crashes or server reboots

    environment:
      - DOCKER_API_VERSION=${DOCKER_API_VERSION}
      # ↑ Reads "1.44" from .env file
      # Fixes compatibility with Docker Engine v29+

    ports:
      - "80:80"                    # Map server port 80 → container port 80
      - "443:443"                  # Map server port 443 → container port 443

    networks:
      - proxy                      # Join the shared "proxy" network

    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
        # ↑ Give Traefik read-only access to Docker (so it can discover containers)
      - "./traefik.yml:/traefik.yml:ro"
        # ↑ Mount the brain file (read-only)
      - "./acme.json:/acme.json"
        # ↑ Mount the SSL certificate storage

    labels:
      - "traefik.enable=true"
        # ↑ Tell Traefik: "Yes, route traffic to ME too"

      - "traefik.http.routers.dashboard.rule=Host(`${TRAEFIK_DOMAIN}`)"
        # ↑ "If the domain is tf.namani.in, this router handles it"
        # ${TRAEFIK_DOMAIN} is replaced with the value from .env

      - "traefik.http.routers.dashboard.entrypoints=websecure"
        # ↑ "Only accept HTTPS traffic (port 443)"

      - "traefik.http.routers.dashboard.service=api@internal"
        # ↑ "Forward the request to Traefik's own built-in dashboard"

      - "traefik.http.routers.dashboard.middlewares=auth"
        # ↑ "Before forwarding, run the middleware named 'auth'"

      - "traefik.http.routers.dashboard.tls.certresolver=myresolver"
        # ↑ "Use Let's Encrypt to get an SSL certificate for this domain"

      - "traefik.http.middlewares.auth.basicauth.users=${DASHBOARD_AUTH}"
        # ↑ "The 'auth' middleware requires this username:password"
        # ${DASHBOARD_AUTH} is replaced with the hashed credentials from .env

networks:
  proxy:
    external: true
    # ↑ This network was created manually with: docker network create proxy
    # All your apps must join this same network to be reachable by Traefik
```

### Label Naming Convention:

Labels follow a pattern: `traefik.http.<type>.<name>.<property>`

| Part | Meaning | Example |
| :--- | :--- | :--- |
| `routers` | Matches incoming requests | `traefik.http.routers.dashboard.rule=...` |
| `middlewares` | Modifies the request before forwarding | `traefik.http.middlewares.auth.basicauth...` |
| `services` | Where to send the request | `traefik.http.routers.dashboard.service=api@internal` |

The `dashboard` in the labels above is just a name we chose — you can call it anything (e.g., `myblog`, `api`, `frontend`).

---

## 7. How `.env` Variables Flow Into Docker Compose

```
.env file                        docker-compose.yml
┌────────────────────┐           ┌─────────────────────────────────────┐
│ TRAEFIK_DOMAIN=    │           │                                     │
│   tf.namani.in     │ ────────► │  Host(`${TRAEFIK_DOMAIN}`)          │
│                    │           │  becomes: Host(`tf.namani.in`)      │
├────────────────────┤           │                                     │
│ DASHBOARD_AUTH=    │           │                                     │
│   admin:$$apr1$... │ ────────► │  users=${DASHBOARD_AUTH}            │
│                    │           │  becomes: users=admin:$apr1$...     │
├────────────────────┤           │                                     │
│ DOCKER_API_VERSION │           │                                     │
│   =1.44            │ ────────► │  DOCKER_API_VERSION=${...}          │
│                    │           │  becomes: DOCKER_API_VERSION=1.44   │
└────────────────────┘           └─────────────────────────────────────┘
```

**Docker Compose automatically reads `.env`** from the same directory. You don't need to specify it.

---

## 8. The `$$` Dollar Sign Rule

This is a common gotcha:

- Docker Compose uses `$` for variable expansion (e.g., `${TRAEFIK_DOMAIN}`)
- Password hashes like `$apr1$xyz$abc` are full of `$` signs
- Docker Compose will try to read `$apr1` as a variable name and replace it with nothing!

**The Fix**: Double every `$` to `$$` in the `.env` file:

| What you generate | What you put in `.env` |
| :--- | :--- |
| `admin:$apr1$xyz$abc` | `admin:$$apr1$$xyz$$abc` |

Docker Compose reads `$$` and converts it back to a single `$` when passing it to the container.

---

## 9. Networking: The `proxy` Network

Docker containers are **isolated by default** — they can't talk to each other unless they share a network.

We created a global network called `proxy`:
```bash
docker network create proxy
```

**Rule**: Every container that needs to be proxied by Traefik must join this network.

```
┌─────────────────────────────────────────────┐
│              "proxy" network                │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Traefik  │  │   Blog   │  │   App    │  │
│  │          │──│          │  │          │  │
│  │ :80/:443 │  │ :3000    │  │ :8080    │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                             │
└─────────────────────────────────────────────┘
```

Traefik can reach `Blog` and `App` because they are all on the same `proxy` network. Traefik then forwards external requests to the correct container.

---

## 10. SSL: How Certificates Are Generated

When you add the label `tls.certresolver=myresolver` to a container:

1. Traefik sees the new domain (e.g., `blog.namani.in`) from the `Host()` rule.
2. Traefik contacts **Let's Encrypt** and says: *"I need a certificate for blog.namani.in."*
3. Let's Encrypt says: *"Prove you own it. I'll send a request to `http://blog.namani.in/.well-known/acme-challenge/xyz`."*
4. Traefik responds to that challenge on port 80 (the `httpChallenge` entrypoint).
5. Let's Encrypt confirms ownership and issues the certificate.
6. Traefik stores it in `acme.json` and starts serving HTTPS.

**This happens completely automatically.** You just add the label, and Traefik handles the rest.

---

## 11. Template: Adding a New Website (with `.env`)

I've created a **reusable template** in `templates/blog-example/`. You never need to edit the `docker-compose.yml` — just edit the `.env` file!

### Step-by-Step: Deploy a New App

**Step 1**: Copy the template folder to the server:
```bash
# On your server
cp -r /root/traefik/templates/blog-example /root/my-new-app
```

**Step 2**: Edit ONLY the `.env` file:
```bash
nano /root/my-new-app/.env
```

**Step 3**: Change these values:
```env
# ── TRAEFIK ROUTING ──────────────────────────
APP_DOMAIN=app.namani.in       # ← Your domain
ROUTER_NAME=myapp              # ← Unique name (no spaces)

# ── APP SETTINGS ─────────────────────────────
APP_IMAGE=nginx:latest         # ← Your Docker image
APP_PORT=80                    # ← Port your app uses inside the container
CONTAINER_NAME=my-app          # ← Friendly name for docker ps
```

**Step 4**: Start it:
```bash
cd /root/my-new-app
docker compose up -d
```

✅ Traefik automatically discovers it, generates SSL, and starts routing.

### The Template's `docker-compose.yml` (you don't need to touch this):

```yaml
services:
  app:
    image: ${APP_IMAGE}                # Reads from .env
    container_name: ${CONTAINER_NAME}  # Reads from .env
    restart: always
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.${ROUTER_NAME}.rule=Host(`${APP_DOMAIN}`)"
      - "traefik.http.routers.${ROUTER_NAME}.entrypoints=websecure"
      - "traefik.http.routers.${ROUTER_NAME}.tls.certresolver=myresolver"
      - "traefik.http.services.${ROUTER_NAME}.loadbalancer.server.port=${APP_PORT}"

networks:
  proxy:
    external: true
```

### Common APP_PORT values:

| App | Image | Port |
| :--- | :--- | :--- |
| Nginx | `nginx:latest` | `80` |
| Node.js / Next.js | `node:latest` | `3000` |
| Ghost Blog | `ghost:latest` | `2368` |
| WordPress | `wordpress:latest` | `80` |
| Grafana | `grafana/grafana:latest` | `3000` |
| Portainer | `portainer/portainer-ce:latest` | `9000` |
| Jellyfin | `jellyfin/jellyfin:latest` | `8096` |
| Gitea | `gitea/gitea:latest` | `3000` |
| Uptime Kuma | `louislam/uptime-kuma:latest` | `3001` |
| MinIO Console | `minio/minio:latest` | `9001` |

---

## 12. Will This Work on My Homelab?

This setup is designed for a **public VPS** (like your server `31.97.233.65`). Here's a compatibility breakdown:

| Your Setup | Works? | Why |
| :--- | :--- | :--- |
| ✅ **VPS with public IP** | Yes | This is what you're using now. Let's Encrypt can reach ports 80/443 directly. |
| ✅ **Homelab + Port Forwarding** | Yes | If your router forwards ports 80 & 443 to your homelab machine, it works exactly the same. |
| ❌ **Homelab behind CGNAT** | No | Your ISP doesn't give you a real public IP. Let's Encrypt can't reach you for the HTTP challenge. |
| ❌ **Tailscale only** | No | Tailscale IPs (like `100.x.x.x`) are private. Let's Encrypt can't reach them. DNS can't point to them. |
| ✅ **Homelab + Cloudflare Tunnel** | Yes (different setup) | Uses `cloudflared` to create a tunnel. No port forwarding needed, but requires a different Traefik config. |

### How to check if your homelab will work:

```bash
# 1. Check if you have a public IP
curl ifconfig.me
# If this returns something like 192.168.x.x or 10.x.x.x → you're behind CGNAT → won't work

# 2. Check if port 80 is reachable from outside
# Forward port 80 on your router to your machine, then test:
curl http://YOUR_PUBLIC_IP
```

### Alternatives for Homelab (if ports are blocked):

1. **Cloudflare Tunnel** — Best option. See Section 12.1 below.
2. **Tailscale + Tailscale Funnel** — Exposes specific services through Tailscale's network. Limited but simple.
3. **DNS Challenge (instead of HTTP)** — If you can't open port 80, use Cloudflare DNS API to prove domain ownership. Requires changing `traefik.yml` to use `dnsChallenge` instead of `httpChallenge`.

---

### 12.1 Is Cloudflare Tunnel Free?

**Yes, 100% free.** Cloudflare Tunnel is included in Cloudflare's free plan. No credit card needed. You just need:
- A free Cloudflare account
- A domain managed by Cloudflare (you already have `namani.in`)

---

### 12.2 Does Cloudflare Tunnel Conflict with Tailscale?

**No, zero conflicts.** They solve completely different problems and can run on the same machine simultaneously:

```
Your Homelab Machine
├── Tailscale (private access)
│   ├── You SSH from your phone via 100.x.x.x     ← Only YOU can access
│   ├── You access Portainer privately             ← Only YOUR devices
│   └── Uses UDP 41641 (WireGuard)
│
├── Cloudflare Tunnel (public access)
│   ├── Anyone visits blog.namani.in               ← The WORLD can access
│   ├── Cloudflare handles SSL                     ← Free certificates
│   └── Uses outbound HTTPS only (no ports to open!)
│
└── Both run side-by-side with zero interference ✅
```

| Feature | Tailscale | Cloudflare Tunnel |
| :--- | :--- | :--- |
| **Purpose** | Private mesh VPN between YOUR devices | Expose services to the PUBLIC internet |
| **Who can access** | Only your Tailscale devices | Anyone on the internet |
| **Network** | Creates `100.x.x.x` overlay | Outbound-only tunnel to Cloudflare |
| **Ports needed** | UDP 41641 | None (outbound HTTPS only) |
| **SSL** | Tailscale's own certs | Cloudflare's edge certs |
| **Cost** | Free (personal) | Free |
| **Use case** | SSH, private admin panels | Public websites, APIs |

**Think of it this way**: Tailscale = your **private back door**. Cloudflare Tunnel = your **public front door**. They don't interfere with each other.

---

### 12.2.1 Cloudflare Tunnel vs Tailscale — Deep Comparison

#### ✅ What Cloudflare Tunnel Does BETTER Than Tailscale

| Strength | Why It Matters |
| :--- | :--- |
| **Public websites** | Tailscale can't host a public website. Cloudflare Tunnel can expose any service to the entire internet with a custom domain. |
| **Free SSL certificates** | Cloudflare automatically provides SSL for every domain. Tailscale has MagicDNS certs but only for private `.ts.net` domains. |
| **DDoS protection** | Cloudflare's global CDN absorbs attacks before they reach your homelab. Tailscale offers zero DDoS protection. |
| **Custom domains** | `blog.namani.in` looks professional. Tailscale gives you `machine-name.tailnet-name.ts.net` — ugly and not shareable. |
| **CDN & Caching** | Cloudflare caches static content at 300+ edge locations worldwide. Your blog loads fast for users in India, USA, Europe. Tailscale has no CDN. |
| **Web Application Firewall** | Cloudflare can block bots, SQL injection, XSS attacks. Tailscale doesn't inspect HTTP traffic. |
| **Analytics** | Cloudflare shows traffic analytics, threat reports, bandwidth stats. Tailscale has none for HTTP. |
| **Zero client setup for visitors** | Anyone with a browser can access your site. Tailscale requires every user to install an app and join your network. |
| **SEO & Discoverability** | Google can index your public site. Tailscale services are invisible to search engines. |

#### ✅ What Tailscale Does BETTER Than Cloudflare Tunnel

| Strength | Why It Matters |
| :--- | :--- |
| **Private access** | Only your devices can connect. No one on the internet can even see your services exist. Cloudflare Tunnel exposes to the world. |
| **SSH access** | Tailscale lets you SSH into any machine from anywhere (phone, laptop, tablet). Cloudflare Tunnel doesn't do SSH natively (you'd need Cloudflare Access, which is more complex). |
| **Device-to-device networking** | Your phone can talk to your homelab can talk to your VPS — all on `100.x.x.x`. Cloudflare Tunnel is one-directional (internet → your machine). |
| **File sharing** | You can access SMB shares, NFS mounts, Syncthing directly over Tailscale. Cloudflare Tunnel only works with HTTP/HTTPS. |
| **No third-party dependency for private access** | Tailscale works peer-to-peer (direct connections between your devices). Cloudflare Tunnel always routes through Cloudflare's servers. |
| **Non-HTTP services** | Tailscale works with any protocol (SSH, RDP, databases, game servers, Minecraft, Plex). Cloudflare Tunnel is primarily HTTP/HTTPS. |
| **Speed for private access** | Tailscale uses WireGuard (direct peer-to-peer). Cloudflare Tunnel routes through their edge → adds latency for private use. |
| **Exit nodes** | Tailscale can route all your internet traffic through a specific machine (like a VPN). Cloudflare Tunnel can't do this. |
| **Access Control Lists (ACLs)** | Tailscale has fine-grained ACLs (e.g., "phone can access NAS but not server"). Cloudflare Tunnel's access control requires Cloudflare Access (paid for advanced features). |

#### 🏆 When to Use Each

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DECISION GUIDE                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  "I want to host a PUBLIC website"                                  │
│     → Cloudflare Tunnel ✅                                          │
│                                                                     │
│  "I want to SSH into my homelab from my phone"                      │
│     → Tailscale ✅                                                  │
│                                                                     │
│  "I want to share files between my devices"                         │
│     → Tailscale ✅                                                  │
│                                                                     │
│  "I want my blog to be fast worldwide"                              │
│     → Cloudflare Tunnel ✅ (CDN)                                    │
│                                                                     │
│  "I want to access my Portainer/Grafana privately"                  │
│     → Tailscale ✅                                                  │
│                                                                     │
│  "I want to protect my site from DDoS"                              │
│     → Cloudflare Tunnel ✅                                          │
│                                                                     │
│  "I want to access my database remotely"                            │
│     → Tailscale ✅ (non-HTTP)                                       │
│                                                                     │
│  "I want BOTH public sites AND private admin access"                │
│     → Use BOTH together ✅✅                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 🎯 The Best Setup (Use Both)

For a homelab, the ideal setup is:

```
Your Homelab
│
├── Tailscale (private)
│   ├── Portainer        → admin.tailnet.ts.net    (only you)
│   ├── Grafana          → grafana.tailnet.ts.net   (only you)
│   ├── SSH              → 100.x.x.x               (only you)
│   └── Database access  → 100.x.x.x:5432          (only you)
│
├── Cloudflare Tunnel (public)
│   ├── Blog             → blog.namani.in           (everyone)
│   ├── Portfolio         → namani.in                (everyone)
│   └── API              → api.namani.in            (everyone)
│
└── Both running simultaneously with zero conflicts ✅
```

**Rule of thumb**: If only YOU need access → Tailscale. If the WORLD needs access → Cloudflare Tunnel.

---

### 12.3 Cloudflare Tunnel Setup Guide (Detailed)

This is a complete, step-by-step guide to expose your homelab services to the internet using Cloudflare Tunnel — **no port forwarding, no public IP needed**.

#### How It Works

```
User visits blog.namani.in
        │
        ▼
  Cloudflare Edge (global CDN)
        │
        │  Tunnel (encrypted, outbound connection)
        │  Your homelab CONNECTS TO Cloudflare, not the other way around
        │
        ▼
  cloudflared (running on your homelab)
        │
        ▼
  Traefik (on port 80/443 internally)
        │
        ▼
  Your App Container
```

The key insight: **your homelab connects OUT to Cloudflare**. No inbound connections needed. No ports to open. Works behind CGNAT, firewalls, anything.

#### Prerequisites

- Cloudflare account (free)
- Domain added to Cloudflare (you have `namani.in` ✅)
- Docker installed on your homelab machine

#### Step 1: Create a Tunnel in Cloudflare Dashboard

1. Go to [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)
2. Sign up / Log in (free plan is fine)
3. Navigate to: **Networks** → **Tunnels**
4. Click **Create a tunnel**
5. Select **Cloudflared** as the connector
6. Give it a name (e.g., `homelab`)
7. **Copy the tunnel token** — you'll need it in the next step

It will look something like:
```
eyJhIjoiNjQ5ZDM0OTBlMjJkMzYyNGE1YzM5YzNiZjI2...
```

#### Step 2: Run `cloudflared` as a Docker Container

On your homelab machine, create a folder and `docker-compose.yml`:

```bash
mkdir -p /root/cloudflare-tunnel
cd /root/cloudflare-tunnel
```

Create the `docker-compose.yml`:
```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: always
    command: tunnel run
    environment:
      - TUNNEL_TOKEN=your-tunnel-token-here
    networks:
      - proxy

networks:
  proxy:
    external: true
```

> ⚠️ Replace `your-tunnel-token-here` with the token from Step 1.

Start it:
```bash
docker compose up -d
```

#### Step 3: Configure the Tunnel Routes in Cloudflare Dashboard

Back in the Cloudflare dashboard, after the tunnel is connected:

1. Click on your tunnel → **Public Hostname** tab
2. Click **Add a public hostname**
3. Fill in:

| Field | Value |
| :--- | :--- |
| **Subdomain** | `blog` (or whatever you want) |
| **Domain** | `namani.in` |
| **Type** | `HTTP` |
| **URL** | `traefik:80` (the container name + port inside Docker) |

4. Click **Save**

**That's it!** Cloudflare will now route `blog.namani.in` → through the tunnel → to your Traefik container → to your app.

#### Step 4: Add More Services

For each new service, just go back to the Cloudflare dashboard:
1. **Public Hostname** → **Add a public hostname**
2. Set the subdomain and point it to `traefik:80`
3. Traefik will handle the routing based on the `Host()` label

#### Important: Traefik Config Changes for Tunnel Mode

When using Cloudflare Tunnel, you need to make a few changes to your Traefik setup:

**1. Remove the HTTPS redirect** (Cloudflare handles SSL):
```yaml
# traefik.yml for HOMELAB with Cloudflare Tunnel
api:
  dashboard: true

entryPoints:
  web:
    address: ":80"
    # NO redirect to HTTPS — Cloudflare handles SSL at the edge

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
```

**2. Remove SSL labels from containers** (no `certresolver` needed):
```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.blog.rule=Host(`blog.namani.in`)"
  - "traefik.http.routers.blog.entrypoints=web"
  # NO tls.certresolver — Cloudflare provides the SSL certificate
```

**3. Remove port 443 from Traefik** (only 80 is needed internally):
```yaml
ports:
  - "80:80"
  # No 443 needed — Cloudflare terminates SSL
```

#### Architecture: VPS vs Homelab Comparison

```
═══════════════════════════════════════════════════════════════
  VPS SETUP (current)              HOMELAB SETUP (with tunnel)
═══════════════════════════════════════════════════════════════

  Internet                         Internet
     │                                │
     ▼                                ▼
  Cloudflare ──DNS──► VPS IP       Cloudflare ──Tunnel──► cloudflared
     │                                │
     ▼                                ▼
  Traefik (port 80/443)            Traefik (port 80 only)
     │                                │
     ▼                                ▼
  SSL: Let's Encrypt               SSL: Cloudflare Edge (free)
     │                                │
     ▼                                ▼
  Your App                         Your App

  ✅ Needs public IP               ✅ No public IP needed
  ✅ Needs ports 80/443 open       ✅ No ports needed
  ❌ Costs money (VPS)             ✅ Runs on your own hardware
═══════════════════════════════════════════════════════════════
```



## 13. Troubleshooting Reference

| Problem | Cause | Fix |
| :--- | :--- | :--- |
| `client version 1.24 is too old` | Docker Engine v29+ API mismatch | Set `DOCKER_API_VERSION=1.44` in `.env` |
| `field not found, node: apiVersion` | Invalid Traefik config key | Remove `apiVersion` from `traefik.yml`, use env var instead |
| SSL certificate not generated | DNS not pointing to server IP | Ensure your domain's A record points to `31.97.233.65` |
| `acme.json` permission error | File permissions too open | Run `chmod 600 acme.json` |
| `$$` becoming garbage numbers | Shell expanding `$` during file write | Use SCP to upload files, or use `cat << 'EOF'` (quoted heredoc) |
| Container not discovered | Missing network or label | Ensure container has `traefik.enable=true` AND is on `proxy` network |
| 401 Unauthorized (can't log in) | Wrong password hash format | Regenerate with `htpasswd -nb user pass`, double all `$` to `$$` |

---

## 14. Quick Commands Cheat Sheet

```bash
# SSH into the server
ssh -i .ssh/aatmanova2 root@31.97.233.65

# Go to Traefik directory
cd /root/traefik

# View live logs
docker logs -f traefik

# Restart Traefik after config changes
docker compose up -d --force-recreate

# Stop Traefik
docker compose down

# Check which containers are running
docker ps

# Check the proxy network
docker network inspect proxy

# Generate a new password hash
htpasswd -nb username password

# View current .env values
cat .env

# View the SSL certificates
cat acme.json | python3 -m json.tool

# Upload local config files to server
scp -i .ssh/aatmanova2 .env docker-compose.yml traefik.yml root@31.97.233.65:/root/traefik/
```

---

## 15. Note for AI Agents

> This section is for any future AI that works on this server.

- **Server**: `31.97.233.65`, accessible via SSH key at `/.ssh/aatmanova2` (relative to project root)
- **Traefik Home**: `/root/traefik/`
- **Docker Engine**: v29.2.1 (API minimum 1.44) — **must** set `DOCKER_API_VERSION=1.44`
- **Traefik Image**: `traefik:latest` (was originally `v3.1`, upgraded for API compatibility)
- **Shared Network**: `proxy` (external, created with `docker network create proxy`)
- **SSL**: HTTP-01 challenge via Let's Encrypt. Domain proxied by Cloudflare. Cloudflare SSL mode should be "Full" or "Full (Strict)".
- **Auth**: Dashboard at `tf.namani.in` uses Basic Auth. Credentials: `admin:admin`. Hash stored in `.env` as `DASHBOARD_AUTH`.
- **Config is Dynamic**: `docker-compose.yml` reads `${VAR}` from `.env`. Edit `.env`, then `docker compose up -d --force-recreate`.
- **`$$` Escaping**: Password hashes in `.env` must have all `$` doubled to `$$` because Docker Compose interpolates `.env` values.
- **Local Copy**: The user keeps a copy of all config files in their local workspace at `Traefik Setup/` for version control. Use SCP to upload changes to the server.
- **App Template**: There is a reusable `.env`-based template at `templates/blog-example/`. To deploy a new app, copy the folder, edit `.env`, and `docker compose up -d`. The `docker-compose.yml` is fully generic and doesn't need editing.
- **Homelab Note**: This setup requires a public IP with ports 80/443 open. It does NOT work with Tailscale-only or behind CGNAT. For homelab without port forwarding, recommend Cloudflare Tunnel or DNS challenge.

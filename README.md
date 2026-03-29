<div align="center">

# 🖥️ Server Manager

**A personal, production-ready server infrastructure toolkit**  
built on top of **Docker** + **Traefik** — deploy any app with a single `.env` change.

[![Traefik](https://img.shields.io/badge/Traefik-v3-blue?logo=traefikproxy&logoColor=white)](https://traefik.io)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)](https://docs.docker.com/compose/)
[![Let's Encrypt](https://img.shields.io/badge/SSL-Let's%20Encrypt-003A70?logo=letsencrypt&logoColor=white)](https://letsencrypt.org/)
[![Cloudflare](https://img.shields.io/badge/DNS-Cloudflare-F38020?logo=cloudflare&logoColor=white)](https://cloudflare.com)

</div>

---

## 📖 Table of Contents

- [Overview](#-overview)
- [How It Works](#-how-it-works)
- [Project Structure](#-project-structure)
- [Quick Start](#-quick-start)
- [Traefik Setup](#-traefik-setup)
- [Applications](#-applications)
- [Deploy a New App](#-deploy-a-new-app)
- [Server Info](#-server-info)

---

## 🌐 Overview

**Server Manager** is a self-hosted infrastructure toolkit that uses **Traefik** as a smart reverse proxy. It automatically handles:

- ✅ **Automatic SSL** via Let's Encrypt (no manual cert management)
- ✅ **One-line deployments** — just edit a single `.env` file
- ✅ **Automatic HTTPS redirect** from HTTP to HTTPS
- ✅ **Auto-discovery** — Traefik detects new Docker containers automatically
- ✅ **Secure dashboard** behind basic auth at your own domain

> **Live Server:** `YOUR_SERVER_IP` &nbsp;|&nbsp; **Traefik Dashboard:** [https://tf.yourdomain.com/dashboard/](https://tf.yourdomain.com/dashboard/)

---

## ⚙️ How It Works

```
  Your Browser
       │
       ▼
  Cloudflare (DNS)
       │
       ▼
  Server: YOUR_SERVER_IP
       │
       ▼
  Port 80  ──► Traefik (auto-redirects to HTTPS)
  Port 443 ──► Traefik (checks domain name)
       │
       ├──► tf.yourdomain.com        →  Traefik Dashboard
       ├──► blog.namani.in      →  Ghost Blog container
       ├──► c.aatmanova.in      →  WordPress container
       └──► anything.domain     →  Any Docker container
```

Traefik reads **Docker labels** on each container to know where to route traffic. Adding a new site is as simple as starting a new container with the right labels — which our templates handle automatically.

---

## 📁 Project Structure

```
Server Manager/
│
├── README.md                        ← You are here
│
├── traefik-setup/                   ← Core reverse proxy configuration
│   ├── traefik.yml                  ← Traefik static config (entry points, SSL, providers)
│   ├── docker-compose.yml           ← Launches the Traefik container
│   ├── .env                         ← Your domain, email, dashboard credentials
│   ├── info.md                      ← Server connection info & quick reference
│   └── Documentation.md             ← Full deep-dive technical documentation
│
└── applications/                    ← Ready-to-deploy application templates
    │
    ├── blog-example/                ← Ghost CMS template (any single-container app)
    │   ├── docker-compose.yml       ← Generic launcher (no edits needed)
    │   ├── .env                     ← App config (domain, image, port)
    │   └── README.md                ← Usage instructions
    │
    └── wordpress/                   ← WordPress + MariaDB template
        ├── docker-compose.yml       ← WordPress + DB multi-service launcher
        ├── .env                     ← All config in one place
        └── README.md                ← Usage instructions
```

> **Philosophy:** Every application template is self-contained. You only ever edit the `.env` file to deploy. The `docker-compose.yml` files are reusable and should not need modification.

---

## 🚀 Quick Start

### Prerequisites

- A Linux VPS with a public IP
- Docker & Docker Compose installed
- A domain managed by Cloudflare (or any DNS provider)

### Step 1 — Create the shared proxy network

Traefik and all your apps communicate through a shared Docker network:

```bash
docker network create proxy
```

> ⚠️ This only needs to be done **once** on the server. All apps join this network automatically.

### Step 2 — Set up Traefik

```bash
# Clone this repo on your server
git clone <your-repo-url> /opt/server-manager
cd /opt/server-manager/traefik-setup

# Create the SSL certificate storage file (required by Let's Encrypt)
touch acme.json && chmod 600 acme.json

# Edit the .env file with your domain and email
nano .env

# Launch Traefik
docker compose up -d
```

### Step 3 — Deploy an Application

```bash
# Copy an application template
cp -r /opt/server-manager/applications/blog-example /opt/my-blog

# Edit ONLY the .env file
nano /opt/my-blog/.env

# Start the app — Traefik discovers it automatically
cd /opt/my-blog && docker compose up -d
```

That's it. Your app is live with HTTPS in under a minute. ✅

---

## 🔧 Traefik Setup

> 📄 **Full documentation:** [`traefik-setup/Documentation.md`](./traefik-setup/Documentation.md)  
> ℹ️ **Server info & credentials:** [`traefik-setup/info.md`](./traefik-setup/info.md)

### Key Files

| File | Purpose |
|------|---------|
| [`traefik-setup/.env`](./traefik-setup/.env) | Set your dashboard domain, email, and auth credentials |
| [`traefik-setup/traefik.yml`](./traefik-setup/traefik.yml) | Static config — entry points, SSL resolver, Docker provider |
| [`traefik-setup/docker-compose.yml`](./traefik-setup/docker-compose.yml) | Runs the Traefik container with the correct volumes and ports |

### `.env` Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `TRAEFIK_DOMAIN` | Domain for Traefik dashboard | `tf.yourdomain.com` |
| `ACME_EMAIL` | Email for SSL cert expiry alerts | `you@gmail.com` |
| `DASHBOARD_AUTH` | `htpasswd`-hashed credentials | `admin:$$apr1$$...` |
| `DOCKER_API_VERSION` | Docker Engine API version | `1.44` |

### Updating the Dashboard Password

```bash
# Generate a new hash on your local machine
htpasswd -nb your-username your-password

# ⚠️ Double every $ sign → $ becomes $$  (required by Docker Compose)
# Paste the result into traefik-setup/.env → DASHBOARD_AUTH=...

# Apply the change
cd /opt/server-manager/traefik-setup
docker compose up -d --force-recreate
```

---

## 📦 Applications

Each folder in `applications/` is a self-contained deployment template.

---

### 🌐 Ghost Blog (Single-Container Template)

> 📄 [`applications/blog-example/README.md`](./applications/blog-example/README.md)

A reusable template for **any single-container Docker app** (Ghost, Nginx, Node.js, Grafana, etc.).

**`.env` variables to change:**

| Variable | Description | Example |
|----------|-------------|---------|
| `APP_DOMAIN` | Your domain for this app | `blog.namani.in` |
| `ROUTER_NAME` | Unique Traefik router name | `my-blog` |
| `APP_IMAGE` | Docker image to run | `ghost:latest` |
| `APP_PORT` | Port the app listens on internally | `2368` |
| `CONTAINER_NAME` | Name shown in `docker ps` | `my-blog` |

**Common images and their ports:**

| App | Image | Port |
|-----|-------|------|
| Nginx | `nginx:latest` | `80` |
| Ghost Blog | `ghost:latest` | `2368` |
| Node.js | `node:latest` | `3000` |
| Grafana | `grafana/grafana` | `3000` |
| Portainer | `portainer/portainer-ce` | `9000` |
| Uptime Kuma | `louislam/uptime-kuma` | `3001` |

---

### 📝 WordPress

> 📄 [`applications/wordpress/README.md`](./applications/wordpress/README.md)

A production-ready **WordPress + MariaDB** stack. Database and WordPress run as separate containers on an isolated internal network, with only WordPress exposed to Traefik.

**`.env` variables to change:**

| Variable | Description | Example |
|----------|-------------|---------|
| `APP_DOMAIN` | Your domain for WordPress | `c.aatmanova.in` |
| `ROUTER_NAME` | Unique Traefik router name | `wordpress_site` |
| `DB_NAME` | Database name | `wordpress_db` |
| `DB_USER` | Database user | `wp_user` |
| `DB_PASSWORD` | Database password — **change this!** | `str0ng_p@ssw0rd` |
| `DB_ROOT_PASSWORD` | MariaDB root password — **change this!** | `r00t_p@ssw0rd` |

**Architecture:**

```
  Internet → Traefik → [wordpress_app container]
                                │
                    (internal network only)
                                │
                       [wordpress_db container]
                       (not exposed externally)
```

---

## ➕ Deploy a New App

Use the `blog-example` template as a base for any new application:

```bash
# 1. Copy the template
cp -r applications/blog-example applications/my-new-app

# 2. Edit ONLY the .env — no other file needs changing
nano applications/my-new-app/.env

# 3. Upload to your server and start
scp -r applications/my-new-app root@YOUR_SERVER_IP:/opt/my-new-app
ssh root@YOUR_SERVER_IP "cd /opt/my-new-app && docker compose up -d"
```

Traefik will **automatically discover** the new container, request an SSL certificate, and start routing traffic within seconds.

> ⚠️ **Important:** Every app's `ROUTER_NAME` and `CONTAINER_NAME` must be **unique** across all deployments. Duplicates will cause routing conflicts.

---

## ℹ️ Server Info

> 📄 Full details in [`traefik-setup/info.md`](./traefik-setup/info.md)

| Property | Value |
|----------|-------|
| **Server IP** | `YOUR_SERVER_IP` |
| **Traefik Dashboard** | [https://tf.yourdomain.com/dashboard/](https://tf.yourdomain.com/dashboard/) |
| **SSH Key** | `.ssh/aatmanova2` |
| **SSH Command** | `ssh -i .ssh/aatmanova2 root@YOUR_SERVER_IP` |
| **Traefik Config Path** | `/opt/traefik/` on server |

---

## 📚 Documentation Index

| File | Description |
|------|-------------|
| [`traefik-setup/Documentation.md`](./traefik-setup/Documentation.md) | Complete Traefik deep-dive: concepts, networking, SSL, troubleshooting, Cloudflare Tunnel guide |
| [`traefik-setup/info.md`](./traefik-setup/info.md) | Quick server reference: IP, credentials, requirements |
| [`applications/blog-example/README.md`](./applications/blog-example/README.md) | How to use the generic single-container app template |
| [`applications/wordpress/README.md`](./applications/wordpress/README.md) | How to deploy WordPress + MariaDB |

---

<div align="center">

**Server Manager** — because infrastructure should be boring to maintain and exciting to extend.

</div>

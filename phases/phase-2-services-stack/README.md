# Phase 2 — Services Stack

Built out a complete self-hosted stack on CT 103, exposing services via subdomain with HTTPS and wiring up monitoring and automation.

**Time invested:** ~3h  
**Completed:** 2026-05-28

---

## What was built

| Service | Internal domain | Port | Purpose |
|---------|----------------|------|---------|
| n8n + Postgres | `gdn8n.duckdns.org` | 5678 | Workflow automation |
| Uptime Kuma | `uptime.lab` | 3001 | Uptime monitoring |
| Homepage | `home.lab` | 3000 | Dashboard |
| Portainer | `portainer.lab` | 9000 | Docker management UI |
| Nginx Proxy Manager | `:80/:443` | — | Reverse proxy + SSL |
| Vaultwarden | (internal) | 8080 | Password manager |

All running as Docker Compose services on CT 103 (`192.168.18.29`).

---

## Setup details

### n8n + Postgres

n8n backed by Postgres instead of SQLite for production-grade persistence across workflow runs and crash recovery.

Public access via NPM + DuckDNS + Let's Encrypt:
- DuckDNS domain: `gdn8n.duckdns.org`
- NPM handles SSL termination and proxies to `n8n:5678` internally

### Uptime Kuma

Monitors configured:
- Proxmox node (HTTP)
- CT 100 AdGuard
- CT 101 Tailscale
- CT 103 services (Docker container monitor — no port exposure needed)
- n8n, Vaultwarden, NPM health endpoints

All monitors alert to Telegram on downtime with < 30s detection time.

### Homepage

Dashboard with live widgets:
- Proxmox node stats (CPU, RAM, disk via API)
- Docker container status
- Quick links to all internal services

### Nginx Proxy Manager

Handles all internal `.lab` subdomain routing and external HTTPS:

```
home.lab      → homepage:3000
uptime.lab    → uptime-kuma:3001
portainer.lab → portainer:9000
```

SSL certificates issued automatically via Let's Encrypt (DuckDNS DNS challenge for internal domains).

---

## Lessons learned

- **DuckDNS + NPM + Let's Encrypt** is the lowest-friction path to HTTPS without a static IP — one env var for the DuckDNS token and NPM handles everything
- **Uptime Kuma's Docker monitor type** checks container state directly without requiring the service to expose a port — useful for internal-only services like Vaultwarden
- **n8n + Postgres** vs n8n + SQLite: SQLite is fine for personal use, but Postgres is worth it if you're running production workflows that must survive crashes or restarts
- **Compose monolith per CT** (one `docker-compose.yml` for all services in a container) vs per-service files: monolith wins for a personal homelab — simpler dependency management, single `docker compose up/down`

---

## Automation built on top

Three n8n workflows wired to the homelab during this phase:

| Workflow | Schedule | What it does |
|----------|----------|-------------|
| Daily monitor | 8am daily | Checks Proxmox reachability + CT status → Telegram alert if anything is wrong (silence = all good) |
| Weekly digest | Sundays 5am | Homelab state summary → Notion + Telegram |
| Maintenance log | Webhook | Receives systemd timer logs from desktop → Notion database |

**Key fix:** Proxmox API `/nodes` endpoint returns zeros in a single-node setup — must use `/nodes/{node}/status` instead. The node name is `Macbook`, not `proxmox`.

---

## Next: [Phase 3 — k3s](../phase-3-k3s/)

# homelab-gustavo

Personal homelab running on a repurposed MacBook Pro — managed with Proxmox, automated with n8n, and progressively hardened toward DevOps/Cloud certifications.

**Goal:** Build a practical lab environment as portfolio evidence for AWS CLF → AWS SAA → CKA.

---

## Hardware

| Component | Detail |
|-----------|--------|
| CPU | Intel Core i5-1038NG7 (4c/8t) |
| RAM | 16 GB |
| Storage | ~100 GB NVMe |
| Hypervisor | Proxmox VE 9.2.3 |
| Offsite backup | Tuxis PBS (Netherlands, encrypted) |

---

## Architecture

```
                        192.168.18.0/24
┌──────────────────────────────────────────────────────────────┐
│  Proxmox VE 9.2.3  (192.168.18.15)                          │
│                                                              │
│  CT 100 · AdGuard        (192.168.18.x)  DNS + ad-block     │
│  CT 101 · Tailscale      (192.168.18.x)  VPN mesh           │
│                                                              │
│  CT 103 · Services       (192.168.18.29) Docker stack       │
│  ├── n8n + Postgres      :5678           Automation          │
│  ├── Vaultwarden         (internal)      Password manager    │
│  ├── Nginx Proxy Manager :80/:443        Reverse proxy       │
│  └── Portainer           :9000           Container UI        │
│                                                              │
│  CT 104 · k3s            (192.168.18.39) Kubernetes         │
│  ├── Traefik             LoadBalancer    Ingress controller  │
│  ├── Homepage            home.lab        Dashboard           │
│  └── Uptime Kuma         uptime.lab      Monitoring          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
         │
         ▼
   Tuxis PBS (offsite, encrypted, Netherlands)
```

---

## Phases

| Phase | Description | Status |
|-------|-------------|--------|
| [Phase 0 — Hardening](phases/phase-0-hardening/) | SSH, firewall, PBS backups, Telegram alerts | ✅ Done |
| [Phase 1 — LXC Migration](phases/phase-1-lxc-migration/) | Replaced VMs with LXC containers, Docker on CT 103 | ✅ Done |
| [Phase 2 — Services Stack](phases/phase-2-services-stack/) | Full self-hosted stack on CT 103 | ✅ Done |
| [Phase 3 — k3s](phases/phase-3-k3s/) | Single-node k3s on CT 104, Traefik ingress, real workloads | ✅ Done |
| [Phase 4 — IaC](phases/phase-4-iac/) | Terraform (bpg/proxmox) + Ansible, 4 CTs in state | ✅ Done |
| [Phase 5 — Observability](phases/phase-5-observability/) | Prometheus + Grafana + Loki + Promtail en k3s | ✅ Done |
| [Phase 7 — AWS Foundations](phases/phase-7-aws-foundations/) | VPC + EC2 + Budget en AWS, base de CLF/SAA | ✅ Done |

---

## Automation

Homelab automation runs on n8n (self-hosted at `gdn8n.duckdns.org`):

- **Daily monitor** — checks Proxmox reachability and CT status; alerts via Telegram if anything is down (silence = all good)
- **Weekly digest** — Sunday summary of homelab state → Notion + Telegram
- **Desktop maintenance log** — systemd timer posts logs to a Notion database via webhook

---

## Architecture Decisions

Non-obvious technical decisions, with context and trade-offs, in [`docs/decisions/`](docs/decisions/).

---

## License

MIT

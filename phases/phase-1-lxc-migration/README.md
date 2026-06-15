# Phase 1 — LXC Migration

Replaced two heavyweight VMs with lean LXC containers, consolidating all services into a single Docker host. This freed ~5.8 GB RAM and ~160 GB of virtual disk space.

**Time invested:** ~4h across multiple sessions  
**Completed:** 2026-05-22

---

## The problem

The original setup had two VMs doing work that didn't require full virtualization:

| Guest | Type | RAM | Disk | Contents |
|-------|------|-----|------|----------|
| VM 102 | VM | 4 GB | 60 GB | Vaultwarden + NPM + ntfy |
| VM 103 | VM | 4 GB | 160 GB | ZimaOS (unused) |

VMs carry ~600 MB overhead per guest vs LXC. With 16 GB total RAM and certification labs planned, every GB counts.

---

## What was done

### 1. Eliminated VM 103 (ZimaOS)

ZimaOS was occupying 160 GB of virtual disk on a ~100 GB NVMe (LVM thin provisioning). It had no useful running services.

```bash
qm shutdown 103
qm destroy 103 --purge --skiplock
```

### 2. Created CT 103 — services

New LXC container to replace both VMs:

```
OS: Debian 12
Cores: 2
RAM: 3 GB + 1 GB swap
Disk: 15 GB (local-lvm)
Features: nesting=1, keyctl=1
```

Bootstrapped with: curl, vim, htop, ncdu, jq, tree, dnsutils, locale fix (en_US.UTF-8).

Installed Docker via official script + docker-compose-plugin. Directory structure:

```
/opt/docker/
├── vaultwarden/
├── npm/
├── n8n/
├── uptime-kuma/
└── homepage/
```

### 3. Migrated Vaultwarden from VM 102

Migration was straightforward — Vaultwarden stores everything in a single data directory:

1. Run new container in CT 103 pointing to `/opt/docker/vaultwarden/data/`
2. Copy data dir from VM 102 via `scp`
3. Validate login and vault access
4. Done

### 4. Migrated NPM from VM 102

Copied NPM's config volume, reconnected proxy hosts to the new backend IPs (CT 103 at `192.168.18.29`).

**Lesson learned:** NPM proxy hosts store the backend IP, not a hostname — if the CT IP changes, every proxy host needs manual update.

### 5. Eliminated VM 102 (Ubuntu)

Kept VM 102 powered off for one week as a rollback option, then destroyed it.

```bash
qm shutdown 102
qm destroy 102 --purge
```

ntfy was not migrated — Telegram already covers alerting, no need for a second system.

---

## Strategic decision

Originally the plan included a media server (LXC 104 + \*arr stack). Dropped because:

1. No external storage — the NVMe can't hold a media library
2. \*arr stack is off-scope for DevOps/Cloud certifications

Instead, CT 104 was reserved for k3s (Phase 3).

---

## State after Phase 1

| Guest | RAM | Services |
|-------|-----|----------|
| CT 100 | 512 MB | AdGuard |
| CT 101 | 512 MB | Tailscale |
| CT 103 | 3 GB | Vaultwarden + NPM + (n8n next) |
| ~~VM 102~~ | freed 4 GB | — |
| ~~VM 103~~ | freed 1.8 GB | — |

**Total RAM freed:** ~5.8 GB  
**Disk junk removed:** ~160 GB virtual allocation

---

## Next: [Phase 2 — Services Stack](../phase-2-services-stack/)

# Phase 0 — Hardening

Quick-win security pass before touching anything else. The goal was to get the Proxmox node to a safe baseline: encrypted offsite backups, firewall, clean kernel state, and reliable alerting.

**Time invested:** ~2.5h  
**Completed:** 2026-05-18

---

## What was done

### 1. Offsite backup (Tuxis PBS)

Configured automated daily backups to a Proxmox Backup Server in the Netherlands (Tuxis):

```
Storage: Tuxis-PBS (pbsfl001.ede.tuxis.nl, encrypted)
Schedule: 03:30 daily
Compression: zstd
```

Retention policy replacing the previous `keep-all=1`:

```
keep-last=3
keep-daily=7
keep-weekly=4
keep-monthly=6
```

The encryption key for the PBS datastore is backed up in 1Password + physical paper. Without it, offsite backups are unrecoverable if the Proxmox node dies.

Removed 2 pre-existing duplicate backup jobs and 11 GB of orphaned local backups in `/var/lib/vz/dump/`.

Test result: AdGuard CT backup in ~10s, 89.9% deduplication, 19.5 MiB/s to Netherlands.

### 2. Firewall

Enabled at Datacenter and Node level with a `admin-access` security group:

```
ACCEPT TCP 22    (SSH)   from 192.168.18.0/24
ACCEPT TCP 8006  (UI)    from 192.168.18.0/24
ACCEPT TCP 5900:5999 (VNC) from 192.168.18.0/24
Default policy: DROP
```

**Lesson learned:** Firewall rules are created with `enable=0` by default in Proxmox — they must be explicitly enabled, which is easy to miss.

### 3. Kernel cleanup

Removed old kernel `proxmox-kernel-6.17.2-1`. Recovered ~8 GB in `/`.

```bash
proxmox-boot-tool kernel list
apt purge proxmox-kernel-6.17.2-1-pve-signed
apt autoremove
proxmox-boot-tool refresh
```

Active kernels after cleanup: `7.0.6-2-pve` (running) + `6.17.13-8-pve` (fallback).

### 4. Telegram alerts

Replaced ntfy with a Telegram bot (`@Homelab_gustavo_bot`) as the primary notification target for Proxmox events (backup results, container state changes).

**Lesson learned:** Telegram's `parse_mode: Markdown` breaks silently on characters like `_`, `*`, `[` in container names or log output. Use plain text for system notifications.

### 5. SSH hardening (added 2026-06-12)

```
PasswordAuthentication no
PermitRootLogin prohibit-password
```

SSH keys scoped by purpose:
- `id_homelab` → Proxmox access only
- `id_ed25519` → GitHub only

---

## State after Phase 0

| Item | Before | After |
|------|--------|-------|
| Offsite backup | `keep-all=1`, no retention | Daily 03:30, 7-day retention |
| Local backup junk | 11 GB orphaned dumps | Cleaned |
| Old kernels | 2 kernels | 1 old removed, ~8 GB recovered |
| Firewall | Off | LAN-only, DROP default |
| Alerting | ntfy (broken) | Telegram bot |
| SSH auth | Password + key | Key only |

---

## Next: [Phase 1 — LXC Migration](../phase-1-lxc-migration/)

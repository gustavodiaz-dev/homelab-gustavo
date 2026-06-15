# Phase 3 — k3s on LXC (CKA Prep)

Running a single-node k3s cluster inside an unprivileged LXC container on Proxmox, with Traefik as the ingress controller and real workloads migrated from Docker.

---

## What was built

- k3s v1.35.5 running inside **CT 104** (Debian 12, 2 cores, 2 GB RAM, 20 GB disk)
- **Traefik** deployed as a LoadBalancer ingress controller (IP: `192.168.18.39`)
- **Homepage** migrated from Docker (CT 103) to a Kubernetes Deployment
- **Uptime Kuma** migrated from Docker (CT 103) to a Kubernetes StatefulSet with persistent storage
- Internal DNS entries in AdGuard pointing `*.lab` to `192.168.18.39`
- **Homepage Kubernetes widget** — reads live pod/deployment status from the k3s API via ClusterRole + in-cluster auth
- **Docker socket proxy** (`tecnativa/docker-socket-proxy`) on CT 103 — exposes Docker API over TCP so Homepage in k3s can still show CT 103 container statuses

---

## Cluster state

```
$ kubectl get nodes
NAME   STATUS   ROLES           AGE    VERSION
k3s    Ready    control-plane   5d2h   v1.35.5+k3s1

$ kubectl get pods -A
NAMESPACE     NAME                                      READY   STATUS
homepage      homepage-56654db55f-dflgd                 1/1     Running
kube-system   coredns-8db54c48d-6p8rm                   1/1     Running
kube-system   local-path-provisioner-5d9d9885bc-cg68f   1/1     Running
kube-system   metrics-server-786d997795-sdlm9            1/1     Running
kube-system   traefik-9bcdbbd9-5vb48                    1/1     Running
uptime-kuma   uptime-kuma-0                             1/1     Running

$ kubectl get ingress -A
NAMESPACE     NAME          CLASS     HOSTS        ADDRESS
homepage      homepage      traefik   home.lab     192.168.18.39
uptime-kuma   uptime-kuma   traefik   uptime.lab   192.168.18.39
```

---

## The challenge: k3s inside an unprivileged LXC

Running k3s inside an unprivileged LXC on Proxmox 9.2 requires specific kernel and namespace configuration. The container cannot run with default settings.

### LXC config (`/etc/pve/lxc/104.conf`)

```ini
arch: amd64
cores: 2
features: nesting=1,keyctl=1
hostname: k3s
memory: 2048
unprivileged: 1

# Required for k3s
lxc.apparmor.profile: unconfined
lxc.cgroup2.devices.allow: a
lxc.cap.drop:
lxc.mount.auto: proc:rw sys:rw cgroup:rw
lxc.mount.entry: /dev/kmsg dev/kmsg none bind,create=file
```

The `/dev/kmsg` bind mount must be created on the **Proxmox host** before starting the container:

```bash
# On Proxmox host — run once, persists via lxc.mount.entry
touch /dev/kmsg
```

### k3s config (`/etc/rancher/k3s/config.yaml`)

```yaml
kubelet-arg:
  - "feature-gates=KubeletInUserNamespace=true"
disable-cloud-controller: true
```

`disable-cloud-controller: true` prevents k3s from waiting for a cloud provider that will never come — without this, nodes get stuck with a `node.cloudprovider.kubernetes.io/uninitialized: NoSchedule` taint.

Remove it manually if it appears:

```bash
kubectl taint nodes k3s node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule-
```

### Install

```bash
curl -sfL https://get.k3s.io | sh -
```

k3s ships with Traefik, CoreDNS, local-path-provisioner, and metrics-server out of the box — no Helm charts needed for basic ingress.

---

## Workloads

### Homepage

A Deployment with a PVC for config persistence. Exposed via Traefik Ingress at `home.lab`.

Key resources:
- `Deployment` — 1 replica, `ghcr.io/gethomepage/homepage:latest`
- `PersistentVolumeClaim` — 50Mi, `local-path` storage class
- `Ingress` — `home.lab` → `homepage:3000`

### Uptime Kuma

A StatefulSet (not a Deployment) because Uptime Kuma needs stable storage identity across restarts.

Key resources:
- `StatefulSet` — 1 replica, `louislam/uptime-kuma:1`
- `PersistentVolumeClaim` — 500Mi, `local-path` storage class
- `Ingress` — `uptime.lab` → `uptime-kuma:3001`

### Storage

k3s's built-in `local-path` provisioner handles PVC provisioning automatically. No external storage backend needed for a single-node setup.

```
$ kubectl get pvc -A
NAMESPACE     NAME                 STATUS   CAPACITY   STORAGECLASS
homepage      homepage-config      Bound    50Mi       local-path
uptime-kuma   data-uptime-kuma-0   Bound    500Mi      local-path
```

---

## Architecture

```
                    AdGuard DNS
                 *.lab → 192.168.18.39
                          │
                          ▼
              CT 104 · k3s (192.168.18.39)
              ┌───────────────────────────┐
              │  Traefik  (LoadBalancer)  │
              │      ┌────────┴────────┐  │
              │      ▼                 ▼  │
              │  home.lab        uptime.lab│
              │  Homepage        Uptime   │
              │  Deployment      Kuma     │
              │  (PVC 50Mi)      StatefulSet│
              │                 (PVC 500Mi)│
              └───────────────────────────┘
```

---

## What I learned

- **Unprivileged LXC + k3s is possible** but needs specific AppArmor, cgroup, and mount configuration. The `lxc.mount.auto: proc:rw sys:rw cgroup:rw` line is the most critical one.
- **`/dev/kmsg` must exist on the host** and be bind-mounted into the container. Without it, kubelet fails to start.
- **`disable-cloud-controller: true` is mandatory** in non-cloud environments. The uninitialized taint silently prevents pods from being scheduled.
- **StatefulSet vs Deployment**: Use StatefulSet when the app depends on stable storage identity (Uptime Kuma loses its database on Deployment restarts). Deployments are stateless by design.
- **Traefik is production-ready out of the box** with k3s — the IngressClass is pre-configured, no extra RBAC setup needed.
- **`local-path` provisioner** is sufficient for a single-node homelab. PVCs are provisioned automatically from node-local storage.

---

## Homepage Kubernetes integration

Since Homepage runs inside the cluster, it uses in-cluster auth automatically — no kubeconfig file needed. The only requirement is a ClusterRole with read permissions bound to Homepage's ServiceAccount.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: homepage-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "nodes", "services", "endpoints", "namespaces"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "replicasets", "daemonsets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["metrics.k8s.io"]
    resources: ["nodes", "pods"]
    verbs: ["get", "list"]
```

In `kubernetes.yaml`:
```yaml
mode: cluster
```

In `services.yaml`, use `pod-selector` (not `app`) — Homepage maps `app:` to the label `app.kubernetes.io/name=`, which doesn't match pods created with the simpler `app=` label:

```yaml
- Kubernetes (CT 104):
    - Uptime Kuma:
        namespace: uptime-kuma
        pod-selector: app=uptime-kuma   # explicit selector, not app: uptime-kuma
```

**Lesson learned:** `app: value` in Homepage services.yaml resolves to `app.kubernetes.io/name=value`. If your pods use the plain `app=value` label (which is common for manually written manifests), you must use `pod-selector` instead.

## Docker socket proxy (CT 103 → CT 104)

Homepage in k3s can't access CT 103's Docker socket directly. Solution: run `tecnativa/docker-socket-proxy` in CT 103, exposing a read-only Docker API over TCP.

```yaml
# In CT 103 docker-compose.yml
socket-proxy:
  image: tecnativa/docker-socket-proxy:latest
  ports:
    - '192.168.18.29:2375:2375'
  environment:
    - CONTAINERS=1
    - INFO=1
    - EVENTS=1
    - POST=0      # read-only
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
```

```yaml
# In Homepage docker.yaml (k3s config PVC)
my-server:
  host: 192.168.18.29
  port: 2375
```

**Note:** Remove `ping:` entries from Homepage services when running inside k3s — ICMP is blocked from pod network to LAN hosts.

---

## Next: Phase 4 — IaC (Terraform + Ansible)

- Terraform provider for Proxmox to codify CT creation
- Ansible playbooks for the full stack (CT 103 + CT 104)
- Terraform remote state in S3 (AWS Free Tier)

# Fase 5 — Observabilidad (Prometheus + Grafana + Loki)

## Objetivo

Visibilidad completa del homelab: métricas de hosts, containers, Proxmox y workloads de k3s, más logs centralizados de todos los pods.

## Arquitectura

```
                    ┌─────────────────────────────────┐
                    │  CT104 — k3s (namespace monitoring) │
                    │                                  │
                    │  Prometheus ←── scrape ──────────┼── CT104 node-exporter (DaemonSet)
                    │      │                           │   kube-state-metrics
                    │      │◄──────────────────────────┼── CT103 node-exporter :9100
                    │      │◄──────────────────────────┼── pve-exporter :9221 → Proxmox API
                    │      │                           │
                    │  Grafana ──── ingress ───────────┼── grafana.lab (Traefik)
                    │      │◄── Prometheus datasource  │
                    │      │◄── Loki datasource        │
                    │                                  │
                    │  Loki ◄──── Promtail (DaemonSet) │
                    └─────────────────────────────────┘

                    ┌──────────────────────────────┐
                    │  CT103 — Docker              │
                    │  node-exporter  :9100        │
                    │  pve-exporter   :9221        │
                    └──────────────────────────────┘
```

## Componentes

| Componente | Chart | Namespace | Función |
|-----------|-------|-----------|---------|
| Prometheus | prometheus-community/prometheus | monitoring | Almacena métricas, scrape automático de k3s |
| Grafana | grafana/grafana | monitoring | Visualización, dashboards, alertas |
| Loki | grafana/loki 3.6.7 | monitoring | Almacena logs (single binary mode) |
| Promtail | grafana/promtail | monitoring | Recolecta logs de todos los pods → Loki |
| node-exporter | Docker (CT103) | — | Métricas del host CT103 |
| pve-exporter | Docker (CT103) | — | Métricas de Proxmox via API token |
| Grafana unified alerting | nativo (chart grafana) | monitoring | Evalúa reglas, dispara webhook a n8n |
| n8n workflow `[Homelab] Alertas Grafana → Telegram` | n8n (CT103) | — | Recibe el webhook, formatea y envía a Telegram |

## Acceso

**Grafana:** http://grafana.lab  
Usuario: `admin` / Contraseña: configurada en `grafana-values.yaml`

> Cambiar la contraseña en el primer login: Profile → Change password

## Métricas disponibles

### Proxmox (via pve-exporter)
- Estado de CTs: `pve_up{id="lxc/103"}` etc.
- CPU y RAM del nodo Proxmox
- Estado de almacenamiento (local-lvm, Tuxis-PBS)

### CT103 (via node-exporter)
- CPU, RAM, disco
- Carga del sistema, I/O

### CT104 / k3s (via node-exporter + kube-state-metrics)
- Métricas del nodo k3s
- Estado de Deployments, Pods, PVCs
- Uso de recursos por namespace

## Dashboards preinstalados (Grafana)

| Dashboard | ID | Datasource |
|-----------|-----|-----------|
| Node Exporter Full | 1860 | Prometheus |
| Kubernetes Cluster | 7249 | Prometheus |
| Docker cAdvisor | 893 | Prometheus |

## Configuración IaC

Los values de Helm viven en `homelab-iac/k8s/monitoring/`:

```bash
# Actualizar Prometheus
helm upgrade prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --values k8s/monitoring/prometheus-values.yaml

# Actualizar Grafana
helm upgrade grafana grafana/grafana \
  --namespace monitoring \
  --values k8s/monitoring/grafana-values.yaml

# Actualizar Loki
helm upgrade loki grafana/loki \
  --namespace monitoring \
  --values k8s/monitoring/loki-values.yaml
```

## Scrape configs externos

El Prometheus scrape a dos targets fuera del cluster k3s:

```yaml
# CT103 — host metrics
- job_name: ct103-node
  static_configs:
    - targets: ['192.168.18.29:9100']

# Proxmox — via API token terraform
- job_name: proxmox
  metrics_path: /pve
  params:
    target: ['192.168.18.15']
  static_configs:
    - targets: ['192.168.18.29:9221']
```

## Decisiones de diseño

### ¿Por qué Loki single binary?
El modo distribuido de Loki requiere memcached (chunks-cache + results-cache) que consume ~500MB+ adicionales. Con CT104 en 3GB, single binary es la opción correcta para homelab.

### ¿Por qué pve-exporter en CT103 y no en Proxmox?
Instalar pip en Proxmox ensucia el sistema base del hipervisor. Correrlo como Docker container en CT103 lo mantiene aislado y aprovecha la conectividad que ya tiene CT103 con la API de Proxmox (mismo token `root@pam!terraform`).

### ¿Por qué no kube-prometheus-stack?
El chart all-in-one instala Alertmanager + muchas reglas de alerta por defecto que consumirían ~800MB RAM. Las alertas ya las manejamos via n8n + Telegram. Prometheus + Grafana por separado es más liviano y controlable.

### ¿Por qué alerting nativo de Grafana en vez de Alertmanager o ntfy?
Grafana ya trae unified alerting (evaluación de reglas + contact points + políticas de notificación) sin necesitar el componente Alertmanager aparte, evitando el costo de RAM mencionado arriba. Para el canal de notificación se consideró ntfy, pero no está instalado en el homelab y hubiera sido una herramienta nueva para resolver algo que el patrón n8n + Telegram ya cubre en el resto de los servicios. El contact point de Grafana es un webhook simple apuntando al n8n existente (`http://192.168.18.29:5678/webhook/grafana-alert`), que reenvía a Telegram. Tres reglas cubren los blind spots reales: instancia caída (`up == 0`, 5 min), disco `/` bajo 15% libre (15 min) y memoria sobre 90% de uso (10 min).

## Estado

- [x] CT104: RAM aumentada a 3072MB (Terraform)
- [x] Prometheus: scrape k3s + CT103 + Proxmox
- [x] Grafana: accesible en grafana.lab, datasources preconfigurados
- [x] Loki: single binary, 7 días retención
- [x] Promtail: logs de todos los pods → Loki
- [x] node-exporter: CT103 host metrics
- [x] pve-exporter: Proxmox metrics via API token
- [x] DNS: grafana.lab en AdGuard
- [x] Alertas en Grafana (webhook → n8n → Telegram): InstanceDown, HighDiskUsage, HighMemoryUsage
- [x] cAdvisor en CT103 para métricas de Docker containers
- [ ] Dashboards custom para n8n y Vaultwarden

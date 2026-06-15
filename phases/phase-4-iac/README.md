# Fase 4 — Infrastructure as Code (Terraform + Ansible)

## Objetivo

Convertir el homelab de configuración manual a IaC: cada container, su configuración de red, recursos y servicios queda declarado en código versionado. El estado real de Proxmox puede recrearse desde cero con un `terraform apply` y un par de playbooks de Ansible.

## Repositorio

Todo el código IaC vive en **`homelab-iac`** (repositorio separado, privado):

```
homelab-iac/
├── terraform/
│   ├── modules/lxc/          # Módulo reutilizable para LXC containers
│   └── environments/homelab/ # Estado y variables del homelab
└── ansible/
    ├── inventory/            # hosts.yml (gitignored) + hosts.yml.example
    ├── playbooks/            # ct103-services.yml, ct104-k3s.yml
    └── files/ct103/          # docker-compose.yml, .env.example
```

## Herramientas

| Herramienta | Versión | Rol |
|-------------|---------|-----|
| Terraform | 1.11.x | Gestiona los LXC containers en Proxmox |
| Provider bpg/proxmox | ~0.78 | API de Proxmox para Terraform |
| Ansible | 2.x | Configura servicios dentro de los containers |

---

## Terraform

### Provider

Se usa `bpg/proxmox` (no el oficial de HashiCorp) porque tiene soporte completo para LXC containers, features y configuración avanzada de Proxmox.

### Autenticación

API token con `privsep=0` (mismo nivel que root@pam, sin separación de privilegios):

```
root@pam!terraform
```

> **Limitación conocida**: Proxmox solo permite cambiar el flag `keyctl` via login directo de root, no via API token. Por eso el módulo LXC tiene `lifecycle { ignore_changes = [features] }`. Los features se configuran manualmente una vez al crear el CT.

### Containers gestionados

| CT | Hostname | CPU | RAM | Disco | IP |
|----|----------|-----|-----|-------|----|
| 100 | adguard | 1c | 768MB | 2G | DHCP (192.168.18.17) |
| 101 | tailscale | 1c | 512MB | 2G | DHCP (192.168.18.20) |
| 103 | services | 3c | 3072MB | 25G | 192.168.18.29 |
| 104 | k3s | 2c | 2048MB | 20G | DHCP (192.168.18.39) |

### Flujo de trabajo

```bash
cd terraform/environments/homelab

# Ver cambios pendientes
terraform plan

# Aplicar
terraform apply

# Importar un CT existente al estado
terraform import module.ct100_adguard.proxmox_virtual_environment_container.this 100
```

### Estado (backend)

Local por ahora (`terraform.tfstate`, gitignored). Pendiente migración a S3 cuando se configure la cuenta AWS.

---

## Ansible

### Inventario

```bash
cp ansible/inventory/hosts.yml.example ansible/inventory/hosts.yml
# Editar con IPs reales
```

### Playbooks disponibles

**CT103 — Stack Docker:**
```bash
ansible-playbook -i inventory/hosts.yml playbooks/ct103-services.yml

# Solo instalar Docker:
ansible-playbook -i inventory/hosts.yml playbooks/ct103-services.yml --tags docker

# Solo redesplegar servicios:
ansible-playbook -i inventory/hosts.yml playbooks/ct103-services.yml --tags deploy
```

**CT104 — k3s:**
```bash
ansible-playbook -i inventory/hosts.yml playbooks/ct104-k3s.yml

# Solo parchear config LXC en Proxmox (requiere reboot de CT104 después):
ansible-playbook -i inventory/hosts.yml playbooks/ct104-k3s.yml --tags lxc

# Solo instalar/validar k3s:
ansible-playbook -i inventory/hosts.yml playbooks/ct104-k3s.yml --tags k3s
```

---

## Decisiones de diseño

### ¿Por qué Terraform en la laptop y no en un CT?

Terraform necesita acceso a la API de Proxmox (puerto 8006). Correrlo en un CT del mismo Proxmox crea una dependencia circular: si el nodo falla, el CT también falla y pierdes acceso al IaC. La laptop es una fuente de control externa e independiente.

A futuro, cuando se agregue un segundo nodo Proxmox o se configure un servidor de CI/CD dedicado, se puede migrar.

### ¿Por qué módulo `lxc` en lugar de recursos directos?

El módulo encapsula los defaults del homelab (debian, local-lvm, nameserver 192.168.18.17, gateway 192.168.18.1) y permite definir cada CT en ~15 líneas en `main.tf` en lugar de ~60.

### `lifecycle { ignore_changes = [operating_system, features] }`

- `operating_system`: el template solo importa al crear el CT. Proxmox no lo expone consistentemente en el API después.
- `features`: Proxmox requiere root@pam directo para cambiar `keyctl`. El API token devuelve HTTP 403. Como los features son estables (nesting+keyctl se configura una sola vez al crear), ignorarlos es la solución correcta.

---

## Lecciones aprendidas

- **Importar CTs existentes**: `terraform import` permite llevar CTs ya creados al state sin recrearlos. Imprescindible cuando el homelab ya existe.
- **`privsep=0` no es suficiente para todo**: el API token con privsep=0 tiene los mismos permisos que root@pam en casi todo, pero no para feature flags como `keyctl`.
- **El nodo se llama "Macbook"**: `pvesh` falla silenciosamente si usas `/nodes/proxmox/`. El nombre real del nodo se obtiene con `pvesh get /nodes`.
- **bpg/proxmox vs hashicorp/proxmox**: el provider de HashiCorp está deprecado. bpg es el mantenido activamente y tiene soporte real para LXC.

---

## Estado

- [x] Terraform instalado y configurado
- [x] 4 containers importados al state (CT100, CT101, CT103, CT104)
- [x] `terraform apply` limpio — sin drift
- [x] Playbook CT103 (Docker stack)
- [x] Playbook CT104 (k3s)
- [ ] Backend S3 para estado remoto (pendiente cuenta AWS)
- [ ] CT105 — container dedicado IaC (futuro, portfolio setup)

# Fase 6 — CI/CD (GitHub Actions + self-hosted runner)

## Objetivo

Automatizar el ciclo `plan`/`check` → `apply`/`deploy` de Terraform y Ansible al hacer push a `homelab-iac`, en vez de ejecutar todo a mano desde el laptop cada vez. Proxmox solo es alcanzable desde la LAN, así que el runner de GitHub Actions tiene que vivir dentro del propio homelab.

## Arquitectura

```
   GitHub (homelab-iac)                  LAN 192.168.18.0/24
┌──────────────────────┐         ┌──────────────────────────────┐
│  Push / PR a main     │         │  CT105 — ci-runner             │
│  ├─ terraform.yml     │  poll   │  GitHub Actions self-hosted    │
│  │   plan (auto)      │◄────────┤  runner                        │
│  │   apply (manual)   │         │       │                        │
│  └─ ansible.yml       │         │       ▼                        │
│      check (auto)     │         │  Proxmox API + SSH a CT103/104 │
│      deploy (manual)  │         └──────────────────────────────┘
└──────────────────────┘
```

## Componentes

| Componente | Recurso | Función |
|-----------|---------|---------|
| CT105 — ci-runner | LXC vía Terraform (`modules/lxc`) | Host del runner, dentro de la LAN |
| GitHub Actions runner | Self-hosted, registrado en CT105 | Ejecuta los workflows con acceso real a Proxmox |
| `.github/workflows/terraform.yml` | `plan` automático en PR, `apply` por `workflow_dispatch` | CI/CD de la infraestructura LXC |
| `.github/workflows/ansible.yml` | `check --diff` automático en PR, `deploy` por `workflow_dispatch` | CI/CD de la configuración de cada host |
| Backend S3 | Bucket dedicado, `use_lockfile`, sin DynamoDB | State remoto — el runner ephemeral borra su workspace después de cada job |
| Repository secrets | `AWS_ACCESS_KEY_ID`/`SECRET`, `PROXMOX_API_TOKEN`, `HOMELAB_SSH_PRIVATE_KEY`, `CT103_ENV_FILE` | Credenciales que los jobs necesitan sin tocar el repo |

## Acceso

Disparo manual de `apply`/`deploy` desde GitHub: pestaña *Actions* → workflow → *Run workflow*. Solo el dueño del repo puede hacerlo (repo privado).

## Configuración IaC

El código vive en `homelab-iac/.github/workflows/` (repo privado, ver su README para el detalle de secrets y estructura).

## Decisiones de diseño

### ¿Por qué `workflow_dispatch` manual en vez del gate nativo de Environments?

GitHub Free no soporta "required reviewers" en Environments para repos privados (devuelve HTTP 422 al intentar configurarlo — es feature de plan pago). En vez de pagar por eso o hacer público el repo, el gate real pasa a ser que solo un colaborador con acceso de escritura puede ejecutar el `workflow_dispatch` — en un repo privado de un solo operador, equivalente en la práctica. Detalle completo en [ADR 0002](../../docs/decisions/0002-manual-dispatch-as-the-approval-gate.md).

### ¿Por qué el state de Terraform vive en S3 sin DynamoDB?

El runner ephemeral de CT105 no conserva nada entre jobs, así que el state no puede quedar en disco local. S3 con `use_lockfile` (locking nativo de Terraform 1.11+) cubre el caso de uso de un solo operador sin necesitar una tabla DynamoDB aparte. Detalle en [ADR 0001](../../docs/decisions/0001-remote-state-in-s3-without-dynamodb.md).

### ¿Por qué el playbook de CT105 está excluido del propio CI?

El playbook `ct105-ci-runner.yml` configura (y puede reiniciar) al mismo runner que lo estaría ejecutando — correrlo desde el propio CI es una receta para que el job se corte a la mitad. Se sigue corriendo a mano desde el laptop. Detalle en [ADR 0003](../../docs/decisions/0003-exclude-self-managing-playbook-from-ci.md).

### ¿Por qué el inventario de Ansible está commiteado en vez de gitignoreado?

Son IPs de la LAN local (`192.168.18.0/24`), no son secret — y tenerlo en el repo evita que cada sesión de CI necesite reconstruirlo. Detalle en [ADR 0004](../../docs/decisions/0004-commit-the-ansible-inventory.md).

### ¿Por qué los secrets se materializan como archivos transitorios en CI?

Algunas herramientas (claves SSH, tokens de Proxmox) solo aceptan rutas de archivo, no variables de entorno. Se escriben a disco al arranque del job y se descartan al terminar — nunca llegan a vivir como archivos del repo. Detalle en [ADR 0005](../../docs/decisions/0005-secrets-as-transient-files-in-ci.md).

## Estado

- [x] CT105 (runner) creado vía Terraform
- [x] GitHub Actions runner instalado y registrado en CT105
- [x] `terraform.yml` + `ansible.yml`: plan/check automático en PR, apply/deploy por disparo manual
- [x] Repository secrets configurados
- [x] Prueba end-to-end real: PR → plan → merge → disparo manual → apply real (`terraform.yml` 0 cambios, `ansible.yml` check + deploy real contra ct104-k3s)
- [x] 4 bugs reales encontrados y corregidos en el camino (precedencia de `--private-key`, scope de `ct105-ci-runner` excluido del CI, distribución de `id_homelab` a CT103/CT104, secret `CT103_ENV_FILE` faltante)

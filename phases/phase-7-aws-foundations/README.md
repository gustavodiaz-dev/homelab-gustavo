# Fase 7 — AWS Foundations (VPC + EC2 + Budget)

## Objetivo

Primera infraestructura real en AWS — fundamentos de VPC y EC2, base directa del temario de AWS CLF-C02 y arranque de SAA-C03. Hasta esta fase, AWS solo existía en el proyecto como backend de state de Terraform (un bucket S3), sin práctica real de redes ni cómputo en la nube.

## Arquitectura

```
                    AWS (us-east-1)
┌──────────────────────────────────────────────────────┐
│  VPC  10.42.0.0/16                                   │
│                                                        │
│  ┌──────────────────────────────────┐                │
│  │  Subred pública  10.42.1.0/24     │                │
│  │                                    │                │
│  │  EC2 t3.micro (Amazon Linux 2023) │◄── SSH (22)    │
│  │  IP pública                       │    solo desde  │
│  │  Security Group: aws-lab-sg       │    mi IP       │
│  └──────────────────────────────────┘                │
│             │                                          │
│  Route Table ── 0.0.0.0/0 ──► Internet Gateway         │
└──────────────────────────────────────────────────────┘

  AWS Budgets: alerta por email al 80% y 100% de $20/mes
```

## Componentes

| Componente | Recurso Terraform | Función |
|-----------|--------------------|---------|
| VPC | `aws_vpc` | Red aislada para el lab |
| Subred pública | `aws_subnet` | Único subnet, con salida directa a internet |
| Internet Gateway + Route Table | `aws_internet_gateway`, `aws_route_table` | Salida/entrada de tráfico público |
| Security Group | `aws_security_group` | SSH restringido a una IP, egress abierto |
| Key Pair | `aws_key_pair` | Importa una clave SSH ya existente, ninguna privada nueva en el state |
| Instancia EC2 | `aws_instance` (t3.micro, AL2023 vía `data.aws_ami`) | Cómputo real para practicar |
| Budget | `aws_budgets_budget` | Alerta de costo, no deja pasar sorpresas |

## Acceso

```bash
terraform output ssh_command
# ssh ec2-user@<ip-pública>
```

## Configuración IaC

El código vive en `homelab-iac/terraform/environments/aws-lab/` (repo privado, ver su README para los comandos exactos de `apply`/`destroy`).

## Decisiones de diseño

### ¿Por qué apply/destroy por sesión en vez de dejarlo corriendo?

La cuenta de AWS arrancó con un plan de $100-200 en créditos con vencimiento fijo, y ya hay otros gastos mensuales fijos que no dejan margen para sumar uno nuevo. Dejar una instancia EC2 corriendo 24/7 consumiría ese crédito sin necesidad — el aprendizaje ocurre en sesiones de estudio puntuales, no de forma continua. El flujo es `terraform apply` al empezar a estudiar, `terraform destroy` al terminar. Detalle completo en [ADR 0007](../../docs/decisions/0007-apply-destroy-discipline-for-aws-lab.md).

### ¿Por qué el Security Group restringe SSH a una sola IP y no `0.0.0.0/0`?

Abrir el puerto 22 al mundo en una instancia que aparece en cualquier escaneo automatizado de internet en minutos no tiene ninguna ventaja práctica para este lab — el acceso real siempre va a venir desde la misma IP de casa. Restringirlo a esa IP es gratis y elimina ese vector por completo.

### ¿Por qué AWS Budgets y no CloudWatch Billing Alarm + SNS?

Ambos cumplen la misma función (avisar antes de gastar de más), pero Budgets ya incluye notificación por email nativa sin necesitar un tema SNS aparte — un solo mecanismo de alerta, no dos haciendo lo mismo.

### ¿Por qué no hay NAT Gateway, ALB, ni RDS en esta fase?

Cada uno de esos cobra por hora estén o no en uso (NAT Gateway ronda $32/mes, ALB ~$16/mes mínimo), y no son necesarios para los fundamentos de VPC/EC2/IAM que cubre esta fase. Se evalúan más adelante, en sesiones puntuales de una hora cuando el temario de SAA los requiera — no como infraestructura permanente.

### ¿Por qué no está en la pipeline de CI/CD?

El resto del proyecto tiene `apply` automático vía `workflow_dispatch` para Proxmox, donde no hay costo por hora. Acá sí lo hay, y automatizar el `apply` de algo que cuesta dinero por sesión de estudio no aporta nada — el control manual y local es la decisión correcta mientras el ciclo siga siendo "levantar para estudiar, destruir al cerrar".

## Estado

- [x] VPC + subred pública + Internet Gateway + Route Table
- [x] Security Group restringido a una IP
- [x] Key Pair importado (clave existente, sin generar una nueva)
- [x] Instancia EC2 t3.micro (Amazon Linux 2023)
- [x] AWS Budget con alertas al 80%/100%
- [x] Permisos IAM acotados (`ec2:*` solo en `us-east-1`, `budgets:*`) en vez de admin
- [ ] Primer ciclo real apply → estudio → destroy, verificado en consola

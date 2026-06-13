# DIS ARG Ops Panel — Estado al 13/06/2026

## URLs y recursos

| Recurso | Valor |
|---|---|
| Panel URL | https://disarg-org.github.io/ops-panel/ |
| Callback OAuth | https://disarg-org.github.io/ops-panel/callback.html |
| API Gateway | https://nr8qjpcl4d.execute-api.us-east-1.amazonaws.com/prod/action |
| Lambda | disarg-ops-panel (Python 3.12, us-east-1) |
| Repo frontend | disarg-org/ops-panel (público, GitHub Pages) |
| IAM Role Lambda | disarg-ops-panel-role |

## Autenticación

- GitHub OAuth App: Client ID `Ov23li09kKMNZA4am0BO`
- Secretos en AWS SSM: `/ops-panel/github-client-id` y `/ops-panel/github-client-secret`
- Usuario permitido: `brunili` (hardcodeado en Lambda — `ALLOWED_GITHUB_USERS`)

## IAM Policies del role

| Policy | Permisos |
|---|---|
| disarg-ops-panel-policy | rds:DescribeDBInstances en `*-rds-prod` |
| disarg-ops-panel-cloudwatch | cloudwatch:GetMetricStatistics |
| disarg-ops-panel-eks | eks:DescribeCluster en prod-eks-cluster |
| disarg-ops-panel-ssm | ssm:GetParameter en /ops-panel/* |

## EKS — acceso de la Lambda

- ClusterRole: `ops-panel-viewer` — get/list pods, logs, deployments, cronjobs, metrics.k8s.io
- ClusterRoleBinding: `ops-panel-viewer-binding` → user `ops-panel`
- aws-auth: agregado vía Terraform en `infra-odoo` (aplicado 13/06/2026)

## Funciones activas en la Lambda

| Action | Qué hace |
|---|---|
| github_exchange | Intercambia code OAuth por token GitHub |
| rds_status | Estado + CPU/RAM/conexiones via CloudWatch |
| pod_status | Lista pods del namespace con fase y reinicios |
| pod_metrics | CPU y RAM de pods via metrics.k8s.io |
| pod_logs | Últimas 50 líneas del pod main-odoo-CLIENT |
| health_check | HTTP GET a /web/health — status y tiempo |

## Dashboard

- Una card por cliente con semáforo
- Umbrales: CPU>15% → naranja, CPU>30% → rojo, RAM>85% → naranja, RAM>95% → rojo
- domstor: solo health check (no tiene EKS downscaling ni RDS downscaling)
- Click en card → detalle completo del cliente

## Terraform (infra-odoo)

- `terraform/terraform.tfvars` — eks_users tiene bmolinari + ops-panel-role
- `terraform/modules/eks/aws.tf` — aws_auth_roles tiene ops-panel-role → system:viewers
- `terraform/modules/eks/node_groups.tf` — db_sync restaurado para evitar destroy accidental
- Workflow: manual, action=plan o apply

## Pendientes próxima sesión

1. **Restart de pod** — agregar RBAC `patch` en EKS + modal de confirmación en UI + permisos kubectl rollout
2. **Start/Stop RDS** — agregar `rds:StartDBInstance` y `rds:StopDBInstance` en IAM + modal confirmación
3. **domstor health check** — da Error, investigar (probablemente timeout de red desde Lambda)
4. **Celular con MFA** — el code OAuth expira durante el flujo MFA; solución: pre-login en GitHub antes de abrir el panel
5. **Thresholds configurables** — hoy hardcodeados en JS, podrían ir en SSM

## Lo que NO tocar sin coordinación

- `domstor` — opera 24/7, cualquier cambio requiere coordinación explícita
- `aws-auth` — cualquier cambio debe ir por Terraform, nunca kubectl directo
- Ventanas de deploy: dls/valimartsrl después 17:00 ARG, sursports antes 09:00 o después 22:00 ARG

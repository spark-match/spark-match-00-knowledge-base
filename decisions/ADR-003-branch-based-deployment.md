# 🟣 ADR-003: Branch-based deployment — `main` + `dev`, **un solo ambiente AWS**

> **Architecture Decision Record** — captura el contexto, opciones y consecuencias de una decisión arquitectónica significativa.

> **Estado**: 🟢 Aceptado
> **Fecha**: 2026-07-09
> **Autores**: @Angel (ahincho)
> **Reviewers**: @David, @Andy, @Fabiola (pendiente)

---

## 🎯 Contexto

El equipo propuso tener **2 ramas (`main` + `dev`) en todos los repos** y **2 ambientes AWS separados (dev + prod)**, sugiriendo además adoptar **Terragrunt** para gestionar la configuración multi-ambiente. La propuesta surgió porque:

- En el flujo actual todo se mergea directo a `main` y se despliega a producción.
- Se quiere un "safety net" para validar cambios antes de la sustentación.
- Backend con AWS SAM está listo para Fase 1 (PR #2 abierto).

Constraints del proyecto:

- **TFP académico**, 5 meses, 2-3 devs backend.
- **Créditos AWS limitados** (AWS Academy Lab, sin producción real).
- **Demo / sustentación**: es UN momento, no un deploy continuo.
- **Cada deploy a Lambda cuesta ~$0** y tarda ~30 segundos (feedback loop rápido).
- **Sin presupuesto** para aprender + mantener Terragrunt.

---

## 🔄 Opciones consideradas

### Opción 1: Branch-based deployment + 1 solo ambiente AWS (recomendada) ✅

```
main ← código probado, congelado para sustentación
 ↑
dev ← integración continua, deploy automático
 ↑
feature/* ← trabajo individual
```

- `dev` se deploya **automáticamente** al único ambiente AWS al mergear.
- `main` se deploya solo con **aprobación manual** (workflow_dispatch).
- Diferenciación por **tag** en recursos AWS (`Environment=dev` vs `Environment=prod`).
- **Coste AWS**: ~$0-15/mes (free tier cubre casi todo).

### Opción 2: Folder duplication `live/dev/` + `live/prod/` (sin Terragrunt)

```
live/
├── dev/   ← main.tf duplicado, environment=dev
└── prod/  ← main.tf duplicado, environment=prod
```

- Mismo código Terraform copiado en 2 carpetas.
- 2 state files en S3 (`dev/terraform.tfstate`, `prod/terraform.tfstate`).
- **Coste AWS**: ~$120/mes (2x RDS, 2x NAT, etc.).
- **Complejidad**: baja (Terraform puro).
- Trade-off: paga 8x más AWS para un "safety net" que probablemente no se use en 5 meses.

### Opción 3: Terragrunt + 2 ambientes (sugerencia del equipo)

- Mismo código HCL en `live/_envcommon/`, instanciado por `live/dev/` y `live/prod/`.
- DRY perfecto a largo plazo.
- **Coste AWS**: ~$120/mes.
- **Complejidad**: alta (Terragrunt agrega before/after hooks, run-all, dependencies, una capa más a debuggear).
- Trade-off: ~1 semana aprendiendo Terragrunt que se podría invertir en features del TFP.

### Opción 0 (no considerar): No hacer nada

- Seguir mergeando a `main` y deployando directo a "prod".
- Riesgo: cambios sin validación rompen la demo de sustentación.
- **Razón para descartar**: el dolor es real (sustentación es el momento crítico).

---

## ✅ Decisión

**Opción 1 — Branch-based deployment con 1 solo ambiente AWS.**

### Repos que adoptan el patrón `main` + `dev`

| Repo | ¿Aplica? | Razón |
|---|---|---|
| `02-infrastructure` | ✅ Sí | Terraform deployable |
| `03-backend` | ✅ Sí | SAM deployable |
| `04-frontend` | ✅ Sí | Angular deployable |
| `05-data-pipeline` | ✅ Sí | Pipeline deployable |
| `06-model-training` | ✅ Sí | Training code |
| `08-deep-agent` | ✅ Sí | AgentCore deployable |
| `00-knowledge-base` | ❌ No | Solo docs |
| `01-devops` | ❌ No | Tooling / reusable workflows |
| `07-article` | ❌ No | LaTeX |
| `spark-match-template` | ❌ No | Template |
| `.github` | ❌ No | Org profile |

### Reglas de branch protection

| Regla | `main` | `dev` |
|---|---|---|
| Required approving reviews | **1** (CODE OWNER) | **1** (CODE OWNER) |
| Require code owner reviews | ✅ | ✅ |
| Enforce admins | ✅ | ❌ (facilita hotfixes) |
| Require conversation resolution | ✅ | ❌ |
| Required status checks | ✅ (todos) | ✅ (smoke) |
| Allow force pushes | ❌ | ❌ (por ahora) |
| Allow deletions | ❌ | ❌ |

### Diferenciación en AWS por tag

- Recursos deployados desde `dev` se tagean con `Environment=dev`.
- Recursos deployados desde `main` se tagean con `Environment=prod`.
- En `02-infrastructure`, el módulo `networking` se aplica **una sola vez** (compartido).
- Lambdas en `03-backend` se deployan a `samconfig.dev.toml` o `samconfig.prod.toml` según la rama.

### Flujo de trabajo

```bash
# Trabajo individual
git checkout dev
git pull
git checkout -b feature/mi-cambio

# PR contra dev (review del CODE OWNER del área)
gh pr create --base dev

# Merge a dev → deploy automático al ambiente compartido
# (tag Environment=dev en AWS)

# Promoción a main: cuando se congela para sustentación
gh pr create --base main --head dev
# Aprobación de product-owners + 1 CODE OWNER adicional
# Merge a main → deploy con aprobación manual
```

### Costes estimados (Opción 1)

| Recurso | Coste/mes |
|---|---|
| Lambda invocations (1M) | $0.20 |
| API Gateway (1M) | $1.00 |
| Aurora Serverless v2 (0.5 ACU avg) | $22.00 |
| CloudWatch Logs (10 GB) | $5.00 |
| S3 storage (50 GB) | $1.15 |
| Secrets Manager (2 secrets) | $1.00 |
| **Total estimado** | **~$30/mes** |

(1 ambiente AWS, no 2; cubierto parcialmente por AWS Academy Lab credits.)

---

## 📋 Consecuencias

### ✅ Positivas

- **Coste AWS mínimo**: 1 solo ambiente, no 2.
- **Sin curva de aprendizaje** de Terragrunt (Terraform puro).
- **Cero dependencias nuevas** (no agregar Go binary, no before/after hooks).
- **Feedback loop rápido**: `dev` se deploya en ~30s, no se espera aprobación manual para iterar.
- **Demo protegida**: `main` solo se actualiza antes de sustentación, queda congelada.
- **Cleanup post-sustentación trivial**: borrar el stack de CloudFormation/SAM = $0/mes.
- **Documenta una decisión consciente** sobre la deuda técnica (futuro Terragrunt queda registrado).

### ⚠️ Negativas (tolerables)

- **No hay "staging" real**: si `dev` falla, el mismo ambiente compartido se rompe. Pero: AWS Academy Lab es sandbox, no afecta a usuarios reales.
- **Sin hotfixes fuera de banda**: cualquier cambio debe pasar por `dev` antes de `main`. Pero: `enforce_admins=false` en `dev` permite bypass de CODE OWNER en emergencias.
- **DRY violation futura**: si en algún momento se agregan 3+ ambientes, habrá que migrar a Terragrunt. Pero: eso es decisión de futuro, no del TFP.
- **Tag-based env** no es tan fuerte como cuentas AWS separadas. Pero: para un TFP es suficiente.

### 🛡️ Mitigaciones

- **Backups automatizados** de RDS Aurora antes de deploys a `main` (snapshot manual o AWS Backup).
- **Feature flags** en backend para activar/desactivar features sin redeploy.
- **Smoke tests post-deploy** en workflow de `dev` (curl al endpoint `/health`).
- **Documentar en README** el flujo de promoción `dev → main`.

---

## 🔗 Referencias

- [`D:\UNI\Spark\BACKEND.md`](../) — decisiones de arquitectura backend
- [ADR-001](./ADR-001-backend-hibrido-lambda-mas-agente.md) — backend híbrido
- [ADR-002](./ADR-002-incident-pr-sin-reviewers.md) — incident, PRs sin CODE OWNER review
- [Review de 02-infrastructure](https://github.com/spark-match/spark-match-02-infrastructure/blob/main/REVIEW.md) — auditoría del repo infra
- PR #2 en 03-backend: https://github.com/spark-match/spark-match-03-backend/pull/2 (será retargeted a `dev`)

---

## 🗓️ Plan de implementación

1. ✅ Crear este ADR-003 (canonical)
2. ⏳ Crear rama `dev` en 02-infrastructure
3. ⏳ Crear rama `dev` en 03-backend
4. ⏳ Configurar branch protection para `dev` (tabla arriba)
5. ⏳ Actualizar workflows de CI/CD:
   - `dev` → auto-deploy con sam/terráform
   - `main` → deploy con `workflow_dispatch` + approval
6. ⏳ Retarget PR #2 a `dev` (en lugar de `main`)
7. ⏳ Documentar flujo en READMEs
8. ⏳ (Opcional) Replicar en 04, 05, 06, 08

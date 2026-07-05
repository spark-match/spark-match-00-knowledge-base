# 🔍 Auditoría y Configuración de Permisos — `spark-match-00-knowledge-base`

> Documento de revisión técnica + bitácora de los cambios de gobernanza aplicados.
> Fecha: 2026-07-04 · Ejecutor: `ahincho`

---

## 1. Resumen ejecutivo

| Aspecto | Antes | Después |
|---|---|---|
| Team general con todos los miembros | ❌ No existía | ✅ `@spark-match/members` (4 miembros) |
| Acceso al repo por team | ❌ Ningún team | ✅ `members` con `push` (Write) |
| `BriyitHT`, `dbarretol`, `FabiTaparaQuispe` | PULL (read-only) | **WRITE** (pueden push + PR) |
| CODEOWNERS catch-all funcional | ❌ `@spark-match/devops` sin acceso | ✅ `@spark-match/members` con acceso |
| Branch protection | ✅ Estricta | ✅ Estricta (sin cambios) |

---

## 2. Estado INICIAL (antes del cambio)

### 2.1 Miembros de la organización

| Usuario | ¿Org member? | ¿En algún team? | Acceso al repo |
|---|:---:|:---:|---|
| `ahincho` | ✅ | 8 teams (todos) | ADMIN (directo) |
| `BriyitHT` | ✅ | ❌ ninguno | PULL (colaborador directo) |
| `dbarretol` | ✅ | ❌ ninguno | PULL (colaborador directo) |
| `FabiTaparaQuispe` | ✅ | ❌ ninguno | PULL (colaborador directo) |

### 2.2 Teams con acceso al repo

| Team | ¿Tiene acceso? |
|---|---|
| `owners`, `product-owners`, `devops`, `backend-devs`, `frontend-devs`, `ai-devs`, `qa`, `article-authors` | ❌ **NO** |

### 2.3 CODEOWNERS (versión vieja)

```
*    @spark-match/devops
/onboarding/    @spark-match/product-owners @spark-match/devops
/decisions/     @spark-match/product-owners @spark-match/devops
... etc
```

**Problema**: `@spark-match/devops` solo tenía a `ahincho` como miembro, y ese team NO tenía acceso al repo. GitHub no podía sugerir reviewers automáticamente.

### 2.4 Matriz efectiva de permisos (antes)

| Usuario | ¿Puede ver? | ¿Puede hacer push? | ¿Puede abrir PR? | ¿Puede revisar? | ¿Puede mergear? |
|---|:---:|:---:|:---:|:---:|:---:|
| `ahincho` | ✅ | ✅ | ✅ | ✅ | ✅ (admin override) |
| `BriyitHT` | ✅ | ❌ | ✅ (READ directo) | ❌ | ❌ |
| `dbarretol` | ✅ | ❌ | ✅ (READ directo) | ❌ | ❌ |
| `FabiTaparaQuispe` | ✅ | ❌ | ✅ (READ directo) | ❌ | ❌ |
| Externo (no miembro) | ✅ (público) | ❌ | ✅ (vía fork) | ❌ | ❌ |

---

## 3. Cambios aplicados

### 3.1 Creación del team `@spark-match/members`

```bash
gh api -X POST orgs/spark-match/teams \
  -f name="members" \
  -f description="Team general que incluye a todos los miembros de la organización Spark Match. Usado como catch-all para CODEOWNERS y como grupo con acceso Write a repositorios compartidos (ej: 00-knowledge-base)." \
  -f privacy="closed"
```

**Resultado**:
- `name: members`
- `privacy: closed` (visible para miembros de la org)
- `id: 18333780`
- `html_url: https://github.com/orgs/spark-match/teams/members`

### 3.2 Adición de los 4 miembros

```bash
for USER in ahincho BriyitHT dbarretol FabiTaparaQuispe; do
  gh api -X PUT orgs/spark-match/teams/members/memberships/$USER -f role=member
done
```

**Resultado**:

| Usuario | Rol en el team |
|---|---|
| `ahincho` | `maintainer` (auto-asignado por ser el owner) |
| `BriyitHT` | `member` |
| `dbarretol` | `member` |
| `FabiTaparaQuispe` | `member` |

### 3.3 Permiso Write al repo

```bash
gh api -X PUT orgs/spark-match/teams/members/repos/spark-match/spark-match-00-knowledge-base \
  -f permission=push
```

**Resultado**: team `members` con `permission: push` (Write).

### 3.4 Actualización de CODEOWNERS

PR #1 mergeado: `chore(governance): add @spark-match/members as CODEOWNERS catch-all`

**Cambios**:
- Catch-all: `*` ahora apunta a `@spark-match/members`
- Mantiene CODEOWNERS específicos para paths sensibles
- Añade notas operativas al final del archivo

---

## 4. Estado FINAL (después del cambio)

### 4.1 Permiso efectivo de cada miembro

| Usuario | Permiso | Fuente | ¿Puede push? | ¿Puede PR? | ¿Puede revisar? | ¿Puede mergear? |
|---|:---:|---|:---:|:---:|:---:|:---:|
| `ahincho` | **admin** | directo + team | ✅ | ✅ | ✅ | ✅ (override) |
| `BriyitHT` | **write** | team `members` | ✅ | ✅ | ✅ | ✅ (con 1 aprobación) |
| `dbarretol` | **write** | team `members` | ✅ | ✅ | ✅ | ✅ (con 1 aprobación) |
| `FabiTaparaQuispe` | **write** | team `members` | ✅ | ✅ | ✅ | ✅ (con 1 aprobación) |

### 4.2 Teams con acceso al repo

| Team | Permiso | Miembros |
|---|:---:|---|
| `@spark-match/members` | **push (write)** | 4 (todos los de la org) |

### 4.3 Branch protection (sin cambios)

| Regla | Valor |
|---|---|
| `enforce_admins` | ✅ true |
| `required_reviews` | 1 |
| `code_owner_reviews` | ✅ true |
| `conversation_resolution` | ✅ true |
| `allow_force_pushes` | ❌ false |
| `allow_deletions` | ❌ false |
| `restrictions` | null (cualquiera con write puede push a feature branches) |

---

## 5. Cómo añadir un nuevo miembro

```bash
# 1. Invitar a la org
gh api -X PUT orgs/spark-match/memberships/<github-user> -f role=member

# 2. Agregar al team members (catch-all)
gh api -X PUT orgs/spark-match/teams/members/memberships/<github-user> -f role=member

# 3. (Opcional) Agregar a un team especializado según su rol
gh api -X PUT orgs/spark-match/teams/<team-slug>/memberships/<github-user> -f role=member
```

El nuevo miembro automáticamente:
- Hereda acceso Write al repo (vía team `members`)
- Aparece como CODEOWNER catch-all (cualquier PR le puede ser asignado)
- Puede hacer `git push -u origin feature/<nombre>`

---

## 6. Flujo de contribución de un miembro típico

```bash
# 1. Clone (solo primera vez)
git clone git@github.com:spark-match/spark-match-00-knowledge-base.git
cd spark-match-00-knowledge-base

# 2. Crear rama
git checkout -b docs/<mi-tema>

# 3. Editar / crear docs
# ...

# 4. Commit + push
git add .
git commit -m "docs(guides): <descripcion>"
git push -u origin docs/<mi-tema>

# 5. Abrir PR
gh pr create --title "docs(guides): <descripcion>" --body "..."

# 6. GitHub auto-asigna reviewers del team `members`
# 7. Otro miembro aprueba (1 review requerido)
# 8. Mergea (cualquiera con write puede, o admin con override)
```

---

## 7. Cómo replicar este patrón a otros repos

Para los repos `01-devops`, `02-infrastructure`, `03-backend`, etc. (cuyo contenido es de un dominio técnico específico), la decisión es:

| Tipo de repo | ¿Team `members` con Write? | Recomendación |
|---|:---:|---|
| `00-knowledge-base` | ✅ Sí | Catch-all (documentación general) |
| `01-devops` (CI/CD) | ⚠️ Opcional | Probablemente solo `@spark-match/devops` |
| `02-infrastructure` (Terraform) | ❌ No | Solo `@spark-match/devops` |
| `03-backend` (código) | ❌ No | Solo `@spark-match/backend-devs` + `@spark-match/ai-devs` |
| `04-frontend` | ❌ No | Solo `@spark-match/frontend-devs` |
| `05-data-pipeline` | ❌ No | Solo `@spark-match/devops` o team data nuevo |
| `06-model-training` | ❌ No | Solo `@spark-match/ai-devs` |
| `07-article` | ❌ No | Solo `@spark-match/article-authors` |
| `.github` (org profile) | ❌ No | Solo `@spark-match/owners` |

**Por qué no Write general para todos**: el código de producción requiere review especializado. Un `frontend-dev` no debería mergear cambios al backend.

---

## 8. Hallazgos residuales (no críticos)

| ID | Hallazgo | Estado |
|---|---|---|
| R-01 | Repo sin `description`, `homepage` ni `topics` | ⏳ Pendiente (cosmético) |
| R-02 | 3 miembros de la org no están en teams especializados | ⏳ Pendiente (depende del rol real de cada uno) |
| R-03 | `default_repo_permission` de la org es `null` | ⏳ Pendiente (cosmético) |
| R-04 | Miembros pueden mergear sus propios PRs (con 1 aprobación de otro) | 🟡 Aceptable (es knowledge base) |
| R-05 | `default_repo_permission` no persiste en GitHub Free plan | 🟡 Limitación del plan — efecto ya logrado por otros medios |

---

## 9. Limitación descubierta: `default_repo_permission` en GitHub Free

### El problema

Al intentar `PATCH /orgs/spark-match` con `default_repository_permission=read`:

```bash
gh api -X PATCH orgs/spark-match -f default_repository_permission=read
# Respuesta: { "default_repository_permission": "read", ... }

gh api orgs/spark-match | jq .default_repo_permission
# Resultado: null  (¡el cambio no persiste!)
```

### Causa

`default_repository_permission` es una configuración **solo disponible en planes de pago** (Team o Enterprise). En el plan **Free**, la API acepta el valor pero no lo persiste.

Confirmación:

```bash
gh api orgs/spark-match --jq '.plan'
# {"name":"free", ...}
```

### ¿El objetivo se logra de todas formas?

✅ **Sí, parcialmente**:

| Caso | ¿Funciona? | Por qué |
|---|:---:|---|
| Miembros actuales ven todos los repos públicos | ✅ | Los 8 repos son `public`, visibles para cualquier org member |
| Miembros actuales tienen Write en 00-knowledge-base | ✅ | Vía team `members` con `permission: push` |
| Futuros miembros ven los repos al unirse | ✅ | Cualquier org member ve los repos `public` automáticamente |
| Futuros miembros tienen Write en 00-knowledge-base | ❌ | Hay que agregarlos manualmente al team `members` |
| Configurar `read` explícito en repos privados | ❌ | No hay repos privados, pero si los hubiera en el futuro, Free no lo soporta |

### Mitigación para el caso real

Agregar nuevos miembros es trivial con 2 comandos:

```bash
# 1. Invitar a la org
gh api -X PUT orgs/spark-match/memberships/<github-user> -f role=member

# 2. Agregar al team `members` (les da Write en 00-knowledge-base)
gh api -X PUT orgs/spark-match/teams/members/memberships/<github-user> -f role=member
```

### Alternativas si en el futuro se requiere `default_repo_permission`

- **Upgrade a GitHub Team** (~$4/user/mes): desbloquea el setting + otras features
- **Mantener la práctica manual**: agregar nuevos miembros al team `members` al invitarlos
- **Usar scripts de bootstrap**: el `CONTRIBUTING.md` puede documentar el flujo

---

## 10. Comandos de referencia rápida

```bash
# Ver todos los miembros de un team
gh api orgs/spark-match/teams/members/members --jq '.[].login'

# Ver teams con acceso a un repo
gh api repos/spark-match/<repo>/teams --jq '.[] | "\(.name) | \(.permission)"'

# Ver permiso efectivo de un usuario en un repo
gh api repos/spark-match/<repo>/collaborators/<user>/permission --jq .permission

# Agregar team a un repo
gh api -X PUT orgs/spark-match/teams/<team-slug>/repos/spark-match/<repo> -f permission=push

# Cambiar permiso de un team en un repo
gh api -X PUT orgs/spark-match/teams/<team-slug>/repos/spark-match/<repo> -f permission=push

# Agregar usuario a un team
gh api -X PUT orgs/spark-match/teams/<team-slug>/memberships/<user> -f role=member
```

---

> _"La mejor gobernanza es la que todos pueden seguir sin pensar en ella."_ — Filosofía Spark Match 🌟
---
title: "Guía de Contribución — Knowledge Base"
author: "@spark-match/devops"
date: 2026-07-09
updated: 2026-07-09
tags:
  - area/conventions
  - status/published
audience:
  - members
status: published
template: contributing
aliases:
  - "CONTRIBUTING"
  - "Cómo contribuir"
---

# 🤝 Guía de Contribución — Spark Match Knowledge Base

> Cómo añadir, editar y mantener documentos en este repositorio.
> Última actualización: 2026-07-09 — ahora incluye sección **🧩 Uso en Obsidian**.

## 📌 Antes de empezar

- Lee el [[README]] para entender la estructura.
- Revisa el [[INDEX]] para ver qué ya existe y evitar duplicados (preferible: usa los MOCs en Obsidian).
- Si tu doc es **cross-team** (afecta a varios equipos), considera abrir primero un **ADR** en [[decisions]] antes de escribir el doc final.
- Schema de frontmatter obligatorio: ver [[admin/frontmatter-schema]].

## 🌿 Flujo de trabajo

### 1. Crear una rama

Usa prefijos descriptivos (Conventional Commits simplificado):

| Tipo | Prefijo | Ejemplo |
|---|---|---|
| Nuevo documento | `docs/` | `docs/how-to-deploy-lambdas` |
| Corrección/edición | `fix/` | `fix/typos-readme` |
| Refactor de docs existentes | `refactor/` | `refactor/restructure-architecture` |
| Mejora de tooling/config | `chore/` | `chore/update-codeowners` |
| Configuración Obsidian | `chore/` | `chore/add-obsidian-config` |

```bash
git checkout main
git pull
git checkout -b docs/<nombre-descriptivo-en-kebab-case>
```

> ⚠️ **IMPORTANTE** según ADR-003 branch-based deployment:
> - Para cambios en este repo: trabajar en `dev`, PR a `dev`, luego merge a `main`
> - Para cambios en producción (código de microservicios): revisar ADR-003 antes

### 2. Ubicar el documento

Coloca tu archivo en la carpeta correcta:

| Si tu doc es... | Carpeta | Plantilla |
|---|---|---|
| Un paso a paso / tutorial / how-to | `guides/` | [[templates/how-to]] |
| Un diseño técnico o diagrama | `architecture/` | — |
| Una decisión cross-team (ADR) | `decisions/` | [[templates/adr]] |
| Una investigación o comparación | `research/` | [[templates/research]] |
| Un análisis de incidente | `postmortems/` | [[templates/postmortem]] |
| Una plantilla reutilizable | `templates/` | — |
| Material de onboarding | `onboarding/` | — |
| Meta-doc (convención/schema) | `.admin/` | — |
| Daily note personal/equipo | `daily/` | [[templates/daily]] |

**¿No encaja?** Abre un issue o pregunta antes de crear una nueva carpeta.

### 3. Usar la plantilla

Todas las plantillas están en `templates/` y tienen frontmatter YAML + placeholders Templater (`{{title}}`, `{{author}}`, etc.):

- ADR → [[templates/adr]]
- How-to → [[templates/how-to]]
- Postmortem → [[templates/postmortem]]
- Research → [[templates/research]]
- MOC (Map of Content) → [[templates/moc]]
- Daily note → [[templates/daily]]
- Meeting note → [[templates/meeting]]

> 💡 **Tip**: si tienes Obsidian + plugin **Templater** instalado, los placeholders se reemplazan automáticamente al crear el archivo desde la plantilla.

### 4. Frontmatter (obligatorio)

Añade metadatos al inicio en YAML según el schema en [[admin/frontmatter-schema]]:

```yaml
---
title: "Cómo desplegar Lambdas con SAM"
author: "@angel-hincho"
date: 2026-07-04
updated: 2026-07-09
tags:
  - area/guides
  - topic/aws/lambda/sam
  - status/published
audience:
  - backend-devs
  - devops
status: published
related:
  - "[[MOC-guides]]"
  - "[[decisions/ADR-001-backend-hibrido]]"
aliases:
  - "SAM deploy"
  - "Lambda deploy"
---
```

### 5. Wikilinks (preferidos sobre Markdown links)

Para enlaces internos a otros docs del vault, **usar wikilinks**:

```markdown
# Sí (preferido)
[[MOC-decisions]]                  → link al MOC de decisiones
[[ADR-001|primer ADR]]             → link con display text
[[admin/obsidian-conventions#Plugins]] → link a una sección específica

# No (legacy)
[decisiones](./decisions/README.md)
```

> 💡 **Ventajas de los wikilinks**:
> - Generan **backlinks** automáticos (qué otros docs referencian este)
> - Activan el **graph view** en Obsidian (visualización de red)
> - No se rompen al renombrar archivos (Obsidian los actualiza)

### 6. Actualizar el catálogo

En Obsidian, los MOCs ([[MOC-decisions]], [[MOC-guides]], etc.) usan queries Dataview que se actualizan automáticamente desde el frontmatter. **No necesitas editar manualmente el MOC**.

Si no usas Obsidian, todavía puedes actualizar el [[INDEX]] manual.

### 7. Abrir el Pull Request

```bash
git add .
git commit -m "docs(guides): add how-to deploy Lambdas with SAM"
git push -u origin docs/deploy-lambdas-sam
gh pr create \
  --title "docs(guides): Cómo desplegar Lambdas con SAM" \
  --body "Añade how-to paso a paso para desplegar Lambdas con SAM. Incluye troubleshooting y links a docs oficiales."
```

> Si abres el PR contra `main`, recuerda que va contra `dev` por ADR-003. Si accidentalmente pusheas a `main`, usa `gh pr edit --base dev`.

### 8. Esperar revisión

- CODE OWNER del área te será asignado automáticamente como reviewer
- Si no tienes respuesta en 2 días, menciona al equipo en el PR
- Tras al menos 1 aprobación (CODE OWNER), podrás mergear a `dev`
- La promoción a `main` es **manual** y requiere aprobación de `@spark-match/product-owners`

## ✅ Checklist antes de pedir review

- [ ] El documento está en la carpeta correcta
- [ ] Tiene **frontmatter completo** (ver [[admin/frontmatter-schema]])
- [ ] Los wikilinks internos funcionan (probados localmente o con `markdown-link-check`)
- [ ] No hay typos evidentes (usar Grammarly o similar)
- [ ] En Obsidian: el doc aparece en su MOC correspondiente
- [ ] He leído el documento desde el punto de vista de un extraño al tema — ¿se entiende?

## 🚫 Qué NO hacer

- ❌ Subir documentos con **información sensible** (claves, tokens, datos personales)
- ❌ Duplicar docs que ya existen — actualízalos en su lugar (usar wikilinks `[[ ]]` para referenciar)
- ❌ Mezclar temas no relacionados en un solo archivo
- ❌ Hacer commits directos a `main` (las protecciones lo impedirán)
- ❌ Subir archivos binarios pesados (PDFs > 5MB, imágenes > 1MB) → usar `attachments/` con wikilinks
- ❌ Plagiar contenido sin citar la fuente
- ❌ Bypassear el CODE OWNER review (la protección de rama lo impedirá)

## 🧩 Uso en Obsidian (opcional, pero recomendado)

> Esta sección describe cómo configurar tu instancia local de Obsidian para colaborar con el vault.

### Setup inicial

1. **Instalar Obsidian**: [obsidian.md/download](https://obsidian.md/download) (Windows, macOS, Linux)
2. **Clonar el repo**: `git clone https://github.com/spark-match/spark-match-00-knowledge-base.git`
3. **Abrir como vault**: Open → "Open folder as vault" → selecciona la carpeta clonada
4. **Confiar en plugins**: cuando Obsidian pregunte, marca "Trust author" (la config está versionada en `.obsidian/`)
5. **Verificar plugins**: Templater, Dataview, Excalidraw, etc. deben aparecer en Settings → Community plugins

### Sincronización

- **Desktop**: usar el plugin **Obsidian Git** (ya instalado) para auto-commit cada 5 minutos
- **Mobile**: ver sección dedicada en [[admin/obsidian-conventions]] (no soportado oficialmente en móvil por limitaciones de Obsidian Git)

### Aliases y renombrado

- Si renombrar archivos desde Obsidian, **usar siempre el comando integrado** (clic derecho → Rename) en lugar de mover manualmente. Obsidian actualiza automáticamente todos los wikilinks.
- Verificar que no quedan "wikilinks huérfanos" con Dataview ([[admin/dataview-queries]] sección "Errores de validación")

## 🔒 Seguridad y secretos

Si descubres un secreto expuesto en un PR:

1. **No lo commitees visiblemente** — usa placeholders como `<SECRET_AQUI>`
2. **Notifica inmediatamente** a `@spark-match/devops`
3. El secreto será rotado y el historial limpiado con `git-filter-repo`

## 🧹 Mantenimiento

### Documentos obsoletos

Si un documento ya no es válido:

1. Cambia el `status` en frontmatter a `archived`
2. Añade el tag `status/archived`
3. Añade una nota al inicio: `> ⚠️ ARCHIVADO: <razón>. Ver <doc-nuevo> en su lugar.`
4. No borres el archivo — puede servir de referencia histórica

### Renombrar o mover

Para renombrar o mover un documento:

1. Crea un PR que haga el cambio
2. Actualiza todos los wikilinks al nuevo nombre (Obsidian lo hace automático)
3. Si renombraste dentro del repo, verifica que el alias esté en `aliases:` para no perder backlinks

### Limpieza periódica

Cada trimestre, `@spark-match/devops` revisará:

- Documentos con `status/draft` por más de 90 días
- Wikilinks huérfanos (queries Dataview en [[admin/dataview-queries]])
- Carpetas vacías o subutilizadas
- Version de plugins comunitarios (compatibilidad)

## 🆘 ¿Necesitas ayuda?

- Abre un **issue** con la etiqueta `question`
- Pregunta en `#spark-match` (canal de Slack/Discord interno)
- Etiqueta a `@spark-match/devops` si es urgente
- O consulta el [[admin/obsidian-conventions|manual de convenciones]]

## 🔗 Wikilinks relacionados

- [[00 - Home]]
- [[README]]
- [[admin/obsidian-conventions]]
- [[admin/frontmatter-schema]]
- [[admin/dataview-queries]]
- [[LICENSE]]

---

> **Recuerda**: si lo aprendiste, alguien más lo necesita aprender también. Documentar es regalar tiempo a tu yo futuro y a tus compañeros. 🌟

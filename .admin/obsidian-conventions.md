---
title: "Conventions — Obsidian vault"
author: "@spark-match/devops"
date: 2026-07-09
updated: 2026-07-09
tags:
  - admin
  - area/conventions
  - status/published
audience:
  - members
status: published
aliases:
  - "obsidian-conventions"
---

# 📐 Convenciones del Vault Obsidian

> Este documento define las convenciones de uso del vault de Spark Match en Obsidian. Es **obligatorio** leerlo antes de contribuir con documentos nuevos.

---

## 📌 Estructura del vault (PARA)

```
vault (00-knowledge-base/)
├── area/...              # Áreas del conocimiento (Knowledge Areas)
│   ├── decisions/        # ADRs
│   ├── architecture/     # Diseños técnicos
│   ├── guides/           # How-tos
│   ├── research/         # Investigaciones
│   ├── postmortems/      # Análisis de incidentes
│   └── onboarding/       # Material de bienvenida
├── resources/            # Recursos externos (futuro)
├── archives/             # Docs deprecados (futuro)
└── daily/                # Daily notes (opcional, opcionalmente privado)
```

> **NOTA**: actualmente seguimos la convención de carpetas planas (no anidadas por área).

## 🏷️ Tags jerárquicos

| Nivel | Formato | Ejemplos |
|---|---|---|
| **Área** (carpeta lógica) | `area/<nombre>` | `area/decisions`, `area/architecture` |
| **Tema técnico** | `topic/<subarea>/<detalle>` | `topic/aws/lambda`, `topic/angular/forms` |
| **Status** | `status/<estado>` | `status/published`, `status/draft`, `status/archived` |
| **Equipo** | `team/<nombre>` | `team/backend-devs`, `team/devops` |
| **Tipo de doc** | `<tipo>` | `moc`, `adr`, `how-to`, `postmortem`, `research` |

### ⚠️ Reglas de uso

- Cada doc debe tener **al menos un tag `area/<X>`** que corresponda a su carpeta
- Cada doc debe tener **al menos un tag `status/<X>`** (default: `status/draft` al crearse)
- Los tags `team/<X>` son para filtrar por audiencia objetivo
- Se pueden agregar tags `topic/<X>` libremente

## 📐 Frontmatter (enforced)

Schema mínimo obligatorio:

```yaml
---
title: ""                # string obligatorio
author: "@github-user"   # string, usuario que escribe
date: YYYY-MM-DD         # fecha de creación
updated: YYYY-MM-DD      # fecha de última edición
tags:                    # list obligatorio
  - area/<X>             # al menos un tag de área
  - status/draft         # status por defecto al crear
audience:                # list de equipos destinatarios
  - backend-devs
status: draft            # string, draft | published | archived
---
```

Schema completo (recomendado):

```yaml
---
title: ""
author: "@github-user"
date: YYYY-MM-DD
updated: YYYY-MM-DD
tags:
  - area/X
  - topic/X/Y
  - status/draft
  - team/role
audience:
  - team1
  - team2
status: draft
template: name-of-template
related:
  - "[[wikilink]]"
  - "[[another-doc]]"
aliases:
  - "Alternative Name"
  - "Another Alias"
---
```

## 🔗 Wikilinks (uso correcto)

| Caso | Sintaxis | Renderiza como |
|---|---|---|
| Link interno | `[[nombre-doc]]` | link al doc `nombre-doc.md` |
| Con display text | `[[nombre-doc\|Display]]` | "Display" como texto |
| Embebido (imagen/pdf) | `![[imagen.png]]` | embebe el archivo |
| A sección | `[[doc#Sección]]` | link directo a la sección |
| Alias | `[[alias-al-doc\|display]]` | link usando alias |

### ⚠️ Reglas

- Preferir **siempre wikilinks** sobre links relativos de Markdown
- Los wikilinks activan el **graph view** y **backlinks**, que son el valor principal de Obsidian
- Para links externos: usar links Markdown `[texto](url)`

## 📊 Dataview (consultas embebidas)

Permite queries SQL-like sobre los documentos con frontmatter. Para activar:

1. Instalar plugin **Dataview** (ya está en `community-plugins.json`)
2. Habilitar en `Settings → Dataview`
3. Usar bloques de código:

````
```dataview
TABLE date, author
FROM "area/decisions"
SORT date DESC
LIMIT 10
```
````

### Queries reutilizables

Ver: [[admin/dataview-queries]]

## 🧰 Plugins activos (compartidos en `community-plugins.json`)

| Plugin | Propósito | Precio |
|---|---|---|
| **Templater** | Reemplazar plantilla con placeholders `{{title}}`, `{{date}}` | Gratis |
| **Dataview** | Queries embebidos sobre frontmatter | Gratis |
| **Excalidraw** | Diagramas nativos en el vault | Gratis |
| **Mind Map** | Vista mental del documento | Gratis |
| **Tasks** | Query de TODOs cross-doc | Gratis |
| **Tag Wrangler** | Renombrar tags masivamente | Gratis |
| **Various Complements** | Autocompletado en editor | Gratis |
| **Obsidian Git** | Auto-commit + sync bidireccional | Gratis |

## 📁 Frontmatter por tipo de doc

| Tipo | Tags obligatorios |
|---|---|
| ADR | `area/decisions` + `status/draft` (o `status/published`) |
| How-to | `area/guides` + `status/draft` |
| Postmortem | `area/postmortems` + `status/draft` |
| Research | `area/research` + `status/in-progress` (o `status/completed`) |
| MOC | `moc` + `area/<X>` + `status/published` |

## 🔄 Workflow de contribución

1. **Crear una rama** en `dev` (siguiendo convención `feat/<nombre>` o `fix/<nombre>`)
2. **Crear el doc** usando la plantilla correspondiente + Templater
3. **Validar** el frontmatter y los wikilinks localmente
4. **Commit + push + PR** a `dev`
5. **CODE OWNERS review** (default: `tech-leads`)
6. **Merge a dev → Promoción a main** (manual, con aprobación de `product-owners`)

## 🔐 Privacidad y datos personales

- ❌ **NO** commitear: `workspace.json`, `graph.json`, `hotkeys.json` (ya están en `.gitignore`)
- ✅ **SÍ** commitear: `app.json`, `appearance.json`, `core-plugins.json`, `community-plugins.json`
- ⚠️ Los daily notes y adjuntos personales: opcionalmente mover a `.gitignore` por usuario

## 📚 Referencias

- [[00 - Home]]
- [[admin/frontmatter-schema]]
- [[admin/dataview-queries]]
- [Obsidian Help — Frontmatter](https://help.obsidian.md/Advanced+topics/YAML+front+matter)
- [Obsidian Help — Linking notes](https://help.obsidian.md/Linking+notes)
- [PARA Method](https://fortelabs.com/blog/para/)

---

> **Mantenedor**: `@spark-match/devops`
> **Reviewers**: `@spark-match/product-owners`
> **Status**: 🟢 published
> **Próxima revisión**: 2026-08-09 (1 mes)

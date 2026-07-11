---
title: "Knowledge Base — Spark Match"
author: "@spark-match/tech-leads"
date: 2026-07-09
updated: 2026-07-09
tags:
  - area/home
  - status/published
audience:
  - members
status: published
template: readme
aliases:
  - "Knowledge Base Spark Match"
  - "Spark Match KB"
---

# 📚 Knowledge Base — Spark Match

> Repositorio compartido de conocimiento para todos los equipos de **Spark Match**.
> Guías, decisiones, investigación, postmortems, plantillas y onboarding en un solo lugar.

> [!info] **Sobre este vault**
> Este repo se puede usar **como vault de Obsidian** además de como repo de docs. Ver [[CONTRIBUTING]] sección "🧩 Uso en Obsidian".

## 🎯 ¿Qué encontrarás aquí?

| Carpeta | Contenido | Audiencia | MOC |
|---|---|---|---|
| 📖 [[guides]] | How-tos, tutoriales paso a paso, procedimientos operativos | Todos | [[MOC-guides]] |
| 🏗️ [[architecture]] | Documentos de diseño, diagramas, decisiones técnicas profundas | Arquitectos, leads | [[MOC-architecture]] |
| ✅ [[decisions]] | Decisiones cross-team (no bounded-context). ADRs organizacionales | Leads, PMs | [[MOC-decisions]] |
| 🔬 [[research]] | Investigaciones, papers, spikes, prototipos, comparativas | Todos | [[MOC-research]] |
| 🚨 [[postmortems]] | Análisis de incidentes y aprendizaje organizacional | DevOps, leads | [[MOC-postmortems]] |
| 📋 [[templates]] | Plantillas reutilizables (ADR, postmortem, how-to, MOC, daily, meeting) | Todos | — |
| 🌱 [[onboarding]] | Material de bienvenida para nuevos miembros | Nuevos miembros | [[MOC-onboarding]] |
| 🔧 [[.admin]] | Convenciones, schema, queries Dataview (meta-docs) | Mantenedores | — |
| 📅 [[daily]] | Daily notes (opcional) | Personal/equipo | — |
| 📎 [[attachments]] | Imágenes y archivos binarios embebidos | Todos | — |

📑 **Catálogo completo**: [[INDEX]] (legacy, usar MOCs en Obsidian).

> 🏠 **Punto de entrada del vault**: [[00 - Home]]

## ✍️ ¿Cómo contribuir?

Este repositorio está abierto a **todos los miembros de Spark Match** para compartir conocimiento.

### Flujo rápido (TL;DR)

1. Crea una rama: `git checkout -b docs/<tu-aporte>`
2. Añade/edita el documento en la carpeta apropiada (usando la [[templates/adr|plantilla]] adecuada)
3. Abre un **Pull Request**
4. CODEOWNERS del área te revisará (default: `@spark-match/tech-leads`)
5. Tras aprobación, mergea 🚀

📘 **Guía detallada**: [[CONTRIBUTING]]

## 🧭 Reglas de oro

> 1. **Documenta lo que aprendiste**, no solo lo que funcionó.
> 2. **Cita fuentes** cuando tomes prestado de internet, papers u otros repos.
> 3. **Usa wikilinks** `[[ ]]` en vez de links Markdown relativos: activa el graph view de Obsidian.
> 4. **Un documento = un tema.** Si tu doc cubre tres cosas, divídelo.
> 5. **Frontmatter obligatorio:** schema en [[admin/frontmatter-schema]].
> 6. **Revisa antes de pedir review.** Ortografía, links, formato.

## 🔍 Búsqueda rápida

### Desde la terminal

```bash
# Buscar por palabra clave en todo el repo
grep -ri "bedrock" --include="*.md" .

# Buscar por tag en front-matter
grep -r "^tags:" --include="*.md" . | grep "aws"
```

### Desde Obsidian (recomendado si tienes instalado)

- `Ctrl + O` → Quick switcher
- `Ctrl + Shift + F` → Búsqueda global
- Graph view (`Ctrl + G`) para visualizar la red de conexiones

### Desde GitHub

[spark-match/spark-match-00-knowledge-base/search](https://github.com/spark-match/spark-match-00-knowledge-base/search).

## 🏛️ Estructura del repo (actualizada)

```
00-knowledge-base/
├── 00 - Home.md                ← MOC raíz, punto de entrada del vault
├── README.md                   ← este archivo
├── CONTRIBUTING.md             ← guía detallada de contribución
├── INDEX.md                    ← catálogo manual (legacy)
├── REVIEW_PERMISSIONS.md       ← auditoría de permisos GitHub
├── LICENSE                     ← MIT
│
├── .obsidian/                  ← config compartida de Obsidian
│   ├── app.json
│   ├── appearance.json
│   ├── core-plugins.json
│   └── community-plugins.json
│
├── .admin/                     ← meta-docs (no son contenido)
│   ├── obsidian-viability-analysis.md
│   ├── obsidian-conventions.md
│   ├── frontmatter-schema.md
│   └── dataview-queries.md
│
├── guides/                     ← how-tos paso a paso (+ MOC-guides.md)
├── architecture/               ← documentos de diseño (+ MOC-architecture.md)
├── decisions/                  ← ADRs cross-team (+ MOC-decisions.md)
├── research/                   ← investigaciones y prototipos (+ MOC-research.md)
├── postmortems/                ← análisis de incidentes (+ MOC-postmortems.md)
├── templates/                  ← plantillas Templater
├── onboarding/                 ← material de bienvenida (+ MOC-onboarding.md)
├── daily/                      ← daily notes (opcional)
├── attachments/                ← imágenes y binarios embebidos
│
└── .github/
    ├── CODEOWNERS              ← ownership por carpeta
    └── pull_request_template.md
```

## 👥 ¿Quién mantiene esto?

- **CODE OWNER por defecto**: `@spark-match/tech-leads`
- **CODE OWNERS por carpeta**: definido en `.github/CODEOWNERS`
- **PRs abiertos** se notifican automáticamente al equipo correspondiente

## 🧩 Uso en Obsidian

Para usar este repo como **vault de Obsidian**:

1. Instala [Obsidian](https://obsidian.md/download) en tu máquina
2. Clona este repo: `git clone https://github.com/spark-match/spark-match-00-knowledge-base.git`
3. Abre Obsidian → "Open folder as vault" → selecciona la carpeta del repo
4. Cuando se te pregunte, **confía en los plugins comunitarios** (Templater, Dataview, etc.)
5. La config del equipo ya está en `.obsidian/`, así que verás los mismos temas y plugins

**Plugins comunitarios recomendados** (ya en `community-plugins.json`):
Templater, Dataview, Excalidraw, Mind Map, Tasks, Tag Wrangler, Obsidian Git.

📘 **Guía completa de Obsidian**: [[admin/obsidian-conventions]]

## 📜 Licencia

MIT — ver [[LICENSE]].

---

> _"Si no está documentado, no existe."_ — Dicho popular del equipo Spark Match 🌟

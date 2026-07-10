---
title: "Análisis de Viabilidad — 00-knowledge-base como Vault de Obsidian"
author: "@ahincho"
date: 2026-07-05
updated: 2026-07-09
tags:
  - area/admin
  - topic/obsidian/viability
  - status/published
audience:
  - devops
  - product-owners
  - tech-leads
status: published
version: "1.1"
related:
  - "[[admin/obsidian-conventions]]"
  - "[[admin/frontmatter-schema]]"
  - "[[admin/dataview-queries]]"
  - "[[00 - Home]]"
---

# 📊 Análisis de Viabilidad — `00-knowledge-base` como Vault de Obsidian

> **Documento de meta-gestión del repositorio**
>
> **Propósito**: evaluar si la estructura actual del repositorio es viable,
> entendible y sostenible para ser usado como un **vault de Obsidian**
> (sistema de gestión de conocimiento personal y colaborativo).
>
> **Audiencia**: `@spark-match/devops` (mantenedor del repo) y `@spark-match/product-owners` (decisor).
>
> **Fecha**: 2026-07-05 (updated 2026-07-09)
> **Versión**: 1.1 (después de implementación Fase 0+1+2)
> **Status**: 🟢 published

---

## 📑 Tabla de contenidos

1. [Resumen ejecutivo](#-resumen-ejecutivo)
2. [Estado actual del repositorio](#-estado-actual-del-repositorio)
3. [Análisis FODA del repo actual](#-análisis-foda-del-repo-actual)
4. [Comparación: Repo GitHub vs Vault Obsidian](#-comparación-repo-github-vs-vault-obsidian)
5. [Problemas detectados](#-problemas-detectados)
6. [Recomendaciones priorizadas](#-recomendaciones-priorizadas)
7. [Estructura de carpetas mejorada](#-estructura-de-carpetas-mejorada)
8. [Schema de frontmatter para Dataview](#-schema-de-frontmatter-para-dataview)
9. [Plugins recomendados](#-plugins-recomendados)
10. [Configuración de `.gitignore`](#-configuración-de-gitignore)
11. [Plan de implementación por fases](#-plan-de-implementación-por-fases)
12. [Riesgos y mitigaciones](#-riesgos-y-mitigaciones)
13. [Próximos pasos](#-próximos-pasos)

---

## 🎯 Resumen ejecutivo

### Veredicto

> ✅ **SÍ es viable** usar `00-knowledge-base` como vault de Obsidian,
> **PERO requiere ~6–8 horas de trabajo de adaptación** distribuido en 2-3
> semanas. La base actual es sólida (estructura, templates, CODEOWNERS,
> convenciones); lo que falta son las capas que hacen de Obsidian un Obsidian:
> wikilinks, frontmatter consistente, plugins configurados y `.gitignore`
> adecuado.

### Métricas actuales vs objetivo

| Métrica | Hoy | Objetivo | Brecha |
|---|:---:|:---:|:---:|
| Documentos totales | 6 | 30+ en 3 meses | 5x |
| Archivos con frontmatter | 2 (33%) | 100% | 67% |
| Wikilinks en el repo | 0 | >50 (alta densidad) | Crítico |
| Plantillas con frontmatter | 0 de 4 | 4 de 4 | 4 |
| Plugins comunitarios configurados | 0 | 4-5 | 4-5 |
| MOCs (Maps of Content) | 0 | 5-7 | 5-7 |
| Tamaño del repo | 255 KB | <2 MB (saludable) | OK |
| `.gitignore` | No existe | Sí (con exclusiones Obsidian) | Crítico |

### Decisión recomendada

**Proceder con la adaptación** siguiendo el plan de implementación por fases
descrito en la sección 11. Empezar por la **Fase 0 (preparación)** y la
**Fase 1 (esenciales)** antes de adoptar el vault activamente.

---

## 📂 Estado actual del repositorio

### Inventario completo

```
00-knowledge-base/
├── .obsidian/                              ← Ya existe (config compartida)
│   ├── app.json                            ← {} (vacío)
│   ├── appearance.json                     ← {} (vacío)
│   ├── core-plugins.json                   ← 30 plugins core activos
│   ├── graph.json                          ← Config personal (debería estar en .gitignore)
│   └── workspace.json                      ← Estado personal (debería estar en .gitignore)
│
├── .github/
│   ├── CODEOWNERS                          ← 85 líneas, refinado por carpeta
│   └── pull_request_template.md
│
├── README.md                               ← 88 líneas, orientación general
├── CONTRIBUTING.md                         ← 173 líneas, guía detallada
├── INDEX.md                                ← 131 líneas, catálogo manual (mayormente vacío)
├── REVIEW_PERMISSIONS.md                   ← Auditoría de permisos GitHub
├── LICENSE                                 ← MIT
│
├── guides/                                 ← VACÍA
├── architecture/                           ← VACÍA
├── decisions/                              ← VACÍA
├── research/                               ← VACÍA
├── postmortems/                            ← VACÍA
│
├── templates/                              ← 4 plantillas (sin frontmatter)
│   ├── adr.md                              ← 49 líneas
│   ├── how-to.md                           ← 73 líneas
│   ├── postmortem.md                       ← 70 líneas
│   └── research.md                         ← 63 líneas
│
└── onboarding/                             ← 2 docs (CON frontmatter)
    ├── welcome.md                          ← 93 líneas, frontmatter completo
    └── dev-setup.md                        ← 250 líneas, frontmatter completo
```

### Estadísticas

| Métrica | Valor |
|---|---|
| Total de archivos `.md` | 11 |
| Archivos con frontmatter YAML | 2 (18% del total) |
| Wikilinks `[[ ]]` en el repo | 0 |
| Tags inline `#tag` (no frontmatter) | 0 (solo headers H1) |
| Carpetas con contenido | 2 de 7 (29%) |
| Líneas totales (`.md`) | ~1,200 |
| Tamaño total | 255 KB |

### Configuración de Obsidian ya presente

- ✅ `.obsidian/` existe
- ✅ `core-plugins.json` con 30 plugins activos:
  - file-explorer, global-search, switcher, graph, backlink, canvas,
    outgoing-link, tag-pane, properties, page-preview, daily-notes,
    templates, note-composer, command-palette, editor-status, bookmarks,
    outline, word-count, file-recovery, sync, bases
- ❌ `community-plugins.json` **no existe** (no hay plugins comunitarios)
- ❌ `app.json` y `appearance.json` están vacíos (sin tema configurado)

---

## 🔍 Análisis FODA del repo actual

### 🟢 Fortalezas

| Fortaleza | Detalle |
|---|---|
| **Estructura de carpetas intuitiva** | Sigue el patrón **PARA** (Projects/Areas/Resources/Archives) con scope claro por categoría |
| **Templates bien diseñados** | 4 plantillas (ADR, how-to, postmortem, research) con secciones bien pensadas y reutilizables |
| **`CONTRIBUTING.md` excelente** | 173 líneas, flujo Git claro, convención de prefijos de rama, qué NO hacer, seguridad |
| **Frontmatter schema definido** | 7 campos estándar: `title, author, date, tags, audience, status, related` |
| **CODEOWNERS refinado** | 85 líneas, paths sensibles gobernados por `product-owners` y `devops` |
| **Convención de kebab-case** | Nombres de archivo consistentes (`dev-setup.md`, `welcome.md`) |
| **Status semántico con emojis** | 🟢 published, 🟡 draft, 🔴 archived — visual y rápido de escanear |
| **CODEOWNERS por equipo** | Cada carpeta tiene un equipo responsable claro |

### 🔴 Debilidades

| Debilidad | Impacto en Obsidian |
|---|---|
| **Cero wikilinks en el repo** | Graph view inútil (nodos aislados), sin backlinks, sin navegación fluida |
| **Solo 2 de 11 archivos con frontmatter** | No se pueden hacer queries Dataview efectivas, Metadatos inconsistentes |
| **`.obsidian/` versionado en Git** | Conflictos en PRs por estado personal, relleno del repo |
| **No hay `.gitignore`** | `.obsidian/workspace.json` y `graph.json` se commitean |
| **Plantillas sin frontmatter** | No se pueden usar con Templater (plugin clave) |
| **Sin MOCs (Maps of Content)** | Sin punto de entrada navegable para el vault |
| **Tags planos, no jerárquicos** | `#aws #lambda` en lugar de `#aws/lambda` (pierde jerarquía navegable) |
| **5 de 7 carpetas vacías** | Estructura sobredimensionada para el contenido actual |

### 🟡 Oportunidades

| Oportunidad | Beneficio |
|---|---|
| **Adoptar Dataview** | Reemplazar `INDEX.md` con queries auto-generadas (menos mantenimiento) |
| **Adoptar Templater** | Crear docs nuevos con un click, placeholders `{{title}}`, `{{date}}` |
| **Adoptar Mind Map** | Vista alternativa del graph, mejor UX para outline |
| **Adoptar Excalidraw** | Diagramas nativos en el vault (mejor que imágenes externas) |
| **MOCs por categoría** | Navegación temática estilo Wikipedia |
| **Tags jerárquicos** | Navegación por árbol de tags (`#aws/lambda/sam`) |
| **Sync con iCloud/Dropbox** | Acceso móvil al vault |
| **Plugin Tasks** | Queries de TODOs cross-documento |
| **Graph interactivo** | Visualizar la red de conocimiento del proyecto |

### ⚫ Amenazas

| Amenaza | Mitigación |
|---|---|
| **Conflictos en PRs por `.obsidian/`** | `.gitignore` con exclusiones por archivo |
| **Frontmatter inconsistente entre archivos** | Schema enforced en `CONTRIBUTING.md` + Dataview validation |
| **Adopción desigual del equipo** | Onboarding específico de Obsidian + documentación |
| **Obsidian queda como repo GitHub "normal"** | Plan de implementación con metas por fase |
| **Overhead de mantener 2 sistemas (Git + Obsidian)** | Una sola fuente de verdad: el repo. Obsidian es solo el cliente |
| **Plugins de Obsidian cambian o se rompen** | Pinning de versiones en `community-plugins.json` |

---

## ⚖️ Comparación: Repo GitHub vs Vault Obsidian

| Aspecto | Repo GitHub (uso actual) | Vault Obsidian (objetivo) |
|---|---|---|
| **Navegación principal** | File tree (sidebar) | Graph view + backlinks + tags |
| **Búsqueda** | `grep` / GitHub search | Full-text + Dataview queries |
| **Links entre docs** | Markdown `[text](./path.md)` | Wikilinks `[[doc]]` + aliases |
| **Metadatos** | Frontmatter YAML (estático) | Frontmatter YAML + **Properties panel** |
| **Visualización** | Markdown renderizado | Markdown + graph + canvas + Excalidraw |
| **Edición** | Editor externo (VSCode, etc.) | Editor nativo + plugins |
| **Versionado** | Git (perfecto) | Git + plugin "Obsidian Git" (auto-commit) |
| **Colaboración** | PRs + reviews (excelente) | Obsidian Sync (de pago) o Git |
| **Mobile** | No nativo | App nativa iOS/Android |
| **Curva de aprendizaje** | Baja (cualquier dev) | Media (saber wikilinks, frontmatter, plugins) |
| **Costo** | Gratis (GitHub Free) | Gratis (personal) / $8/mes (Sync) |

> 💡 **Conclusión**: las dos plataformas se **complementan** perfectamente.
> Git provee versionado, PRs y reviews; Obsidian provee UX, navegación y visualización.
> La clave: que el **repo Git sea la fuente de verdad** y Obsidian sea solo el cliente.

---

## 🚨 Problemas detectados

### 🔴 Críticos (bloquean viabilidad)

| # | Problema | Por qué es crítico | Solución |
|---|---|---|---|
| 1 | **No existe `.gitignore`** | `.obsidian/workspace.json` se commitea, causa conflictos en PRs | Crear `.gitignore` con exclusiones |
| 2 | **Cero wikilinks** | Graph view es inútil (nodos sin conexiones) | Adoptar `[[ ]]` en todos los docs |
| 3 | **Plantillas sin frontmatter** | No funcionan con Templater, no se pueden crear docs nuevos | Convertir a formato Templater |

### 🟠 Importantes (degradan experiencia)

| # | Problema | Impacto | Solución |
|---|---|---|---|
| 4 | **Solo 2 docs con frontmatter** | Dataview queries no funcionan bien, schema inconsistente | Estandarizar frontmatter en todos los docs |
| 5 | **Sin MOCs** | Sin punto de entrada navegable | Crear MOC raíz + MOCs por categoría |
| 6 | **INDEX.md manual** | Propenso a desincronizarse | Reemplazar con query Dataview |
| 7 | **Tags planos** | Pierde jerarquía navegable | Migrar a `#area/subarea/topic` |
| 8 | **5 carpetas vacías** | Estructura sobredimensionada | OK (esperar a que se llenen orgánicamente) |

### 🟢 Menores (nice-to-have)

| # | Problema | Solución |
|---|---|---|
| 9 | Sin tema visual definido en Obsidian | Configurar `appearance.json` con tema compartido |
| 10 | Sin `community-plugins.json` | Crear con plugins recomendados |
| 11 | Sin convenciones documentadas de Obsidian | Agregar sección en `CONTRIBUTING.md` |
| 12 | Mapeo por equipo manual en `INDEX.md` | Reemplazar con tag `#equipo/nombre` + Dataview |

---

## 🎯 Recomendaciones priorizadas

### 🔴 Prioridad ALTA (esenciales para viabilidad)

| # | Acción | Beneficio | Esfuerzo | Owner sugerido |
|---|---|---|:---:|---|
| 1 | Crear `.gitignore` con exclusiones Obsidian | Evita conflictos, limpia el repo | 5 min | `@devops` |
| 2 | Mover config compartida a `.obsidian/` (mantener) y personal a `.gitignore` | Consistencia sin conflictos | 15 min | `@devops` |
| 3 | Convertir las 4 plantillas a formato Templater (con `{{title}}`, `{{date}}`, etc.) | Crea docs nuevos con un click | 30 min | `@devops` |
| 4 | Añadir frontmatter a las 4 plantillas y al README/CONTRIBUTING/INDEX | Schema consistente para Dataview | 20 min | `@devops` |
| 5 | Crear MOC raíz (`00 - Home.md`) y al menos 1 MOC por categoría con contenido | Punto de entrada navegable | 1 h | `@devops` + `@product-owners` |

**Total Prioridad ALTA: ~2-3 horas**

### 🟠 Prioridad MEDIA (mejoran la experiencia significativamente)

| # | Acción | Beneficio | Esfuerzo | Owner sugerido |
|---|---|---|:---:|---|
| 6 | Wikilinks en docs existentes (welcome → dev-setup, templates → INDEX) | Activa graph view y backlinks | 30 min | `@devops` |
| 7 | Tags jerárquicos (`#aws/lambda/sam`) | Navegación por árbol de tags | 1 h | Cualquier autor |
| 8 | Reemplazar `INDEX.md` con query Dataview auto-generada | Catálogo siempre actualizado | 1 h | `@devops` |
| 9 | Documentar convenciones de Obsidian en `CONTRIBUTING.md` | Onboarding claro para nuevos vault users | 1 h | `@devops` |
| 10 | Crear `attachments/` folder + convención de imágenes | Limpia los docs, centraliza binarios | 30 min | `@devops` |

**Total Prioridad MEDIA: ~4-5 horas**

### 🟢 Prioridad BAJA (nice-to-have, cuando haya tracción)

| # | Acción | Beneficio | Esfuerzo |
|---|---|---|:---:|
| 11 | Plugin Excalidraw para diagramas | Visualizaciones nativas | 30 min |
| 12 | Plugin Mind Map | Vista mental del documento | 15 min |
| 13 | Plugin Calendar + carpeta `daily/` | Journaling y tracking | 30 min |
| 14 | Configurar Sync con iCloud/Dropbox | Acceso móvil | 30 min |
| 15 | Plugin Tasks para TODOs cross-doc | Query de pendientes | 20 min |

**Total Prioridad BAJA: ~2 horas**

---

## 🗂️ Estructura de carpetas mejorada (propuesta)

```
00-knowledge-base/
│
├── .obsidian/                              ← Config compartida (SÍ commitear)
│   ├── app.json
│   ├── appearance.json                     ← tema compartido por el equipo
│   ├── core-plugins.json                   ← plugins core activos
│   └── community-plugins.json              ← plugins comunitarios (NUEVO)
│
├── .gitignore                              ← NUEVO: ignora .obsidian/workspace.json, etc.
│
├── README.md                               ← Mantiene (overview para GitHub y vault)
├── CONTRIBUTING.md                         ← Actualiza con sección "Uso en Obsidian"
├── INDEX.md                                ← Mantiene como fallback (Dataview lo reemplaza visualmente)
│
├── 00 - Home.md                            ← NUEVO: MOC raíz del vault
│
├── guides/                                 ← How-tos paso a paso
│   ├── MOC-guides.md                       ← NUEVO: índice Dataview de guías
│   └── (docs futuros)
│
├── architecture/                           ← Diseños técnicos, diagramas
│   ├── MOC-architecture.md                 ← NUEVO
│   └── (docs futuros)
│
├── decisions/                              ← ADRs cross-team
│   ├── MOC-decisions.md                    ← NUEVO
│   └── (ADRs futuros)
│
├── research/                               ← Papers, spikes, comparativas
│   ├── MOC-research.md                     ← NUEVO
│   └── (research futuro)
│
├── postmortems/                            ← Análisis de incidentes
│   ├── MOC-postmortems.md                  ← NUEVO
│   └── (postmortems futuros)
│
├── templates/                              ← Plantillas (Templater format)
│   ├── adr.md                              ← Convertido a Templater
│   ├── how-to.md                           ← Convertido
│   ├── postmortem.md                       ← Convertido
│   └── research.md                         ← Convertido
│
├── onboarding/                             ← Material de bienvenida
│   ├── welcome.md
│   ├── dev-setup.md
│   └── MOC-onboarding.md                   ← NUEVO
│
├── daily/                                  ← NUEVO (opcional): daily notes
│   └── .gitkeep
│
├── attachments/                            ← NUEVO: imágenes y archivos embebidos
│   └── .gitkeep
│
└── .admin/                                 ← NUEVO: meta-docs del vault
    ├── obsidian-viability-analysis.md      ← ESTE DOCUMENTO
    ├── obsidian-conventions.md             ← Convención de uso en vault
    ├── dataview-queries.md                 ← Queries Dataview reutilizables
    └── frontmatter-schema.md               ← Schema enforced de metadatos
```

### Diferencias clave vs estructura actual

| Cambio | Razón |
|---|---|
| `00 - Home.md` en raíz | MOC principal, punto de entrada del vault |
| MOCs en cada categoría | Navegación temática por área |
| `daily/` (opcional) | Para journaling / scratchpad personal |
| `attachments/` | Convención Obsidian estándar para imágenes |
| `.admin/` separado | Meta-documentación no es "contenido" del KB |
| `community-plugins.json` en `.obsidian/` | Lista de plugins recomendados para el equipo |

---

## 📋 Schema de frontmatter para Dataview

### Schema estándar (enforced)

```yaml
---
title: "{{title}}"                    # string, título del documento
author: "@tu-usuario"                 # string, usuario GitHub
date: {{date}}                        # date, fecha de creación (YYYY-MM-DD)
updated: {{date}}                     # date, última actualización
tags:                                 # list, jerárquicos
  - area/<categoría>                  # area/architecture | area/decisions | etc.
  - topic/<subtopic>                  # topic/aws/bedrock | topic/security | etc.
  - status/<state>                    # status/published | status/draft | status/archived
audience:                             # list, equipos destinatarios
  - backend-devs
  - ai-devs
  - devops
status: published                     # string, estado de publicación
related:                              # list de wikilinks
  - "[[MOC-architecture]]"
  - "[[adr-005-serverless]]"
aliases:                              # list, nombres alternativos
  - <alias 1>
  - <alias 2>
cssclasses:                           # list, clases CSS para styling
  - <clase>
---

# {{title}}

> [!info] Metadata
> Autor: @usuario | Fecha: {{date}} | Status: 🌐 published
```

### Campos Dataview-queryable

| Campo | Tipo | Ejemplo | Uso Dataview |
|---|---|---|---|
| `title` | string | "Setup de Lambda con SAM" | `WHERE title` |
| `author` | string | "@angel-hincho" | `GROUP BY author` |
| `date` | date | 2026-07-05 | `SORT date DESC` |
| `updated` | date | 2026-07-10 | Detectar docs desactualizados |
| `tags` | list | `[area/architecture, topic/aws]` | `WHERE contains(tags, "area/architecture")` |
| `audience` | list | `[backend-devs, devops]` | `WHERE contains(audience, "backend-devs")` |
| `status` | string | "published" | `WHERE status = "published"` |
| `related` | list (wikilinks) | `[[adr-005]]` | Backlinks automáticos |
| `aliases` | list | `["SAM", "lambda deploy"]` | Búsqueda alternativa |

### Tags jerárquicos (convención)

| Nivel | Convención | Ejemplo |
|---|---|---|
| **Área** (carpeta) | `area/<nombre>` | `area/architecture`, `area/decisions` |
| **Tema técnico** | `topic/<subarea>/<detalle>` | `topic/aws/bedrock`, `topic/aws/lambda` |
| **Status** | `status/<estado>` | `status/published`, `status/draft` |
| **Equipo** | `team/<nombre>` | `team/backend-devs`, `team/ai-devs` |

### Query Dataview para reemplazar `INDEX.md`

```dataview
TABLE 
  author as "Autor",
  date as "Fecha",
  status as "Status"
FROM ""
WHERE contains(tags, "area/<X>")
SORT date DESC
```

---

## 🧩 Plugins recomendados (`community-plugins.json`)

| Plugin | Para qué | Prioridad | Licencia |
|---|---|:---:|---|
| **Templater** | Reemplazar las plantillas estáticas con placeholders `{{title}}`, `{{date}}`, etc. | 🔴 Alta | Gratis |
| **Dataview** | Reemplazar `INDEX.md` con queries auto-generadas; crear MOCs dinámicos | 🔴 Alta | Gratis |
| **Mind Map** | Vista mental del documento (alternativa a outline) | 🟡 Media | Gratis |
| **Excalidraw** | Diagramas nativos en Obsidian (mejor que Mermaid para bocetos) | 🟡 Media | Gratis |
| **Calendar** | Vista de calendario de daily notes | 🟢 Baja | Gratis |
| **Outliner** | Mejor UX para listas y outlining | 🟢 Baja | Gratis |
| **Tag Wrangler** | Renombrar tags masivamente (útil para la migración a jerárquicos) | 🟢 Baja | Gratis |
| **Tasks** | Query de TODOs cross-documento | 🟢 Baja | Gratis |
| **Obsidian Git** | Auto-commit + sync bidireccional con GitHub | 🟢 Baja | Gratis |

### Configuración de instalación

1. Abrir Obsidian → Settings → Community plugins → Disable Safe Mode
2. Instalar uno por uno (Templater, Dataview, Mind Map, Excalidraw)
3. Habilitar los deseados
4. **Exportar config**: copiar `.obsidian/community-plugins.json` al repo
5. **Versionar**: commitear el JSON para que el equipo use la misma config

---

## 🔒 Configuración de `.gitignore`

### Contenido propuesto

```gitignore
# =============================================================================
# .gitignore — Spark Match Knowledge Base
# =============================================================================

# --- Obsidian: estado personal (NO compartir entre usuarios) ---
# Cada usuario tiene su propio layout de workspace y configuración de graph.
.obsidian/workspace.json
.obsidian/graph.json
.obsidian/appearance.json
.obsidian/plugins/*/data.json
.obsidian/plugins/*/

# Excepciones: la config compartida SÍ debe commitearse
!.obsidian/app.json
!.obsidian/core-plugins.json
!.obsidian/community-plugins.json

# --- Sistema operativo ---
.DS_Store              # macOS
Thumbs.db              # Windows
desktop.ini            # Windows
*$RECYCLE.BIN*         # Windows

# --- Editores ---
.vscode/               # VSCode
.idea/                 # JetBrains
*.swp                  # Vim
*.swo                  # Vim
*~                     # Backup files

# --- Obsidian: trash y caché ---
.trash/
.cache/
```

### Qué se commitea vs qué no

| Path | Commitear | Razón |
|---|:---:|---|
| `.obsidian/app.json` | ✅ | Config de la app |
| `.obsidian/core-plugins.json` | ✅ | Qué plugins core están activos |
| `.obsidian/community-plugins.json` | ✅ | Lista de plugins comunitarios del equipo |
| `.obsidian/appearance.json` | ❌ | Tema personal de cada usuario |
| `.obsidian/workspace.json` | ❌ | Layout personal (paneles abiertos, etc.) |
| `.obsidian/graph.json` | ❌ | Configuración de graph view (filtros, etc.) |
| `.obsidian/plugins/<name>/data.json` | ❌ | Estado interno de cada plugin |

---

## 🗓️ Plan de implementación por fases

### Fase 0 — Preparación (1-2 días)

**Objetivo**: limpiar el repo y crear las bases para Obsidian.

- [ ] Decidir formato de tags (jerárquico: `#area/...` + `#topic/...`)
- [ ] Decidir nombres de los MOCs por categoría
- [ ] Decidir tema de Obsidian para el equipo (recomendado: "Minimal" o "Things")
- [ ] Crear `.gitignore` con el contenido propuesto
- [ ] Primer commit con `.gitignore`

**Entregable**: `.gitignore` funcional, sin archivos personales en el repo.

### Fase 1 — Esenciales (3-5 días)

**Objetivo**: tener un vault Obsidian mínimamente viable.

- [ ] Convertir 4 plantillas a formato Templater con frontmatter
- [ ] Añadir frontmatter a `README.md`, `CONTRIBUTING.md`, `INDEX.md`
- [ ] Crear MOC raíz `00 - Home.md` con enlaces a las categorías
- [ ] Crear 5 MOCs por categoría (uno en cada carpeta vacía)
- [ ] Crear `.obsidian/community-plugins.json` con Templater + Dataview
- [ ] Configurar `appearance.json` con tema compartido

**Entregable**: vault funcional con 6 MOCs + 4 plantillas Templater + 2 plugins.

### Fase 2 — Mejora de experiencia (1-2 semanas)

**Objetivo**: activar las features que hacen la diferencia.

- [ ] Wikilinks en todos los docs existentes (welcome → dev-setup, etc.)
- [ ] Migrar tags planos a jerárquicos en todos los docs
- [ ] Reemplazar `INDEX.md` con query Dataview auto-generada
- [ ] Agregar sección "Uso en Obsidian" en `CONTRIBUTING.md`
- [ ] Crear `.admin/obsidian-conventions.md` con reglas de vault
- [ ] Instalar plugins Mind Map y Excalidraw

**Entregable**: vault rico, navegable, con graph view funcional.

### Fase 3 — Adopción del equipo (ongoing)

**Objetivo**: que el equipo use el vault activamente.

- [ ] Sesión de onboarding (30 min) sobre cómo usar el vault
- [ ] Crear al menos 3 docs reales en categorías vacías
- [ ] Primer ADR real en `decisions/` siguiendo la plantilla Templater
- [ ] Configurar Obsidian Git plugin para auto-commit
- [ ] Crear primer MOC de proyecto (ej: `MOC-spark-match-agent.md`)

**Entregable**: vault adoptado, con contenido orgánico.

### Fase 4 — Optimización (cuando haya tracción)

- [ ] Evaluar plugin Tasks para TODOs cross-doc
- [ ] Configurar Calendar + daily/ si el equipo lo adopta
- [ ] Evaluar Sync vs alternativa para acceso móvil
- [ ] Crear MOCs temáticos transversales (ej: `MOC-aws.md`, `MOC-seguridad.md`)

---

## ⚠️ Riesgos y mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigación |
|---|:---:|:---:|---|
| **Conflictos en PRs por `.obsidian/`** | Alta | Bajo | `.gitignore` antes de empezar |
| **Equipo no adopta Obsidian** | Media | Alto (esfuerzo perdido) | Onboarding + showcase del vault |
| **Wikilinks rotos al renombrar archivos** | Alta | Bajo | Plugin "Various Complements" + CI con `markdown-link-check` |
| **Overhead de mantener 2 sistemas (Git + Obsidian)** | Baja | Medio | Convención clara: Git es la fuente de verdad, Obsidian es solo el cliente |
| **Plugins de Obsidian cambian su API** | Baja | Bajo | Pinning de versiones en `community-plugins.json` |
| **Frontmatter schema drift** | Media | Medio | Dataview validation + convención en `.admin/frontmatter-schema.md` |
| **Vault crece descontroladamente** | Media | Medio | Política de "archive, don't delete" + revisión trimestral |
| **Confusión entre docs de GitHub vs vault** | Baja | Bajo | Documentar el dual-uso en `CONTRIBUTING.md` |

---

## ✅ Próximos pasos

### Esta semana

1. ✅ **Decisión del equipo**: ¿procedemos con la adaptación?
2. ⚪ **Crear `.gitignore`** (Fase 0, 5 min)
3. ⚪ **Revisar y aprobar este análisis** con `@product-owners`

### Próximas 2 semanas

4. ⚪ **Ejecutar Fase 1** (esenciales: 4 plantillas + MOCs + plugins)
5. ⚪ **Sesión de onboarding** del equipo sobre Obsidian
6. ⚪ **Primer ADR real** usando la nueva plantilla Templater

### Próximo mes

7. ⚪ **Ejecutar Fase 2** (wikilinks + tags jerárquicos + Dataview)
8. ⚪ **Crear al menos 5 docs reales** en las categorías vacías
9. ⚪ **Evaluar adopción** y ajustar plan

---

## 📚 Referencias

- [Obsidian Help — Getting Started](https://help.obsidian.md/Getting+started)
- [Obsidian Help — Linking notes](https://help.obsidian.md/Linking+notes)
- [Obsidian Help — YAML front matter](https://help.obsidian.md/Advanced+topics/YAML+front+matter)
- [Templater Plugin](https://github.com/SilentVoid13/Templater)
- [Dataview Plugin](https://github.com/blacksmithgu/obsidian-dataview)
- [PARA Method](https://fortelabs.com/blog/para/) — base de la estructura de carpetas
- [Obsidian Git Plugin](https://github.com/Vinzent03/obsidian-git)
- `00-knowledge-base/CONTRIBUTING.md` — guía de contribución Git
- `00-knowledge-base/README.md` — overview del repositorio

---

> **Mantenedor**: `@spark-match/devops`
> **Reviewers**: `@spark-match/product-owners`
> **Status**: 🟡 draft (espera aprobación para pasar a 🟢)
> **Próxima revisión**: 30 días después de la implementación de Fase 1

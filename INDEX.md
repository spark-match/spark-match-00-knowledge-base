---
title: "Catálogo de Documentos"
author: "@spark-match/devops"
date: 2026-07-09
updated: 2026-07-09
tags:
  - area/conventions
  - status/published
audience:
  - members
status: published
aliases:
  - "INDEX"
  - "Catálogo"
  - "Índice de documentos"
---

# 📑 Índice de Documentos — Knowledge Base

> **Catálogo completo de documentos disponibles.**
>
> ⚠️ **LEGACY / MANTENIDO COMO FALLBACK**: si tienes Obsidian instalado, **usa los MOCs** ([[00 - Home]], [[MOC-decisions]], [[MOC-guides]], etc.) que se auto-actualizan con queries Dataview. Este archivo queda como catálogo estático alternativo.

> **Leyenda de status**:
> - 🟢 `published` — vigente y revisado
> - 🟡 `draft` — en construcción, no tomar como definitivo
> - 🔴 `archived` — obsoleto, conservado por referencia

---

## 🔍 Catálogo auto-generado (Dataview)

### Todos los docs publicados

```dataview
TABLE
  file.link as "Documento",
  author as "Autor",
  date as "Fecha",
  audience as "Audiencia"
FROM ""
WHERE status = "published" AND file.name != this.file.name
SORT tags ASC, date DESC
```

### Docs por área

```dataview
TABLE length(rows) as "Total"
FROM ""
WHERE status = "published" AND file.name != this.file.name
GROUP BY tags
```

### Docs por audiencia

```dataview
TABLE length(rows) as "Total"
FROM ""
WHERE status = "published"
GROUP BY audience
```

---

## 📖 Guías (`guides/`)

How-tos, tutoriales paso a paso, procedimientos operativos.

```dataview
TABLE
  author as "Autor",
  date as "Fecha",
  status as "Status"
FROM "guides"
WHERE contains(tags, "area/guides") AND tags != "moc"
SORT date DESC
```

> 📌 Ver [[MOC-guides]] para una lista visual.

---

## 🏗️ Arquitectura (`architecture/`)

Documentos de diseño, diagramas, decisiones técnicas profundas.

```dataview
TABLE
  author as "Autor",
  date as "Fecha",
  status as "Status"
FROM "architecture"
WHERE contains(tags, "area/architecture") AND tags != "moc"
SORT date DESC
```

> 📌 Ver [[MOC-architecture]].

---

## ✅ Decisiones (`decisions/`)

ADRs (Architecture Decision Records) cross-team. Decisiones que afectan a toda la organización.

```dataview
TABLE
  author as "Autor",
  date as "Fecha",
  status as "Status"
FROM "decisions"
WHERE contains(tags, "area/decisions") AND tags != "moc"
SORT date DESC
```

> 📌 Ver [[MOC-decisions]].

---

## 🔬 Investigación (`research/`)

Investigaciones, papers, spikes, prototipos, comparativas.

```dataview
TABLE
  author as "Autor",
  date as "Fecha",
  status as "Status"
FROM "research"
WHERE contains(tags, "area/research") AND tags != "moc"
SORT date DESC
```

> 📌 Ver [[MOC-research]].

---

## 🚨 Postmortems (`postmortems/`)

Análisis de incidentes y aprendizaje organizacional.

```dataview
TABLE
  severity as "Severidad",
  author as "Autor",
  date as "Fecha"
FROM "postmortems"
WHERE contains(tags, "area/postmortems") AND tags != "moc"
SORT date DESC
```

> 📌 Ver [[MOC-postmortems]].

---

## 📋 Plantillas (`templates/`)

Plantillas reutilizables para crear documentos consistentes (formato Templater + YAML frontmatter).

| Plantilla | Para qué | Wiki link |
|---|---|---|
| [[templates/adr]] | Decisiones arquitectónicas | [[templates/adr]] |
| [[templates/how-to]] | Guías paso a paso | [[templates/how-to]] |
| [[templates/postmortem]] | Análisis de incidentes | [[templates/postmortem]] |
| [[templates/research]] | Investigaciones y comparativas | [[templates/research]] |
| [[templates/moc]] | Crear nuevos MOCs | [[templates/moc]] |
| [[templates/daily]] | Daily notes (opcional) | [[templates/daily]] |
| [[templates/meeting]] | Notas de reuniones | [[templates/meeting]] |

---

## 🌱 Onboarding (`onboarding/`)

Material de bienvenida para nuevos miembros.

```dataview
TABLE
  author as "Autor",
  date as "Fecha"
FROM "onboarding"
WHERE contains(tags, "area/onboarding") AND tags != "moc"
SORT date DESC
```

> 📌 Ver [[MOC-onboarding]].

---

## 🗺️ Mapa por equipo

### Backend Devs

- [[MOC-decisions]] — ADRs de arquitectura
- [[decisions/ADR-001-backend-hibrido-lambda-mas-agente]]
- [[docs/SDD/4_reglas-negocio-agente|Reglas de negocio del agente]]

### Frontend Devs

- [[docs/SDD/4_reglas-negocio-agente|Reglas de negocio del agente]]

### AI Devs

- [[MOC-decisions]] — ADRs
- [[decisions/ADR-001-backend-hibrido-lambda-mas-agente]]
- [[docs/SDD/4_reglas-negocio-agente]]

### Data Engineers

- [[docs/SDD/4_reglas-negocio-agente|Reglas de negocio del agente]] — §7 esquema de `features.csv`

### DevOps

- [[MOC-guides]] — How-tos
- [[onboarding/dev-setup|Setup del entorno de desarrollo]]
- [[admin/obsidian-conventions|Convenciones del vault]]

### QA

- [[docs/SDD/4_reglas-negocio-agente|Reglas de negocio del agente]]

### Product Owners

- [[onboarding/welcome|Bienvenida]]
- [[MOC-decisions]] — ADRs que requieren aprobación

### Article Authors

- _(pendiente añadir docs en [[MOC-research|investigaciones]])_

---

## 📊 Estadísticas del vault

```dataview
TABLE
  length(rows) as "Total"
FROM ""
WHERE status = "published" AND file.name != this.file.name
GROUP BY tags
SORT length(rows) DESC
```

### Por status

```dataview
TABLE length(rows) as "Cantidad"
FROM ""
GROUP BY status
SORT length(rows) DESC
```

### Por autor (top 5)

```dataview
TABLE length(rows) as "Total docs"
FROM ""
WHERE status = "published"
GROUP BY author
SORT length(rows) DESC
LIMIT 5
```

---

## 🔗 Wikilinks relacionados

- [[00 - Home]]
- [[README]]
- [[CONTRIBUTING]]
- [[admin/obsidian-conventions]]
- [[admin/frontmatter-schema]]
- [[admin/dataview-queries]]

---

> _Última actualización: 2026-07-09 — ahora con soporte completo de Dataview_

---
title: "Home — Knowledge Base"
author: "@spark-match/tech-leads"
date: 2026-07-09
updated: 2026-07-09
tags:
  - moc
  - area/home
  - status/published
audience:
  - members
status: published
template: moc
mocType: root
aliases:
  - "MOC-Home"
  - "Spark Match Knowledge Base"
---

# 🏠 Welcome to the Spark Match Knowledge Base

> [!info] **Acerca de este vault**
> Este repositorio es **el centro de conocimiento** del proyecto **Spark Match**, organizado siguiendo el método **PARA** (Projects/Areas/Resources/Archives) + **Zettelkasten** con wikilinks y frontmatter.
>
> **Última actualización**: <% tp.date.now("YYYY-MM-DD", 0, "day", "YYYY-MM-DD HH:mm") %>

---

## 🎯 ¿Qué encontrarás aquí?

Este vault está organizado en **6 categorías temáticas**, cada una con su propio MOC (Map of Content):

| MOC | Propósito |
|---|---|
| [[MOC-decisions]] | 📜 ADRs y decisiones arquitectónicas |
| [[MOC-architecture]] | 🏗️ Diseños técnicos y diagramas |
| [[MOC-guides]] | 📖 How-tos paso a paso para el equipo |
| [[MOC-research]] | 🔬 Investigaciones y comparativas |
| [[MOC-postmortems]] | 🚨 Análisis de incidentes |
| [[MOC-onboarding]] | 👋 Material de bienvenida para nuevos miembros |

## 📚 Categorías adicionales

| Documento | Propósito |
|---|---|
| [[onboarding/welcome]] | Mensaje de bienvenida al equipo |
| [[onboarding/dev-setup]] | Setup del entorno de desarrollo |
| [[REVIEW_PERMISSIONS]] | Auditoría de permisos en GitHub |
| [[INDEX]] | Catálogo alternativo de documentos (legacy) |
| [[CONTRIBUTING]] | Guía para contribuir al vault |
| [[README]] | Overview del repositorio |
| [[LICENSE]] | Licencia MIT |

## 🔧 Plantillas Templater disponibles

- [[templates/adr]] — Para decisiones arquitectónicas
- [[templates/how-to]] — Para guías paso a paso
- [[templates/postmortem]] — Para análisis de incidentes
- [[templates/research]] — Para investigaciones
- [[templates/moc]] — Para nuevos MOCs (como este)
- [[templates/daily]] — Para daily notes
- [[templates/meeting]] — Para notas de reuniones

## 🧩 Administración

- [[admin/obsidian-viability-analysis]] — Análisis de viabilidad del vault
- [[admin/obsidian-conventions]] — Convenciones de uso en vault
- [[admin/frontmatter-schema]] — Schema de metadatos
- [[admin/dataview-queries]] — Queries Dataview reutilizables

## ⚡ Quick links

### Para empezar ahora

```dataview
TABLE
  author as "Autor",
  date as "Fecha",
  status as "Status"
FROM "onboarding"
SORT date DESC
LIMIT 5
```

### ADRs recientes

```dataview
TABLE
  author as "Autor",
  date as "Fecha",
  status as "Status"
FROM "decisions"
WHERE contains(tags, "area/decisions") AND status = "published"
SORT date DESC
LIMIT 10
```

## 🚀 Cómo contribuir

1. Lee [[CONTRIBUTING]]
2. Decidir entre escribir un ADR, un how-to, una investigación, un postmortem, o un MOC
3. Clonar el repo localmente
4. Abrir en Obsidian como vault (carpeta raíz = el repo clonado)
5. Usar la plantilla adecuada con el plugin **Templater**
6. Hacer commit + push + PR (CODE OWNERS por defecto: `tech-leads` para cambios importantes)

## 📊 Stats del vault

> Total de documentos y autores actualizados automáticamente.

```dataview
TABLE length(rows) as "Total docs"
FROM ""
WHERE status = "published"
GROUP BY author
SORT length(rows) DESC
```

---

> **Mantenedor**: `@spark-match/tech-leads`
> **Status**: 🟢 published
> **Próxima revisión**: 2026-08-09 (1 mes)

---
title: "MOC — Decisiones Arquitectónicas"
author: "@spark-match/tech-leads"
date: 2026-07-09
updated: 2026-07-09
tags:
  - moc
  - area/decisions
  - status/published
audience:
  - tech-leads
  - members
status: published
template: moc
mocType: category
aliases:
  - "MOC-decisions"
  - "MOC-ADRs"
---

# 📜 MOC — Decisiones Arquitectónicas (ADRs)

> Un **Architecture Decision Record (ADR)** captura el contexto, opciones y consecuencias de una decisión arquitectónica significativa.

---

## 📊 Resumen

```dataview
TABLE
  author as "Autor",
  date as "Fecha",
  status as "Status"
FROM "decisions"
WHERE contains(tags, "area/decisions") AND tags != "moc"
SORT date DESC
```

## 📈 Distribución por status

```dataview
TABLE length(rows) as "Cantidad"
FROM "decisions"
WHERE contains(tags, "area/decisions") AND tags != "moc"
GROUP BY status
```

## 🎯 ADRs aceptados (status = "published")

### 1. Decisiones de arquitectura

```dataview
LIST
FROM "decisions"
WHERE contains(tags, "area/decisions") AND contains(tags, "topic/architecture")
SORT date DESC
```

### 2. Decisiones de gobernanza

```dataview
LIST
FROM "decisions"
WHERE contains(tags, "area/decisions") AND contains(tags, "topic/governance")
SORT date DESC
```

### 3. Decisiones de proceso

```dataview
LIST
FROM "decisions"
WHERE contains(tags, "area/decisions") AND contains(tags, "topic/process")
SORT date DESC
```

## 🆕 Últimas 5 decisiones

```dataview
TABLE
  author as "Autor",
  date as "Fecha"
FROM "decisions"
WHERE contains(tags, "area/decisions") AND tags != "moc"
SORT date DESC
LIMIT 5
```

## 📌 Pendientes (status = "draft")

```dataview
LIST
FROM "decisions"
WHERE contains(tags, "area/decisions") AND status = "draft"
SORT date DESC
```

## 🔗 Wikilinks relacionados

- [[00 - Home]]
- [[MOC-architecture]]
- [[admin/obsidian-conventions]] — convención de nomenclatura de ADRs

## 📝 Changelog

| Fecha | Cambio | Autor |
|---|---|---|
| 2026-07-09 | Creación inicial del MOC | @tech-leads |

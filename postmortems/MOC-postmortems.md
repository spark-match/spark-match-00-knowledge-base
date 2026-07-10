---
title: "MOC — Postmortems"
author: "@spark-match/devops"
date: 2026-07-09
updated: 2026-07-09
tags:
  - moc
  - area/postmortems
  - status/published
audience:
  - tech-leads
  - devops
  - members
status: published
template: moc
mocType: category
aliases:
  - "MOC-postmortems"
  - "MOC-incidents"
---

# 🚨 MOC — Postmortems

> Análisis retrospectivo de incidentes, su causa raíz y plan de prevención.

---

## 📊 Resumen

```dataview
TABLE
  author as "Autor",
  date as "Fecha",
  severity as "Severidad"
FROM "postmortems"
WHERE contains(tags, "area/postmortems") AND tags != "moc"
SORT date DESC
```

## 🚨 Por severidad

```dataview
TABLE length(rows) as "Cantidad"
FROM "postmortems"
WHERE contains(tags, "area/postmortems") AND tags != "moc"
GROUP BY severity
SORT length(rows) DESC
```

## 📈 Resueltos pendientes

```dataview
TABLE length(rows) as "Cantidad"
FROM "postmortems"
WHERE contains(tags, "area/postmortems") AND tags != "moc"
GROUP BY status
```

## 🆕 Últimos 5 postmortems

```dataview
TABLE
  severity as "Severidad",
  author as "Autor",
  date as "Fecha"
FROM "postmortems"
WHERE contains(tags, "area/postmortems") AND tags != "moc"
SORT date DESC
LIMIT 5
```

## 🛠️ TODOs abiertos de prevención

```dataview
TASK
FROM "postmortems"
WHERE contains(tags, "area/postmortems")
WHERE !completed
```

## 🔗 Wikilinks relacionados

- [[00 - Home]]
- [[MOC-decisions]] — para ADRs derivados de incidentes
- [[REVIEW_PERMISSIONS]] — incidente documentado sobre gobernanza GitHub
- [[admin/obsidian-conventions]] — cómo escribir un postmortem

## 📝 Changelog

| Fecha | Cambio | Autor |
|---|---|---|
| 2026-07-09 | Creación inicial del MOC | @devops |

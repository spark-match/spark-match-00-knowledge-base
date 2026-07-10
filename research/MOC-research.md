---
title: "MOC — Investigaciones"
author: "@spark-match/tech-leads"
date: 2026-07-09
updated: 2026-07-09
tags:
  - moc
  - area/research
  - status/published
audience:
  - tech-leads
  - members
status: published
template: moc
mocType: category
aliases:
  - "MOC-research"
---

# 🔬 MOC — Investigaciones

> Investigaciones, comparativas, spikes y análisis profundo de alternativas.

---

## 📊 Resumen

```dataview
TABLE
  author as "Autor",
  date as "Fecha",
  status as "Status"
FROM "research"
WHERE contains(tags, "area/research") AND tags != "moc"
SORT date DESC
```

## 🎯 Por tema técnico

### Cloud Providers (AWS, GCP, Azure)

```dataview
LIST
FROM "research"
WHERE contains(tags, "area/research") AND contains(tags, "topic/cloud")
SORT date DESC
```

### Lenguajes y Frameworks

```dataview
LIST
FROM "research"
WHERE contains(tags, "area/research") AND contains(tags, "topic/frameworks")
SORT date DESC
```

### Serverless

```dataview
LIST
FROM "research"
WHERE contains(tags, "area/research") AND contains(tags, "topic/serverless")
SORT date DESC
```

### AI / LLM

```dataview
LIST
FROM "research"
WHERE contains(tags, "area/research") AND contains(tags, "topic/ai")
SORT date DESC
```

### Observabilidad

```dataview
LIST
FROM "research"
WHERE contains(tags, "area/research") AND contains(tags, "topic/observability")
SORT date DESC
```

## 📈 Por status

```dataview
TABLE length(rows) as "Cantidad"
FROM "research"
WHERE contains(tags, "area/research") AND tags != "moc"
GROUP BY status
SORT length(rows) DESC
```

## 🔗 Wikilinks relacionados

- [[00 - Home]]
- [[MOC-decisions]] — para los ADRs derivados de investigaciones
- [[admin/obsidian-conventions]] — cómo escribir una investigación

## 📝 Changelog

| Fecha | Cambio | Autor |
|---|---|---|
| 2026-07-09 | Creación inicial del MOC | @tech-leads |

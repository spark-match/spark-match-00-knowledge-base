---
title: "MOC — Guías y How-tos"
author: "@spark-match/devops"
date: 2026-07-09
updated: 2026-07-09
tags:
  - moc
  - area/guides
  - status/published
audience:
  - members
status: published
template: moc
mocType: category
aliases:
  - "MOC-guides"
  - "MOC-how-to"
  - "MOC-guías"
---

# 📖 MOC — Guías y How-tos

> Paso a paso para tareas comunes: setup, despliegue, troubleshooting, etc.

---

## 📊 Resumen

```dataview
TABLE
  author as "Autor",
  date as "Fecha",
  estimatedTime as "Tiempo"
FROM "guides"
WHERE contains(tags, "area/guides") AND tags != "moc"
SORT date DESC
```

## 🚀 Onboarding

```dataview
LIST
FROM "guides"
WHERE contains(tags, "area/guides") AND contains(tags, "topic/onboarding")
SORT date DESC
```

## 🔧 Setup y Configuración

```dataview
LIST
FROM "guides"
WHERE contains(tags, "area/guides") AND contains(tags, "topic/setup")
SORT date DESC
```

## 🚢 Despliegue (CI/CD)

```dataview
LIST
FROM "guides"
WHERE contains(tags, "area/guides") AND contains(tags, "topic/deployment")
SORT date DESC
```

## 🐛 Troubleshooting

```dataview
LIST
FROM "guides"
WHERE contains(tags, "area/guides") AND contains(tags, "topic/troubleshooting")
SORT date DESC
```

## 🧪 Testing y QA

```dataview
LIST
FROM "guides"
WHERE contains(tags, "area/guides") AND contains(tags, "topic/testing")
SORT date DESC
```

## 🔗 Wikilinks relacionados

- [[00 - Home]]
- [[MOC-onboarding]]
- [[onboarding/dev-setup]]
- [[admin/obsidian-conventions]] — cómo escribir un how-to

## 📝 Changelog

| Fecha | Cambio | Autor |
|---|---|---|
| 2026-07-09 | Creación inicial del MOC | @devops |

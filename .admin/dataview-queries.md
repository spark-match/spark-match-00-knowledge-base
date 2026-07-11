---
title: "Dataview Queries — Catálogo"
author: "@spark-match/devops"
date: 2026-07-09
updated: 2026-07-09
tags:
  - admin
  - area/conventions
  - topic/queries
  - status/published
audience:
  - members
status: published
aliases:
  - "dataview-queries"
---

# 🔍 Catálogo de Queries Dataview Reutilizables

> Queries listas para usar en MOCs y docs que necesiten listar otros documentos del vault.

## 📌 Cómo usar

En cualquier doc de Markdown, agregar un bloque de código con tipo `dataview`:

````
```dataview
TABLE author, date
FROM "decisions"
SORT date DESC
```
````

**Output esperado**: una tabla generada en tiempo real cuando Obsidian renderice el doc.

---

## 📜 ADRs

### Listar todos los ADRs (default MOC)

```dataview
TABLE
  author as "Autor",
  date as "Fecha",
  status as "Status"
FROM "decisions"
WHERE contains(tags, "area/decisions") AND tags != "moc"
SORT date DESC
```

### ADRs pendientes (draft)

```dataview
LIST
FROM "decisions"
WHERE contains(tags, "area/decisions") AND status = "draft"
SORT date DESC
```

### ADRs por autor

```dataview
TABLE length(rows) as "Cantidad"
FROM "decisions"
WHERE contains(tags, "area/decisions") AND tags != "moc"
GROUP BY author
SORT length(rows) DESC
```

### ADRs por tag de tema

```dataview
LIST
FROM "decisions"
WHERE contains(tags, "area/decisions") AND contains(tags, "topic/<X>")
SORT date DESC
```

## 🔬 Investigaciones

### Investigaciones en curso

```dataview
LIST
FROM "research"
WHERE contains(tags, "area/research") AND status = "in-progress"
SORT date DESC
```

### Investigaciones completadas

```dataview
TABLE
  author as "Autor",
  date as "Fecha"
FROM "research"
WHERE contains(tags, "area/research") AND status = "completed"
SORT date DESC
```

## 📖 Guías

### How-tos más recientes

```dataview
TABLE
  author as "Autor",
  estimatedTime as "Tiempo (min)",
  date as "Fecha"
FROM "guides"
WHERE contains(tags, "area/guides") AND tags != "moc"
SORT date DESC
LIMIT 5
```

### How-tos por audiencia

```dataview
LIST
FROM "guides"
WHERE contains(tags, "area/guides") AND contains(audience, "<equipo>")
SORT date DESC
```

## 🚨 Postmortems

### Críticos (severity = "Crítica")

```dataview
TABLE
  author as "Autor",
  date as "Fecha"
FROM "postmortems"
WHERE contains(tags, "area/postmortems") AND severity = "Crítica"
SORT date DESC
```

### TODOs abiertos de prevención

```dataview
TASK
FROM "postmortems"
WHERE contains(tags, "area/postmortems")
WHERE !completed
```

## 📊 Visión general del vault

### Docs por status

```dataview
TABLE length(rows) as "Cantidad"
FROM ""
WHERE status
GROUP BY status
SORT length(rows) DESC
```

### Docs por autor

```dataview
TABLE length(rows) as "Total docs"
FROM ""
WHERE status = "published"
GROUP BY author
SORT length(rows) DESC
```

### Docs por equipo (audiencia)

```dataview
TABLE length(rows) as "Docs publicados"
FROM ""
WHERE status = "published" AND audience
FLATTEN audience
GROUP BY audience as "Equipo"
SORT length(rows) DESC
```

### Docs recientes (global)

```dataview
LIST
FROM ""
WHERE date AND status = "published"
SORT date DESC
LIMIT 10
```

## 🏷️ Por tags

### Docs por tema técnico (topic/*)

```dataview
TABLE length(rows) as "Cantidad"
FROM ""
WHERE contains(tags, "topic/<X>") AND tags != "moc"
GROUP BY file.link
SORT length(rows) DESC
```

### Docs por equipo (team/*)

```dataview
TABLE
  audience as "Audiencia",
  date as "Fecha"
FROM ""
WHERE contains(tags, "team/<X>") AND status = "published"
SORT date DESC
```

## 🔗 Wikilinks salientes

### Top docs por cantidad de wikilinks salientes

```dataview
TABLE
  outgoingLinks as "Wikilinks salientes",
  file.link as "Doc"
FROM ""
WHERE outgoingLinks AND status = "published"
SORT length(outgoingLinks) DESC
LIMIT 10
```

### Top docs con más backlinks

```dataview
TABLE
  file.link as "Doc",
  backlinks as "Backlinks"
FROM ""
WHERE backlinks
SORT length(backlinks) DESC
LIMIT 10
```

## 🐛 Errores de validación

### Docs sin tag de área

```dataview
LIST
FROM ""
WHERE !tags OR !any(filter(tags, (t) => startswith(t, "area/")))
```

### Docs sin status

```dataview
LIST
FROM ""
WHERE !status
```

## 🔗 Wikilinks relacionados

- [[00 - Home]]
- [[admin/obsidian-conventions]]
- [[admin/frontmatter-schema]]

## 📚 Referencias

- [Dataview documentation oficial](https://blacksmithgu.github.io/obsidian-dataview/)
- [Dataview snippets](https://github.com/mgmeyers/obsidian-snippets)

---

> **Mantenedor**: `@spark-match/devops`
> **Status**: 🟢 published
> **Próxima revisión**: 2026-08-09 (1 mes)

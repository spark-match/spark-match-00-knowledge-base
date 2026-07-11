---
title: "MOC: {{title}}"
author: "{{author}}"
date: <% tp.date.now("YYYY-MM-DD") %>
updated: <% tp.date.now("YYYY-MM-DD") %>
tags:
  - moc
  - area/<area>
  - status/published
audience: []
status: published
template: moc
mocType: <category>
related: []
aliases:
  - "MOC-{{title}}"
  - "{{title}} Map of Content"
---

# 📍 {{title}} — Map of Content

> [!info] **Acerca de este MOC**
> Un **Map of Content (MOC)** es una página índice temática que agrupa enlaces a otros documentos del vault relacionados. Sigue la metodología de [[Niklas Luhmann - Zettelkasten]] aplicada al [[PARA Method]].
>
> **Audiencia**: todos los del equipo
> **Última actualización**: <% tp.date.now("YYYY-MM-DD HH:mm") %>

---

## 🎯 Propósito de este MOC

<!-- 1-2 oraciones: para qué existe este MOC y quién debería consultarlo. -->

## 📑 Tabla de contenido

- [Sección 1](#-sección-1)
- [Sección 2](#-sección-2)
- [Sección 3](#-sección-3)

---

## 📚 Sección 1

```dataview
TABLE
  author as "Autor",
  date as "Fecha",
  status as "Status"
FROM ""
WHERE contains(tags, "area/<area>") AND tags != "moc"
SORT date DESC
LIMIT 10
```

## 📚 Sección 2

<!-- Agregar manualmente wikilinks a docs específicos -->

- [[nombre-doc-1]]
- [[nombre-doc-2]]

## 📚 Sección 3

- Pendiente...

---

## 🔗 Wikilinks relacionados

- [[00 - Home]]
- [[MOC-<otra-categoría>]]

## 📝 Changelog del MOC

| Fecha | Cambio | Autor |
|---|---|---|
| <% tp.date.now("YYYY-MM-DD") %> | Creación inicial del MOC | @<% tp.user.username() %> |

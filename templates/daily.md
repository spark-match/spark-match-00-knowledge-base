---
title: "Daily: <% tp.date.now("YYYY-MM-DD") %>"
author: "{{author}}"
date: <% tp.date.now("YYYY-MM-DD") %>
tags:
  - area/daily
  - status/published
audience: []
status: published
template: daily-note
weather: ""
mood: ""
---

# 📅 Daily: <% tp.date.now("YYYY-MM-DD", 0, "day", "dddd, MMMM D") %>

> [!info] **Hoy es**: <% tp.date.now("dddd, D [de] MMMM [de] YYYY", "es-PE") %>

---

## 🎯 Principales objetivos del día

- [ ] Objetivo 1
- [ ] Objetivo 2
- [ ] Objetivo 3

## 📝 Notas de la mañana

<!-- Reuniones, sincronización con el equipo, alineamientos. -->

## 🔧 Trabajo en curso

<!-- Logs de avance, blockers, decisiones tomadas en el día. -->

### Mañana

- ...

### Tarde

- ...

## 💡 Aprendido / Lecciones

- ...

## 🐛 Blockers

- ...

## ✅ Resumen al cierre del día

- **Completado**: ...
- **Pendiente para mañana**: ...
- **Necesito ayuda con**: ...

## 🔗 Wikilinks del día

- [[<% tp.date.now("YYYY-MM-DD", -1, "day", "YYYY-MM-DD") %>]] ← ayer
- [[<% tp.date.now("YYYY-MM-DD", 1, "day", "YYYY-MM-DD") %>]] → mañana

## 👀 Highlights

<!-- Lo más relevante del día, se puede usar para generar weekly summary -->

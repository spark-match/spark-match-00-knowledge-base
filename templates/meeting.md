---
title: "Reunión: {{title}}"
author: "{{author}}"
date: <% tp.date.now("YYYY-MM-DD") %>
updated: <% tp.date.now("YYYY-MM-DD") %>
tags:
  - area/meetings
  - status/published
audience:
  - tech-leads
status: published
template: meeting-note
meetingDate: <% tp.date.now("YYYY-MM-DD HH:mm") %>
meetingType: ""  # sync | planning | retrospective | review | brainstorming
participants: []
duration: ""
related: []
---

# 🗣️ Reunión: {{title}}

> [!info] **Metadata**
> **Fecha y hora**: {{meetingDate}}
> **Tipo**: {{meetingType}}
> **Duración**: {{duration}}
> **Participantes**: {{participants}}
> **Líder / Facilitador**: @<% tp.user.username() %>

---

## 🎯 Objetivo de la reunión

<!-- 1-2 oraciones: por qué nos reunimos, qué decisión hay que tomar. -->

## 📋 Agenda

1. Punto 1
2. Punto 2
3. Punto 3

## 💬 Notas por punto

### Punto 1: <Título>

**Discusión**: ...

**Decisión**: ...

### Punto 2: <Título>

**Discusión**: ...

**Decisión**: ...

### Punto 3: <Título>

**Discusión**: ...

**Decisión**: ...

## ✅ Action items (TODOs)

| # | Tarea | Owner | Fecha objetivo |
|---|---|---|---|
| 1 | ... | @usuario | YYYY-MM-DD |
| 2 | ... | @usuario | YYYY-MM-DD |
| 3 | ... | @usuario | YYYY-MM-DD |

## 🔗 Wikilinks relacionados

- [[00 - Home]]
- [[MOC-team-coordination]]

## 📎 Anexos

- [Slides usadas](#)
- [Documento de referencia](#)
- [Video de la reunión](#)

---
title: "Daily Notes"
author: "@spark-match/devops"
date: 2026-07-09
updated: 2026-07-09
tags:
  - area/conventions
  - topic/conventions/daily-notes
  - status/published
audience:
  - members
status: published
related:
  - "[[templates/daily]]"
  - "[[admin/obsidian-conventions]]"
---

# 📅 Daily Notes

> Este directorio contiene los **daily notes** del equipo. Cada día un archivo `YYYY-MM-DD.md`.

## 📌 Convención

- **Nombre**: `YYYY-MM-DD.md` (formato ISO 8601)
- **Ubicación**: este directorio
- **Plantilla**: [[templates/daily]]
- **Frecuencia**: 1 por día (opcional pero recomendado)

## ⚠️ Notas importantes

- Los daily notes son contenido **personal** por naturaleza. Considera anonimizar o limitar a reuniones técnicas.
- Opcionalmente, puedes sincronizar con un plugin "Daily Note Mover" para mantenerlas privadas (no trackeadas por Git).
- Para uso estrictamente personal, ajusta `.gitignore` y comenta este directorio.
- Para evitar ruido en PRs, una alternativa es crear un subdirectorio `daily/.personal/` en .gitignore.

## 🔗 Wikilinks relacionados

- [[00 - Home]]
- [[admin/obsidian-conventions]]
- [[templates/daily]]

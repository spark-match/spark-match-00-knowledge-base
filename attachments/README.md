---
title: "Adjuntos (attachments)"
author: "@spark-match/devops"
date: 2026-07-09
updated: 2026-07-09
tags:
  - area/conventions
  - topic/conventions/attachments
  - status/published
audience:
  - members
status: published
related:
  - "[[admin/obsidian-conventions]]"
  - "[[README]]"
---

# 📎 Adjuntos (attachments)

> Este directorio contiene las imágenes, PDFs y archivos binarios embebidos en los documentos del vault.

## 📌 Convención

- **Por defecto**: configurable en `.obsidian/app.json` (`attachmentFolderPath: "attachments/"`)
- **Convención al nombrar**: `YYYY-MM-DD-descripcion-corta.png`
- **Tamaño**: mantener cada archivo < 2 MB para no inflar el repo
- **Formatos comunes**: PNG, JPG, SVG, PDF, XLSX

## ⚠️ Notas importantes

- Cuando uses una imagen en un MD: `![[2026-07-09-arquitectura-aws.png]]` (con doble `!` para embebido)
- Las imágenes también se pueden referenciar con wikilink simple: `[[2026-07-09-arquitectura-aws.png]]`
- Si el vault crece mucho (>10 MB de adjuntos), considerar hosting externo (S3 + CloudFront) y referenciar por URL

## 🔗 Wikilinks relacionados

- [[00 - Home]]
- [[admin/obsidian-conventions]]

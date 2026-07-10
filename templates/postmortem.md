---
title: "Postmortem: {{title}}"
author: "{{author}}"
date: <% tp.date.now("YYYY-MM-DD") %>
updated: <% tp.date.now("YYYY-MM-DD") %>
tags:
  - area/postmortems
  - status/draft
audience:
  - tech-leads
  - devops
status: draft
template: postmortem
severity: "Media"
incidentDate: ""
duration: ""
detectedBy: ""
resolvedBy: ""
related: []
aliases: []
---

# 🚨 Postmortem: {{title}}

> [!warning] **Metadata del incidente**
> **Severidad**: 🔴 Crítica | 🟠 Alta | 🟡 Media | 🟢 Baja
> **Fecha del incidente**: {{incidentDate}}
> **Duración**: {{duration}}
> **Detectado por**: {{detectedBy}}
> **Resuelto por**: {{resolvedBy}}
> **Status**: 🟡 Borrador | 🟢 Publicado

---

## 📋 Resumen ejecutivo

<!-- 2-3 oraciones: qué pasó, impacto, cómo se resolvió. -->

## ⏱️ Línea de tiempo

| Hora (UTC) | Evento |
|---|---|
| HH:MM | Primer síntoma detectado |
| HH:MM | Alerta disparada |
| HH:MM | Equipo notificado |
| HH:MM | Mitigación aplicada |
| HH:MM | Servicio restaurado |

## 🔍 Causa raíz

<!-- Explica la causa técnica raíz. Usa "5 Whys" si ayuda. -->

**5 Whys**:

1. ¿Por qué...? Porque...
2. ¿Por qué...? Porque...

## 💥 Impacto

- **Usuarios afectados**: N usuarios / X% del tráfico
- **Duración**: X minutos de indisponibilidad
- **Downstream effects**: ...

## ✅ Acciones de mitigación inmediatas

- [x] Acción 1
- [x] Acción 2

## 🛠️ Plan de prevención (TODOs)

| Acción | Owner | Fecha objetivo |
|---|---|---|
| ... | @usuario | YYYY-MM-DD |
| ... | @usuario | YYYY-MM-DD |

## 📚 Lecciones aprendidas

### ✅ Qué hicimos bien

- ...

### ❌ Qué podemos mejorar

- ...

### 🤯 Qué nos sorprendió

- ...

## 🔗 Wikilinks relacionados

- [[00 - Home]]
- [[MOC-postmortems]]
- [[MOC-incidents]]

## 📎 Anexos

- [Logs relevantes](#)
- [Dashboard de Grafana](#)
- [Captura de alerta](#)

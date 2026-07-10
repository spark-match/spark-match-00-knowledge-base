---
title: "<% tp.user.prefix_title('ADR') %>: {{title}}"
author: "{{author}}"
date: <% tp.date.now("YYYY-MM-DD") %>
updated: <% tp.date.now("YYYY-MM-DD") %>
tags:
  - area/decisions
  - status/draft
audience:
  - tech-leads
status: draft
template: adr
adrNumber: ""
related: []
aliases: []
---

# 🟣 {{title}}

> [!info] **Architecture Decision Record**
> Captura el contexto, opciones y consecuencias de una decisión arquitectónica significativa.

> [!quote] **Metadata del ADR**
> **Estado**: 🟡 Propuesto | 🟢 Aceptado | 🟠 Deprecado | ⚫ Superseded by ADR-NNN
> **Fecha**: <% tp.date.now("YYYY-MM-DD") %>
> **Autores**: @<% tp.user.username() %>
> **Reviewers**: @usuario2 (opcional)

---

## 🎯 Contexto

<!-- ¿Qué problema estamos resolviendo? ¿Qué constraints (tiempo, presupuesto, equipo) tenemos? -->

## 🔄 Opciones consideradas

| Opción | Pros | Contras |
|---|---|---|
| **A. ...** | ... | ... |
| **B. ...** | ... | ... |
| **C. ...** | ... | ... |

## ✅ Decisión

<!-- ¿Qué elegimos? 1-2 oraciones claras. -->

## 🎬 Consecuencias

### Positivas

- ...

### Negativas

- ...

### Mitigaciones

- ...

## 🔗 Wikilinks relacionados

<!-- Si esta decisión se vincula con otros docs del vault, listarlos aquí como wikilinks para activar backlinks -->
- [[00 - Home]]
- [[MOC-decisions]]

## 📚 Referencias

- [Link 1](https://...)
- [Link 2](https://...)

## 💬 Discusión

<!-- Resumen del debate que llevó a esta decisión (opcional). -->

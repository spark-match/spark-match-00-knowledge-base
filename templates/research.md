---
title: "Investigación: {{title}}"
author: "{{author}}"
date: <% tp.date.now("YYYY-MM-DD") %>
updated: <% tp.date.now("YYYY-MM-DD") %>
tags:
  - area/research
  - status/in-progress
audience:
  - tech-leads
status: in-progress
template: research
investigator: "{{author}}"
startDate: ""
endDate: ""
hypothesis: ""
related: []
aliases: []
---

# 🔬 Investigación: {{title}}

> [!info] **Metadata de la investigación**
> **Status**: 🟡 En curso | 🟢 Completa | ⚫ Abandonada
> **Fecha inicio**: <% tp.file.creation_date("YYYY-MM-DD") %>
> **Fecha fin**: (si aplica)
> **Investigadores**: @<% tp.user.username() %>

---

## 🎯 Pregunta / Hipótesis

<!-- ¿Qué queremos resolver o descubrir? -->

**Hipótesis**: Creemos que <X> es mejor que <Y> para <caso de uso>.

## 🎯 Criterios de éxito

<!-- ¿Cómo sabremos si la hipótesis es cierta? -->

- [ ] Métrica 1: ...
- [ ] Métrica 2: ...

## 🔄 Alternativas evaluadas

| Alternativa | Pros | Contras |
|---|---|---|
| **A. ...** | ... | ... |
| **B. ...** | ... | ... |
| **C. ...** | ... | ... |

## 🧪 Metodología

<!-- ¿Cómo probamos cada alternativa? ¿Qué datos usamos? -->

## 📊 Resultados

<!-- Tablas, gráficos, métricas comparativas. -->

### Métricas cuantitativas

| Métrica | Alternativa A | Alternativa B | Diferencia |
|---|---|---|---|
| Latencia p50 | X ms | Y ms | +Z% |
| Latencia p99 | X ms | Y ms | +Z% |
| Coste mensual | $X | $Y | -Z% |

### Observaciones cualitativas

- ...

## ✅ Conclusiones

<!-- Recomendación basada en los resultados. -->

## 🔗 Wikilinks relacionados

- [[00 - Home]]
- [[MOC-research]]

## 📚 Referencias

- [Documentación oficial de X](https://...)
- [Benchmark de la industria](https://...)
- [Artículos relacionados](#)

## 📎 Anexos

<!-- Datos crudos, scripts, notebooks, etc. -->

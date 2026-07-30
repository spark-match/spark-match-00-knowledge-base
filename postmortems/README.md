# 🚨 Postmortems

> Análisis de incidentes y aprendizaje organizacional.

## ¿Qué va aquí?

Documentos con la forma **"qué pasó, por qué, qué aprendimos, qué
cambios aplicamos"**:

- Postmortem del outage del 2026-07-12 (DB connection pool exhausted)
- Análisis del PR mergeado sin reviewer (ver [`../decisions/ADR-002-incident-pr-sin-reviewers.md`](../decisions/ADR-002-incident-pr-sin-reviewers.md))

## Plantilla

Usa [`../templates/postmortem.md`](../templates/postmortem.md).
Estructura mínima:

- **Resumen ejecutivo** (1 párrafo)
- **Timeline** (qué pasó cuándo)
- **Root cause** (5 Whys)
- **Impacto** (usuarios afectados, SLO breach, etc.)
- **Action items** (cambios concretos con owner + due date)

## Estado

Esta carpeta está **vacía por diseño** — se creó en PR-#79 (carpeta
stub) y se llenará conforme ocurran incidentes dignos de
documentar.

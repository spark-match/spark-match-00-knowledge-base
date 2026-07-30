# 🏗️ Architecture

> Documentos de diseño profundo, diagramas de sistema, decisiones
> técnicas que afectan a múltiples componentes pero que NO son
> cross-team (esas van en `decisions/`).

## ¿Qué va aquí?

Documentos con la forma **"cómo funciona X a nivel de sistema"**:

- Diagramas C4 del backend
- Topología de red
- Estrategias de observabilidad
- Patrones de error handling
- Diseños de seguridad (threat models)

**No confundir con `decisions/`**: una decisión cross-team (ej.
"usar Serverless en vez de ECS") va en `decisions/`. La
**descripción** de cómo Serverless se implementa técnicamente va
aquí.

## Convención

Usa el front-matter estándar (ver [`../CONTRIBUTING.md`](../CONTRIBUTING.md)):

```markdown
---
title: Topología de red de Spark Match
author: @tu-usuario
date: YYYY-MM-DD
tags: [network, vpc, architecture]
audience: [backend-devs, devops]
status: published
---
```

## Estado

Esta carpeta está **vacía por diseño** — se creó en PR-#79 (carpeta
stub) y se llenará conforme se migren documentos desde `docs/SDD/`
o se creen diseños nuevos.

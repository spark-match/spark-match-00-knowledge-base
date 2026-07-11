---
title: "MOC — Arquitectura"
author: "@spark-match/tech-leads"
date: 2026-07-09
updated: 2026-07-09
tags:
  - moc
  - area/architecture
  - status/published
audience:
  - tech-leads
  - backend-devs
  - frontend-devs
  - ai-devs
  - devops
status: published
template: moc
mocType: category
aliases:
  - "MOC-architecture"
  - "MOC-arquitectura"
---

# 🏗️ MOC — Arquitectura

> Diseños técnicos, diagramas y especificaciones de alto nivel del sistema Spark Match.

---

## 📊 Resumen

```dataview
TABLE
  author as "Autor",
  date as "Fecha",
  status as "Status"
FROM "architecture"
WHERE contains(tags, "area/architecture") AND tags != "moc"
SORT date DESC
```

## 🎯 Por componente

### Backend (microservicios)

```dataview
LIST
FROM "architecture"
WHERE contains(tags, "area/architecture") AND contains(tags, "topic/backend")
SORT date DESC
```

### Frontend (Angular)

```dataview
LIST
FROM "architecture"
WHERE contains(tags, "area/architecture") AND contains(tags, "topic/frontend")
SORT date DESC
```

### AI / Deep Agent

```dataview
LIST
FROM "architecture"
WHERE contains(tags, "area/architecture") AND contains(tags, "topic/ai")
SORT date DESC
```

### Infraestructura (Terraform)

```dataview
LIST
FROM "architecture"
WHERE contains(tags, "area/architecture") AND contains(tags, "topic/infrastructure")
SORT date DESC
```

### Agentes / EDA

```dataview
LIST
FROM "architecture"
WHERE contains(tags, "area/architecture") AND contains(tags, "topic/agents")
SORT date DESC
```

## 📐 Stack tecnológico (PFC)

```dataview
LIST
FROM "architecture"
WHERE contains(tags, "topic/aws") OR contains(tags, "topic/serverless")
SORT date DESC
```

## 🔗 Wikilinks relacionados

- [[00 - Home]]
- [[MOC-decisions]]
- [[MOC-research]]
- [[01-backend/ARCHITECTURE]] — arquitectura del backend
- [[07-deep-agent/DEPLOYMENT]] — arquitectura del agente IA

## 📝 Changelog

| Fecha | Cambio | Autor |
|---|---|---|
| 2026-07-09 | Creación inicial del MOC | @tech-leads |

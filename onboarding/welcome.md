---
title: "Bienvenida a Spark Match"
author: "@spark-match/product-owners"
date: 2026-07-04
updated: 2026-07-09
tags:
  - area/onboarding
  - topic/onboarding/welcome
  - status/published
audience:
  - members
status: published
aliases:
  - "Welcome"
  - "Bienvenida"
---

# 🌟 ¡Bienvenido/a a Spark Match!

> Tu primera parada como nuevo miembro del equipo.

## 👋 ¿Qué es Spark Match?

Spark Match es un proyecto académico del **Trabajo de Fin de Programa** de la UNI (Universidad Nacional de Ingeniería) — II Programa de Especialización en IA Generativa y Machine Learning Ops.

Construimos un **copiloto de orientación vocacional** basado en IA Generativa que ayuda a estudiantes a descubrir carreras afines a su perfil.

## 🗺️ ¿Dónde está cada cosa?

### Repositorios principales (organización `@spark-match`)

| Repo | Para qué |
|---|---|
| [spark-match/.github](https://github.com/spark-match/.github) | Perfil de la organización + info general |
| [spark-match/00-knowledge-base](https://github.com/spark-match/spark-match-00-knowledge-base) | **Este vault** — base de conocimiento compartida |
| [spark-match/01-devops](https://github.com/spark-match/spark-match-01-devops) | Workflows reutilizables (CI/CD) |
| [spark-match/02-infrastructure](https://github.com/spark-match/spark-match-02-infrastructure) | Terraform de AWS |
| [spark-match/03-backend](https://github.com/spark-match/spark-match-03-backend) | Backend serverless (Lambda + EventBridge) |
| [spark-match/04-frontend](https://github.com/spark-match/spark-match-04-frontend) | SPA en Angular |
| [spark-match/05-data-pipeline](https://github.com/spark-match/spark-match-05-data-pipeline) | ETL de datos vocacionales |
| [spark-match/06-model-training](https://github.com/spark-match/spark-match-06-model-training) | Entrenamiento y serving de modelos |
| [spark-match/07-article](https://github.com/spark-match/spark-match-07-article) | Artículo académico LaTeX |
| [spark-match/08-deep-agent](https://github.com/spark-match/spark-match-08-deep-agent) | Agente IA de counselling (LangChain) |

### Equipos (CODE OWNERS)

| Equipo | Responsabilidad |
|---|---|
| `owners` | Propietarios de la organización |
| `product-owners` | Gestión de prioridades y aprobaciones |
| `tech-leads` | Líderes técnicos de cada dominio |
| `devops` | Infraestructura AWS y Terraform |
| `backend-devs` | API REST en FastAPI/Lambda |
| `frontend-devs` | SPA en Angular |
| `ai-devs` | IA/ML: LangChain, prompts, embeddings, agente |
| `qa` | Aseguramiento de calidad y testing |
| `article-authors` | Autores del artículo académico |

## 🚀 Tus primeros 3 días

### Día 1: Conoce el proyecto

- [ ] Lee el [[README]] de este vault
- [ ] Lee [[CONTRIBUTING]] para saber cómo contribuir
- [ ] Explora el [[00 - Home]] y los MOCs principales:
  - [[MOC-decisions]] — decisiones arquitectónicas
  - [[MOC-architecture]] — diseños técnicos
  - [[MOC-guides]] — how-tos paso a paso
  - [[MOC-research]] — investigaciones
- [ ] Si te toca código, lee la arquitectura de tu equipo

### Día 2: Setup técnico

- [ ] Sigue [[onboarding/dev-setup|dev-setup]] para preparar tu entorno
- [ ] (Opcional) Instala [Obsidian](https://obsidian.md) y abre este repo como vault
- [ ] Activa los plugins comunitarios del equipo (Templater, Dataview, Excalidraw)
- [ ] Haz un PR de prueba (corrige un typo, por ejemplo)

### Día 3: Tu primera tarea

- [ ] Pide a tu lead una tarea `good first issue` o `good first doc`
- [ ] Abre tu primer PR
- [ ] Celebra 🎉

## 💬 Cultura del equipo

- **Documenta lo que aprendes** — Si te took 2 horas descubrir algo, documéntalo para el siguiente ([[admin/obsidian-conventions|convenciones de docs]]).
- **Pregunta antes de asumir** — Mejor una pregunta "tonta" que un PR roto.
- **Revisa PRs con empatía** — Critica el código, no a la persona.
- **Serverless first** — Para nuevos servicios, Lambda + EventBridge por defecto.
- **Security first** — [[.github/CODEOWNERS|CODEOWNERS]] + branch protection están por algo. No los saltées.

## 🆘 ¿Necesitas ayuda?

- **Slack/Discord**: canal `#spark-match`
- **Issues en GitHub**: con etiqueta `question`
- **Tu lead directo**: primer punto de contacto
- **`@spark-match/devops`**: para temas de infraestructura o repo
- **`@spark-match/tech-leads`**: para temas técnicos de fondo

## 🎁 Recursos extra

- [[CONTRIBUTING]] — cómo contribuir a este vault
- [[INDEX]] — catálogo completo de docs (legacy, prefiere los MOCs)
- [[templates]] — plantillas para crear docs
- [CONTRIBUTING en GitHub](https://github.com/spark-match/spark-match-00-knowledge-base/blob/main/CONTRIBUTING.md)

---

> _"Solo se aprende aquello que se comparte."_ — Filosofía de Spark Match 🌟

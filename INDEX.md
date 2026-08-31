# 📑 Índice de Documentos — Knowledge Base

> Catálogo completo de documentos disponibles.
> **Mantén este archivo actualizado** al añadir/eliminar docs.

> **Leyenda de status**:
> - 🟢 `published` — vigente y revisado
> - 🟡 `draft` — en construcción, no tomar como definitivo
> - 🔴 `archived` — obsoleto, conservado por referencia

---

## 📖 Guías (`guides/`)

How-tos, tutoriales paso a paso, procedimientos operativos.

| Documento | Autor | Fecha | Audiencia | Status |
|---|---|---|---|---|
| _(vacío — añade el primero)_ | — | — | — | — |

---

## 🏗️ Arquitectura (`architecture/`)

Documentos de diseño, diagramas, decisiones técnicas profundas.

| Documento | Autor | Fecha | Audiencia | Status |
|---|---|---|---|---|
| _(vacío — añade el primero)_ | — | — | — | — |

---

## ✅ Decisiones (`decisions/`)

ADRs (Architecture Decision Records) cross-team. Decisiones que afectan a toda la organización.

| Documento | Autor | Fecha | Audiencia | Status |
|---|---|---|---|---|
| [ADR-001: Backend híbrido (Lambda + servidor agente)](./decisions/ADR-001-backend-hibrido-lambda-mas-agente.md) | @Angel, @Fabiola | 2026-07-05 | Backend, AI Devs | 🔴 ARCHIVADO 2026-07-29 |
| [ADR-002: Incident, PRs merged without CODE OWNER review](./decisions/ADR-002-incident-pr-sin-reviewers.md) | @Angel | 2026-07-05 | DevOps, todos | 🟢 |

---

## 🔬 Investigación (`research/`)

Investigaciones, papers, spikes, prototipos, comparativas.

| Documento | Autor | Fecha | Audiencia | Status |
|---|---|---|---|---|
| _(vacío — añade el primero)_ | — | — | — | — |

---

## 🚨 Postmortems (`postmortems/`)

Análisis de incidentes y aprendizaje organizacional.

| Documento | Autor | Fecha | Audiencia | Status |
|---|---|---|---|---|
| _(vacío — añade el primero)_ | — | — | — | — |

---

## 📋 Plantillas (`templates/`)

Plantillas reutilizables para crear documentos consistentes.

| Documento | Autor | Fecha | Audiencia | Status |
|---|---|---|---|---|
| [Plantilla de ADR](./templates/adr.md) | @spark-match/devops | 2026-07-04 | Todos | 🟢 |
| [Plantilla de Postmortem](./templates/postmortem.md) | @spark-match/devops | 2026-07-04 | DevOps, leads | 🟢 |
| [Plantilla de Investigación](./templates/research.md) | @spark-match/devops | 2026-07-04 | Todos | 🟢 |
| [Plantilla de How-to](./templates/how-to.md) | @spark-match/devops | 2026-07-04 | Todos | 🟢 |

---

## 🌱 Onboarding (`onboarding/`)

Material de bienvenida para nuevos miembros.

| Documento | Autor | Fecha | Audiencia | Status |
|---|---|---|---|---|
| [Bienvenida a Spark Match](./onboarding/welcome.md) | @spark-match/product-owners | 2026-07-04 | Nuevos miembros | 🟢 |
| [Setup del entorno de desarrollo](./onboarding/dev-setup.md) | @spark-match/devops | 2026-07-04 | Nuevos devs | 🟢 |

---

## 📘 SDD — Solution Design Documents (`docs/SDD/`)

Documentos de producto/diseño del MVP. NO son governance, son
**contrato de producto** (PRD, requirements, design, arquitectura,
reglas de negocio). Audiencia amplia: producto + engineering.

| Documento | Autor | Fecha | Audiencia | Status |
|---|---|---|---|---|
| [1. PRD actualizado v1.1](./docs/SDD/1_PRD.md) | @spark-match/product-owners | 2026-06 | Todos | 🟡 draft |
| [2. Requirements](./docs/SDD/2_requirements.md) | @spark-match/product-owners | 2026-06 | Todos | 🟡 draft |
| [3. Design](./docs/SDD/3_design.md) | @spark-match/product-owners | 2026-06 | Todos | 🟡 draft |
| [4. Reglas de negocio del agente y scoring](./docs/SDD/4_reglas-negocio-agente.md) | @spark-match/product-owners | 2026-07-05 | AI Devs, Backend, Data | 🟡 draft |
| [Arquitectura de referencia](./docs/SDD/ARCHITECTURE.md) | @spark-match/product-owners | 2026-06 | Todos | 🟡 draft |

---

## 🗺️ Mapa por equipo

### Backend Devs
- [ADR-001: Backend híbrido (Lambda + servidor agente)](./decisions/ADR-001-backend-hibrido-lambda-mas-agente.md)
- [Reglas de negocio del agente y scoring](./docs/SDD/4_reglas-negocio-agente.md)

### Frontend Devs
- [Reglas de negocio del agente y scoring](./docs/SDD/4_reglas-negocio-agente.md)

### AI Devs
- [Reglas de negocio del agente y scoring](./docs/SDD/4_reglas-negocio-agente.md)
- [ADR-001: Backend híbrido (Lambda + servidor agente)](./decisions/ADR-001-backend-hibrido-lambda-mas-agente.md)

### Data Engineers
- [Reglas de negocio del agente y scoring](./docs/SDD/4_reglas-negocio-agente.md) — §7 esquema de `features.csv`

### DevOps
- [Plantilla de ADR](./templates/adr.md)
- [Plantilla de Postmortem](./templates/postmortem.md)
- [Plantilla de Investigación](./templates/research.md)
- [Plantilla de How-to](./templates/how-to.md)
- [Setup del entorno de desarrollo](./onboarding/dev-setup.md)

### QA
- [Reglas de negocio del agente y scoring](./docs/SDD/4_reglas-negocio-agente.md)

### Product Owners
- [Bienvenida a Spark Match](./onboarding/welcome.md)

### Article Authors
- _(ningún doc por ahora)_

---

## 🔍 Búsqueda por tag

_(se actualizará conforme se añadan docs con front-matter)_

## 📊 Estadísticas

- **Documentos totales**: 13 (4 plantillas + 2 onboarding + 2 ADRs + 5 SDD)
- **Documentos por status (governance/templates/onboarding)**:
  - 🟢 published: 7 (4 plantillas + 2 onboarding + ADR-002)
  - 🟡 draft: 0
  - 🔴 archived: 1 (ADR-001)
- **SDD docs** (sin status explícito en front-matter, tratados como 🟡 draft por estar en revisión): 5
  - `4_reglas-negocio-agente.md` es el único con status explícito (`draft`)

> _Última actualización: 2026-07-29 (PR-#79: carpetas stub, ADR-001 archivado, ADR-002 añadido al index)_
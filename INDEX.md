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
| [ADR-001: Backend híbrido (Lambda + servidor agente)](./decisions/ADR-001-backend-hibrido-lambda-mas-agente.md) | @Angel, @Fabiola | 2026-07-05 | Backend, AI Devs | 🟡 |

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

- **Documentos totales**: 8 (4 plantillas + 2 onboarding + 1 ADR + 1 SDD reglas)
- **Documentos por status**:
  - 🟢 published: 6
  - 🟡 draft: 2
  - 🔴 archived: 0

> _Última actualización: 2026-07-05 (ADR-001 + reglas de negocio del agente)_
# 🟣 ADR-001: Backend híbrido — Lambda (CRUD/EDA) + servidor Python dedicado (agente)

> **Architecture Decision Record** — captura el contexto, opciones y consecuencias de una decisión arquitectónica significativa.

> **Estado**: 🔴 ARCHIVADO — *Deprecated 2026-07-29 (PR-#79 en `spark-match-00-knowledge-base`)*
> **Fecha original**: 2026-07-05
> **Autores**: @Angel, @Fabiola (registro)
> **Reviewers**: @David (pendiente)
>
> ⚠️ **Esta decisión fue revertida**. El 2026-07-28,
> `spark-match-03-backend` PR #62
> ([`chore(repo): remove all Python references from this Node.js/TS-only repo`](https://github.com/spark-match/spark-match-03-backend/pull/62))
> removió **todas** las referencias a Python del backend.
>
> **Estado actual**:
> - El backend `spark-match-03-backend` es 100% TypeScript / Node.js.
> - El agente cognitivo vive en un repo separado:
>   `spark-match-07-deep-agent` (Python + FastAPI + LangGraph + Bedrock).
> - El "híbrido" se materializa a nivel de **repos**, no de un solo
>   proceso (Lambda CRUD en un repo, FastAPI agent en otro).
>
> **Por qué se conserva este archivo**: contexto histórico de por
> qué se llegó a la decisión de "Lambda para CRUD + Python para
> agente" y por qué luego se separó en dos repos.

---

## 🎯 Contexto

El reparto original pedía FastAPI + Docker, pero el repo `03-backend` documenta AWS Lambda
serverless, y el POC `08-deep-agent` corre como servidor FastAPI + LangGraph con **streaming**.
Hay que cerrar una sola dirección porque **bloquea todo el avance de backend**.

Constraints:

- **Créditos AWS limitados** — la preocupación explícita es no gastar en infraestructura ociosa.
- **Agilidad** — se necesita levantar y probar rápido, sin mantener un servidor 24/7.
- **Streaming** — la parte de IA (Deep Agent) necesita respuesta en stream (SSE / AG-UI).
- Un servidor tradicional siempre-encendido (Spring Boot / Quarkus / Nest) implica costo continuo
  (~US$20 / 2 semanas solo por tener API Gateway + microservicios arriba).

## 🔄 Opciones consideradas

| Opción | Pros | Contras |
|---|---|---|
| **A. Todo Lambda** | Costo ~0 en baja carga, cero ops | Streaming en Lambda es incómodo; el agente sufre |
| **B. Todo servidor tradicional (Nest/Quarkus/Spring)** | Sin cold start, streaming natural | Servidor 24/7 = costo continuo, menos ágil para el equipo |
| **C. Híbrido: Lambda (CRUD/EDA) + servidor Python dedicado (agente)** | Cada carga en su sitio; Lambda barato para CRUD, servidor solo para IA/streaming | Dos entornos de despliegue que coordinar |

## ✅ Decisión

**Opción C — Backend híbrido:**

- **AWS Lambda pura** para operaciones CRUD y eventos (EDA) del lado usuario: login/registro,
  listado de universidades e institutos, catálogo, georreferencia. Con un **mini-framework interno**
  que estandarice el manejo de errores y el formato de respuesta (evitar Lambda "cruda").
- **Servidor Python dedicado** para el agente cognitivo (Deep Agent): razonamiento, memoria y
  **streaming** (SSE / AG-UI). Se levanta solo para la parte de IA.

## 🎬 Consecuencias

### Positivas

- Costo controlado: el CRUD no paga servidor ocioso; el servidor de IA se dimensiona aparte.
- El streaming del agente funciona sin forzar Lambda a algo para lo que no está pensada.
- Separación limpia por responsabilidad (CRUD/EDA vs cognición).

### Negativas

- Dos toolchains / dos despliegues (Lambda + servidor).
- Contratos entre ambos mundos deben quedar claros (eventos / API).

### Mitigaciones

- Mini-framework Lambda compartido para errores y respuestas homogéneas.
- El flujo del agente se define en `docs/SDD/4_reglas-negocio-agente.md` (valida qué expone cada lado).

## 📚 Referencias

- `03-backend/DECISIONS.md` (ADR-001 serverless, pgvector, EDA)
- `08-deep-agent/README.md` (POC FastAPI + LangGraph + Bedrock)
- `docs/SDD/4_reglas-negocio-agente.md`

## 💬 Discusión

Pendiente de cerrar en la misma reunión: **LLM definitivo (Bedrock vs Gemini)**, que impacta
directamente el consumo de créditos. Ángel valida la decisión final tras revisar el flujo con
Andy (frontend) y Nikolai (datos).

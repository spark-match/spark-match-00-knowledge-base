# Plan de Trabajo — CareerMatch Perú (Matching Context + Frontend + Integración)

## 0. Supuestos y alcance

- **Duración**: 21 días (3 semanas) de trabajo + 7 días de colchón (semana 4) = 28 días calendario.
- **Equipo**: 5 personas, repartidas en 3 roles mínimos:   
**Backend**,  
**Frontend**,   
**Integración**.  

  Se asume una distribución de referencia (ajustable):  
  2 Backend,  
  2 Frontend,  
  1 Integración.  
  
  El rol de Integración puede apoyarse puntualmente en Backend/Frontend para resolver incompatibilidades.

- **Fuente de las tareas de Backend**: se derivan y consolidan directamente de `tasks.md`
  (Matching Context + Data Pipeline, 29 tareas de alto nivel) y `design.md`. Cada tarea de este
  plan referencia entre paréntesis el/los ID(s) original(es) de `tasks.md` para trazabilidad
  (ej. `[tasks.md 6.x]`).

- **Frontend e Integración**: `tasks.md`/`design.md` **no** detallan tareas de Frontend (solo
  mencionan un `frontend/app.js` de referencia) ni un plan de integración explícito — esas tareas
  se han diseñado aquí desde cero, en base a los endpoints y eventos reales descritos en `design.md`
  §2 (Identity, Assessment, Matching, AI Advisor). **Deben validarse con el equipo.**

- **Decisiones de producto aún pendientes** (heredadas de `design.md`, marcadas con ❄️/⚠️ ahí) se
  listan en la sección 7 como **riesgos**, porque pueden bloquear tareas de Frontend/Integración a
  mitad de sprint si no se resuelven en la Semana 1.

---

## 1. Leyenda

**Roles**: 

`BE` = Backend · `FE` = Frontend · `INT` = Integración

**Estado**: 

`Pendiente` · `En progreso` · `Bloqueada` · `Completada`

**Prioridad implícita**: las tareas de la Semana 1 son bloqueantes para casi todo lo demás — no
tiene sentido paralelizar Frontend "de verdad" (conectado a datos reales) hasta que exista al menos
un endpoint de Matching funcionando con datos fixture.

---

## 2. Vista de fases

| Fase | Días | Objetivo |
|---|---|---|
| **F0 — Planificación** | 1–2 | Alcance MVP, setup de entornos, wireframes |
| **F1 — Fundación de datos + scoring** | 2–7 | Pipeline de datos completo + `ScoringEngine` funcionando sobre fixtures |
| **F2 — Construcción paralela (Backend eventos/API + Frontend UI)** | 6–14 | Endpoints/eventos de Matching + pantallas de Frontend conectadas a mocks |
| **F3 — Integración y pruebas** | 11–19 | Conexión real Frontend↔Backend↔eventos, pruebas end-to-end, corrección de incompatibilidades |
| **F4 — Despliegue** | 18–21 | Despliegue a ambiente demo, smoke tests, documentación final |
| **F5 — Colchón** | 22–28 | Bugs críticos, ensayo de demo, reserva para imprevistos |

---

## 3. Semana 1 (Días 1–7) — Planificación + Fundación de datos

| ID | Tarea | Descripción | Responsable | Depende de | Estado | Observaciones |
|---|---|---|---|---|---|---|
| P-01 | Kickoff y alcance MVP | Confirmar qué se demuestra al final de las 3 semanas (recorte de alcance dado el tamaño real de la arquitectura Spark Match, que excede lo razonable para 5 personas/3 semanas). | INT | — | Pendiente | Ver riesgo R1 (sección 7): decidir aquí Assessment estructurado vs. conversacional — bloquea F-04, F-05, B-06. |
| P-02 | Setup monorepo y entornos | Estructura `contexts/matching/`, `.env.example`, `uv init`, dependencias. `[tasks.md 1.1–1.5]` | BE | P-01 | Pendiente | — |
| P-03 | Setup proyecto Frontend | Scaffold (framework a elección del equipo), estructura de carpetas, cliente HTTP base. | FE | P-01 | Pendiente | — |
| P-04 | `Dockerfile` + `docker-compose.yml` (dev local) | Backend + Postgres local para desarrollo. `[tasks.md 1.6–1.7]` | BE | P-02 | Pendiente | Solo desarrollo local del Matching Context, no representa el backend completo. |
| P-05 | Wireframes de pantallas clave | Login/registro, cuestionario RIASEC/pesos/filtros, resultados Top-5, feedback Likert. | FE | P-01 | Pendiente | Depende de la decisión R1 para saber si el "cuestionario" es un formulario o un chat. |
| B-01 | `DataLoader` (carga desde snapshot/raw.xlsx) | `[tasks.md 2.1–2.3]` | BE | P-02 | Pendiente | Selenium (`ingestion.py`) es opcional, no bloqueante. |
| B-02 | `FeatureEngineer` (imputación + normalización) | `[tasks.md 3.1–3.3]` | BE | B-01 | Pendiente | Verificar que `income_norm/cost_norm/admission_norm/duration_norm` quedan en `[0,1]`, todas "mayor = mejor". |
| B-03 | Definir y crear `career.career_offerings` en Aurora | Schema del aggregate carrera×universidad. `[design.md §CareerOffering]` | BE | P-02 | Pendiente | Bloqueante para B-04 y para todo Matching. |
| B-04 | Etiquetado RIASEC (Bedrock, 554 carreras únicas) + join | `[tasks.md 4.1–4.4]` | BE | B-02 | Pendiente | Tagging sobre carreras únicas, no sobre las 6,208 filas. |
| B-05 | Escritura batch a `career.career_offerings` | `[tasks.md 25.1]` | BE | B-03, B-04 | Pendiente | Reemplaza el snapshot completo en cada corrida batch. |
| I-01 | Checkpoint Fase 1 — pipeline validado | Ejecutar suite de tests del pipeline sobre fixture reducido (50 carreras). `[tasks.md 7.1]` | INT | B-05 | Pendiente | Primer punto de control real del sprint. |

---

## 4. Semana 2 (Días 8–14) — Construcción paralela

### Backend: scoring, eventos y endpoints

| ID | Tarea | Descripción | Responsable | Depende de | Estado | Observaciones |
|---|---|---|---|---|---|---|
| B-06 | `calculate_riasec_code` + `calculate_affinity` | `[tasks.md 6.1–6.3]` | BE | I-01 | Pendiente | Afinidad: divisor `/6` (no `/18`), ya corregido en design.md. |
| B-07 | `calculate_score` + `rank_and_filter` | `[tasks.md 6.4–6.7]` | BE | B-06 | Pendiente | Sin inversiones adicionales — las 4 normalizadas ya son "mayor = mejor". |
| B-08 | Definir eventos `AssessmentCompleted` / `RecommendationGenerated` | Contratos de payload. `[tasks.md 10.2, 10.4]` | BE | B-07 | Pendiente | **Depende de R1** (sección 7) si el payload lo genera un formulario o un chat. |
| B-09 | `matching_handler` (Lambda reactivo a `AssessmentCompleted`) | `[tasks.md 10.3]` | BE | B-08 | Pendiente | — |
| B-10 | `feedback_handler` (Lambda reactivo a `FeedbackSubmitted`) | `[tasks.md 10.5]` | BE | B-03 | Pendiente | No es endpoint REST — solo evento. |
| B-11 | Endpoints locales de desarrollo (`GET /v1/match/recommendations`, `GET /v1/match/{careerId}/affinity`) | `[tasks.md 11.1–11.5]` | BE | B-07 | Pendiente | Para que Frontend pueda integrarse antes de tener EventBridge real. |
| B-12 | Integrar validación JWT (consumir Identity Context) | `[tasks.md 16.x–18.x]` | BE | P-02 | Pendiente | Matching **no emite** JWT, solo valida contra el secreto compartido de Identity. |
| I-02 | Checkpoint Fase 2 — Matching local con curl/Postman | `[tasks.md 12.x]` | INT | B-11 | Pendiente | Publicar colección de Postman/contratos para que Frontend integre. |

### Frontend: pantallas conectadas a mocks/endpoints locales

| ID | Tarea | Descripción | Responsable | Depende de | Estado | Observaciones |
|---|---|---|---|---|---|---|
| F-01 | Pantallas login/registro | Consumir endpoints de Identity Context (`/v1/auth/register`, `/v1/auth/login`). | FE | P-03, P-05 | Pendiente | Si Identity Context de otro equipo no está listo, mockear respuesta JWT. |
| F-02 | Manejo de sesión (guardar/enviar JWT) | Interceptor HTTP que agrega `Authorization: Bearer`. | FE | F-01 | Pendiente | — |
| F-03 | Pantalla de captura RIASEC/pesos/filtros | Formulario o chat según se resuelva R1. | FE | P-05 | Bloqueada | **Bloqueada por R1** — no iniciar la UI real hasta la decisión de producto; mientras tanto, avanzar solo el layout estático. |
| F-04 | Pantalla de resultados (Top-5) | Consume `GET /v1/match/recommendations`, muestra score, desglose por criterio, datos verificables. | FE | I-02, F-03 | Pendiente | Usar el mock/endpoint local de B-11 mientras no exista EventBridge real. |
| F-05 | Pantalla de feedback (Likert 1–5) | Envía evento/registro de validación. | FE | F-04 | Pendiente | Confirmar con Backend si se envía como evento (`FeedbackSubmitted`) o llamada directa en el MVP de demo. |
| F-06 | Manejo de errores/estados de carga | Loading states, mensajes de error (401, 404, 500) para las pantallas anteriores. | FE | F-01–F-05 | Pendiente | — |

---

## 5. Semana 3 (Días 15–21) — Integración real y pruebas

| ID | Tarea | Descripción | Responsable | Depende de | Estado | Observaciones |
|---|---|---|---|---|---|---|
| B-13 | Persistencia Aurora: `feedback_storage.py` + tablas `feedback`/`rankings` | `[tasks.md 14.x, 15.x]` | BE | B-03 | Pendiente | — |
| B-14 | Tests de propiedades (determinismo, pesos válidos, confianza) | `[tasks.md 6.9–6.10, 10.8, 27.1]` | BE | B-07 | Pendiente | Propiedad de confianza (10.1) sigue ❄️ congelada — depende de R1. |
| B-15 | Benchmark de rendimiento (6,208 filas ≤ 1 s) | `[tasks.md 27.2]` | BE | B-07 | Pendiente | — |
| I-03 | **Integración real** Frontend ↔ Backend ↔ eventos | Conectar Frontend a los endpoints/eventos reales (no mocks); validar el flujo `AssessmentCompleted → RecommendationGenerated → FeedbackSubmitted` de punta a punta. | INT | B-09, B-10, B-11, F-04, F-05 | Pendiente | **Tarea central del rol de Integración** — no es solo "juntar código". |
| I-04 | Pruebas de integración end-to-end | Ejecutar el flujo completo con datos reales/fixtures y verificar cada contrato de API/evento. `[tasks.md 26.x]` | INT | I-03 | Pendiente | — |
| I-05 | **Detección y documentación de incompatibilidades** | Registrar en un documento vivo cualquier caso donde Backend y Frontend, funcionando bien por separado, fallen al integrarse (formatos de fecha, nombres de campo, códigos de estado HTTP no manejados, payloads de evento distintos a lo documentado, etc.). | INT | I-04 | Pendiente | Debe producir un artefacto propio (`docs/INTEGRATION_ISSUES.md`) — no solo mencionarse de palabra. |
| I-06 | Coordinar resolución de incompatibilidades | Triage con Backend/Frontend: quién corrige qué, prioridad, plazo. | INT | I-05 | Pendiente | Rol de coordinación, no de ejecución directa del fix. |
| B-16 / F-07 | Corrección de incompatibilidades detectadas | Fixes concretos en Backend/Frontend según lo acordado en I-06. | BE / FE | I-06 | Pendiente | Se abre un ID por cada incompatibilidad real detectada; aquí va como placeholder. |
| I-07 | Pruebas de usabilidad / validación de flujo con usuario piloto | Recorrido completo por una persona ajena al equipo. | INT + FE | B-16/F-07 | Pendiente | — |
| I-08 | Documentación final (`README`, `ARCHITECTURE.md`, `DEPLOYMENT.md`) | `[tasks.md 28.x]` | INT | B-16/F-07 | Pendiente | — |
| I-09 | **Checkpoint Fase 3** — sistema integrado y probado | `[tasks.md 29.1–29.6]` | INT | I-07, I-08, B-14, B-15 | Pendiente | Cobertura ≥ 60%, todas las propiedades no congeladas pasan. |

---

## 6. Semana 4 — Despliegue y colchón (Días 22–28)

| ID | Tarea | Descripción | Responsable | Depende de | Estado | Observaciones |
|---|---|---|---|---|---|---|
| D-01 | Despliegue a ambiente demo | Consumir Aurora/EventBridge/Secrets Manager ya existentes (no crear infra nueva). `[tasks.md 28.4]` | BE + INT | I-09 | Pendiente | — |
| D-02 | Smoke tests post-despliegue | Verificar los 2 endpoints de Matching + flujo de eventos en el ambiente real. | INT | D-01 | Pendiente | — |
| D-03 | Corrección de bugs críticos (colchón) | Reserva de tiempo sin tareas fijas — se llena según lo que falle en D-02. | Todos | D-02 | Pendiente | Esta es la semana de colchón explícita que pediste; **no** planificar trabajo nuevo aquí. |
| D-04 | Ensayo de presentación/demo final | Recorrido del flujo completo en vivo. | Todos | D-03 | Pendiente | — |

---

## 7. Riesgos y decisiones abiertas (pueden bloquear tareas de este plan)

| ID | Riesgo/decisión | Tareas que bloquea si no se resuelve en Semana 1 |
|---|---|---|
| **R1** | Assessment estructurado (formulario) vs. captura conversacional (Bloques A/B/C vía LLM) — sigue marcada como ❄️/pendiente de decisión de producto en `design.md` §1 y §3. | F-03, F-04, B-08, B-14 (test de confianza) |
| **R2** | Nombre de tabla y dimensión real de embeddings del RAG (`career_chunks`, 1024 vs 1536, Titan v1/v2) — pendiente de verificar contra el repo `08-deep-agent`. | No bloquea el MVP de 3 semanas si el chat/RAG queda fuera de alcance de la demo (recomendado dado el tiempo disponible). |
| **R3** | Disponibilidad real de Identity Context (JWT) de otro equipo/repo durante el sprint. | F-01, F-02, B-12 — si no está disponible a tiempo, mockear JWT localmente y documentarlo como deuda técnica. |
| **R4** | Alcance real vs. tiempo disponible: la arquitectura completa de Spark Match (5 bounded contexts + EventBridge + AI Advisor separado) es significativamente mayor a lo que 5 personas cubren en 3 semanas. | Todo el plan — se recomienda que P-01 (día 1) cierre explícitamente qué queda **fuera** de la demo (ej. AI Advisor conversacional, RAG, notificaciones). |

---

## 8. Cómo mantener este documento

- Actualizar la columna **Estado** conforme avanza el sprint (`Pendiente → En progreso → Completada`,
  o `Bloqueada` si depende de un riesgo de la sección 7).
- Cuando Integración detecte una incompatibilidad real (I-05), agregar una fila nueva bajo `B-16/F-07`
  con ID propio (ej. `B-16.1`, `F-07.1`) para que quede trazable quién la corrigió y cuándo.
- Si se resuelve un riesgo de la sección 7, anotar la fecha y desbloquear las tareas asociadas.
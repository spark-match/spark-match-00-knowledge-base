# Plan de pruebas de integración (QA)

> **Responsable:** @Fabiola (testing / integración) · **Estado:** vivo · **Última actualización:** 2026-07-24
> Complementa a [`INTEGRATION_ISSUES.md`](INTEGRATION_ISSUES.md): aquí se **verifica** que cada
> incompatibilidad quede resuelta al integrar los componentes.

## 1. Objetivo

Validar que los componentes de Spark Match —que hoy funcionan por separado— se comuniquen
correctamente de extremo a extremo, y dejar **evidencia** para el criterio de resultados/demo de la
rúbrica (puntos 7 y 8).

## 2. Arquitectura y puntos de integración

```
UI (Angular) --> API (backend serverless TS) --> Agente (Python/deepagents)
                                                     |            |
                                                     v            v
                                              LLM (Bedrock)   Motor de scoring --> features.csv
```

Cada flecha es un **punto de integración** que se prueba de forma independiente y luego en conjunto.

## 3. Matriz de pruebas de integración

| # | Integración | Qué se verifica | Resultado esperado | INT | Estado |
|---|---|---|---|---|---|
| I-1 | UI → API | Escala de feedback | El front envía **1–5** (Likert), la API lo acepta (no 422) | INT-001 | ⬜ |
| I-2 | UI → API | Valor de región | El front envía **"Lima"** (y los 25 dptos), no "Lima Metropolitana" | INT-002 | ⬜ |
| I-3 | UI → API | Filtros | Se puede enviar sin filtros obligatorios ("me da igual") | INT-003 | ⬜ |
| I-4 | API → Agente | Mensaje del usuario | El agente recibe el texto y responde perfil + pesos | INT-005 | ⬜ |
| I-5 | Agente → Scoring | Pesos | Los 5 pesos **suman 1** | INT-011 | ⬜ |
| I-6 | Agente → Scoring | Afinidad | La afinidad llega **normalizada en [0,1]** (no 0–100) | INT-011 | ⬜ |
| I-7 | Scoring → Datos | Ranking | El Top-N sale de `features.csv` y respeta región y presupuesto | INT-002 | ⬜ |
| I-8 | Salida → UI | Cantidad | El reporte muestra **Top-5** (no Top-3) | INT-004 | ⬜ |
| I-9 | Agente → UI | Justificación | La explicación cita **datos reales** del ranking (no inventados) | — | ⬜ |
| I-10 | Observabilidad | LangSmith | Cada interacción queda **registrada** (mensajes, tokens, herramientas) | — | ⬜ |
| I-11 | Costos | KPI por usuario | Se puede atribuir el consumo (tokens/costo) a cada usuario registrado | — | ⬜ |

> Marca ✅ / ❌ en "Estado" a medida que se integra cada pieza.

## 4. Casos de prueba end-to-end (guiones de conversación)

Cada caso simula una conversación completa (multi-turno). Sirven también como **evals** (LLM-judge).

| Caso | Entrada del estudiante | Qué debe pasar |
|---|---|---|
| **E2E-1 (canónico)** | "Vivo en Cusco, presupuesto 600 soles/mes, quiero buen sueldo y quedarme cerca." | Perfil con alta prioridad salario + región Cusco; Top-5 de Cusco dentro del presupuesto; explicación con datos reales; feedback 1–5 disponible. |
| **E2E-2 (info faltante)** | "No sé qué estudiar." | El agente **repregunta** (entrevista escalonada) por intereses, presupuesto, región, antes de recomendar. |
| **E2E-3 (Lima, sin límite)** | "Soy de Lima, no tengo problema de presupuesto, quiero la mejor universidad." | Región = "Lima" (match en datos); peso de costo bajo; Top-5 coherente. |
| **E2E-4 (mensaje ambiguo)** | "Quiero algo bueno y fácil." | El agente pide aclaración; no inventa pesos incoherentes. |
| **E2E-5 (afinidad)** | "Me encantan las matemáticas y programar." | RIASEC con componente investigativo/realista; carreras cuantitativas arriba del ranking. |

## 5. Qué se puede probar AHORA (antes de la integración total)

- **Frontend (rama `feat/screen-designe`):** regiones correctas, feedback 1–5, muestra Top-5, sin emojis.
- **Contrato de datos:** comparar el JSON que envía el front vs. el que espera la API (detecta I-1..I-3).
- **Datos:** verificar en `features.csv` que las regiones y columnas (`career`, `career_family`,
  `*_norm`) existan y que "Lima Metropolitana" NO aparezca.
- **Preparar los 5 guiones** de la sección 4 como dataset de evals.

## 6. Herramientas

| Para probar | Herramienta |
|---|---|
| La UI | Navegador (Angular en `localhost:4200`) |
| Los endpoints de la API | Postman / Thunder Client |
| Las trazas del agente | LangSmith |
| Los datos / regiones | Script Python sobre `features.csv` |

## 7. Criterio de "listo para la demo" (Definition of Done)

- [ ] El flujo E2E-1 corre de punta a punta sin errores (aunque los pesos no estén afinados).
- [ ] Front y backend **se comunican** (una request real llega y vuelve).
- [ ] LangSmith muestra la traza de la conversación.
- [ ] El feedback 1–5 se guarda sin error 422.
- [ ] Se tiene al menos **1 conversación completa grabable** para el video.

## 8. Registro de ejecución

| Fecha | Caso probado | Resultado | Notas / bug encontrado |
|---|---|---|---|
| | | | |

# Observaciones al documento tasks.md

## Fabiola:

> Nota general: el plan recoge bien lo conversado (RIASEC, scoring de 5 factores, Bedrock,
> Aurora+pgvector, DynamoDB, etiquetado RIASEC por lotes). Las observaciones de abajo son
> puntos de correctitud y simplificaciones, no un rechazo del plan.

### Correctitud (posibles bugs)

- **Afinidad: el divisor está mal (÷18, debería ser ÷6).** Tarea 6.2 y test 6.8. Con el
  algoritmo descrito, el máximo real de `score_crudo` es 6 (3+2+1), no 18. Efecto: una afinidad
  perfecta da 0.33 en vez de 1.0 (el propio test 6.8 lo confirma: `"RIA" vs "RIA" → 6/18 ≈ 0.33`).
  Esto hunde el peso de la afinidad en todo el score, que es el núcleo vocacional del producto.
  → Dividir entre 6. (Extra: el puntaje depende de la posición de la letra en el código del
  estudiante, así que "RIA" y "AIR" dan lo mismo; se pierde el match posicional — revisar si es lo
  deseado.)

- **Choque con el pipeline ya construido de Nikolai → doble inversión.** En tasks.md
  `costo_norm`/`duracion_norm` NO se invierten (3.2) y luego el scoring hace `1 − costo_norm` /
  `1 − duracion_norm` (6.5). Pero el `features.csv` real (repo 05, rama `dev`) ya trae
  `cost_norm`/`duration_norm` invertidos (mayor = mejor) y la columna se llama `admission_norm`,
  no `selectividad_norm`. Si el scoring usa el CSV real tal cual → doble inversión → ranking al
  revés en costo y duración. → Fijar UNA sola convención; lo más simple (los datos ya existen):
  usar las columnas de Nikolai directas, sin `1 − x`.

- **Los nombres de columna del ScoringEngine no coinciden con el CSV real.** Tarea 6.6 exige
  `ingreso_norm, costo_norm, selectividad_norm, duracion_norm, career_id, career_name…` (español);
  el CSV real tiene `income_norm, cost_norm, admission_norm, duration_norm, id, career, annual_cost…`
  (inglés) y NO tiene `riasec_profile`. Con esos nombres, la validación de schema lanzaría
  `ValueError` contra el CSV real. → Unificar nombres.

- **Costo: mensual vs anual.** Tareas 5.2, 6.6 y 6.7 usan `costo_mensualidad` y filtran
  `costo_mensualidad <= presupuesto_max`, pero el dato es anual (`annual_cost`), el Figma muestra
  "Costo Anual" y el presupuesto es anual. → Unificar todo a base anual.

### Simplificaciones (para la demo)

- **Etiquetar RIASEC solo a las 554 carreras únicas (o 81 familias), no a las 6.208 filas.** El
  RIASEC es propiedad de la carrera, no de cada combinación carrera×universidad. Tareas 4.1/25.1
  etiquetan las 6.208 → ~11× llamadas de más a Bedrock y riesgo de códigos distintos para la misma
  carrera. → Etiquetar únicas una vez y hacer join. Más barato, simple y consistente.

- **Auth: quizá no hace falta JWT + Lambda para la demo.** Es un chat anónimo de orientación (no
  hay login en el Figma). Bastaría un `session_id` anónimo (uuid). Si se quiere identidad, Google
  OAuth; pero para la demo lo más simple es sesión anónima → quita tareas 16, 17.1, 18 y parte de
  infra.

- **Un solo servicio para la demo.** Lambda + Fargate + API Gateway agrega despliegue; para la demo,
  todo en un FastAPI es más rápido, y el híbrido queda como meta de producción (ADR-001).

- **Reusar el pipeline y el `raw.xlsx` de Nikolai en vez de re-scrapear MINEDU.** Ya existen (rama
  `dev`); Selenium al portal es frágil (timeouts, cambios de HTML). → Usar el snapshot como fuente
  y dejar el scraping como refresco opcional.

### Menores

- El clamp de duración `[3,7]` (3.2) difiere del criterio de Nikolai (`≤10`); revisar que no altere
  valores válidos.
- Las tareas referencian `_Requerimientos: X.Y_`, pero `requirements.md` y `PRD.md` aún no están
  actualizados; verificar que esos IDs existan tras propagar.

### Alineación de stack: el backend de Angel vs tasks.md (importante)

Angel ya avanzó el backend en `03-backend` (rama `feature/scaffolding-fase-1`): un scaffold
serverless DDD+EDA con el contexto **Identity** funcional (register/login/JWT), EventBridge,
AWS SAM y **14 ADRs** propios. Muy sólido, pero su stack **no coincide con el `tasks.md`**:

| Tema | tasks.md (David) | Backend de Angel |
|---|---|---|
| Lenguaje | Python en todo el backend | **TypeScript** (Python solo el agente) |
| Compute | Fargate + FastAPI para `/chat` | **Lambda** para todo (su ADR-001 descarta Fargate) |
| Framework Lambda | boto3 puro | **Middy + Zod + Powertools** (su ADR-013) |
| Gestor | uv (Python) | **npm workspaces** |
| Auth | sesión / JWT simple | **Identity completo** (register/login/usuarios) |

Coinciden en: serverless AWS, Aurora+pgvector, e híbrido con servidor Python para el agente
(su ADR-012 ≈ ADR-001 de este repo). El `Identity/login` de Angel también va en sentido contrario
a la simplificación de "sesión anónima" (ver arriba).

→ **Definir cuál es la fuente de verdad del stack backend antes de repartir tareas** (Python/FastAPI
del tasks.md vs TypeScript-serverless ya construido por Angel). Si no, hay retrabajo asegurado.

## nombre 2:

- observacion 01
- observacion 02
...


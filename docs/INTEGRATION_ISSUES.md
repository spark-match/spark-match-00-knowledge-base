# Incompatibilidades de integración

> **Artefacto de la tarea I-05** del [plan de trabajo](WORK_PLAN/wp.md).
> **Responsable**: @Fabiola (testing / integración) · **Última actualización**: 2026-07-09

Registro vivo de los casos donde **dos componentes funcionan bien por separado pero fallan al
integrarse**: nombres de campo distintos, contratos de API que no coinciden, formatos, rangos de
valores, stacks divergentes, o datos que un lado espera y el otro no produce.

No es una lista de bugs internos de cada repo — eso va en los issues de cada repositorio.

## Cómo usarlo

- Un hallazgo = una entrada, con ID propio (`INT-00X`).
- **Estado**: `🔴 Abierto` · `🟡 En curso` · `🟢 Resuelto`.
- Al resolverse, anotar la fecha y cómo se resolvió (no borrar la entrada: la trazabilidad importa).

## Resumen

| ID | Componentes | Problema | Impacto | Estado |
|---|---|---|---|---|
| [INT-001](#int-001) | Frontend ↔ Backend | Feedback binario (👍/👎) vs `validation_score` 1–5 | La API rechaza el feedback (HTTP 422) | 🔴 Abierto |
| [INT-002](#int-002) | Frontend ↔ Datos | La UI usa "Lima Metropolitana"; el dataset solo tiene "Lima" | El filtro de región no devuelve resultados | 🔴 Abierto |
| [INT-003](#int-003) | Frontend ↔ Reglas de negocio | Los filtros son obligatorios en la UI; las reglas los definen opcionales | El estudiante no puede decir "me da igual" | 🔴 Abierto |
| [INT-004](#int-004) | Frontend ↔ Backend | El reporte muestra Top-3; el backend devuelve Top-5 | Se pierden 2 recomendaciones | 🔴 Abierto |
| [INT-005](#int-005) | Datos ↔ Agente | `features.csv` no tiene `riasec_profile` | La afinidad no se puede calcular sobre las 6.208 filas | 🟡 En curso |
| [INT-006](#int-006) | Backend ↔ tasks.md | Stack real en TypeScript; el `tasks.md` asume Python/FastAPI | Riesgo de retrabajo al repartir tareas | 🔴 Abierto |
| [INT-007](#int-007) | Frontend ↔ Prototipo | Scaffold en Angular 21; el prototipo del Figma es Lovable (React) | El código del prototipo no se reutiliza | 🔴 Abierto |
| [INT-008](#int-008) | Frontend ↔ Datos | La UI afirma "datos actualizados a diciembre 2024", sin respaldo | Afirmación pública posiblemente falsa | 🔴 Abierto |
| [INT-009](#int-009) | Gobernanza ↔ Data pipeline | El `.gitignore` de la org tiene `*.csv` | `git add` ignora en silencio los entregables del pipeline | 🟡 En curso |
| [INT-010](#int-010) | CI ↔ Testing | Lint y tests del frontend son no-bloqueantes (`\|\| echo`) | "CI verde" no significa que los tests pasen | 🔴 Abierto |
| [INT-011](#int-011) | Agente ↔ Scoring | La afinidad del agente va en 0–100; la fórmula espera [0,1] | Score inflado ×100 si se combinan sin normalizar | 🔴 Abierto |
| [INT-012](#int-012) | ADR-003 ↔ Infra | El ADR-003 eligió 1 ambiente AWS; la infra implementa 2 (dev+prod) | Doc contradice el código + mayor gasto de créditos AWS | 🔴 Abierto |

---

## INT-001

**Feedback binario en la UI vs escala 1–5 en el backend** · Frontend ↔ Backend · 🔴 Abierto

Cada tarjeta del reporte ofrece **"¿Útil? Sí / No"** (binario). Pero el backend define
`validation_score: int` en el rango **[1, 5]** y **responde HTTP 422** si el valor cae fuera
(`tasks.md` 11.3 y 17.2). El `wp.md` (F-05) pide explícitamente *"Pantalla de feedback (Likert 1–5)"*.

Ambos lados funcionan por separado; al integrarse, **todo feedback será rechazado**.

- **Decidir**: ¿la UI pasa a estrellas / escala 1–5, o el backend acepta un booleano?
- Si se mantiene el binario, hay que cambiar el contrato de la API y su validación.

## INT-002

**"Lima Metropolitana" no existe en el dataset** · Frontend ↔ Datos · 🔴 Abierto

La UI muestra `Lima Metropolitana` como región (cabecera del chat y perfil del reporte). La columna
`location` de `features.csv` tiene **25 departamentos**, y el valor real es **`Lima`** a secas.

Verificado sobre `data/features.csv` (repo `05-data-pipeline`): no existe ninguna región cuyo nombre
contenga "Metropolitana".

- El filtro por región hace match exacto contra `location` → **devolvería 0 carreras**.
- **Acción**: usar exactamente los 25 valores del dataset como opciones del desplegable.

## INT-003

**Los filtros son obligatorios en la UI** · Frontend ↔ Reglas de negocio · 🔴 Abierto

La pantalla de filtros bloquea el botón con el mensaje *"Completa los 3 filtros"*. Las reglas de
negocio (§6.4 de `4_reglas-negocio-agente.md`) dicen que los filtros del Bloque C **descartan
opciones pero son opcionales**: si el estudiante responde *"me da igual"*, ese filtro no se aplica.

Los filtros de tipo ya tienen su escape ("Ambas" / "Ambos"), pero **región no tiene opción
"Cualquiera / me da igual"** y además bloquea el avance.

## INT-004

**Top-3 en el reporte vs Top-5 en el backend** · Frontend ↔ Backend · 🔴 Abierto

El reporte anuncia *"Se encontraron 3 carreras con alta compatibilidad"*. El backend trunca a
**Top-5** (`tasks.md` 10.3: `ranked_list[:5]`), y el `4_reglas-negocio-agente.md` §6.7 define
`Top-N (default N=5)`.

Hay que fijar **un solo N** y que ambos lados lo respeten.

## INT-005

**`features.csv` no tiene `riasec_profile`** · Datos ↔ Agente · 🟡 En curso

El dataset económico (6.208 filas) no trae la dimensión vocacional, así que el término `afinidad`
de la fórmula de scoring **no se puede calcular** sobre los datos reales. Hoy la afinidad solo
existe sobre las 10 carreras hardcodeadas del POC `08-deep-agent`.

- **En curso**: PR `feature/riasec-tagging` en `05-data-pipeline` — etiqueta las 554 carreras
  únicas vía Bedrock y hace el join a las 6.208 filas.

## INT-006

**El backend real es TypeScript; el `tasks.md` asume Python** · Backend ↔ tasks.md · 🔴 Abierto

El scaffold de `03-backend` es **serverless DDD+EDA en TypeScript** (AWS SAM, Lambda, EventBridge,
Middy + Zod + Powertools), con el contexto Identity ya funcional. El `tasks.md` fija como
restricciones transversales **Python + `uv` + FastAPI + boto3 puro**.

El `tasks.md` acotó después su alcance al *Matching Context* (Python), lo que reduce el choque, pero
**el stack del resto del backend sigue sin estar declarado como fuente de verdad única**. Conviene
cerrarlo antes de repartir tareas.

## INT-007

**Frontend en Angular; prototipo en Lovable/React** · Frontend ↔ Prototipo · 🔴 Abierto

El scaffold de `04-frontend` es **Angular 21 + Material + SCSS**. El prototipo validado del diseño
está hecho en **Lovable**, que genera **React**.

El diseño sigue siendo válido como referencia visual, pero **el código del prototipo no se reutiliza**:
hay que reimplementar los componentes en Angular. Conviene que el equipo lo asuma explícitamente y no
lo descubra a mitad de sprint.

## INT-008

**La UI afirma una fecha de datos sin respaldo** · Frontend ↔ Datos · 🔴 Abierto

La portada afirma que los datos *"provienen de la plataforma oficial Ponte en Carrera del Ministerio
de Educación del Perú, **actualizados a diciembre 2024**"*, y las tarjetas del reporte citan
*"Ponte en Carrera 2024"*.

Ni el `README.md` ni el `Diccionario de datos.md` del repo `05-data-pipeline` indican la fecha de
corte del `raw.xlsx`. Es una afirmación pública sobre datos oficiales.

- **Acción**: confirmar la fecha real con @Nikolai antes de publicarla, o retirarla de la UI.
- **Actualización (2026-07-09)**: @Nikolai confirmará la fecha; el portal de descarga estaba caído.
  Además, el **MTPE consolidó [Mi Carrera](https://micarrera.trabajo.gob.pe/)** como observatorio
  oficial, relegando a Ponte en Carrera. La UI atribuye los datos a "Ponte en Carrera", pero la
  fuente podría estar en transición → revisar atribución y fecha de corte antes de publicar.

## INT-011

**Escala de afinidad: 0–100 en el agente vs [0,1] en la fórmula** · Agente ↔ Scoring · 🔴 Abierto

El tool `calculate_affinity` del agente (`08-deep-agent`, `src/tools/matching.py`) devuelve la
afinidad en **porcentaje (0–100)** — el `reason` dice literalmente *"{score}% de afinidad"*. La
fórmula de scoring de 5 factores (`4_reglas-negocio-agente.md` §3.1) trabaja con todas las variables
en **[0, 1]**.

Si el término de afinidad se combina con `income_norm`, `cost_norm`, etc. sin dividir entre 100, el
score queda **inflado ×100** respecto a los demás factores y el ranking se distorsiona.

- **Acción**: normalizar la afinidad a [0, 1] en el punto de integración agente ↔ scoring (dividir
  entre 100, o que el tool ya la devuelva normalizada).

## INT-009

**`*.csv` en el `.gitignore` ignora los entregables del pipeline** · Gobernanza ↔ Data pipeline · 🟡 En curso

El `.gitignore` estándar añadido a `05-data-pipeline` incluye la regla **`*.csv`** bajo el comentario
*"NUNCA commitear data cruda"*. Pero **este repo sí versiona su dataset** (`data/features.csv`,
`data/filtered.csv`, `snapshots/features/*.csv`).

Los archivos ya trackeados no se pierden, pero **cualquier CSV nuevo se ignora en silencio**.
Verificado con `git check-ignore`:

```
.gitignore:119:*.csv    data/riasec_tags.csv
.gitignore:119:*.csv    data/riasec_validation_sample.csv
```

Esos dos son justamente los entregables de la tarea **B-04** (las 554 carreras etiquetadas y la
muestra para revisión humana). Sin excepciones, `git add` los descarta sin avisar.

- Incoherencia adicional: la regla dice "nunca commitear data cruda", pero **`data/raw.xlsx` sí está
  commiteado** y no se ignora (`.xlsx` no aparece en las reglas).
- **En curso**: el PR `feature/riasec-tagging` adopta el `.gitignore` de la org y le añade una sección
  de excepciones (`!data/features.csv`, `!data/riasec_tags.csv`, `!snapshots/**/*.csv`…).
- **Pendiente**: decidir si la plantilla de la org debe llevar `*.csv` para repos de datos.

## INT-010

**Lint y tests del frontend no pueden fallar el CI** · CI ↔ Testing · 🔴 Abierto

En `04-frontend`, el workflow de CI ejecuta ambos pasos con `|| echo`, así que **siempre terminan en
verde**, aun cuando fallen:

```yaml
run: npm run lint || echo "Lint warnings present (non-blocking)"
run: npm test -- --watch=false --browsers=ChromeHeadlessCI || echo "Tests skipped (browser config pending — non-blocking)"
```

El propio mensaje sugiere que los tests **ni siquiera se ejecutan**: falta la dependencia
`@vitest/browser-playwright` para el setup de Angular 21 + Vitest.

Está marcado como temporal y documentado en el PR, pero mientras siga así **"CI verde" no significa
"los tests pasan"** en el frontend. Riesgo de llegar a la demo con la red de seguridad desactivada.

- **Acción**: instalar la dependencia que falta y quitar los `|| echo` antes de la Fase 3 (integración).

---

## INT-012

**El ADR-003 eligió 1 ambiente AWS, pero la infra implementa 2 (dev + prod)** · ADR-003 ↔ Infra · 🔴 Abierto

El **ADR-003** eligió explícitamente **un solo ambiente AWS** (diferenciado por tags), y descartó la
opción de carpetas `live/dev` + `live/prod` por costo (~$30/mes con 1 ambiente vs. ~$120/mes con 2).

En la reunión del 11-jul el equipo decidió tener **dev + prod**, y los PRs de infra ya lo
implementan:

- `01-devops` — `feat/terraform-pipelines-n-env-aware`: workflows de Terraform conscientes de N ambientes.
- `02-infrastructure` — `feat/multi-env-and-n-env-pipelines`: crea `live/dev` + `live/prod` (justo la
  opción que el ADR-003 había descartado).

El código sigue la decisión nueva, pero el **documento (ADR-003) ahora dice lo contrario**.

- **Acción 1:** actualizar el ADR-003 (o crear un ADR-004) que ratifique los 2 ambientes y revise la
  estimación de costo.
- **Acción 2 (créditos AWS):** los créditos de AWS Academy son limitados; 2 ambientes con
  networking/NAT duplicados consumen más rápido. Confirmar que el ambiente `dev` sea liviano o se
  apague cuando no se use.

---

## Desviaciones documentadas (no son incompatibilidades)

Cambios conscientes respecto a `tasks.md`, registrados aquí para que todos estén enterados.

| Origen | Desviación | Motivo |
|---|---|---|
| PR `feature/riasec-tagging` (repo 05) | Model ID de Bedrock: `anthropic.claude-opus-4-8` en vez de `anthropic.claude-3-5-sonnet-20241022` | Los IDs actuales de Bedrock no llevan sufijo de fecha; el del `tasks.md` está desactualizado |
| PR `feature/riasec-tagging` (repo 05) | Cliente `AnthropicBedrockMantle` (SDK de Anthropic) en vez de `boto3` + `invoke_model` | `invoke_model` es el camino legacy; el SDK evita construir el JSON a mano |

> Nota relacionada: la **Batches API no está disponible en Amazon Bedrock**, así que no aplica el
> descuento del 50% por procesamiento en lote para el etiquetado RIASEC.

# reglas-negocio-agente.md

> **Status**: 🟢 vigente
> **Fecha**: 2026-07-05 · **Última verificación contra el código**: 2026-09-01
> **Audiencia**: AI Devs, Backend, Data Engineers
> Documento que fija el **flujo del agente** y las **reglas de negocio del scoring**, y reconcilia
> `2_requirements.md` y `3_design.md` en los puntos donde el diseño evolucionó (ver §8).
>
> **Cómo mantener este documento:** las §3, §7 y §9 describen código que existe. Verificarlas
> contra `spark-match-07-deep-agent` (no contra la versión anterior de este documento) y fechar la
> verificación. Lo que se diseñó pero no se implementó va a la tabla de brechas de §9.2, no al
> cuerpo del documento en presente.

---

## 1. Propósito y alcance (MVP)

El producto acompaña al estudiante en su decisión de carrera mediante un **agente cognitivo**
(Deep Agent) que hace, de forma conversacional, el trabajo que normalmente haría un orientador
vocacional: descubre el perfil del estudiante, extrae qué le importa, cruza esa información con
datos de carreras/universidades y entrega un informe que el estudiante aprueba o rechaza.

El proyecto define 5 etapas de acompañamiento. **El MVP cubre las etapas 1–3:**

| Etapa | Nombre | ¿En MVP? |
|---|---|---|
| 1 | Exploración y autodescubrimiento | ✅ Sí |
| 2 | Decisión y match (núcleo de recomendación) | ✅ Sí |
| 3 | Acción y validación humana | ✅ Sí |
| 4 | Seguimiento universitario | ❌ Fuera de MVP |
| 5 | Transición laboral | ❌ Fuera de MVP |

---

## 2. Principio de diseño: ingeniería del conocimiento

El motor de recomendación **no se abandona**; se **adecúa**. En lugar de un formulario rígido, el
conocimiento del proceso de orientación (qué preguntar, cómo pesar cada factor) se traslada a un
agente. El agente:

1. Conversa y **extrae** los datos necesarios para el scoring (perfil vocacional + filtros).
2. **Razona** cruzando esos datos con el catálogo real de programas y datos oficiales.
3. Entrega un **artefacto/informe** (Top-N carreras + universidades) que el humano valida.

> **Nota sobre los pesos.** La idea original (Nikolai) era que el agente extrajera de la
> conversación un **peso por persona** para cada factor. La implementación actual usa **pesos
> fijos** (§3.2) por las razones que documenta el propio código. Los pesos por persona siguen
> siendo una dirección válida, pero hoy **no están implementados**: ver la brecha en §9.2.

---

## 3. Modelo de scoring

> Fuente de verdad: `spark-match-07-deep-agent`, `src/tools/recommendation/scoring.py`.
> `SCORING_VERSION = "1.0.0"`. La versión viaja en cada respuesta y se guarda en el informe, para
> que un informe emitido hoy siga siendo explicable si mañana cambian los pesos.

### 3.1 Fórmula

```
match_score = 0.50 · riasec_affinity
            + 0.20 · income
            + 0.20 · admission_accessibility
            + 0.10 · affordability
```

- Cada componente ∈ `[0, 1]`; el resultado se reporta como `match_score` ∈ `[0, 100]`
  (`round(total * 100, 1)`).
- **Determinismo**: mismos inputs → mismo ranking.
- La respuesta incluye `score_breakdown` con los 4 componentes por separado, para poder explicar
  al estudiante de dónde sale su puntuación.

**Tres reglas que hay que entender antes de tocar nada:**

1. **Los filtros no puntúan, excluyen.** Región, gestión, tipo de institución y presupuesto se
   aplican *antes* de puntuar (§5, Bloque C). Meterlos en el score los convertiría en preferencias
   negociables, y no lo son: quien pide Arequipa pública no debe ver Piura privada con menos
   puntos, debe **no verla**.
2. **Una cifra imputada no puntúa.** El pipeline rellena lo que falta con la mediana de la familia
   de carrera, y cada fila trae su bandera `*_measured`. Si la cifra no se midió, ese componente
   vale `NEUTRO = 0.5`: ni ayuda ni penaliza. Afecta al **73 % de los ingresos** y al **65 % de las
   tasas de admisión**.
3. **Los rangos de referencia son fijos, no relativos al resultado.** Se normaliza contra
   percentiles del dataset completo, no contra el mín/máx de los candidatos filtrados. Con min-max
   sobre el subconjunto, añadir un candidato cambiaría la nota de todos los demás y un 0.8
   significaría algo distinto en cada búsqueda.

**Rangos de referencia** (percentiles 5 y 95, calculados el 2026-08-09 sobre `programs.csv` usando
**solo filas medidas**: 1.680 ingresos, 3.112 costos, 2.160 tasas de admisión de 6.208 filas):

| Magnitud | p5 | p95 | Sentido |
|---|---|---|---|
| `monthly_income` | 1 598,8 | 4 195,0 | mayor = mejor |
| `annual_cost` | 52,0 | 6 680,0 | **invertido**: menor = mejor |
| `admission_rate` | 9,0 | 89,0 | mayor = mejor (**accesibilidad**, no selectividad) |

> Se usa p5/p95 y no mín/máx porque el máximo de `annual_cost` son S/ 32 530, un caso extremo que
> aplastaría a todos los demás contra el cero.

### 3.2 Pesos: fijos, no por persona

Los pesos son **constantes del sistema** (`WEIGHTS` en `scoring.py`), iguales para todos los
estudiantes. La afinidad domina (0.50) porque el producto es orientación **vocacional**: su
premisa es que encajar importa más que cobrar.

Cambiarlos es una decisión de producto: se edita `WEIGHTS`, se sube `SCORING_VERSION` y los
informes antiguos siguen siendo interpretables.

### 3.3 Por qué la duración NO puntúa

`duration_years` **se informa pero no entra en el score**. Puntuarla exigiría decidir que menos
años es mejor, y eso no es cierto para nadie en general: hay estudiantes para quienes una carrera
más larga es justo lo que quieren. Se muestra como dato verificable en la tarjeta y el estudiante
decide.

> ⚠️ Esto invalida cualquier versión de este documento que hable de una **fórmula de 5 factores**
> con `w_duracion`. No existe, y es deliberado.

### 3.4 Origen de cada componente

| Componente | Columna del dataset | Qué mide |
|---|---|---|
| `riasec_affinity` | `riasec_profile` × perfil del estudiante | Ajuste vocacional (pesos posicionales 3-2-1), llevado a `[0,1]` dividiendo entre 100 |
| `income` | `monthly_income` + `income_measured` | Ingreso mensual del egresado (S/ / mes) |
| `admission_accessibility` | `admission_rate` + `admission_measured` | Qué tan alcanzable es entrar (%) |
| `affordability` | `annual_cost` + `cost_measured` | Costo anual, invertido (S/ / año) |

> **El código RIASEC de cada carrera lo asignó un modelo de lenguaje**, no el MINEDU. Sirve para
> orientar, no como clasificación oficial, y así se declara en la herramienta.

---

## 4. Flujo conversacional del agente

```
[Etapa 1] Exploración        → extrae PERFIL RIASEC (riasec_code de 3 letras)
        │
[Etapa 2] Filtros            → extrae FILTROS duros (región, gestión, tipo, presupuesto)
        │
        ▼
   recommend_programs: filtra → puntúa (§3.1) → ordena → Top-N
        │
[Etapa 3] Validación humana  → el estudiante acepta o rechaza el informe
```

**Umbral de avance**: no se calcula ranking hasta tener los 6 scores RIASEC y el `riasec_code`.
El gate cuantitativo `confidence ≥ 0.70` y el tope de 4 repreguntas de `3_design.md` **no están
implementados** (§9.2).

---

## 5. Cuestionario cognitivo (qué debe preguntar el agente)

> Las preguntas son **guía conversacional**, no un formulario. El agente las formula de forma
> natural y no repite lo ya respondido.

### Bloque A — Exploración vocacional → perfil RIASEC *(Etapa 1)*

| # | Pregunta guía | Alimenta |
|---|---|---|
| A1 | ¿Qué actividades te hacen perder la noción del tiempo? | intereses |
| A2 | Cuando resuelves un problema, ¿prefieres hacerlo con las manos, analizando datos, creando algo, ayudando a alguien, liderando, u organizando? | R / I / A / S / E / C |
| A3 | ¿Qué cursos del colegio disfrutabas y cuáles evitabas? | intereses / dislikes |
| A4 | ¿Qué te mantiene motivado cuando trabajas en algo? | refuerza E / S / I |

**Salida del bloque:** 6 scores RIASEC (1–10) + `riasec_code` (3 letras).

### Bloque B — Contexto y expectativas *(Etapa 2)*

> ⚠️ **Este bloque ya no produce pesos.** Con pesos fijos (§3.2), las respuestas de B no entran en
> la fórmula. Siguen siendo útiles para **conversar y explicar** el resultado, y para inferir el
> presupuesto del Bloque C, pero el agente no debe prometer que ajustará el ranking según ellas.

| # | Pregunta guía | Para qué sirve hoy |
|---|---|---|
| B1 | Al elegir una carrera, ¿qué pesa más para ti: el sueldo, lo que cuesta, que te apasione, o qué tan fácil es entrar? | Encuadrar la explicación del `score_breakdown` |
| B2 | ¿Estarías dispuesto a invertir más si la carrera te asegura mejores ingresos? | Inferir `max_annual_cost` (filtro C4) |
| B3 | ¿Qué tan importante es que te guste, aunque pague menos? | Contexto para el informe |
| B4 | ¿Te animarías a una carrera muy selectiva, o prefieres opciones más accesibles? | Contexto; la accesibilidad ya pesa 0.20 |
| B5 | ¿Qué tan importante es la duración de la carrera? | **Solo informativo** — la duración no puntúa (§3.3) |

### Bloque C — Filtros duros *(descartan opciones, no pesan)*

| # | Pregunta guía | Parámetro de `recommend_programs` | Columna |
|---|---|---|---|
| C1 | ¿En qué región quieres estudiar, o te da igual? | `region` | `location` (departamento, 25 valores) |
| C2 | ¿Universidad pública, privada o indistinto? | `management_type` | `management_type` |
| C3 | ¿Universidad, instituto o indistinto? | `institution_type` | `institution_type` |
| C4 | ¿Cuánto podrías pagar al año? | `max_annual_cost` | `annual_cost` |

> **Obligatorio al filtrar:** la respuesta trae `candidates_without_each_filter` — cuántos
> programas quedarían soltando cada filtro. Un filtro no da una respuesta mala, **borra opciones en
> silencio**. El agente debe decir «con tu presupuesto quedan 43 de los 411 de Arequipa» siempre
> que un filtro deje fuera a la mayoría, para que el estudiante pueda corregirlo.

---

## 6. Reglas de negocio (implementadas)

1. **Filtros primero:** los del Bloque C descartan antes del scoring. «Me da igual» / `"ambas"` /
   `"ambos"` / `None` = sin filtro.
2. **Filtro vacío = error explicativo:** si la combinación no deja candidatos, el error indica qué
   filtro soltar y cuántos programas aparecerían.
3. **Componente imputado = `NEUTRO` (0.5):** ver §3.1, regla 2.
4. **Una recomendación por carrera:** el ranking colapsa por carrera y muestra su mejor
   institución, para no llenar el Top-N con la misma carrera repetida.
5. **Desempate**, en este orden:
   1. `match_score` descendente;
   2. **menos cifras estimadas** (lo que sabemos antes que lo que estimamos);
   3. nombre de carrera alfabético;
   4. institución alfabética.
6. **Top-N:** `DEFAULT_TOP_N = 3`, `MAX_TOP_N = 10`.
7. **Salida:** cada recomendación trae la lista `estimated` con los campos que **no** son datos
   medidos de ese programa sino la mediana de su familia. **Deben presentarse como estimados o no
   mencionarse.** `match_score` es una puntuación propia del sistema, no un dato del MINEDU.

---

## 7. Esquema de datos

### 7.1 Campos del estudiante (perfil)

Soportados por `StudentProfile` (`src/models/profile.py`): 6 scores RIASEC, `riasec_code`,
`interests`, `strengths`, `preferred_fields`, `dislikes`, identidad básica, `target_career`,
`career_goals`.

Los filtros del Bloque C **no se persisten en el perfil**: viajan como argumentos de
`recommend_programs` en cada llamada.

### 7.2 Dataset de programas — `data/programs/programs.csv`

**Estado: ✅ existe.** 6.208 filas = carrera × institución. Fuente: Ponte en Carrera (MINEDU).

| Columna | Tipo | Uso |
|---|---|---|
| `source_id` | str | id de la combinación carrera–institución |
| `career` | str | nombre de carrera — título de la tarjeta |
| `career_family` | str | familia; base de la imputación por mediana |
| `riasec_profile` | str (3 letras) | **fórmula** — afinidad. Asignado por LLM (repo 05) |
| `institution` | str | universidad/instituto — subtítulo de la tarjeta |
| `institution_type` | enum | filtro C3 |
| `management_type` | enum | filtro C2 |
| `location` | str (25) | filtro C1 — **departamento**, no ciudad |
| `duration_years` | float | dato verificable — **no puntúa** (§3.3) |
| `monthly_income` | float (S/ /mes) | **fórmula** — `income` |
| `annual_cost` | float (S/ /año) | **fórmula** — `affordability` (invertido) + filtro C4 |
| `admission_rate` | float (%) | **fórmula** — `admission_accessibility` |
| `duration_measured` | bool | trazabilidad de imputación |
| `income_measured` | bool | si es `false` → componente = `NEUTRO` |
| `cost_measured` | bool | si es `false` → componente = `NEUTRO` |
| `admission_measured` | bool | si es `false` → componente = `NEUTRO` |

> **Unidades:** el ingreso es **mensual** y el costo es **anual** (así los muestra la UI); el
> presupuesto que declara el estudiante también es **anual**, y se compara contra `annual_cost`.
>
> **Nomenclatura:** las columnas están en **inglés**; el resto del documento usa español por
> legibilidad. Esta tabla es la equivalencia autoritativa.
>
> **Mapeo con el frontend:** `institutionType` (pública/privada) del frontend corresponde a
> `management_type` del CSV, y `academicType` (universidad/instituto) a `institution_type`. Los
> nombres están cruzados respecto a la intuición — verificarlo al conectar la UI.

---

## 8. Deltas respecto a `2_requirements.md` y `3_design.md`

| Tema | Docs SDD antiguos | Estado real (2026-09-01) |
|---|---|---|
| Modelo de perfil | Preferencias + confidence | **RIASEC** (6 scores + código de 3 letras) |
| LLM | Gemini | **AWS Bedrock (Claude)** — ver ADR-001 |
| Vector DB | FAISS / Chroma / Pinecone | **Sin vector store.** No hay RAG implementado; el catálogo se consulta como CSV en memoria |
| Base de datos | — | **RDS PostgreSQL** (`aws_db_instance`), no Aurora |
| Backend | FastAPI monolito | **Híbrido**: Lambda TypeScript (CRUD/EDA) + servicio Python en ECS Fargate (agente) — ver ADR-001 |
| Georreferencia | No contemplada | Filtro por `location` (departamento) sobre el CSV; sin mapa ni DynamoDB |

---

## 9. Estado de implementación (`spark-match-07-deep-agent`)

> Contraste entre este documento y lo que **está codificado**. Verificado el 2026-09-01 contra la
> rama `main` del repo del agente.

### 9.1 Implementado ✅

| Regla / concepto | Dónde vive |
|---|---|
| Scores RIASEC 1–10 + `riasec_code` (top-3) | `src/tools/assessment.py` → `evaluate_riasec_profile` |
| Afinidad con pesos posicionales **3-2-1**, normalizada contra el auto-match | `src/tools/matching/handler.py` → `_riasec_similarity` (devuelve 0–100) |
| **Fórmula multicriterio de 4 factores** (§3.1) | `src/tools/recommendation/scoring.py` → `score_program` |
| Rangos de referencia p5/p95 fijos | `scoring.py` → `REFERENCE_RANGES` |
| Componente imputado = `NEUTRO` 0.5 | `scoring.py` → `_componente` |
| Versionado del criterio de scoring | `scoring.py` → `SCORING_VERSION` |
| **Filtros duros** región / gestión / tipo / presupuesto (§5-C) | `src/tools/recommendation/handler.py` → `_construir_filtros` |
| Transparencia de filtros (`candidates_without_each_filter`) | `handler.py` |
| Desempate por cifras medidas + alfabético (§6.5) | `handler.py` → `_orden` |
| Top-N con tope (`3` / máx. `10`), una por carrera | `handler.py` → `DEFAULT_TOP_N`, `MAX_TOP_N` |
| Dataset real de 6.208 programas **con RIASEC** | `data/programs/programs.csv` (etiquetado en repo 05) |
| Marcado de cifras estimadas en la salida (`estimated`) | `handler.py` |

### 9.2 Diseñado pero NO implementado ❌

| Regla del documento | Estado real | Qué faltaría |
|---|---|---|
| **Pesos dinámicos por persona** (§2, §3.2) | Los pesos son constantes en `WEIGHTS` | Campos `w_*` en `StudentProfile`, extracción conversacional y paso de pesos a `score_program` |
| Gate `confidence ≥ 0.70` y máximo 4 repreguntas (§4) | Existen `profile_completeness` y `has_riasec_profile`, pero no el umbral ni el contador | Implementar el gate en el flujo del agente |
| Filtro por modalidad (presencial/virtual) | No existe la columna en el dataset | Añadirla en el pipeline (repo 05) antes de poder filtrar |

### 9.3 Decisiones cerradas (no son brechas)

- **La duración no puntúa** (§3.3) — deliberado, razonado en el código.
- **El Bloque B no produce pesos** (§5-B) — consecuencia de los pesos fijos.
- **No hay vector store ni RAG** — el catálogo cabe en memoria; el RAG es trabajo futuro.

---

## 10. Preguntas abiertas

1. ¿Se recuperan los **pesos por persona** (§9.2) o se dan por descartados y se cierra el diseño
   con pesos fijos? Es la única brecha de §9.2 que cambia el producto.
2. ¿Se ratifica `DEFAULT_TOP_N = 3` o el informe final debe entregar 5?
3. ¿Se añade `modalidad` al pipeline para habilitar el filtro?
4. Los **rangos de referencia** se calcularon el 2026-08-09 sobre el dataset actual. Si el pipeline
   vuelve a correr y cambian las distribuciones, ¿quién los recalcula y sube `SCORING_VERSION`?
5. El portal Ponte en Carrera está **dado de baja**: la ingesta está congelada. ¿Se busca fuente
   alternativa o se declara el dataset como corte fijo del proyecto?

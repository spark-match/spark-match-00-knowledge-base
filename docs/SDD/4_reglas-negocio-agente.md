# reglas-negocio-agente.md

> **Status**: 🟡 draft 
> **Fecha**: 2026-07-05
> **Audiencia**: AI Devs, Backend, Data Engineers
> Documento que fija el **flujo del agente** y las **reglas de negocio del scoring**, y reconcilia
> `2_requirements.md` y `3_design.md` en los puntos donde el diseño evolucionó (ver §8).

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

1. Conversa y **extrae** los datos necesarios para el scoring (perfil vocacional + prioridades + filtros).
2. **Razona** cruzando esos datos con el catálogo de carreras y datos oficiales.
3. Entrega un **artefacto/informe** (Top-N carreras + universidades) que el humano valida.

Esto significa que el scoring de pesos dinámicos (idea de Nikolai) sigue vigente: el agente es
quien alimenta sus variables de entrada de forma conversacional.

---

## 3. Modelo de scoring

### 3.1 Fórmula

```
score(carrera_universidad) = w_afinidad · afinidad_norm
                           + w_ingreso  · income_norm
                           + w_costo    · cost_norm        # ya invertido: mayor = más barato
                           + w_admision · admission_norm   # mayor = más fácil ingresar
                           + w_duracion · duration_norm    # ya invertido: mayor = más corta
```

- Todas las variables normalizadas a `[0, 1]`; el score resultante ∈ `[0, 1]`.
- **⚠️ No re-invertir:** `cost_norm` y `duration_norm` **ya vienen invertidos** desde el pipeline
  (`data-pipeline`, rama `dev`) mediante `1 − score`, y `admission_norm` ya está en sentido
  "mayor = más fácil". Se usan **tal cual** salen del `features.csv`; aplicar `(1 − …)` de nuevo
  sería un bug de doble inversión.
- **Normalización ya hecha:** `income_norm`, `cost_norm`, `admission_norm`, `duration_norm` vienen
  precalculadas en `features.csv` (ver §7.2). Solo falta `afinidad_norm` (RIASEC), que **aún no
  existe en el dataset** — ver §9.
- **Determinismo**: mismos inputs → mismo ranking (requisito heredado de `3_design.md`).

### 3.2 Pesos dinámicos

Los **5 pesos** son **por persona**:

- Cada `w ∈ [0, 1]`.
- **Suman 1** (se normalizan tras la conversación).
- **Pueden ser 0** (ej.: "el costo me da igual" → `w_costo = 0` y se re-normaliza el resto).

### 3.3 Origen de cada variable

| Variable (fórmula) | Columna en `features.csv` | Qué mide | Fuente de dato |
|---|---|---|---|
| `afinidad_norm` | *(no existe aún)* | Ajuste vocacional perfil ↔ carrera | Perfil RIASEC del estudiante × código RIASEC de la carrera — **falta añadir `riasec_profile` al dataset (§9)** |
| `income_norm` | `income_norm` | Atractivo del ingreso mensual del egresado (log1p + min-max) | Ponte en Carrera (MINEDU) → `features.csv` |
| `cost_norm` | `cost_norm` | Accesibilidad económica; **ya invertido** (mayor = más barato) | Ponte en Carrera / universidad |
| `admission_norm` | `admission_norm` | Facilidad de ingreso (mayor = más fácil) | Ponte en Carrera / universidad |
| `duration_norm` | `duration_norm` | Duración; **ya invertido** (mayor = más corta) | Ponte en Carrera / universidad |

> **Afinidad (RIASEC):** el agente ya calcula el código Holland de 3 letras del estudiante y lo
> compara con el `riasec_profile` de cada carrera (peso posicional 3-2-1). Ver
> `src/tools/matching.py` (`calculate_affinity`) en el POC `08-deep-agent`.
> **Nota de implementación:** hoy `calculate_affinity` devuelve el score en **porcentaje (0–100)**;
> para la fórmula de §3.1 hay que normalizarlo dividiendo entre 100. De los 4 términos de la
> fórmula, **solo `afinidad` está implementado**; ver el detalle en §9.

---

## 4. Flujo conversacional del agente

```
[Etapa 1] Exploración        → extrae PERFIL RIASEC (afinidad)
        │
[Etapa 2] Prioridades        → extrae los 5 PESOS dinámicos
        │
[Etapa 2] Filtros            → extrae FILTROS duros (región, tipo, presupuesto)
        │
        ▼
   Motor de scoring calcula ranking (Top-N) sobre carreras que pasan los filtros
        │
[Etapa 3] Validación humana  → el estudiante acepta o rechaza el informe
```

**Umbral de avance**: no se calcula ranking hasta tener los 6 scores RIASEC + al menos los 5 pesos
(equivalente al `confidence ≥ 0.70` de `3_design.md`). Máximo **4 repreguntas** antes de forzar el
ranking con la información disponible (degradación controlada).

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

### Bloque B — Prioridades → los 5 pesos *(Etapa 2)*

| # | Pregunta guía | Alimenta |
|---|---|---|
| B1 | Al elegir una carrera, ordena de más a menos importante: (a) el sueldo que podrías ganar, (b) lo que cuesta estudiarla, (c) que sea lo que te apasiona, (d) qué tan fácil es entrar, (e) cuánto dura la carrera. | ranking base de pesos |
| B2 | ¿Estarías dispuesto a invertir más si la carrera te asegura mejores ingresos? | relación costo ↔ ingreso |
| B3 | ¿Qué tan importante es que te guste, aunque pague menos? | afinidad vs ingreso |
| B4 | ¿Te animarías a una carrera muy selectiva/difícil de ingresar a la universidad/instituto, o prefieres opciones más accesibles? | tolerancia a admisión |
| B5 | ¿Qué tan importante es la duración de la carrera? | duración |

**Salida del bloque:** `w_afinidad, w_ingreso, w_costo, w_admision, w_duracion` (normalizados a suma 1).

### Bloque C — Filtros duros *(descartan opciones, no pesan)*

| # | Pregunta guía | Alimenta |
|---|---|---|
| C1 | ¿En qué región o ciudad quieres estudiar, o te da igual? | `region` (georreferencia) |
| C2 | ¿Universidad pública, privada o indistinto? | `tipo_gestion` |
| C3 | ¿Universidad , instituto o indistinto? | `tipo_institucion` |

---

## 6. Reglas de negocio (para implementación)

1. **Cálculo de pesos desde el ranking (B1):** asignar puntajes 5-4-3-2-1 según el orden de los
   5 factores, y dividir cada uno entre la suma → pesos que suman 1.
2. **Peso cero:** si el estudiante dice explícitamente "no me importa X", ese peso = 0 y se
   re-normaliza el resto.
3. **Defaults:** si no logra priorizar, usar 0.20 en cada peso (5 factores).
4. **Filtros:** los del Bloque C descartan carreras/universidades antes del scoring; si el
   estudiante responde "me da igual", ese filtro no se aplica.
5. **Umbral / repreguntas:** ranking solo con 6 RIASEC + 5 pesos presentes; máximo 4 repreguntas.
6. **Desempate:** ante scores iguales (diff < 0.001), orden alfabético por institución (heredado
   de `3_design.md`).
7. **Salida:** Top-N (default N=5) con datos verificables (ingreso mensual, costo anual, tasa de
   admisión, duración) y una explicación por recomendación. **Usar siempre las columnas `*_imputed`
   (soles/años/%), nunca las `*_norm`**, para explicar al usuario (ver diccionario de datos, repo 05).

---

## 7. Esquema de datos requerido

### 7.1 Campos del estudiante (perfil)

Ya soportados por el POC (`StudentProfile`): 6 scores RIASEC, `riasec_code`, `interests`,
`strengths`, `preferred_fields`, `dislikes`, identidad básica.

**Faltan (añadir):** `w_afinidad`, `w_ingreso`, `w_costo`, `w_admision`, `region`,
`tipo_gestion` (pública/privada), `tipo_institucion` (universidad/instituto),
`presupuesto_anual` (S/. / año — el filtro de presupuesto de la UI es **anual**), `modalidad`.

### 7.2 Campos por carrera/universidad → `features.csv` (repo 05, Nikolai — rama `dev`)

**Estado: ✅ el dataset ya existe** (`data/features.csv`, **6.208 filas** = carrera × institución,
fuente Ponte en Carrera). Esquema real (ver `Diccionario de datos.md` del repo 05):

| Columna real | Tipo | Uso en el sistema | Mapeo a la fórmula / UI |
|---|---|---|---|
| `id` | int | id combinación carrera–institución | — |
| `career_family` | str (81 cat.) | consulta/explicación | familia (ej. "Ingeniería Civil") |
| `career` | str (554 cat.) | consulta/explicación | nombre de carrera (título tarjeta) |
| `institution` | str (1071 cat.) | consulta/explicación | universidad/instituto (subtítulo tarjeta) |
| `location` | str (25 cat.) | filtro C1 | **departamento** (no ciudad) — filtro "Región" |
| `management_type` | enum (Pública/Privada) | filtro C2 | filtro "Tipo de universidad" |
| `institution_type` | enum (Universidad/Instituto) | filtro C3 | — (pendiente en UI) |
| `monthly_income_imputed` | float (S/./mes) | explicación | UI "Ingreso Mensual Prom." |
| `annual_cost_imputed` | float (S/./año) | explicación | UI "Costo Anual" |
| `admission_rate_imputed` | float (%) | explicación | UI "Tasa de Admisión" |
| `duration_years_imputed` | float (años) | explicación | UI "Duración" |
| `income_norm` | float [0,1] | **fórmula** (§3.1) | `income_norm` (log1p+min-max) |
| `cost_norm` | float [0,1] | **fórmula** (§3.1) | `cost_norm` (**invertido**: mayor = más barato) |
| `admission_norm` | float [0,1] | **fórmula** (§3.1) | `admission_norm` (mayor = más fácil) |
| `duration_norm` | float [0,1] | **fórmula** (§3.1) | `duration_norm` (**invertido**: mayor = más corta) |
| `*_imputed_flag` | bool | auditoría de calidad | trazabilidad de imputación |

> **Normalización:** ya está hecha en el pipeline (columnas `*_norm`), no hay que recalcularla.
> Para explicar al usuario úsense las `*_imputed` (soles/años/%), **nunca** las `*_norm`.
>
> **❗ Falta lo vocacional:** `features.csv` **no tiene `riasec_profile` por carrera**, así que el
> término `afinidad_norm` de §3.1 **no se puede calcular hoy** sobre estas 6.208 filas. Hay que
> añadir un mapeo carrera → código RIASEC al dataset (o hacer el join con el catálogo del POC). Ver §9.
>
> **Nomenclatura:** las columnas del CSV están en **inglés** (`career`, `location`,
> `management_type`, `institution_type`, `annual_cost`…); el resto de este doc usa nombres en
> español por legibilidad — esta tabla es la equivalencia autoritativa.

---

## 8. Deltas respecto a `2_requirements.md` y `3_design.md`

Puntos donde el diseño evolucionó y hay que reconciliar los documentos de David:

| Tema | Docs SDD actuales | Dirección acordada (este doc) |
|---|---|---|
| Modelo de perfil | Preferencias + confidence | **RIASEC** + prioridades para pesos |
| LLM | Gemini | **Bedrock (Claude)** — *pendiente ratificar (ver ADR-001)* |
| Vector DB | FAISS / Chroma / Pinecone | **pgvector (Aurora)** |
| Backend | FastAPI monolito | **Híbrido: Lambda (CRUD/EDA) + servidor Python dedicado (agente)** — ver ADR-001 |
| Georreferencia | No contemplada | DynamoDB para universidades + mapa |

---

## 9. Estado de implementación (POCs `08-deep-agent` y `05-data-pipeline`)

> Contraste entre las reglas de este documento y lo que **ya está codificado** en el agente
> (`08-deep-agent`, rama `main`) y en el pipeline de datos (`05-data-pipeline`, rama `dev`).
> Sirve para no asumir que el spec = implementación.

### 9.1 Ya implementado ✅

| Regla / concepto | Dónde vive en el código | Notas |
|---|---|---|
| Scores RIASEC 1–10 + derivación `riasec_code` (top-3) | `src/tools/assessment.py` → `evaluate_riasec_profile` | Ordena las 6 dimensiones desc. y toma las 3 mayores |
| Afinidad con pesos posicionales **3-2-1** | `src/tools/matching.py` → `_riasec_similarity` | `weights = [3.0, 2.0, 1.0]` |
| Cálculo del ranking Top-N por afinidad | `src/tools/matching.py` → `calculate_affinity` | `top_n=5` por defecto; ordena por score desc. |
| Catálogo de carreras (MVP) | `src/tools/catalog.py` → `CAREER_CATALOG` (10 carreras) | Campos: `id, name, riasec_profile, description, skills, field, outlook` |
| Búsqueda por texto/campo | `src/tools/catalog.py` → `search_careers` | Match de texto; **sin** filtros de descarte |
| Perfil del estudiante (langmem) | `src/models/profile.py` → `StudentProfile` | 6 RIASEC, `riasec_code`, `interests`, `strengths`, `preferred_fields`, `dislikes`, identidad, `target_career`, `career_goals` |
| Orquestación del ranking | `src/agent/subagents/matching.py` | Subagente `matching` (usa `search_careers` + `calculate_affinity`) |
| **Dataset económico + normalización** (repo 05) | `05-data-pipeline` rama `dev` → `data/features.csv` | 6.208 filas; `income/cost/admission/duration_norm` ya en `[0,1]`; imputación + flags de calidad |
| **Pipeline reproducible** (repo 05) | `05-data-pipeline` → `src/{ingestion,data_clean,feature_engineering}.py` | Snapshots versionados + `Diccionario de datos.md` |

### 9.2 Aún NO implementado (spec pendiente) ❌

| Regla del documento | Estado real | Qué falta |
|---|---|---|
| **Unir los dos mundos** (afinidad ↔ económico) | El agente tiene RIASEC pero no datos económicos; el `features.csv` tiene datos económicos pero **no RIASEC** | ❗ Añadir `riasec_profile` por carrera al dataset (mapeo/join) — bloquea la fórmula de 5 factores |
| Fórmula de scoring de 5 factores (§3.1) | El agente solo calcula `afinidad`; los 4 factores económicos existen como columnas `*_norm` pero no se combinan | Implementar la suma ponderada (servicio de scoring o tool extendida) sobre `features.csv` |
| Pesos dinámicos por persona `w_*` (§3.2) | No hay campos ni lógica; `calculate_affinity` no recibe pesos | 5 campos `w_*` en `StudentProfile` + normalización a suma 1 |
| Escala normalizada `[0,1]` de la afinidad (§3.1) | `calculate_affinity` devuelve **% (0–100)**; los factores económicos ya están en `[0,1]` | Normalizar la afinidad (÷100) al integrar la fórmula |
| Filtros duros del Bloque C (§4, §6.4) | `search_careers` no filtra; el `features.csv` sí tiene `location`/`management_type`/`institution_type` | Lógica de filtrado (los datos ya lo permiten) |
| Umbral `confidence ≥ 0.70` / máx. 4 repreguntas (§4) | El modelo tiene `profile_completeness` y `has_riasec_profile`, pero no un gate 0.70 ni contador de repreguntas | Implementar el gate en el flujo del agente |
| Desempate alfabético por institución (§6.6) | El orden es solo por score desc. | Añadir criterio de desempate |
| Catálogo del agente vs dataset real | El POC usa 10 carreras hardcodeadas con RIASEC; el dataset real tiene 6.208 filas sin RIASEC | Migrar el agente a consultar `features.csv` (vía pgvector en prod) |

> **Implicación:** el agente entrega ranking **vocacional (RIASEC)** sobre 10 carreras; el pipeline
> entrega **datos económicos + normalización** sobre 6.208 carreras. **Ninguno de los dos hace la
> fórmula de 5 factores todavía**, y el bloqueante principal es que los datos **no tienen RIASEC**.

---

## 10. Preguntas abiertas para la reunión

1. LLM definitivo: **Bedrock vs Gemini** (impacta créditos).
2. Fuente de datos única: **Ponte en Carrera vs SUNEDU (scraping) vs catálogo semilla**.
3. ¿El scoring corre dentro del agente (tool `calculate_affinity` extendida) o como servicio aparte?
4. N del Top-N para el informe final (¿3 o 5?).
5. **¿Cómo asignar el `riasec_profile` a las 6.208 filas de `features.csv`?** (mapeo por
   `career_family`, LLM que etiqueta, tabla semilla manual…) — es el bloqueante de la fórmula.
6. ¿Se mergea la rama `dev` de `05-data-pipeline` a `main`? (hoy el dataset solo vive en `dev`).

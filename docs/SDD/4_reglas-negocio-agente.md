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
                           + w_ingreso  · ingreso_norm
                           + w_costo    · (1 − costo_norm)
                           + w_admision · (1 − selectividad_norm)
```

- Todas las variables normalizadas a `[0, 1]`; el score resultante ∈ `[0, 1]`.
- **Determinismo**: mismos inputs → mismo ranking (requisito heredado de `3_design.md`).

### 3.2 Pesos dinámicos

Los 4 pesos son **por persona**:

- Cada `w ∈ [0, 1]`.
- **Suman 1** (se normalizan tras la conversación).
- **Pueden ser 0** (ej.: "el costo me da igual" → `w_costo = 0` y se re-normaliza el resto).

### 3.3 Origen de cada variable

| Variable | Qué mide | Fuente de dato |
|---|---|---|
| `afinidad_norm` | Ajuste vocacional perfil ↔ carrera | Perfil RIASEC del estudiante × código RIASEC de la carrera |
| `ingreso_norm` | Ingreso esperado del egresado | Ponte en Carrera (MINEDU) → `features.csv` |
| `costo_norm` | Costo de la carrera (mensualidad / ingreso) | Ponte en Carrera / catálogo universidad |
| `selectividad_norm` | Qué tan difícil es entrar (tasa de admisión) | Ponte en Carrera / dato universidad |

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
[Etapa 2] Prioridades        → extrae los 4 PESOS dinámicos
        │
[Etapa 2] Filtros            → extrae FILTROS duros (región, tipo, presupuesto)
        │
        ▼
   Motor de scoring calcula ranking (Top-N) sobre carreras que pasan los filtros
        │
[Etapa 3] Validación humana  → el estudiante acepta o rechaza el informe
```

**Umbral de avance**: no se calcula ranking hasta tener los 6 scores RIASEC + al menos los 4 pesos
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

### Bloque B — Prioridades → los 4 pesos *(Etapa 2)*

| # | Pregunta guía | Alimenta |
|---|---|---|
| B1 | Al elegir una carrera, ordena de más a menos importante: (a) el sueldo que podrías ganar, (b) lo que cuesta estudiarla, (c) que sea lo que te apasiona, (d) qué tan fácil es entrar. | ranking base de pesos |
| B2 | ¿Estarías dispuesto a invertir más si la carrera te asegura mejores ingresos? | relación costo ↔ ingreso |
| B3 | ¿Qué tan importante es que te guste, aunque pague menos? | afinidad vs ingreso |
| B4 | ¿Te animarías a una carrera muy selectiva/difícil de ingresar a la universidad/instituto, o prefieres opciones más accesibles? | tolerancia a admisión |
| B5 | ¿Que tan importante es la duracion de la carrera? | duracion |

**Salida del bloque:** `w_afinidad, w_ingreso, w_costo, w_admision` (normalizados a suma 1).

### Bloque C — Filtros duros *(descartan opciones, no pesan)*

| # | Pregunta guía | Alimenta |
|---|---|---|
| C1 | ¿En qué región o ciudad quieres estudiar, o te da igual? | `region` (georreferencia) |
| C2 | ¿Universidad pública, privada o indistinto? | `tipo_gestion` |
| C3 | ¿Universidad , instituto o indistinto? | `tipo_institucion` |

---

## 6. Reglas de negocio (para implementación)

1. **Cálculo de pesos desde el ranking (B1):** asignar puntajes 4-3-2-1 según el orden, y dividir
   cada uno entre la suma → pesos que suman 1.
2. **Peso cero:** si el estudiante dice explícitamente "no me importa X", ese peso = 0 y se
   re-normaliza el resto.
3. **Defaults:** si no logra priorizar, usar 0.25 en cada peso.
4. **Filtros:** los del Bloque C descartan carreras/universidades antes del scoring; si el
   estudiante responde "me da igual", ese filtro no se aplica.
5. **Umbral / repreguntas:** ranking solo con 6 RIASEC + 4 pesos presentes; máximo 4 repreguntas.
6. **Desempate:** ante scores iguales (diff < 0.001), orden alfabético por institución (heredado
   de `3_design.md`).
7. **Salida:** Top-N (default N=5) con datos verificables (ingreso, costo, tasa de admisión) y una
   explicación por recomendación.

---

## 7. Esquema de datos requerido

### 7.1 Campos del estudiante (perfil)

Ya soportados por el POC (`StudentProfile`): 6 scores RIASEC, `riasec_code`, `interests`,
`strengths`, `preferred_fields`, `dislikes`, identidad básica.

**Faltan (añadir):** `w_afinidad`, `w_ingreso`, `w_costo`, `w_admision`, `region`,
`tipo_institucion`, `presupuesto_max`, `modalidad`.

### 7.2 Campos por carrera/universidad → define `features.csv` (repo 05, Nikolai)

El catálogo actual del POC solo tiene `riasec_profile`, `skills`, `field`, `outlook`.
**No tiene ni un campo económico ni geográfico.** Para el scoring hacen falta:

| Campo | Tipo | Fuente | Uso |
|---|---|---|---|
| `career_id` / `career_name` | str | catálogo | id |
| `institution` | str | catálogo/SUNEDU | universidad |
| `riasec_profile` | str (3 letras) | catálogo | afinidad |
| `ingreso_promedio` | num (S/. / mes) | Ponte en Carrera | `ingreso_norm` |
| `costo_mensualidad` | num (S/.) | Ponte en Carrera / universidad | `costo_norm` |
| `tasa_admision` | num (%) | Ponte en Carrera / universidad | `selectividad_norm` |
| `region` | str | universidad / georef | filtro C1 |
| `tipo_institucion` | enum (pública/privada) | universidad | filtro C2 |
| `duracion_anios` | num | catálogo | dato verificable |

> Normalización sugerida: min-max por columna sobre el dataset → todo a `[0, 1]`.

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

## 9. Estado de implementación en el POC (`08-deep-agent`)

> Contraste entre las reglas de este documento y lo que **ya está codificado** en el POC
> `08-deep-agent` (rama `main`). Sirve para no asumir que el spec = implementación.

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

### 9.2 Aún NO implementado (spec pendiente) ❌

| Regla del documento | Estado real en el código | Qué falta |
|---|---|---|
| Fórmula de scoring completa (§3.1) | Solo existe el término `afinidad`; los 3 términos económicos/admisión no | Añadir `ingreso`, `costo`, `selectividad` al cálculo |
| Pesos dinámicos por persona `w_*` (§3.2) | No hay campos ni lógica; `calculate_affinity` no recibe pesos | Campos en `StudentProfile` + extender el tool o crear servicio de scoring |
| Escala normalizada `[0,1]` de la afinidad (§3.1) | `calculate_affinity` devuelve **% (0–100)** | Normalizar (÷100) al integrar la fórmula |
| Filtros duros del Bloque C (§4, §6.4) | `search_careers` no descarta por `region`/`tipo`/`presupuesto` | Lógica de filtrado previa al scoring |
| Umbral `confidence ≥ 0.70` / máx. 4 repreguntas (§4) | El modelo tiene `profile_completeness` y `has_riasec_profile`, pero no un gate 0.70 ni contador de repreguntas | Implementar el gate en el flujo del agente |
| Desempate alfabético por institución (§6.6) | El orden es solo por score desc. | Añadir criterio de desempate |
| `features.csv` con campos económicos/geográficos (§7.2) | No existe; el `05-data-pipeline` aún no tiene datos | Construir el dataset (repo 05) |

> **Implicación:** hoy el POC entrega un ranking **puramente vocacional (RIASEC)**. Las etapas 2–3
> (pesos, filtros, datos económicos) del flujo de §4 son diseño pendiente de implementar.

---

## 10. Preguntas abiertas para la reunión

1. LLM definitivo: **Bedrock vs Gemini** (impacta créditos).
2. Fuente de datos única: **Ponte en Carrera vs SUNEDU (scraping) vs catálogo semilla**.
3. ¿El scoring corre dentro del agente (tool `calculate_affinity` extendida) o como servicio aparte?
4. N del Top-N para el informe final (¿3 o 5?).

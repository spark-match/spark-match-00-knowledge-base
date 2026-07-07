# design.md — CareerMatch Perú (Consolidado, foco AWS)

> **⚠️ PENDIENTE: Definir fuente de verdad del stack backend.**  
> El backend de Angel (rama `feature/scaffolding-fase-1`) usa **TypeScript + Lambda + Identity completo (register/login/JWT)**, mientras que este documento asume **Python + Fargate + sesión anónima**. Ambos coexisten en el repo. **NO editar las secciones de Auth/JWT de este documento** hasta que el equipo decida qué stack adoptar. Ver `OBSERVACIONES_tasks.md` § Alineación de stack para contexto.

## 1. Descripción General

**CareerMatch Perú** es un sistema de recomendación de carreras universitarias que opera mediante un asistente conversacional basado en LLM, integrado con un motor de evaluación multi-criterio determinístico y auditable. El flujo principal es:

1. Usuario inicia sesión autenticada vía JWT.
2. Expresa preferencias en lenguaje natural a través de interfaz web conversacional.
3. El **LLM_Layer** (Amazon Bedrock/Claude) interpreta preferencias y detecta información incompleta en 3 bloques:
   - **Bloque A**: RIASEC (6 dimensiones Holland)
   - **Bloque B**: Pesos dinámicos (5 criterios: afinidad, ingreso, costo, admisión, duración)
   - **Bloque C**: Filtros duros (región, tipo institución, presupuesto, modalidad)
4. Si falta información, el sistema genera preguntas contextuales iterativamente hasta alcanzar confianza ≥ 0.70 o máximo 4 repreguntas.
5. Una vez interpretadas, el **Scoring_Engine** calcula indicadores de concordancia multi-criterio para todas las combinaciones carrera–universidad, ordena por score descendente y retorna **Top-5**.
6. El **LLM_Layer** redacta explicaciones personalizadas.
7. El usuario valida con escala Likert y la sesión se persiste en **Feedback_Storage** para análisis futuro.

**Énfasis en:**
- **Determinismo**: Cualquier input idéntico genera ranking idéntico en 100% de ejecuciones.
- **Reproducibilidad**: Snapshots versionados permiten auditar exactamente qué datos y configuración se usó.
- **Aislamiento de datos** por `session_id`.
- **Degradación controlada**: Si componentes fallan, el sistema retorna la mejor respuesta posible con información disponible.
- **Arquitectura modular AWS**: Para la demo, un solo servicio FastAPI sirve todos los endpoints (`/chat`, `/feedback`, `/universities`). La arquitectura Lambda + Fargate + API Gateway queda como **meta de producción** (ver ADR-001 en sección 2.2).

---

## 2. Arquitectura de Componentes

### 2.1 Diagrama de Alto Nivel

> **Demo (alcance MVP):** Un solo servicio FastAPI sirve todos los endpoints.
> **Producción (meta, ver ADR-001):** Lambda + Fargate + API Gateway como se describe abajo.

```
─── Demo (un solo servicio FastAPI) ──────────────────────────────────
Cliente (browser)
    ↓ (HTTP) 
localhost:8000
    ├→ /chat        (FastAPI → Orchestrator → LLM/Scoring)
    ├→ /feedback    (FastAPI → FeedbackStorage)
    ├→ /universities (FastAPI → DynamoDB / stub local)
    └→ /health      (FastAPI health check)

FastAPI Service (single container):
    Orchestrator 
      ├→ LLM_Layer (Bedrock/Claude-3.5-Sonnet)
      ├→ Session_Manager (in-memory, TTL=1800s)
      ├→ Scoring_Engine (determinístico, en-process)
      ├→ Auth_Service (session_id anónimo en-memoria)
      ├→ Feedback_Storage (PostgreSQL local)
      └→ RAG_Module (opcional, con pgvector)

Persistencia:
    └→ PostgreSQL local (simula Aurora): feedback, rankings, pgvector

─── Producción (meta post-demo, ver ADR-001) ─────────────────────────
Cliente (browser)
    ↓ (HTTP)
API Gateway
    ├→ Lambda: /session/create, /feedback, /universities       [stateless, CRUD]
    └→ VPC Link → ALB → Fargate: /chat                         [stateful, agente]

Fargate Service (Deep Agent):
    Orchestrator 
      ├→ LLM_Layer (Bedrock/Claude-3.5-Sonnet)
      ├→ Session_Manager (in-memory, TTL=1800s)
      ├→ Scoring_Engine (determinístico, en-process)
      └→ RAG_Module (opcional)
           └→ Embedding_Service (Bedrock Titan Embeddings)
                └→ pgvector (Aurora PostgreSQL)

Persistencia:
    ├→ Aurora PostgreSQL: feedback, rankings, pgvector (career_chunks)
    └→ DynamoDB: universities (georreferencia, GSI location-index)

Pipeline de Datos (batch, EventBridge cron → AWS Batch/Fargate task):
    Ingestion (Selenium, Ponte en Carrera)
      → Data Clean
      → Feature Engineering (normalización MinMax)
      → Extraer carreras únicas (554 distinct `career`, NO las 6.208 filas)
      → RIASEC Tagging sobre unique careers (Bedrock few-shot)
      → Join RIASEC tags contra features.csv por `career`
      → features.csv (snapshot versionado, ahora con columna `riasec_profile`)
```

### 2.2 Arquitectura para la Demo (un solo FastAPI)

Para el MVP/demo, todo corre en un **solo servicio FastAPI** (contenedor Docker). Esto evita la complejidad operativa de Lambda + API Gateway + VPC Link, acelerando el desarrollo iterativo.

| Aspecto | Demo (alcance MVP) | Producción (meta, ADR-001) |
|---|---|---|
| Compute | 1 contenedor FastAPI (Fargate o local) | Lambda (stateless) + Fargate (stateful) |
| Endpoints | `/chat`, `/feedback`, `/universities` todo en FastAPI | API Gateway → Lambda (/session, /feedback, /universities) + VPC Link → Fargate (/chat) |
| Sesión | `session_id` anónimo en memoria | JWT sin estado compartido |
| Persistencia | PostgreSQL local (o Aurora si disponible) | Aurora PostgreSQL + DynamoDB |
| Frontend | Sirve desde FastAPI (static files) o CDN | API Gateway + CloudFront |
| Despliegue | `docker-compose up` | AWS SAM / Terraform multi-stack |

### 2.2.b Producción: Lambda + Fargate (meta post-demo)

Cuando se requiera escalar a producción, la arquitectura objetivo es:

El loop conversacional requiere:
- Múltiples llamadas a Bedrock por turno.
- Mantener contexto en memoria entre turnos de la misma sesión.
- Posible duración variable (4+ turnos).

Lambda no es adecuado: timeout corto (15 min), stateless forzado, costo por llamada. **Fargate proporciona:**
- Contenedor Python persistente.
- Memoria en-process ilimitada (para sesiones).
- Long polling / WebSocket support (para chat fluido).

**Stateless (Lambda) mantiene:**
- `/session/create` → JWT
- `/feedback` → validaciones
- `/universities` → lectura de GIS

> **Decisión (ADR-001):** Para la demo se prioriza velocidad de desarrollo sobre separación de servicios. La migración a Lambda+Fargate se hará post-demo si la carga lo justifica. Ver `OBSERVACIONES_tasks.md` § Simplificaciones.

---



## 3. Requisitos (Trazabilidad)

| Req | Criterio | Implementación |
|---|---|---|
| **R1 — RIASEC** | 1.1 Captura conversacional de 6 scores | Bloque A, LLM interpreta |
| | 1.2 Cálculo de `riasec_code` (3 letras) | `calculate_riasec_code` |
| | 1.3 Captura opcional: interests, strengths | `StudentProfile.interests` |
| **R2 — Pesos** | 2.1 Captura de 5 pesos via orden (B1) | `calculate_weights_from_ranking` |
| | 2.2 Peso explícito en 0 + re-normalización | `validate_weights` |
| | 2.3 Defaults 0.2 si no se logra priorizar | Fallback en Orchestrator |
| **R3 — Filtros** | 3.1 Filtros C descartan antes scoring | `rank_and_filter` fase 1 |
 | | 3.2 "Me da igual" desactiva filtro | `profile.location = None` |
| **R4 — Confianza** | 4.1 Fórmula `calculate_confidence` | 11 campos, [0,1] |
| | 4.2 Avance si confidence >= 0.70 | `should_transition_to_recommendation` |
| | 4.3 Máximo 4 repreguntas | `turn_count >= 4` fuerza ranking |
| **R5 — Scoring** | 5.1 Fórmula 5 criterios | `calculate_score` |
| | 5.2 Determinismo | Mismo input → mismo ranking 100% |
| | 5.3 Desempate alfabético | `diff < 0.001` → `institution` ASC |
| | 5.4 Top-5 con datos verificables | `get_top_n(ranked_list, 5)` |
| **R6 — Datos** | 6.1 Schema `features.csv` | `CareerFeature` modelo |
| | 6.2 Etiquetado Bedrock few-shot | `tag_careers_with_bedrock` |
| | 6.3 Validación muestral 300 carreras | `validate_sample` → CSV |
| | 6.4 Fallback familia | `apply_family_fallback` |
| **R7 — LLM** | 7.1 Interpretación Bloques A/B/C | `LLMLayer.interpret_*` |
| | 7.2 Validación JSON 3 reintentos | `_parse_and_validate_json` |
| | 7.3 Generación explicaciones | `generate_explanation` |
| **R8 — Backend** | 8.1 Lambda `/session`, `/feedback`, `/universities` | 3 handlers |
| | 8.2 Fargate `/chat` + SessionContext memoria | `Orchestrator.handle_turn` |
| | 8.3 JWT sin estado compartido | `Auth_Service` + validación local |
| **R9 — Persistencia** | 9.1 Aurora: feedback, rankings aislados | `FeedbackRecord` + índices |
| | 9.2 pgvector Aurora: career_chunks | `career_chunks` tabla |
| | 9.3 DynamoDB: universities + GSI location-index | `universities` tabla |
| **R10 — No funcionales** | 10.1 Scoring 6k+ combos en ≤ 1s | Benchmark test |
| | 10.2 Aislamiento por `session_id` | Queries filtrando siempre |
| | 10.3 Fallback templado si fallan componentes | Degradación controlada |
| | 10.4 Logging sin PII | No tokens completos, nombres OK |

---

## Restricciones Transversales

- **Package manager:** `uv` (no Poetry/Conda, no pip puro).
- **Framework Fargate:** FastAPI + Uvicorn.
- **Framework Lambda:** `boto3` puro (sin frameworks adicionales).
- **Cobertura tests:** ≥ 60% en `scoring`, `llm_service`, `auth`, `data_pipeline`.
- **Credenciales:** Variables de entorno / AWS Secrets Manager (nunca hardcodeadas).
---

## 6. Módulos Detallados

### `backend/embedding.py` — Embedding_Service (Amazon Bedrock Titan)

**Responsabilidad:** Generar embeddings de texto mediante Amazon Bedrock Titan Embeddings (agnóstico de lógica, solo pasathrough a API).

**API Pública:**

```python
class EmbeddingService:
    def __init__(
        self,
        provider: str = "bedrock",  # FIXED: solo 'bedrock' en este ciclo
        model: str = "amazon.titan-embed-text-v2",
        region: str = "us-east-1",  # Región AWS (default)
    ):
        """
        Inicializa servicio de embeddings con Amazon Bedrock.
        
        Args:
            provider: 'bedrock' (único proveedor en MVP)
            model: 'amazon.titan-embed-text-v2' (modelo Bedrock Titan)
            region: Región AWS donde ejecutar
        
        Comportamiento:
        - Inicializa cliente Bedrock (`boto3.client("bedrock-runtime")`)
        - Valida conectividad con Bedrock
        
        Errores:
        - BotoClientError si credenciales no disponibles o región inválida
        """
    
    def embed(self, text: str) -> list[float]:
        """
        Genera embedding de texto (1024 dimensiones).
        
        Args:
            text: Texto a embedder (en español o inglés)
        
        Returns:
            Vector numérico: list[float] de 1024 elementos
        
        Timeout:
        - Máximo 5 segundos
        
        Comportamiento:
        - Llama a Bedrock Titan Embeddings API
        - Retorna vector normalizado
        
        Errores:
        - APITimeoutError: si timeout > 5s
        - BotoClientError: si API falla
        - En ambos casos: retorna None (manejado por caller para degradación)
        """
    
    def embed_batch(self, texts: list[str]) -> list[list[float]] | None:
        """
        Genera embeddings para múltiples textos (optimización batch).
        
        Args:
            texts: Lista de textos
        
        Returns:
            Lista de vectores (mismo orden que inputs), o None si falla
        
        Comportamiento:
        - Agrupa textos en batches si API lo permite
        - Más eficiente que embed() llamado iterativamente
        - Si falla cualquier texto, loguea warning pero continúa
        
        Timeout:
        - Máximo 10 segundos total
        """
    
    def similarity(self, vec1: list[float], vec2: list[float]) -> float:
        """
        Calcula similitud coseno entre dos vectores.
        
        Args:
            vec1, vec2: Vectores [0, ..., 1023]
        
        Returns:
            float [0.0, 1.0]
        
        Comportamiento:
        - Operación local (no llamada a API)
        - Si dimensiones no coinciden, raise ValueError
        """
```

**Notas de comportamiento:**
- EmbeddingService es **inyectable** en RAG_Module (para pgvector search).
- Para afinidad RIASEC, **no se usa embedding**; se usa peso posicional directo (ver `### Algoritmos` → `#### Algoritmo 2: Cálculo de Afinidad RIASEC`).
- Si Bedrock falla, degradación controlada: RAG desactivado, flujo principal sin cambios.

---

### `backend/llm_service.py` — LLM_Layer (Amazon Bedrock Claude-3.5-Sonnet)

**Responsabilidad:** Interpretar preferencias en lenguaje natural mediante Claude, detectar información incompleta, generar preguntas y explicaciones.

**API Pública:**

```python
class LLMService:
    def __init__(
        self,
        provider: str = "bedrock",  # FIXED: solo 'bedrock' en este ciclo
        model: str = "anthropic.claude-3-5-sonnet-20241022",
        region: str = "us-east-1",
    ):
        """
        Inicializa servicio LLM con Amazon Bedrock/Claude.
        
        Args:
            provider: 'bedrock' (único)
            model: Modelo Claude (default: Claude-3.5-Sonnet)
            region: Región AWS
        
        Comportamiento:
        - Inicializa cliente Bedrock
        - Carga system prompts v1 (mínimo 3 ejemplos few-shot por método)
        """
    
    def interpret_bloque_a(
        self,
        message: str,
        conversation_history: list[dict],
    ) -> dict:
        """
        Interpreta preferencias RIASEC desde entrada natural (Bloque A).
        
        Args:
            message: Entrada del usuario en español
            conversation_history: Historial para continuidad
        
        Returns:
            {
                "riasec_scores": {
                    "R": int [1,10],  # Realista
                    "I": int [1,10],  # Investigador
                    "A": int [1,10],  # Artístico
                    "S": int [1,10],  # Social
                    "E": int [1,10],  # Empresarial
                    "C": int [1,10]   # Convencional
                },
                "missing_letters": list[str],  # Letras sin estimar
                "confidence": float [0,1]
            }
        
        Comportamiento:
        - Usa system prompt de few-shot con 3+ ejemplos de perfiles RIASEC
        - Extrae scores parciales (puede no tener los 6)
        - JSON validado; si inválido (3 reintentos), retorna estructura templada
        
        Timeout:
        - Máximo 5 segundos
        
        Ejemplos en prompt:
            "Si el usuario dice 'Me encantan los números y resolver problemas', 
             entonces: R=8, I=9, A=3, S=4, E=6, C=7"
        """
    
    def interpret_bloque_b(
        self,
        message: str,
        conversation_history: list[dict],
    ) -> dict:
        """
        Interpreta prioridades de criterios (Bloque B).
        
        Args:
            message: Entrada del usuario
            conversation_history: Historial
        
        Returns:
            {
                "priority_order": list[str] | None,  
                    # Ejemplo: ["afinidad", "ingreso", "costo", "admision", "duracion"]
                    # Orden de prioridad descendente (1er elemento = más importante)
                    # None si el usuario aún no prioriza
                "explicit_weights": dict | None,
                    # Alternativa: {w_afinidad: 0.3, w_ingreso: 0.2, ...}
                    # None si se usa order en lugar de pesos explícitos
                "confidence": float [0,1]
            }
        
        Comportamiento:
        - Detecta si usuario da orden explícita ("lo más importante es X, luego Y")
        - O detecta pesos implícitos ("me interesa ingresos pero costo no"
        - Retorna order si es claro; weights si es explícito; None si indeciso
        - JSON validado; fallback templado si falla
        
        Timeout:
        - Máximo 5 segundos
        """
    
    def interpret_bloque_c(
        self,
        message: str,
        conversation_history: list[dict],
    ) -> dict:
        """
        Interpreta filtros duros (Bloque C).
        
        Args:
            message: Entrada del usuario
            conversation_history: Historial
        
        Returns:
            {
                "location": str | None,  
                    # Región específica o None (= "me da igual")
                "management_type": "publica" | "privada" | None,
                "presupuesto_max": float | None,  # En soles
                "modalidad": "presencial" | "virtual" | None,
                "confidence": float [0,1]
            }
        
        Comportamiento:
        - Detecta si usuario menciona "me da igual" para algún filtro → None
        - Detecta ubicación: "Lima", "Arequipa", etc. → normaliza
        - Detecta tipo gestión: "pública", "privada"
        - Detecta presupuesto: números mencionados
        - Detecta modalidad: "online", "virtual", "presencial"
        
        Timeout:
        - Máximo 5 segundos
        """
    
    def generate_follow_up(
        self,
        profile: StudentProfile,
        missing_info: list[str],
        conversation_history: list[dict],
    ) -> str:
        """
        Genera pregunta abierta para completar información faltante.
        
        Args:
            profile: Perfil parcial actual
            missing_info: Lista de campos que faltan (ej ["afinidad_E", "w_ingreso"])
            conversation_history: Historial para evitar repeticiones
        
        Returns:
            Pregunta conversacional en español peruano
        
        Restricciones (Requerimiento 4):
        - DEBE ser pregunta abierta (no SÍ/NO)
        - DEBE mencionar por qué es útil la información
        - NO DEBE repetir lo ya mencionado en historial
        - DEBE ser contextual al perfil parcial
        
        Ejemplos correctos:
            "Veo que te interesan las matemáticas. ¿Hay algún aspecto de 
             una carrera que sea especialmente importante para ti? Por ejemplo, 
             algunos priorizan los ingresos futuros, otros el costo, y otros 
             la facilidad de admisión."
        
        Ejemplos incorrectos:
            "¿Cuál es tu región?" (directa, no abierta)
            "Dijiste que te gustan mates, ¿y qué más?" (repite lo ya dicho)
        
        Timeout:
        - Máximo 3 segundos
        
        Comportamiento:
        - Usa few-shot de preguntas naturales
        - Si no hay missing_info claro, retorna string vacío
        """
    
    def generate_explanation(
        self,
        ranking_position: int,  # 1, 2, ..., 5
        career: str,
        institution: str,
        concordancia_score: float,  # [0,1]
        scores_by_criterion: dict,  # {afinidad, ingreso, costo, admision, duracion}
        user_profile: StudentProfile,
        verifiable_data: dict,  # {monthly_income_imputed, annual_cost_imputed, admission_rate_imputed, duration_years_imputed}
    ) -> str:
        """
        Redacta explicación personalizada de recomendación.
        
        Args:
            ranking_position: Posición en ranking (1–5)
            career, institution: Nombres de carrera y universidad
            concordancia_score: Score final [0,1]
            scores_by_criterion: Desglose por criterio
            user_profile: Perfil del usuario
            verifiable_data: Datos oficiales del MINEDU
        
        Returns:
            Texto de 3–5 oraciones en español peruano
        
        Contenido obligatorio:
        - Qué criterio/característica coincide con perfil
        - Al menos 1 dato verificable (ingresos, costo, tasa admisión)
        - Conexión beneficio-usuario
        
        Restricción: NO inventa datos; todos vienen de verifiable_data
        
        Ejemplo correcto:
            "Te recomendamos Estadística en la UNMSM porque tu afinidad 
             con el análisis de datos es muy alta (85/100). Esta carrera 
             ofrece uno de los mejores ingresos (S/. 3,200/mes) según datos 
             oficiales del MINEDU. Aunque es selectiva (18% de admisión), 
             es factible con tu fortaleza en matemáticas."
        
        Fallback (si LLM falla):
            "Recomendamos [carrera] en [universidad] porque alinea bien 
             con tus intereses (score: [score]). Ingresos promedio: S/. [X]/mes. 
             Costo: S/. [Y]/año. Tasa de admisión: [Z]%."
        
        Timeout:
        - Máximo 3 segundos
        
        Errores:
        - Si LLM falla, retorna fallback templated (no excepción)
        """
    
    def validate_weights(self, weights: dict) -> dict:
        """
        Valida que pesos sean válidos (todos en [0,1], suma = 1.0 ± 0.01).
        
        Args:
            weights: {w_afinidad, w_ingreso, w_costo, w_admision, w_duracion}
        
        Returns:
            Weights validados y normalizados (si suma ≠ 1, re-normaliza)
        
        Comportamiento:
        - Si suma < 0.99 o > 1.01: divide cada peso entre suma actual
        - Soporta pesos en 0 explícitos (re-normaliza el resto)
        - Todos pesos quedan en [0,1]
        """
```

**Notas de comportamiento:**
- Todas las respuestas JSON son validadas con **máx 3 reintentos**; si falla, fallback templado.
- System prompts incluyen mínimo 3 ejemplos few-shot por método (definidos en `prompts/` subdirectorio).
- `LLM_PROVIDER=bedrock` y `LLM_MODEL=anthropic.claude-3-5-sonnet-20241022` en `.env`.

---

### `backend/scoring.py` — Scoring_Engine

**Responsabilidad:** Calcular concordancia multi-criterio determinística, generar rankings ordenados.

**API Pública:**

```python
class ScoringEngine:
    def __init__(
        self,
        features_csv_path: str,
        config_json_path: str,
    ):
        """
        Inicializa motor de scoring.
        
        Args:
            features_csv_path: Ruta a data/features.csv
            config_json_path: Ruta a data/feature_config.json
        
        Comportamiento:
        - Carga features.csv en memoria (Pandas DataFrame)
        - Carga config.json (parámetros de normalización)
        - Valida esquema (columnas de features.csv): id, career, institution, 
          riasec_profile (agregado vía join desde tabla RIASEC de 554 carreras únicas),
          income_norm, cost_norm, admission_norm, 
          duration_norm, location, management_type, y las 4 imputadas:
          monthly_income_imputed, annual_cost_imputed, admission_rate_imputed, duration_years_imputed
        
        Errores:
        - FileNotFoundError: si archivos no existen
        - ValueError: si esquema incorrecto
        """
    
    def calculate_riasec_code(self, riasec_scores: dict[str, int]) -> str:
        """
        Calcula código RIASEC (3 letras) desde 6 scores.
        
        Args:
            riasec_scores: {"R": 8, "I": 9, "A": 7, "S": 6, "E": 5, "C": 4}
        
        Returns:
            Código de 3 letras, ej "IRA"
        
        Lógica:
        - Ordena 6 scores descendente
        - Toma las 3 primeras letras
        - Empates resueltos por orden alfabético
        """
    
    def calculate_affinity(
        self,
        student_code: str,
        career_profile: str,
    ) -> float:
        """
        Calcula afinidad RIASEC mediante peso posicional.
        
        Args:
            student_code: "RIA" (perfil estudiante)
            career_profile: "RIA" (perfil carrera)
        
        Returns:
            float [0, 1]
        
        Fórmula:
        - Posición 1 en student_code: +3 puntos
        - Posición 2: +2 puntos
        - Posición 3: +1 punto
        - No aparece: 0 puntos
        
        score_crudo = suma de puntos para cada letra de career_profile
        afinidad_norm = score_crudo / 6.0  (máximo posible: 3+2+1 = 6)
        
        Determinismo:
        - Mismo input → mismo output en todas las ejecuciones
        """
    
    def calculate_score(
        self,
        weights: dict,  # {w_afinidad, w_ingreso, w_costo, w_admision, w_duracion}
        afinidad: float,  # [0,1]
        ingreso_norm: float,  # [0,1]
        costo_norm: float,  # [0,1] — mayor = más barato (ya invertido en pipeline)
        admission_norm: float,  # [0,1] — mayor = más fácil acceso
        duracion_norm: float,  # [0,1] — mayor = más corta (ya invertido en pipeline)
    ) -> float:
        """
        Calcula concordancia score de una carrera.
        
        Fórmula:
            score = w_afinidad * afinidad
                  + w_ingreso * ingreso_norm
                  + w_costo * costo_norm
                  + w_admision * admission_norm
                  + w_duracion * duracion_norm
        
        Returns:
            float [0, 1]
        
        Notas:
        - Todas las normas ya están orientadas [0,1] con "mayor = mejor para el estudiante"
        - El pipeline ya invierte costo y duración, y orienta admisión como facilidad
        - Scoring NO aplica inversiones adicionales
        """
    
    def rank_and_filter(
        self,
        profile: StudentProfile,
        features_df: pd.DataFrame,
    ) -> list[dict]:
        """
        Genera ranking completo aplicando filtros y scoring.
        
        Args:
            profile: Perfil estudiante con RIASEC, pesos, filtros
            features_df: Dataset de carreras-universidades
        
        Returns:
            List de dicts ordenado por score DESC:
            [
                {
                    "rank": 1,
                    "id": "estadistica_001",
                    "career": "Estadística",
                    "institution": "UNMSM",
                    "concordancia_score": 0.847,
                    "scores_by_criterion": {
                        "afinidad": 0.85,
                        "ingreso": 0.82,
                        "costo": 0.90,
                        "admision": 0.60,
                        "duracion": 0.75
                    },
                    "verifiable_data": {
                        "monthly_income_imputed": 3200,
                        "annual_cost_imputed": 100,
                        "admission_rate_imputed": 18,
                        "duration_years_imputed": 5
                    }
                },
                ...
            ]
        
        Lógica (7 pasos):
        1. Calcula RIASEC code desde profile.riasec_scores
        2. Para cada fila en features_df:
           a. Aplica filtros duros (C: región, tipo inst, presupuesto, modalidad)
           b. Si usuario dijo "me da igual", ignora ese filtro
           c. Si supera filtros: calcula afinidad RIASEC
           d. Calcula score final
        3. Ordena resultados por score DESC
        4. Desempate (abs(score_a - score_b) < 0.001): orden alfabético por institution
        5. Asigna rank
        6. Retorna completo (sin truncar)
        
        Fallback (degradación):
        - Si features_df vacío: raise FileNotFoundError explícito
        - Si algún campo normalizador falta: logea warning, asume 0.5
        - Si riasec_profile de carrera falta: afinidad = 0.5
        
        Timeout:
        - Para dataset ~6,208 carreras: ≤ 1 segundo
        """
    
    def get_top_n(
        self,
        ranked_list: list[dict],
        n: int = 5,
    ) -> list[dict]:
        """
        Extrae Top-N de ranking completo.
        
        Args:
            ranked_list: Resultado de rank_and_filter
            n: Número de resultados (default 5, rango [1,10])
        
        Returns:
            Primeros N elementos de ranked_list
        
        Validación:
        - Si n < 1: clampea a 1
        - Si n > 10: clampea a 10
        """
    
    def reload_features(self) -> None:
        """
        Recarga features.csv desde disco sin reiniciar servidor.
        
        Comportamiento:
        - Lee features.csv nuevamente
        - Valida schema
        - Reemplaza dataset en memoria
        
        Errores:
        - FileNotFoundError, ValueError: propagados a caller
        """
```

**Notas de comportamiento:**
- Determinismo absoluto: mismo input → mismo ranking en 100% ejecuciones.
- Scoring es en-process (sin llamadas a APIs externas).
- Dataset cargado en memoria al startup; para datasets > 100MB, considerar carga lazy.

---

### `backend/rag.py` — RAG_Module (pgvector/Aurora)

**Responsabilidad:** Permitir consultas conversacionales sobre detalles de carreras mediante recuperación + generación.

**API Pública:**

```python
class RAGModule:
    def __init__(
        self,
        db_url: str,  # Conexión Aurora PostgreSQL con pgvector
        embedding_service: EmbeddingService,
    ):
        """
        Inicializa módulo RAG.
        
        Args:
            db_url: URI de PostgreSQL (ej psycopg://user:pass@host/db)
            embedding_service: Instancia de EmbeddingService (Bedrock Titan)
        
        Comportamiento:
        - Conecta a Aurora PostgreSQL
        - Valida que tabla career_chunks existe
        - Valida que extensión pgvector está activa
        """
    
    def is_career_available(self, career: str) -> bool:
        """
        Verifica si carrera tiene RAG disponible.
        
        Args:
            career: Nombre de carrera (ej "Estadística")
        
        Returns:
            True si chunks indexados en pgvector, False en otro caso
        
        Comportamiento:
        - Query a PostgreSQL: SELECT COUNT(*) FROM career_chunks WHERE career_id = ?
        - Búsqueda rápida sin embeddings
        """
    
    def query(
        self,
        career: str,
        question: str,
        top_k: int = 3,
    ) -> dict:
        """
        Realiza consulta RAG sobre detalles de carrera.
        
        Args:
            career: Nombre de carrera
            question: Pregunta en español (ej "¿Qué cursos lleva?")
            top_k: Número de chunks a recuperar (default 3)
        
        Returns:
            {
                "status": "success" | "no_documents" | "error",
                "answer": str,
                "sources": list[str],
                "confidence": float [0,1]
            }
        
        Lógica:
        1. Verifica si career disponible vía is_career_available()
           → Si no: retorna status="no_documents"
        2. Genera embedding de question vía EmbeddingService.embed()
           → Si falla: retorna status="error" (RAG no bloquea flujo)
        3. Busca Top-K chunks similares en pgvector:
           SELECT content, metadata FROM career_chunks 
           WHERE career_id = ?
           ORDER BY embedding <=> ? LIMIT K
        4. Pasa chunks + question a LLMService.generate_explanation()
           (re-usa mismo LLM para generar respuesta)
        5. Extrae y retorna fuentes desde metadata
        
        Timeout:
        - Máximo 3 segundos (similarity search + LLM)
        
        Degradación:
        - Si embedding falla: retorna status="error", answer="No hay información..."
        - Si LLM falla: retorna status="error"
        - RAG nunca bloquea el flujo principal (/chat)
        """
    
    def index_career_documents(
        self,
        career: str,
        documents: list[str],
        metadata_dict: dict | None = None,
    ) -> None:
        """
        Indexa documentos de una carrera en pgvector.
        
        Args:
            career: Nombre de carrera
            documents: Textos ya procesados (no PDFs crudos)
            metadata_dict: {plan_curricular: bool, perfil_profesional: bool, ...}
        
        Comportamiento:
        - Divide textos en chunks (~500 caracteres, 50 char overlap)
        - Genera embeddings vía EmbeddingService.embed_batch()
        - Inserta en career_chunks con metadata:
          INSERT INTO career_chunks (career_id, content, embedding, metadata)
          VALUES (?, ?, ?, ?)
        
        Notas:
        - Llamada durante setup/importación de datos (offline)
        - Para demo: solo 3–5 carreras piloto
        
        Timeout:
        - Máximo 30 segundos (para ~10 documentos)
        """
    
    def get_available_careers(self) -> list[str]:
        """
        Retorna lista de carreras con RAG disponible.
        
        Returns:
            List de nombres de carrera (ej ["Estadística", "Ingeniería"])
        
        Comportamiento:
        - Query: SELECT DISTINCT career_id FROM career_chunks
        """
```

**Notas de comportamiento:**
- RAG es **opcional**: si no disponible para carrera, sistema continúa sin error.
- Vector DB es **pgvector en Aurora PostgreSQL** (no FAISS, Pinecone, Chroma).
- EmbeddingService es inyectable: permite cambiar proveedor sin modificar RAG_Module.

---

### `backend/feedback.py` — Feedback_Storage (Aurora PostgreSQL)

**Responsabilidad:** Persistir consultas, rankings, explicaciones y validaciones en Aurora PostgreSQL, construyendo histórico.

**API Pública:**

```python
class FeedbackStorage:
    def __init__(self, db_url: str):
        """
        Inicializa almacenamiento de feedback en Aurora.
        
        Args:
            db_url: URI PostgreSQL (ej psycopg://user:pass@host/careermatch)
        
        Comportamiento:
        - Conecta a Aurora PostgreSQL
        - Crea tablas si no existen (feedback, rankings)
        - Valida schema
        """
    
    def save_ranking(
        self,
        session_id: str,
        user_input: str,
        profile_interpreted: dict,
        weights_generated: dict,
        ranking_generated: list[dict],
        timestamp: datetime,
    ) -> str:
        """
        Guarda ranking generado (sin validación usuario aún).
        
        Args:
            session_id: ID única de sesión
            user_input: Consulta original del usuario
            profile_interpreted: Perfil interpretado (JSON)
            weights_generated: Pesos dinámicos
            ranking_generated: Top-5 ranking (JSON)
            timestamp: Fecha/hora de generación
        
        Returns:
            ranking_id: Identificador único para vincular feedback posterior
        
        Comportamiento:
        - Inserta en tabla rankings:
          INSERT INTO rankings 
          (session_id, ranking_json, reproducibility_metadata, created_at)
          VALUES (?, ?, ?, ?)
        
        ranking_json contiene: {profile, weights, top_5_items}
        reproducibility_metadata contiene: {snapshot_id, config_id, timestamp}
        
        - Genera ranking_id único: uuid4()
        - Retorna ranking_id
        
        Timeout:
        - Máximo 5 segundos (incluye retry si DB temporalmente no disponible)
        
        Fallback (degradación):
        - Si DB no disponible: logea error, almacena en queue local en-memory
        - Retorna ranking_id provisional
        - En siguiente oportunidad, sincroniza queue a DB
        """
    
    def update_validation(
        self,
        ranking_id: str,
        validation_score: int,  # [1, 5]
        selected_career: str | None = None,
        notes: str | None = None,
        timestamp: datetime | None = None,
    ) -> None:
        """
        Actualiza ranking con validación del usuario.
        
        Args:
            ranking_id: ID del ranking a actualizar
            validation_score: Escala Likert 1–5
            selected_career: Carrera elegida por usuario (opcional)
            notes: Comentarios libres (opcional)
            timestamp: Fecha de validación (auto si None)
        
        Comportamiento:
        - Localiza ranking por ranking_id en tabla rankings
        - Actualiza campos en JSONB ranking_json:
          ranking_json['user_validation'] = {
              score: validation_score,
              selected_career: selected_career,
              notes: notes,
              timestamp: timestamp
          }
        - UPDATE rankings SET ranking_json = ? WHERE id = ?
        
        Validaciones:
        - validation_score ∈ [1, 5]; si fuera: ValueError
        - ranking_id debe existir; si no: ValueError
        
        Timeout:
        - Máximo 5 segundos (incluye retry)
        
        Fallback (degradación):
        - Si DB no disponible: queue local, sincroniza después
        """
    
    def get_ranking_by_id(self, ranking_id: str) -> dict | None:
        """
        Recupera ranking completo (para reproducibilidad, auditoría).
        
        Args:
            ranking_id: ID del ranking
        
        Returns:
            Dict con todos los campos del ranking, o None si no existe
        
        Comportamiento:
        - SELECT ranking_json FROM rankings WHERE id = ?
        - Retorna JSONB completo deserializado
        
        Notas:
        - No expone datos de otros usuarios (aislamiento garantizado por WHERE session_id)
        """
    
    def query_feedback_aggregated(
        self,
        filter_career: str | None = None,
        filter_region: str | None = None,
        date_range: tuple[datetime, datetime] | None = None,
    ) -> list[dict]:
        """
        Consulta agregada de feedback para análisis (sin PII).
        
        Args:
            filter_career: Carrera específica (None = todas)
            filter_region: Región (None = todas)
            date_range: Rango de fechas (None = todas)
        
        Returns:
            List de registros agregados:
            [
                {
                    "career": "Estadística",
                    "institution": "UNMSM",
                    "validation_scores": [4, 5, 3, 4],
                    "avg_validation": 4.0,
                    "count": 4
                },
                ...
            ]
        
        Comportamiento:
        - NO expone datos individuales de usuarios
        - Agrupa por (career, institution)
        - Calcula estadísticas desde ranking_json['user_validation']['score']
        - Query:
          SELECT 
              ranking_json->'career' AS career,
              ranking_json->'institution' AS institution,
              COUNT(*) AS count,
              AVG((ranking_json->'user_validation'->>'score')::int) AS avg_validation,
              ... GROUP BY career, institution
        
        Usado para análisis de calidad posterior
        """
```

**Notas de comportamiento:**
- Aislamiento: queries filtran implícitamente por `session_id` en contexto de seguridad.
- Fallback: si DB no disponible, queue local persiste en memoria hasta sincronizar.
- Schema: índices en (session_id, ranking_id, career) para queries rápidas.

---

### `backend/orchestration.py` — Orchestration_Layer

**Responsabilidad:** Orquestar flujo completo: autenticación → interpretación (Bloques A/B/C) → evaluación → explicación → persistencia. Manejar errores y fallback.

**API Pública:**

```python
class Orchestration:
    def __init__(
        self,
        auth_service,
        session_manager,
        llm_service,
        scoring_engine,
        rag_module,
        feedback_storage,
    ):
        """Inyecta todas las dependencias del sistema."""
    
    def handle_chat(
        self,
        session_token: str,
        message: str,
        metadata: dict | None = None,
    ) -> dict:
        """
        Maneja solicitud de chat completa (entrypoint principal).
        
        Args:
            session_token: Token de sesión (validado por HTTP layer)
            message: Mensaje del usuario en español
            metadata: Filtros opcionales {location, budget_max (presupuesto anual), management_type}
        
        Returns:
            ChatResponse:
            {
                "status": "success" | "error",
                "conversation": {
                    "turn": int,
                    "bot_message": str,
                    "requires_follow_up": bool
                },
                "ranking": {
                    "status": "ready" | "awaiting_info",
                    "top_5": [
                        {
                            "rank": 1,
                            "id": str,
                            "career": str,
                            "institution": str,
                            "concordancia_score": float,
                            "monthly_income_imputed": float,
                            "annual_cost_imputed": float,
                            "admission_rate_imputed": float,
                            "duration_years_imputed": float,
                            "explanation": str
                        },
                        ...
                    ]
                },
                "rag_available_for": list[str],
                "ranking_id": str | None,
                "error_message": str | None
            }
        
        Flujo detallado:
        
        1. VALIDAR TOKEN
           auth_service.validate_token(session_token)
           → Si inválido: retorna {status: "error", error_message: "Token inválido"}
        
        2. RECUPERAR SESIÓN
           context = session_manager.get_context(session_id)
           → Si no existe: crear nuevo vacío
           → Agregar message a conversation_history
        
        3. DETERMINAR BLOQUE ACTIVO
           - Si riasec_scores incompleto: BLOQUE A
           - Elif pesos no priorizados: BLOQUE B
           - Elif filtros no definidos: BLOQUE C
           - Else: proceder a SCORING
        
        4. INTERPRETAR BLOQUE ACTIVO
           Si BLOQUE A:
               result_a = llm_service.interpret_bloque_a(message, history)
               Actualizar profile.riasec_scores con result_a
               Recalcular profile.riasec_code
           
           Si BLOQUE B:
               result_b = llm_service.interpret_bloque_b(message, history)
               Si result_b['priority_order']: 
                   weights = calculate_weights_from_ranking(result_b['priority_order'])
                   Actualizar profile.w_*, validar con validate_weights()
               Elif result_b['explicit_weights']:
                   Actualizar profile.w_*, validar
           
           Si BLOQUE C:
               result_c = llm_service.interpret_bloque_c(message, history)
                Actualizar profile.location, management_type, presupuesto_max, modalidad
        
        5. ACTUALIZAR CONTEXTO Y CALCULAR CONFIANZA
           session_manager.update_profile(session_id, nuevos_datos)
           confidence = session_manager.calculate_confidence(profile)
           Incrementar turn_count
        
        6. EVALUAR AVANCE
           
           IF confidence >= 0.70:
               → Proceder a SCORING
           
           ELIF turn_count >= 4:
               → Proceder a SCORING (degradación controlada)
               → Loguar {graceful_degradation: true}
           
           ELSE:
               → Generar pregunta seguimiento
               → Retornar {status: "success", requires_follow_up: true, bot_message: "¿...?"}
        
        7. SCORING (si avanza)
           
           a. Calcular afinidades:
              affinity_scores = [
                  scoring_engine.calculate_affinity(profile.riasec_code, carrera['riasec_profile'])
                  for carrera in features_df
              ]
           
           b. Calcular ranking completo:
              ranked = scoring_engine.rank_and_filter(profile, features_df)
           
           c. Extraer Top-5:
              top_5 = scoring_engine.get_top_n(ranked, 5)
           
           d. Generar explicaciones:
              for item in top_5:
                  explanation = llm_service.generate_explanation(
                      rank=item['rank'],
                      career=item['career'],
                      institution=item['institution'],
                      score=item['concordancia_score'],
                      scores_by_criterion=item['scores_by_criterion'],
                      user_profile=profile,
                      verifiable_data=item['verifiable_data']
                  )
                  item['explanation'] = explanation
           
           e. Persistir:
              ranking_id = feedback_storage.save_ranking(
                  session_id,
                  user_input=message,
                  profile_interpreted=profile.to_dict(),
                  weights_generated=profile.weights_dict(),
                  ranking_generated=top_5,
                  timestamp=now()
              )
           
            f. Obtener carreras con RAG:
               rag_available = [
                   item['career'] for item in top_5
                   if rag_module.is_career_available(item['career'])
               ]
           
           g. Retornar:
              {
                  "status": "success",
                  "conversation": {bot_message: "Te recomendamos...", requires_follow_up: false},
                  "ranking": {status: "ready", top_5: top_5},
                  "rag_available_for": rag_available,
                  "ranking_id": ranking_id
              }
        
        Manejo de errores:
        - LLM interpreta inválido (3 reintentos): continúa con lo que tiene, incrementa turn_count
        - Scoring falla (features_df vacío): retorna error HTTP 500
        - Embedding falla (para afinidad): afinidad = 0.5 (fallback)
        - Feedback persist falla: queue local, continúa
        
        Timeout:
        - Total máximo 8 segundos (incluye LLM, scoring, persistencia)
        """
    
    def handle_feedback(
        self,
        session_token: str,
        ranking_id: str,
        validation_score: int,
        selected_career: str | None = None,
        notes: str | None = None,
    ) -> dict:
        """
        Maneja validación del usuario.
        
        Args:
            session_token: Token validado
            ranking_id: ID del ranking a validar
            validation_score: Escala Likert 1–5
            selected_career, notes: Opcionales
        
        Returns:
            {
                "status": "success" | "error",
                "message": str
            }
        
        Lógica:
        1. Validar score ∈ [1, 5]
           → Si no: retorna {status: "error", message: "Score inválido"}
        
        2. Verificar que ranking_id pertenece a session_id
           (security: usuario no puede validar ranking de otro)
           ranking = feedback_storage.get_ranking_by_id(ranking_id)
           → Si ranking.session_id ≠ session_id: error
        
        3. Persistir validación:
           feedback_storage.update_validation(
               ranking_id,
               validation_score,
               selected_career,
               notes,
               timestamp=now()
           )
        
        4. Retornar éxito
        """
    
    def handle_rag_query(
        self,
        session_token: str,
        career: str,
        question: str,
    ) -> dict:
        """
        Maneja consulta RAG sobre detalles de carrera.
        
        Args:
            session_token: Token validado
            career: Nombre de carrera recomendada
            question: Pregunta del usuario
        
        Returns:
            {
                "status": "success" | "no_documents" | "error",
                "answer": str,
                "sources": list[str],
                "confidence": float
            }
        
        Lógica:
        1. Verificar que career está en ranking previo del usuario
           (security: usuario no puede consultar carreras random)
           session_context = session_manager.get_context(session_id)
           → Si ranking_history vacío o career no está: error
        
        2. Invocar RAG:
           response = rag_module.query(career, question, top_k=3)
        
        3. Retornar respuesta (incluso si status="no_documents")
        """
```

**Notas de comportamiento:**
- Orchestration **NO lanza excepciones** al caller; retorna respuestas estructuradas con status.
- Errores loguedos completamente (para auditoría, sin PII).
- Fallbacks permiten degradación controlada en cada paso.

---

### `backend/app.py` — FastAPI Entrypoint (Fargate)

```python
from fastapi import FastAPI, HTTPException, Depends, Request
from fastapi.responses import JSONResponse
import logging

app = FastAPI(title="CareerMatch Perú Agent API")
logger = logging.getLogger("careermatch")

# Variable global (inicializada en startup)
orchestration = None

def get_session_token(request: Request) -> str:
    """Extrae session_token del header Authorization."""
    auth_header = request.headers.get("Authorization", "")
    if not auth_header.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Authorization header requerido")
    return auth_header[7:]

@app.post("/chat")
async def endpoint_chat(
    request: dict,  # {message, metadata}
    session_token: str = Depends(get_session_token),
) -> dict:
    """
    POST /chat
    
    Delega a orchestration.handle_chat() y retorna respuesta.
    HTTP 200 siempre (status en JSON), 401 si token inválido.
    """
    try:
        response = orchestration.handle_chat(
            session_token,
            request.get("message"),
            request.get("metadata")
        )
        return response
    except Exception as e:
        logger.error(f"Error en /chat: {e}", exc_info=True)
        return {"status": "error", "error_message": "Error interno del servidor"}

@app.post("/feedback")
async def endpoint_feedback(
    request: dict,  # {ranking_id, validation_score, ...}
    session_token: str = Depends(get_session_token),
) -> dict:
    """
    POST /feedback
    
    Delega a orchestration.handle_feedback().
    HTTP 422 si validation_score fuera de rango.
    """
    try:
        score = request.get("validation_score")
        if not (1 <= score <= 5):
            raise HTTPException(status_code=422, detail="Score debe estar entre 1 y 5")
        
        response = orchestration.handle_feedback(
            session_token,
            request.get("ranking_id"),
            score,
            request.get("selected_career"),
            request.get("notes")
        )
        return response
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error en /feedback: {e}", exc_info=True)
        return {"status": "error", "message": "Error interno"}

@app.post("/rag")
async def endpoint_rag(
    request: dict,  # {career, question}
    session_token: str = Depends(get_session_token),
) -> dict:
    """
    POST /rag
    
    Delega a orchestration.handle_rag_query().
    """
    try:
        response = orchestration.handle_rag_query(
            session_token,
            request.get("career"),
            request.get("question")
        )
        return response
    except Exception as e:
        logger.error(f"Error en /rag: {e}", exc_info=True)
        return {"status": "error", "answer": "", "sources": []}

@app.on_event("startup")
async def startup():
    """Inicializa componentes al levantarse el servidor."""
    global orchestration
    
    # Inicializar servicios...
    # (código de inicialización)
    
    logger.info("Orquestador iniciado correctamente")
```

---

### `frontend/app.js` — Gestor de Aplicación (JavaScript)

```javascript
class CareerMatchApp {
    constructor() {
        this.sessionToken = null;
        this.baseUrl = "http://localhost:8000"; // O URL de servidor
    }

    async init() {
        this.sessionToken = localStorage.getItem("session_token");
        if (!this.sessionToken) {
            // Demo: crear sesión anónima local (sin Lambda)
            // En producción, se usaría POST /session/create vía Lambda
            await this.createSessionStub();
        }
        this.attachEventListeners();
    }

    async createSessionStub() {
        // Stub: generar token local para demo
        this.sessionToken = "token_" + Math.random().toString(36).substr(2, 9);
        localStorage.setItem("session_token", this.sessionToken);
    }

    async sendMessage(message) {
        if (!message.trim()) return;
        
        this.appendMessage(message, "user");
        
        try {
            const response = await fetch(`${this.baseUrl}/chat`, {
                method: "POST",
                headers: {
                    "Authorization": `Bearer ${this.sessionToken}`,
                    "Content-Type": "application/json"
                },
                body: JSON.stringify({
                    message: message,
                    metadata: {}
                })
            });
            
            const data = await response.json();
            
            if (data.status === "success") {
                this.appendMessage(data.conversation.bot_message, "bot");
                
                if (data.ranking.status === "ready") {
                    this.rankingId = data.ranking_id;
                    this.displayRanking(data.ranking.top_5);
                }
            } else {
                this.showError(data.error_message);
            }
        } catch (error) {
            console.error("Error:", error);
            this.showError("Error al procesar tu consulta");
        }
    }

    displayRanking(topN) {
        // Renderizar Top-5 ranking con datos verificables
        // Incluir botones Likert para feedback
    }

    appendMessage(text, role) {
        // Agregar mensaje al chat
    }

    attachEventListeners() {
        // Event listeners para botones
    }
}
```

---

### `data_pipeline/ingestion.py` — Descarga Automatizada

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import shutil
from pathlib import Path
from datetime import datetime
import logging

class DataIngestion:
    def __init__(
        self,
        minedu_url: str = "https://ponte.minedu.gob.pe",
        output_dir: str = "data",
        snapshots_dir: str = "snapshots"
    ):
        """Inicializa ingestion con Selenium."""
        self.minedu_url = minedu_url
        self.output_dir = Path(output_dir)
        self.snapshots_dir = Path(snapshots_dir)
        self.logger = logging.getLogger(__name__)
    
    def download(self) -> str:
        """
        Descarga archivo Excel de Ponte en Carrera.
        
        Returns:
            Ruta a data/raw.xlsx
        
        Timeout: ≤ 120 segundos total
        """
        options = webdriver.ChromeOptions()
        options.add_argument("--no-sandbox")
        prefs = {"download.default_directory": str(self.output_dir)}
        options.add_experimental_option("prefs", prefs)
        
        driver = webdriver.Chrome(options=options)
        
        try:
            driver.get(self.minedu_url)
            wait = WebDriverWait(driver, 30)
            
            # Clic en "Buscar"
            btn_buscar = wait.until(EC.element_to_be_clickable((By.ID, "btnBuscar")))
            btn_buscar.click()
            
            # Esperar descarga
            import time
            time.sleep(5)  # Espera a que descargue
            
            # Clic en "Descargar"
            btn_descargar = wait.until(
                EC.element_to_be_clickable((By.ID, "descargarDondeEstudioExcel"))
            )
            btn_descargar.click()
            
            # Esperar archivo
            time.sleep(10)
            raw_file = self.output_dir / "raw.xlsx"
            if raw_file.exists():
                # Crear snapshot
                self.create_snapshot(str(raw_file))
                self.logger.info(f"Descarga completada: {raw_file}")
                return str(raw_file)
            else:
                raise FileNotFoundError("Descarga no completó")
        
        finally:
            driver.quit()
    
    def create_snapshot(self, source_file: str) -> str:
        """Crea snapshot versionado."""
        timestamp = datetime.utcnow().strftime("%Y%m%d_%H%M%S")
        snapshot_path = self.snapshots_dir / "raw" / f"raw_{timestamp}.xlsx"
        snapshot_path.parent.mkdir(parents=True, exist_ok=True)
        shutil.copy(source_file, snapshot_path)
        self.logger.info(f"Snapshot creado: {snapshot_path}")
        return str(snapshot_path)
```

---

### `data_pipeline/data_clean.py` — Limpieza

```python
import pandas as pd
import logging

class DataCleaner:
    def __init__(self, raw_file: str):
        """Carga archivo Excel (header=6)."""
        self.df = pd.read_excel(raw_file, header=6)
        self.logger = logging.getLogger(__name__)
    
    def standardize_columns(self) -> pd.DataFrame:
        """Renombra columnas del Excel crudo a snake_case estándar.
        
        Columnas de salida (features.csv real):
          id, career, institution, career_family,
          location, management_type, institution_type,
          monthly_income, annual_cost, admission_rate, duration_years
        """
        mapping = {
            "N°": "id",
            "Carrera": "career",
            "Universidad": "institution",
            "Familia Carrera": "career_family",
            "Ubicación": "location",
            "Gestión": "management_type",       # Pública/Privada
            "Tipo Institución": "institution_type",  # Universidad/Instituto (NO es el filtro)
            "Ingreso Mensual": "monthly_income",
            "Costo Anual": "annual_cost",
            "Tasa Admisión": "admission_rate",
            "Duración (años)": "duration_years",
        }
        self.df = self.df.rename(columns=mapping)
        return self.df
    
    def remove_empty_rows(self) -> pd.DataFrame:
        """Elimina filas incompletas críticas."""
        initial_count = len(self.df)
        self.df = self.df.dropna(subset=["career", "institution"])
        removed = initial_count - len(self.df)
        self.logger.info(f"Filas removidas: {removed}")
        return self.df
    
    def convert_types(self) -> pd.DataFrame:
        """Convierte columnas numéricas."""
        numeric_cols = ["duration_years", "monthly_income", "annual_cost", "admission_rate"]
        for col in numeric_cols:
            if col in self.df.columns:
                self.df[col] = pd.to_numeric(self.df[col], errors="coerce")
        return self.df
    
    def clean(self) -> pd.DataFrame:
        """Ejecuta pipeline completo."""
        return (self
                .standardize_columns()
                .remove_empty_rows()
                .convert_types())
    
    def save(self, output_path: str) -> None:
        """Guarda CSV (UTF-8-sig)."""
        self.df.to_csv(output_path, index=False, encoding="utf-8-sig")
        self.logger.info(f"Guardado: {output_path}")
```

---

### `data_pipeline/feature_engineering.py` — Normalización

```python
import pandas as pd
import numpy as np
import json
import logging
from pathlib import Path
from datetime import datetime

class FeatureEngineer:
    def __init__(self, cleaned_csv: str, config_path: str = "data/feature_config.json"):
        """Inicializa ingeniero de features."""
        self.df = pd.read_csv(cleaned_csv)
        self.config_path = config_path
        self.logger = logging.getLogger(__name__)
        self.config = self._load_config()
    
    def _load_config(self) -> dict:
        """Carga o crea config de imputación."""
        if Path(self.config_path).exists():
            with open(self.config_path, "r") as f:
                return json.load(f)
        return {
            "duration_years_fallback": 4.0,
            "monthly_income_fallback": 2500.0,
            "annual_cost_fallback": 1000.0,
            "admission_rate_fallback": 30.0
        }
    
    def create_imputation_flags(self) -> pd.DataFrame:
        """Crea flags para variables a imputar.
        
        Columnas crudas de entrada: monthly_income, annual_cost, admission_rate, duration_years.
        Columnas imputadas de salida: monthly_income_imputed, annual_cost_imputed, 
        admission_rate_imputed, duration_years_imputed.
        """
        self.df["duration_imputed_flag"] = (
            (self.df["duration_years"] <= 0) |
            (self.df["duration_years"] > 10) |
            (self.df["duration_years"].isna())
        ).astype(int)
        
        self.df["income_imputed_flag"] = (
            (self.df["monthly_income"] <= 0) |
            (self.df["monthly_income"].isna())
        ).astype(int)
        
        # ... más flags
        
        return self.df
    
    def hierarchical_imputation(self) -> pd.DataFrame:
        """Imputación en cascada (familia → fallback)."""
        for col in ["duration_years", "monthly_income", "annual_cost", "admission_rate"]:
            # Nivel 1: mediana por family
            family_medians = self.df.groupby("career_family")[col].median()
            
            # Nivel 2: fallback desde config
            fallback = self.config[f"{col}_fallback"]
            
            self.df[f"{col}_imputed"] = self.df[col].fillna(
                self.df["career_family"].map(family_medians).fillna(fallback)
            )
        
        return self.df
    
    def validate_ranges(self) -> pd.DataFrame:
        """Valida y ajusta rangos."""
        self.df["duration_years_imputed"] = self.df["duration_years_imputed"].clip(3, 7)
        self.df["admission_rate_imputed"] = self.df["admission_rate_imputed"].clip(0, 90)
        return self.df
    
    def normalize_features(self) -> pd.DataFrame:
        """Genera 4 variables normalizadas [0,1] (mayor = mejor para el estudiante).
        
        Convención de nombres (features.csv real):
          income_norm, cost_norm, admission_norm, duration_norm
        Las columnas imputadas de entrada son:
          monthly_income_imputed, annual_cost_imputed, admission_rate_imputed, duration_years_imputed
        """
        # income_norm = MinMax(log1p(income))  — mayor ingreso = mejor (no invertir)
        self.df["income_norm"] = np.log1p(self.df["monthly_income_imputed"])
        self.df["income_norm"] = (
            (self.df["income_norm"] - self.df["income_norm"].min()) /
            (self.df["income_norm"].max() - self.df["income_norm"].min())
        )
        
        # cost_norm = 1 - MinMax(log1p(cost))  — mayor = más barato (invertir)
        cost_log = np.log1p(self.df["annual_cost_imputed"])
        self.df["cost_norm"] = (
            1 - ((cost_log - cost_log.min()) / (cost_log.max() - cost_log.min()))
        )
        
        # duration_norm = 1 - MinMax(duration)  — mayor = más corta (invertir)
        duration = self.df["duration_years_imputed"]
        self.df["duration_norm"] = (
            1 - ((duration - duration.min()) / (duration.max() - duration.min()))
        )
        
        # admission_norm = MinMax(admission)  — mayor = más fácil (no invertir)
        admission = self.df["admission_rate_imputed"]
        self.df["admission_norm"] = (
            (admission - admission.min()) / (admission.max() - admission.min())
        )
        
        return self.df
    
    def save_features(self, output_path: str) -> None:
        """Guarda features a CSV."""
        self.df.to_csv(output_path, index=False, encoding="utf-8-sig")
        self.logger.info(f"Features guardadas: {output_path}")
    
    def save_config(self) -> None:
        """Persiste config JSON."""
        with open(self.config_path, "w") as f:
            json.dump(self.config, f, indent=2)
    
    def create_snapshot(self) -> None:
        """Genera snapshots versionados."""
        timestamp = datetime.utcnow().strftime("%Y%m%d_%H%M%S")
        
        features_snapshot = Path("snapshots/features") / f"features_{timestamp}.csv"
        features_snapshot.parent.mkdir(parents=True, exist_ok=True)
        self.df.to_csv(features_snapshot, index=False, encoding="utf-8-sig")
        
        config_snapshot = Path("snapshots/configs") / f"config_{timestamp}.json"
        config_snapshot.parent.mkdir(parents=True, exist_ok=True)
        with open(config_snapshot, "w") as f:
            json.dump(self.config, f, indent=2)
    
    def run_pipeline(self) -> pd.DataFrame:
        """Ejecuta pipeline de imputación y normalización.
        
        NOTA: RIASEC tagging es un paso SEPARADO que opera sobre las 554
        carreras únicas (no sobre las 6.208 filas). Después de este pipeline,
        se extraen unique careers → se etiquetan con Bedrock → se hace
        join por `career` contra features_df para agregar `riasec_profile`.
        """
        return (self
                .create_imputation_flags()
                .hierarchical_imputation()
                .validate_ranges()
                .normalize_features())
```

---

### Modelos de Datos

### `StudentProfile` (Bloques A, B, C)

```python
class StudentProfile:
    """Perfil interpretado del estudiante tras conversación (Bloques A/B/C)."""
    
    # Bloque A — RIASEC (Etapa 1: gathering_info)
    riasec_scores: dict[str, int]  # {"R": 8, "I": 9, "A": 7, "S": 6, "E": 5, "C": 4} [1,10]
    riasec_code: str | None  # 3 letras, ej "RIA", calculado desde riasec_scores
    
    # Bloque B — pesos dinámicos (Etapa 2), suman 1.0
    w_afinidad: float     # [0,1], peso criterio afinidad RIASEC
    w_ingreso: float      # [0,1], peso criterio ingresos esperados
    w_costo: float        # [0,1], peso criterio costo
    w_admision: float     # [0,1], peso criterio selectividad admisión
    w_duracion: float     # [0,1], peso criterio duración en años
    
    # Bloque C — filtros duros (no pesan, descartan)
    location: str | None                # Región/ubicación del estudiante (None = "me da igual")
    management_type: str | None         # 'publica' | 'privada' | None (tipo de gestión, columna features.csv)
    presupuesto_max: float | None       # Máximo costo anual (soles)
    modalidad: str | None               # 'presencial' | 'virtual' | None
    
    # Datos adicionales contextuales
    interests: list[str]        # Áreas de interés mencionadas (ej ["matemáticas", "análisis"])
    strengths: list[str]        # Fortalezas percibidas (ej ["lógica", "resolución problemas"])
    preferred_fields: list[str] # Campos específicos preferidos (ej ["data science"])
    dislikes: list[str]         # Áreas de rechazo (ej ["memorización"])
    
    # Estado del perfil
    confidence_score: float  # [0,1], calculado por calculate_confidence()
    confidence_reasoning: str  # Explicación de baja confianza si aplica
```

**Defaults (degradación controlada):**
- Si Bloque B no se prioriza tras 4 repreguntas: cada peso = `0.2` (suma 1.0).
- Si usuario dice "no me importa X": ese peso = `0` y se re-normaliza resto sumando 1.0.
- Si usuario no menciona aspecto: campo = `None`.

---

### `CareerFeature` (fila de `features.csv`)

```python
class CareerFeature:
    """Carrera-Universidad individual con features normalizadas.
    
    Mapeo desde features.csv real (Ponte en Carrera).
    `riasec_profile` NO viene en el CSV crudo — se agrega vía join con una
    tabla RIASEC de 554 carreras únicas (deduplicadas por `career`), etiquetadas
    con Bedrock few-shot. El join se hace por la columna `career`.
    `management_type` (Pública/Privada) es distinto de `institution_type` (Universidad/Instituto).
    """
    
    id: str  # Identificador único de la carrera (de features.csv)
    career: str  # Nombre de carrera (ej "Estadística")
    institution: str  # Universidad que ofrece
    
    # RIASEC etiquetado (agregado vía join externo)
    riasec_profile: str  # 3 letras, ej "RIA"
    riasec_source: str  # 'seed' | 'llm_tagged' | 'family_fallback' | 'pending'
    
    # Campos geo-administrativos (del CSV crudo)
    location: str  # Región donde está la universidad (columna features.csv)
    management_type: str  # 'Pública' | 'Privada' — tipo de gestión
    # NOTA: Existe también `institution_type` (Universidad/Instituto) que NO es lo mismo.
    #       El filtro del Figma pregunta por gestión (Pública/Privada), no por tipo institucional.
    
    # Features normalizadas [0,1] para scoring (todas orientadas: mayor = mejor para el estudiante)
    income_norm: float    # MinMax(log1p(monthly_income_imputed)) — mayor ingreso = mejor
    cost_norm: float      # 1 - MinMax(log1p(annual_cost_imputed)) — mayor = más barato (invertido)
    admission_norm: float  # MinMax(admission_rate_imputed) — mayor = más fácil acceso (no invertido)
    duration_norm: float   # 1 - MinMax(duration_years_imputed) — mayor = más corta (invertido)
    
    # Datos verificables imputados (valores imputados, no crudos, para mostrar al usuario)
    monthly_income_imputed: float  # S/. / mes (imputado)
    annual_cost_imputed: float  # S/. / año (imputado)
    admission_rate_imputed: float  # % (imputado)
    duration_years_imputed: float  # Años (imputado)
```

---

### `RankingItem` (item en el ranking Top-5)

```python
class RankingItem:
    """Un item en el ranking Top-5."""
    
    rank: int  # 1, 2, 3, 4, 5
    id: str  # career id from features.csv
    career: str  # "Estadística" (antes career_name)
    institution: str  # "UNMSM"
    
    # Score de concordancia
    concordancia_score: float  # [0,1]
    
    # Desglose por criterio
    scores_by_criterion: dict  # {
                                #   afinidad: 0.85,
                                #   ingreso: 0.82,
                                #   costo: 0.90,
                                #   admision: 0.60,
                                #   duracion: 0.75
                                # }
    
    # Datos verificables imputados del MINEDU
    datos_verificables: dict  # {
                               #   monthly_income_imputed: 3200,  # S/. mes
                               #   annual_cost_imputed: 100,     # S/. año
                               #   admission_rate_imputed: 18,   # %
                               #   duration_years_imputed: 5
                               # }
    
    # Explicación personalizada generada por LLM
    explicacion: str  # "Te recomendamos Estadística en la UNMSM porque..."
```

---

### `SessionContext` (en memoria, Fargate, TTL=1800s)

```python
class SessionContext:
    """Contexto conversacional de sesión (en memoria en Fargate)."""
    
    session_id: str  # UUID
    profile: StudentProfile  # Perfil parcial/completo interpretado
    conversation_history: list[dict]  # [
                                       #   {role: 'user', content: str, timestamp: datetime},
                                       #   {role: 'bot', content: str, timestamp: datetime},
                                       #   ...
                                       # ]
    
    turn_count: int  # Contador de repreguntas (máx 4)
    dialogue_state: str  # 'gathering_info' | 'ready_for_recommendation'
    
    created_at: float  # Timestamp creación (para TTL)
    ttl_seconds: int  # 1800 (30 minutos)
    
    # Histórico de rankings generados en esta sesión
    ranking_history: list[dict]  # [{ranking_id, top_5, timestamp}, ...]
```

---

### `FeedbackRecord` (persistido en Aurora PostgreSQL)

```python
class FeedbackRecord:
    """Registro persistido de una consulta + ranking + validación."""
    
    # Identificadores
    session_id: str  # UUID de sesión
    ranking_id: str  # UUID único del ranking
    
    # Entrada original del usuario
    user_input: str  # Consulta en español
    user_input_location: str | None  # Filtro ubicación/región si especificó
    user_input_budget_max: float | None  # Presupuesto máximo ANUAL (en soles, se compara con annual_cost_imputed)
    user_input_management_type: str | None  # 'Pública' | 'Privada' (antes institution_type)
    
    # Perfil interpretado (snapshot)
    profile_interpreted: dict  # StudentProfile serializado a JSON
    
    # Pesos dinámicos generados
    weights_generated: dict  # {
                              #   w_afinidad: 0.3,
                              #   w_ingreso: 0.25,
                              #   w_costo: 0.2,
                              #   w_admision: 0.15,
                              #   w_duracion: 0.1
                              # }
    
    # Ranking generado (Top-5)
    ranking_generated: list[dict]  # [RankingItem serializado, ...] x 5
    
    # Validación del usuario (Likert 1–5) — puede ser None si no validó aún
    validation_score: int | None  # [1, 5]
    selected_career: str | None  # Carrera que eligió
    notes: str | None  # Comentarios libres
    
    # Reproducibilidad (para auditoría y debugging)
    reproducibility_metadata: dict  # {
                                     #   snapshot_id: "features_20260715_143022",
                                     #   config_id: "config_20260715_143022",
                                     #   prompt_version: "v1",
                                     #   llm_model_used: "anthropic.claude-3-5-sonnet",
                                     #   bedrock_region: "us-east-1"
                                     # }
    
    # Timestamps
    created_at: datetime  # ISO 8601 UTC (guardar ranking)
    updated_at: datetime  # ISO 8601 UTC (actualizar con validación)
```

---

### `ChatResponse` (respuesta de `/chat`)

```python
class ChatResponse:
    """Respuesta de endpoint POST /chat."""
    
    status: str  # "success" | "error"
    
    conversation: dict  # {
                         #   turn: 1,
                         #   bot_message: "¿Hay región donde prefieras...?",
                         #   requires_follow_up: true
                         # }
    
    ranking: dict | None  # {
                           #   status: "ready" | "awaiting_info",
                           #   top_5: [RankingItem, ...] | None
                           # }
    
    rag_available_for: list[str]  # ["Estadística", "Ingeniería Civil"] — carreras con RAG
    
    ranking_id: str | None  # UUID si ranking listo, None si sigue preguntando
    
    error_message: str | None  # "Token inválido" o None si éxito
```

---

### `RagResponse` (respuesta de `/rag`)

```python
class RagResponse:
    """Respuesta de endpoint POST /rag."""
    
    answer: str  # Respuesta generada por LLM + contexto RAG
    sources: list[str]  # Documentos citados (ej ["Plan Curricular Oficial UNMSM"])
    confidence: float  # [0, 1]
    status: str  # "success" | "no_documents" | "error"
```

---

### `FeedbackResponse` (respuesta de `/feedback`)

```python
class FeedbackResponse:
    """Respuesta de endpoint POST /feedback."""
    
    status: str  # "success" | "error"
    message: str  # "Feedback guardado" o descripción de error
```

---

### Algoritmos

#### Algoritmo 1: Cálculo de RIASEC Code

**Pseudocódigo:**

```
FUNCTION calculate_riasec_code(
    riasec_scores: dict[str, int]  # {"R": 8, "I": 9, "A": 7, "S": 6, "E": 5, "C": 4}
) -> str

    // Paso 1: Crear pares (letra, score)
    pairs = []
    FOR cada (letra, score) in riasec_scores:
        pairs.append((letra, score))
    
    // Paso 2: Ordenar descendente por score, ascendente por letra (desempate)
    pairs.sort(
        key = lambda x: (-x[1], x[0])  // DESC score, ASC letter
    )
    
    // Paso 3: Tomar 3 primeras letras
    code = pairs[0][0] + pairs[1][0] + pairs[2][0]
    
    RETURN code

END FUNCTION
```

**Ejemplo:**
```
Input: {R: 8, I: 9, A: 7, S: 6, E: 5, C: 4}
Sorted pairs: [(I, 9), (R, 8), (A, 7), (S, 6), (E, 5), (C, 4)]
Output: "IRA"
```

---

#### Algoritmo 2: Cálculo de Afinidad RIASEC

**Pseudocódigo:**

```
FUNCTION calculate_affinity(
    student_code: str,  // "RIA" (perfil estudiante)
    career_profile: str  // "RIA" (perfil carrera)
) -> float

    score_crudo = 0
    
    // Para cada letra del perfil de carrera
    FOR i, career_letter in enumerate(career_profile):
        
        // Buscar posición en código estudiante
        student_pos = student_code.find(career_letter)
        
        IF student_pos == -1:
            // Letra no está en perfil estudiante
            points = 0
        ELIF student_pos == 0:
            // Posición 1 (índice 0) en estudiante
            points = 3
        ELIF student_pos == 1:
            // Posición 2 (índice 1)
            points = 2
        ELIF student_pos == 2:
            // Posición 3 (índice 2)
            points = 1
        ELSE:
            // No debería ocurrir (code tiene 3 letras)
            points = 0
        
        score_crudo += points
    
    // Normalizar a [0,1]
    // Máximo posible: 3+2+1 = 6
    afinidad_norm = score_crudo / 6.0
    
    RETURN clamp(afinidad_norm, 0.0, 1.0)

END FUNCTION
```

**Ejemplo:**
```
student_code = "RIA"
career_profile = "RIA"  → R:pos0(+3), I:pos1(+2), A:pos2(+1) = 6 → 6/6 = 1.0
career_profile = "AIR"  → A:pos2(+1), I:pos1(+2), R:pos0(+3) = 6 → 6/6 = 1.0
career_profile = "SIA"  → S:not_found(0), I:pos1(+2), A:pos2(+1) = 3 → 3/6 = 0.5
```

> **Nota abierta a decidir con el equipo:** Con el algoritmo actual `student_code="RIA"` y `career_profile="AIR"` dan el mismo puntaje (1.0) porque se busca la letra en cualquier posición del perfil de carrera (`str.find`), no se compara posición-a-posición. Si se desea que el orden exacto importe (ej. que "RIA" empate perfecto puntúe más que "AIR"), habría que implementar comparación posicional. Confirmar con el equipo antes de modificar código.

---

#### Algoritmo 3: Cálculo de Confidence Score

**Pseudocódigo:**

```
FUNCTION calculate_confidence(profile: StudentProfile) -> float

    // Contar campos completados
    riasec_completos = 1 si (profile.riasec_scores contiene 6 keys y todos != None)
                       0 si no
    
    pesos_completos = 0
    FOR cada peso in [w_afinidad, w_ingreso, w_costo, w_admision, w_duracion]:
        IF profile[peso] != None:
            pesos_completos += 1
    
    // Máximo posible: 6 (RIASEC) + 5 (pesos) = 11
    confidence = (riasec_completos * 6 + pesos_completos) / 11.0
    
    RETURN clamp(confidence, 0.0, 1.0)

END FUNCTION
```

**Tabla de ejemplos:**
| RIASEC | Pesos | Confidence |
|---|---|---|
| 6/6 | 5/5 | (6 + 5) / 11 = 1.0 |
| 6/6 | 3/5 | (6 + 3) / 11 ≈ 0.82 |
| 6/6 | 0/5 | (6 + 0) / 11 ≈ 0.55 |
| 0/6 | 3/5 | (0 + 3) / 11 ≈ 0.27 |
| 0/6 | 0/5 | (0 + 0) / 11 = 0.0 |

---

#### Algoritmo 4: Cálculo de Pesos desde Orden de Prioridad

**Pseudocódigo:**

```
FUNCTION calculate_weights_from_ranking(
    order: list[str]  // ["afinidad", "ingreso", "costo", "admision", "duracion"]
) -> dict[str, float]

    // Asignar puntaje 5-4-3-2-1 según posición
    scores = {
        order[0]: 5,  // 1er elemento: 5 puntos
        order[1]: 4,
        order[2]: 3,
        order[3]: 2,
        order[4]: 1   // Último: 1 punto
    }
    
    total = 5 + 4 + 3 + 2 + 1 = 15
    
    // Normalizar a [0,1]
    weights = {
        "w_afinidad": scores.get("afinidad", 0) / 15.0,
        "w_ingreso": scores.get("ingreso", 0) / 15.0,
        "w_costo": scores.get("costo", 0) / 15.0,
        "w_admision": scores.get("admision", 0) / 15.0,
        "w_duracion": scores.get("duracion", 0) / 15.0
    }
    
    RETURN weights

END FUNCTION
```

**Ejemplo:**
```
order = ["afinidad", "ingreso", "costo", "admision", "duracion"]
→ w_afinidad = 5/15 = 0.333
→ w_ingreso = 4/15 = 0.267
→ w_costo = 3/15 = 0.200
→ w_admision = 2/15 = 0.133
→ w_duracion = 1/15 = 0.067
suma = 1.0 ✓
```

---

#### Algoritmo 5: Normalización MinMax con Transformaciones

**Pseudocódigo:**

```
FUNCTION normalize_variable(
    series: Series,
    log_transform: bool = false,
    invert: bool = false
) -> Series

    // Paso 1: Transformación logarítmica (opcional)
    IF log_transform:
        series = log(1 + series)  // log1p
    
    // Paso 2: Min-Max scaling
    min_val = series.min()
    max_val = series.max()
    
    IF min_val == max_val:
        normalized = Series([1.0] * len(series))  // Todos iguales → 1.0
    ELSE:
        normalized = (series - min_val) / (max_val - min_val)
    
    // Paso 3: Inversión (opcional)
    IF invert:
        normalized = 1 - normalized
    
    RETURN normalized

END FUNCTION

// Aplicación específica para features.csv (todas orientadas: mayor = mejor para el estudiante):
// NOTA: Los inputs provienen del DataFrame después de standardize_columns e imputación,
// por lo que usan las columnas imputadas.
income_norm = normalize_variable(monthly_income_imputed, log=True, invert=False)
    // Mayor ingreso = mejor → no invertir
cost_norm = normalize_variable(annual_cost_imputed, log=True, invert=True)
    // Menor costo = mejor → invertir (mayor cost_norm = más barato)
admission_norm = normalize_variable(admission_rate_imputed, log=False, invert=False)
    // Mayor tasa = más fácil = mejor → no invertir
duration_norm = normalize_variable(duration_years_imputed, log=False, invert=True)
    // Menor duración = mejor → invertir (mayor duration_norm = más corta)
```

---

#### Algoritmo 6: Cálculo de Score de Concordancia (5 Criterios)

**Pseudocódigo:**

```
FUNCTION calculate_score(
    weights: dict,  // {w_afinidad, w_ingreso, w_costo, w_admision, w_duracion}
    afinidad_norm: float [0,1],
    ingreso_norm: float [0,1],
    costo_norm: float [0,1],      // mayor = más barato (ya invertido en pipeline)
    admission_norm: float [0,1],  // mayor = más fácil acceso
    duracion_norm: float [0,1]    // mayor = más corta (ya invertido en pipeline)
) -> float

    // Validar que pesos sumen 1.0 ± 0.01
    sum_weights = sum(weights.values())
    IF abs(sum_weights - 1.0) > 0.01:
        // Normalizar automáticamente
        FOR key in weights:
            weights[key] /= sum_weights
        LOG_WARNING("Pesos renormalizados: suma no era 1.0")
    
    // Todas las normas ya están orientadas "mayor = mejor" por el pipeline
    // NO se aplican inversiones adicionales
    score = (
        weights["w_afinidad"] * afinidad_norm +
        weights["w_ingreso"] * ingreso_norm +
        weights["w_costo"] * costo_norm +
        weights["w_admision"] * admission_norm +
        weights["w_duracion"] * duracion_norm
    )
    
    RETURN clamp(score, 0.0, 1.0)

END FUNCTION
```

---

#### Algoritmo 7: Ranking Determinístico (Top-5)

**Pseudocódigo:**

```
FUNCTION rank_and_filter(
    profile: StudentProfile,
    features_df: DataFrame
) -> list[RankingItem]

    // Paso 1: Calcular código RIASEC estudiante
    riasec_code = calculate_riasec_code(profile.riasec_scores)
    
    // Paso 2: Aplicar filtros duros (Bloque C)
    // Columnas de features.csv: location, management_type, annual_cost_imputed, modalidad
    filtered_df = features_df.copy()
    
    IF profile.location != None:
        filtered_df = filtered_df[filtered_df["location"] == profile.location]
    
    IF profile.management_type != None:
        // Normalizar: StudentProfile guarda 'publica'/'privada', CSV guarda 'Pública'/'Privada'
        csv_val = "Pública" if profile.management_type == "publica" else "Privada"
        filtered_df = filtered_df[filtered_df["management_type"] == csv_val]
    
    IF profile.presupuesto_max != None:
        filtered_df = filtered_df[filtered_df["annual_cost_imputed"] <= profile.presupuesto_max]
    
    IF profile.modalidad != None:
        filtered_df = filtered_df[filtered_df["modalidad"] == profile.modalidad]
    
    // Paso 3: Calcular score para cada fila
    results = []
    
    FOR idx, row in filtered_df.iterrows():
        
        // Calcular afinidad RIASEC
        afinidad = calculate_affinity(riasec_code, row["riasec_profile"])
        
        // Si riasec_profile falta, afinidad = 0.5 (fallback)
        IF row["riasec_profile"] == None or row["riasec_profile"] == "":
            afinidad = 0.5
            LOG_WARNING(f"RIASEC faltante para {row['id']}")
        
        // Calcular score final (todas las normas ya están orientadas "mayor = mejor")
        score = calculate_score(
            weights={
                "w_afinidad": profile.w_afinidad,
                "w_ingreso": profile.w_ingreso,
                "w_costo": profile.w_costo,
                "w_admision": profile.w_admision,
                "w_duracion": profile.w_duracion
            },
            afinidad_norm=afinidad,
            ingreso_norm=row["income_norm"],
            costo_norm=row["cost_norm"],
            admission_norm=row["admission_norm"],
            duracion_norm=row["duration_norm"]
        )
        
        results.append({
            "id": row["id"],
            "career": row["career"],
            "institution": row["institution"],
            "concordancia_score": score,
            "scores_by_criterion": {
                "afinidad": afinidad,
                "ingreso": row["income_norm"],
                "costo": row["cost_norm"],
                "admision": row["admission_norm"],
                "duracion": row["duration_norm"]
            },
            "verifiable_data": {
                "monthly_income_imputed": row["monthly_income_imputed"],
                "annual_cost_imputed": row["annual_cost_imputed"],
                "admission_rate_imputed": row["admission_rate_imputed"],
                "duration_years_imputed": row["duration_years_imputed"]
            }
        })
    
    // Paso 4: Ordenar descendente por score
    // Desempate: alfabético por institution (determinismo)
    results.sort(
        key = lambda x: (-x["concordancia_score"], x["institution"])
    )
    
    // Paso 5: Asignar ranks y retornar Top-5
    top_5 = []
    FOR i, item in enumerate(results[:5]):
        item["rank"] = i + 1
        top_5.append(item)
    
    RETURN top_5

END FUNCTION
```

**Determinismo:**
- Mismo input (profile, features_df) → mismo output ranking en 100% ejecuciones.
- Desempate alfabético por `institution` garantiza orden único.

---

#### Algoritmo 8: Imputación Jerárquica (Data Pipeline)

**Pseudocódigo:**

```
FUNCTION impute_hierarchical(
    variable: str,  // "duration_years", "monthly_income", "annual_cost", "admission_rate"
    df: DataFrame,
    config: dict
) -> Series

    result = df[variable].copy()
    
    // Identificar valores inválidos
    invalid_mask = (
        (df[variable] <= 0) |
        (df[variable].isna()) |
        ((variable == "duration_years") AND (df[variable] > 10)) |
        ((variable == "admission_rate") AND (df[variable] > 90))
    )
    
    // Nivel 1: Mediana por career_family
    level_1_fallback = df.groupby("career_family")[variable].median()
    
    FOR idx WHERE invalid_mask[idx]:
        cf = df["career_family"][idx]
        
        IF cf in level_1_fallback AND level_1_fallback[cf] != None:
            result[idx] = level_1_fallback[cf]
            CONTINUE
        
        // Nivel 2: Fallback desde config
        fallback_key = variable + "_fallback"
        IF fallback_key in config:
            result[idx] = config[fallback_key]
        ELSE:
            // No debería ocurrir si config está completo
            LOG_ERROR(f"Fallback no definido para {variable}")
            result[idx] = None
    
    RETURN result

END FUNCTION
```

---

## Diseño de API REST

### Endpoint: POST /chat

**Request:**
```json
{
    "message": "Me interesan las matemáticas y el análisis de datos",
    "metadata": {
        "location": null,         # Ubicación/región del estudiante
        "budget_max": null,       # Presupuesto máximo ANUAL (soles), se compara con annual_cost_imputed
        "management_type": null   # 'Pública' | 'Privada'
    }                             # NOTA: budget_max es anual, NO mensual
}
```

**Headers:**
```
Authorization: Bearer {session_token}
Content-Type: application/json
```

**Response (200 OK):**
```json
{
    "status": "success",
    "conversation": {
        "turn": 1,
        "bot_message": "¡Excelente! Veo que te interesan áreas cuantitativas. Para darte mejores recomendaciones, ¿hay alguna región del Perú donde prefieras estudiar?",
        "requires_follow_up": true
    },
    "ranking": {
        "status": "awaiting_info",
        "top_5": null
    },
    "rag_available_for": [],
    "ranking_id": null,
    "error_message": null
}
```

**Response con Ranking (después de confianza >= 0.70):**
```json
{
    "status": "success",
    "conversation": {
        "turn": 4,
        "bot_message": "Perfecto, tengo una visión clara de lo que buscas. Aquí están mis 5 mejores recomendaciones:",
        "requires_follow_up": false
    },
    "ranking": {
        "status": "ready",
        "top_5": [
            {
                "rank": 1,
                "career": "Estadística",
                "institution": "UNMSM",
                "concordancia_score": 0.847,
                "scores_by_criterion": {
                    "afinidad": 0.85,
                    "ingreso": 0.82,
                    "costo": 0.90,
                    "admision": 0.60,
                    "duracion": 0.75
                },
                    "datos_verificables": {
                        "monthly_income_imputed": 3200,
                        "annual_cost_imputed": 100,
                        "admission_rate_imputed": 18,
                        "duration_years_imputed": 5
                    },
                    "explicacion": "Te recomendamos Estadística en la UNMSM porque tu afinidad con el análisis de datos es muy alta (85/100). Esta carrera ofrece uno de los mejores ingresos en el mercado (S/. 3,200/mes según datos del MINEDU) con un costo relativamente accesible. Aunque es selectiva (18% de admisión), es factible con tu fortaleza en matemáticas."
            },
            {
                "rank": 2,
                "career": "Ingeniería Civil",
                "institution": "UNI",
                "concordancia_score": 0.812,
                "scores_by_criterion": {
                    "afinidad": 0.78,
                    "ingreso": 0.88,
                    "costo": 0.85,
                    "admision": 0.65,
                    "duracion": 0.70
                },
                "datos_verificables": {
                    "monthly_income_imputed": 3500,
                    "annual_cost_imputed": 120,
                    "admission_rate_imputed": 22,
                    "duration_years_imputed": 5
                },
                "explicacion": "Ingeniería Civil en la UNI es otra excelente opción. Combina razonamiento analítico con aplicaciones prácticas. Los ingresos son superiores (S/. 3,500/mes) y la demanda en el mercado es consistente. Un poco más selectiva que Estadística, pero muy accesible para estudiantes con tu perfil."
            },
            {
                "rank": 3,
                "career": "Ciencia de Datos",
                "institution": "Pontificia Universidad Católica",
                "concordancia_score": 0.795,
                "scores_by_criterion": {
                    "afinidad": 0.92,
                    "ingreso": 0.85,
                    "costo": 0.60,
                    "admision": 0.70,
                    "duracion": 0.68
                },
                "datos_verificables": {
                    "monthly_income_imputed": 3100,
                    "annual_cost_imputed": 280,
                    "admission_rate_imputed": 35,
                    "duration_years_imputed": 4
                },
                "explicacion": "Ciencia de Datos tiene la máxima afinidad con tu perfil (92/100) y es más nueva en el mercado. El programa es más corto (4 años) y menos selectivo. Sin embargo, es una institución privada con costos mayores (S/. 280/mes). Es ideal si buscas especialización rápida."
            },
            {
                "rank": 4,
                "career": "Matemática Aplicada",
                "institution": "UNMSM",
                "concordancia_score": 0.778,
                "scores_by_criterion": {
                    "afinidad": 0.90,
                    "ingreso": 0.72,
                    "costo": 0.92,
                    "admision": 0.55,
                    "duracion": 0.82
                },
                "datos_verificables": {
                    "monthly_income_imputed": 2800,
                    "annual_cost_imputed": 80,
                    "admission_rate_imputed": 15,
                    "duration_years_imputed": 5
                },
                "explicacion": "Matemática Aplicada es la opción más accesible económicamente (S/. 80/mes) con altísima afinidad (90/100). Los ingresos son menores que otras opciones, pero ofrece flexibilidad para postgrados especializados. Recomendada si el presupuesto es una limitación importante."
            },
            {
                "rank": 5,
                "career": "Actuaría",
                "institution": "Pontificia Universidad Católica",
                "concordancia_score": 0.742,
                "scores_by_criterion": {
                    "afinidad": 0.80,
                    "ingreso": 0.92,
                    "costo": 0.55,
                    "admision": 0.68,
                    "duracion": 0.65
                },
                "datos_verificables": {
                    "monthly_income_imputed": 4100,
                    "annual_cost_imputed": 300,
                    "admission_rate_imputed": 40,
                    "duration_years_imputed": 5
                },
                "explicacion": "Actuaría ofrece los ingresos más altos del ranking (S/. 4,100/mes), combinando matemáticas con aplicación en seguros y finanzas. Es menos selectiva que otras opciones, pero también es privada con costos elevados. Ideal si el potencial salarial es prioritario."
            }
        ]
    },
    "rag_available_for": ["Estadística", "Ingeniería Civil", "Ciencia de Datos"],
    "ranking_id": "550e8400-e29b-41d4-a716-446655440000",
    "error_message": null
}
```

---

### Endpoint: POST /feedback

**Request:**
```json
{
    "ranking_id": "550e8400-e29b-41d4-a716-446655440000",
    "validation_score": 5,
    "selected_career": "Estadística",
    "notes": "Me gustó mucho. Voy a investigar más sobre esta carrera."
}
```

**Response (200 OK):**
```json
{
    "status": "success",
    "message": "¡Gracias por tu feedback! Nos ayuda a mejorar nuestras recomendaciones."
}
```

**Response (422 Unprocessable Entity):**
```json
{
    "status": "error",
    "message": "validation_score debe estar entre 1 y 5"
}
```

---

### Endpoint: POST /rag

**Request:**
```json
{
    "career": "Estadística",
    "question": "¿Qué cursos lleva esta carrera?"
}
```

**Response (200 OK):**
```json
{
    "answer": "Estadística comprende cursos teóricos y aplicados divididos en bloques: \n\n1. Probabilidad y Cálculo (primer año): Cálculo I–III, Álgebra Lineal, Probabilidades. \n2. Estadística Matemática (segundo año): Inferencia Estadística, Muestreo, Teoría de Decisiones. \n3. Aplicaciones (tercero-cuarto año): Análisis Multivariante, Series de Tiempo, Modelos Lineales. \n4. Especialización (quinto año): Cursos electivos de análisis bayesiano, econometría, estadística computacional.",
    "sources": ["Plan Curricular Oficial UNMSM 2024", "Catálogo de Cursos Requeridos"],
    "confidence": 0.94,
    "status": "success"
}
```

**Response (no_documents):**
```json
{
    "answer": "",
    "sources": [],
    "confidence": 0.0,
    "status": "no_documents"
}
```

---

## Estructura de Persistencia

### Aurora PostgreSQL — Tablas Principales

```sql
-- Tabla: feedback (contiene rankings + validaciones)
CREATE TABLE IF NOT EXISTS feedback (
    id BIGSERIAL PRIMARY KEY,
    session_id UUID NOT NULL,
    ranking_id UUID NOT NULL UNIQUE,
    
    -- Entrada del usuario
    user_input TEXT NOT NULL,
    user_input_location VARCHAR(100),  -- región/ubicación (columna features.csv: location)
    user_input_budget_max DECIMAL(10,2),  -- Presupuesto máximo ANUAL (soles), se compara con annual_cost_imputed
    user_input_management_type VARCHAR(20),  -- 'Pública' | 'Privada' (antes tipo_institucion)
    -- NOTA: institution_type (Universidad/Instituto) es un campo distinto en features.csv
    
    -- Perfil interpretado (JSON)
    profile_riasec_scores JSONB,  -- {"R": 8, "I": 9, ...}
    profile_riasec_code VARCHAR(3),
    profile_w_afinidad DECIMAL(3,2),
    profile_w_ingreso DECIMAL(3,2),
    profile_w_costo DECIMAL(3,2),
    profile_w_admision DECIMAL(3,2),
    profile_w_duracion DECIMAL(3,2),
    profile_confidence DECIMAL(3,2),
    
    -- Ranking generado (JSON completo)
    ranking_generated JSONB,  -- Array de RankingItem [...]
    
    -- Validación del usuario
    validation_score SMALLINT,
    validation_selected_career VARCHAR(255),
    validation_notes TEXT,
    
    -- Reproducibilidad
    snapshot_id VARCHAR(50),  -- features_20260715_143022
    config_id VARCHAR(50),    -- config_20260715_143022
    prompt_version VARCHAR(10),
    llm_model_used VARCHAR(100),
    bedrock_region VARCHAR(30),
    
    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Índices
    INDEX idx_session ON feedback(session_id),
    INDEX idx_ranking ON feedback(ranking_id),
    INDEX idx_created ON feedback(created_at)
);

-- Tabla: career_chunks (para RAG, pgvector)
CREATE TABLE IF NOT EXISTS career_chunks (
    id BIGSERIAL PRIMARY KEY,
    career_id TEXT NOT NULL,
    content TEXT NOT NULL,
    embedding VECTOR(1024),
    metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT now(),
    
    INDEX idx_career ON career_chunks(career_id),
    INDEX idx_embedding ON career_chunks USING ivfflat (embedding vector_cosine_ops)
);
```

### DynamoDB — Tabla universities

```
Tabla: universities

Atributos:
  PK: institution_id (S)  — "unmsm_001"
  
  Atributos principales:
    name (S)                   — "Universidad Nacional Mayor de San Marcos"
    location (S)               — "Lima" (antes region)
    management_type (S)        — "Pública" | "Privada" (antes tipo_institucion)
    latitude (N)               — -12.0450
    longitude (N)              — -77.0822
    careers_ids (SS)           — {"estadistica_001", "ingenieria_civil_001", ...}
    website (S)                — "https://www.unmsm.edu.pe"
    contact_email (S)          — "admisiones@unmsm.edu.pe"
    
GSI: location-index
  PK: location (S) (antes region)
  SK: institution_id (S)
  
Permite queries tipo:
  SELECT * FROM universities WHERE location = "Lima"
```

### SQLite (Demo Local)

Para desarrollo local sin AWS, usar SQLite:

```python
# backend/config.py
DATABASE_URL = os.getenv(
    "DATABASE_URL",
    "sqlite:///./feedback.db"  # Default para demo
)
```

```sql
-- Base de datos de demo (SQLite)
CREATE TABLE feedback (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL,
    ranking_id TEXT NOT NULL UNIQUE,
    
    user_input TEXT NOT NULL,
    user_input_location TEXT,  -- región/ubicación
    user_input_budget_max REAL,  -- Presupuesto máximo ANUAL (soles), se compara con annual_cost_imputed
    user_input_management_type TEXT,  -- 'Pública' | 'Privada'
    
    profile_riasec_scores TEXT,  -- JSON serializado
    profile_w_afinidad REAL,
    profile_w_ingreso REAL,
    profile_w_costo REAL,
    profile_w_admision REAL,
    profile_w_duracion REAL,
    profile_confidence REAL,
    
    ranking_generated TEXT,  -- JSON serializado
    
    validation_score INTEGER,
    validation_selected_career TEXT,
    validation_notes TEXT,
    
    snapshot_id TEXT,
    config_id TEXT,
    prompt_version TEXT,
    llm_model_used TEXT,
    
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_feedback_session ON feedback(session_id);
CREATE INDEX idx_feedback_ranking ON feedback(ranking_id);
```

---

## Restricciones y Validaciones

| Campo | Restricción |
|---|---|
| `riasec_scores[*]` | [1, 10] |
| `riasec_code` | Exactamente 3 letras |
| `w_afinidad...w_duracion` | [0, 1], suma ≈ 1.0 |
| `concordancia_score` | [0, 1] |
| `validation_score` | [1, 5] entero |
| `monthly_income_imputed` | > 0 |
| `annual_cost_imputed` | ≥ 0 |
| `admission_rate_imputed` | [0, 90] % |
| `duration_years_imputed` | [3, 7] años |
| Timestamps | ISO 8601 UTC |

---
# Stack Tecnológico y Dependencias (Actualizado — AWS)

## Dependencias por Componente

### Core: Python Runtime

| Dependencia | Versión mínima | Propósito | Notas |
|---|---|---|---|
| **Python** | 3.10+ | Runtime | Requerido para FastAPI, AWS SDK, etc. |
| **uv** | 0.4.0+ | Package manager | Gestión moderna de dependencias (`uv add`, `uv run`, `uv sync`) |

---

### Backend API (Fargate + Lambda)

| Dependencia | Versión mínima | Propósito | Justificación |
|---|---|---|---|
| **FastAPI** | 0.104.0+ | Framework REST API (Fargate) | Async, type hints, OpenAPI automático |
| **Uvicorn** | 0.24.0+ | ASGI server (Fargate) | HTTP server ligeropara FastAPI |
| **Pydantic** | 2.0.0+ | Validación de modelos | Schemas de request/response |
| **boto3** | 1.28.0+ | AWS SDK (Bedrock, Aurora, DynamoDB, Lambda) | Interacción con servicios AWS |
| **psycopg2-binary** | 2.9.0+ | PostgreSQL adapter (Aurora) | Conexiones a Aurora PostgreSQL |
| **pgvector** | 0.1.8+ | pgvector Python client | Queries de similarity search en Aurora |
| **SQLAlchemy** | 2.0.0+ | ORM (opcional, Aurora) | Abstracción DB, pero boto3 puede ser suficiente |
| **python-dotenv** | 1.0.0+ | Gestión de .env | Variables de entorno locales (dev) |
| **PyJWT** | 2.8.0+ | JWT tokens | Autenticación sin estado Lambda ↔ Fargate |
| **cryptography** | 41.0.0+ | Encriptación (JWT) | Soporte para PyJWT |

---

### Data Pipeline (Batch/Fargate Task)

| Dependencia | Versión mínima | Propósito | Justificación |
|---|---|---|---|
| **pandas** | 2.0.0+ | Manipulación de datos | Lectura/limpieza/transformación de carreras |
| **numpy** | 1.24.0+ | Operaciones vectorizadas | Cálculos de normalización MinMax |
| **openpyxl** | 3.10.0+ | Lectura/escritura Excel | Procesamiento de descarga de Ponte en Carrera |
| **selenium** | 4.15.0+ | Automatización de navegador | Descarga automatizada de Ponte en Carrera (MINEDU) |
| **boto3** | 1.28.0+ | AWS SDK | Acceso a S3 (snapshots), Bedrock (tagging RIASEC) |

---

### Testing

| Dependencia | Versión mínima | Propósito | Instalación |
|---|---|---|---|---|
| **pytest** | 7.0.0+ | Test runner | `uv add --dev "pytest>=7.0.0"` |
| **pytest-asyncio** | 0.21.0+ | Tests async (para FastAPI) | `uv add --dev "pytest-asyncio>=0.21.0"` |
| **pytest-cov** | 4.1.0+ | Cobertura de tests | `uv add --dev "pytest-cov>=4.1.0"` |
| **requests** | 2.31.0+ | HTTP client (tests) | `uv add --dev "requests>=2.31.0"` |
| **moto** | 4.2.0+ | Mock AWS services | `uv add --dev "moto>=4.2.0"` |

---

### Frontend (Navegador)

| Dependencia | Versión mínima | Propósito | Notas |
|---|---|---|---|
| **JavaScript (Vanilla)** | ES6+ | Lógica frontend | Sin framework (reducir dependencias) |
| **HTML5** | — | Markup | Respetado en todo navegador moderno |
| **CSS3** | — | Estilos responsive | Flexbox, Grid para responsividad |

---

### Infraestructura (AWS + Local)

| Servicio/Herramienta | Versión mínima | Propósito | Notas |
|---|---|---|---|
| **AWS Account** | — | Acceso a servicios AWS | Requerido para Bedrock, Aurora, DynamoDB, Lambda, Fargate, etc. |
| **AWS CLI** | 2.13.0+ | Configuración y debugging AWS | Opcional, facilita despliegue local |
| **Docker** | 20.10.0+ | Containerización (Fargate) | Requerido para ejecutar Fargate localmente |
| **Docker Compose** | 2.0.0+ | Orquestación local | Para desarrollo con DB local simulada |
| **Chrome** | 90.0.0+ | Navegador (Selenium) | Requerido para descargar datos de Ponte |
| **Node.js** | 18.0.0+ | Runtime frontend (opcional) | Solo si se desea usar build tools (webpack, etc.) |

---

## Gestión de Dependencias con uv

El proyecto usa `uv` como gestor de dependencias. No se usa `requirements.txt`.

**Inicializar proyecto:**

```bash
uv init
```

**Agregar dependencias:**

```bash
# Core AWS + Backend
uv add "boto3>=1.28.0" "fastapi>=0.104.0" "uvicorn>=0.24.0" "pydantic>=2.0.0" \
       "psycopg2-binary>=2.9.0" "pgvector>=0.1.8" "sqlalchemy>=2.0.0" \
       "PyJWT>=2.8.0" "cryptography>=41.0.0" "python-dotenv>=1.0.0"

# Data Pipeline
uv add "pandas>=2.0.0" "numpy>=1.24.0" "openpyxl>=3.10.0" "selenium>=4.15.0"

# Dev dependencies
uv add --dev "pytest>=7.0.0" "pytest-asyncio>=0.21.0" "pytest-cov>=4.1.0" \
            "requests>=2.31.0" "moto>=4.2.0" \
            "ipython>=8.0.0" "notebook>=7.0.0" \
            "black>=23.0.0" "flake8>=6.0.0" "mypy>=1.0.0"
```

**Sincronizar entorno:**

```bash
uv sync
```

Esto genera `uv.lock` automáticamente con versiones fijas de todas las dependencias transitivas, garantizando entornos reproducibles.

---

## Variables de Entorno (.env)

```bash
# AWS Configuration
AWS_REGION=us-east-1
AWS_PROFILE=default  # o nombre del perfil AWS configurado

# Bedrock (LLM)
LLM_PROVIDER=bedrock
LLM_MODEL=anthropic.claude-3-5-sonnet-20241022
BEDROCK_REGION=us-east-1

# Bedrock Embeddings
EMBEDDING_PROVIDER=bedrock
EMBEDDING_MODEL=amazon.titan-embed-text-v2
EMBEDDING_REGION=us-east-1

# Aurora PostgreSQL
AURORA_ENDPOINT=careermatch-db.xxxxxxxxxxxx.us-east-1.rds.amazonaws.com
AURORA_PORT=5432
AURORA_USER=postgres
AURORA_PASSWORD=your_secure_password_here
AURORA_DATABASE=careermatch

# DynamoDB
DYNAMODB_TABLE_UNIVERSITIES=universities
DYNAMODB_REGION=us-east-1

# JWT Authentication
JWT_SECRET=your-super-secret-key-min-32-chars-xxxxxxx
JWT_ALGORITHM=HS256
SESSION_TIMEOUT_MINUTES=480  # 8 horas

# S3 (snapshots opcional)
S3_BUCKET_SNAPSHOTS=careermatch-snapshots
S3_REGION=us-east-1

# Logging
LOG_LEVEL=INFO  # DEBUG, INFO, WARNING, ERROR, CRITICAL

# Development
ENVIRONMENT=development  # production | development | staging
DEBUG=false
```

**Archivo .env.example:**

Copiar y llenar con valores reales:

```bash
cp .env.example .env
# Editar .env con valores de AWS
```

---

## Cambios Principales vs. Diseño Anterior

### 1. LLM: De Gemini/OpenAI a Amazon Bedrock

**Anterior:**
```python
from google.generativeai import generative_ai  # Gemini
from openai import OpenAI  # OpenAI
```

**Ahora:**
```python
import boto3

bedrock_client = boto3.client('bedrock-runtime', region_name='us-east-1')
# Usar Claude-3.5-Sonnet vía Bedrock
response = bedrock_client.invoke_model(
    modelId='anthropic.claude-3-5-sonnet-20241022',
    body=json.dumps({...})
)
```

**Ventajas:**
- Integración nativa con AWS
- Seguridad: claves en AWS IAM
- Costos previsibles
- Sin SDK externo para cada proveedor

---

### 2. Embeddings: De sentence-transformers local a Amazon Bedrock Titan

**Anterior:**
```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('multilingual-MiniLM-L12-v2')
embedding = model.encode('texto')  # Local, 384 dims
```

**Ahora:**
```python
import boto3

bedrock_client = boto3.client('bedrock-runtime', region_name='us-east-1')
response = bedrock_client.invoke_model(
    modelId='amazon.titan-embed-text-v2:0',
    body=json.dumps({
        'inputText': 'texto',
        ...
    })
)
# Retorna 1024 dims
```

**Ventajas:**
- Embeddings en cloud (no depender de memoria local)
- Dimensionalidad 1024 (vs 384 local)
- Integración con pgvector en Aurora

---

### 3. Vector DB: De FAISS local a pgvector en Aurora

**Anterior:**
```python
import faiss

index = faiss.IndexFlatL2(1024)
# Búsqueda local
```

**Ahora:**
```sql
CREATE EXTENSION pgvector;
SELECT * FROM career_chunks
ORDER BY embedding <=> query_embedding
LIMIT 3;
```

**Ventajas:**
- Persistencia automática en Aurora
- Escalabilidad sin cargar índices en memoria
- SQL nativo, sin librería externa

---

### 4. Base de Datos: De SQLite a Aurora PostgreSQL

**Anterior:**
```python
DATABASE_URL = "sqlite:///./feedback.db"
```

**Ahora:**
```python
DATABASE_URL = "postgresql://user:pass@aurora-endpoint:5432/careermatch"
```

**Ventajas:**
- Managed service (AWS RDS Aurora)
- Backups automáticos
- Escalabilidad
- pgvector soporte nativo

---

### 5. Arquitectura: De monolito a Lambda + Fargate

**Anterior:**
- Todo en un servidor FastAPI

**Ahora:**
- **Lambda**: Stateless (`/session/create`, `/feedback`, `/universities`)
- **Fargate**: Stateful agent (`/chat`)
- Comunicación vía JWT sin estado compartido

**Ventajas:**
- Escalabilidad independiente
- Costo: pagar por uso real
- Sesiones en memoria sin overhead de Redis

---

## Manejo de Errores (Actualizado para AWS)

### Estrategia General de Degradación Controlada

```python
try:
    # Intentar operación
    response = bedrock_client.invoke_model(...)
except botocore.exceptions.ClientError as e:
    if e.response['Error']['Code'] == 'ThrottlingException':
        # Reintentar exponencial
        await asyncio.sleep(2 ** attempt)
        response = await invoke_model_retry(...)
    elif e.response['Error']['Code'] == 'ValidationException':
        # Fallback templado
        logger.warning(f"Bedrock validation error: {e}")
        response = generate_templated_response(...)
    else:
        # Fallar explícitamente
        logger.error(f"Bedrock error: {e}", exc_info=True)
        raise HTTPException(status_code=500)
except Exception as e:
    logger.error(f"Unexpected error: {e}", exc_info=True)
    raise HTTPException(status_code=500)
```

---

### Tabla de Excepciones AWS

| Excepción | Servicio | Causa | Manejo |
|---|---|---|---|
| `ThrottlingException` | Bedrock | Rate limit excedido | Reintentar exponencial (1s, 2s, 4s) |
| `ValidationException` | Bedrock | Input inválido | Loguear, retornar fallback templado |
| `AccessDeniedException` | IAM | Credenciales sin permisos | Loguear error critico, fallar HTTP 500 |
| `EndpointConnectionError` | RDS/Aurora | BD no disponible | Queue local, reintentar sincronización |
| `ResourceNotFoundException` | DynamoDB | Tabla no existe | Loguear error critico, fallar HTTP 500 |
| `BotoClientError` | AWS SDK | Error general AWS | Reintentar exponencial, luego fallar |
| `TimeoutError` | Bedrock/Aurora | Timeout > 5s (LLM) o 10s (DB) | Loguear, retornar error o fallback |
| `JSONDecodeError` | LLM_Layer | Bedrock retorna JSON inválido | Reintentar 3 veces, luego fallback templado |

---

### Casos de Uso: Degradación en Acción

| Situación | Comportamiento | Resultado Esperado |
|---|---|---|
| **Bedrock no disponible** | Fallback templado para explicaciones | Top-5 sin explicación personalizada (solo datos) |
| **Aurora temporalmente inaccesible** | Queue local de feedback en Fargate (en-memoria) | Feedback guardado localmente, sincronizado cuando BD vuelva |
| **Bedrock RIASEC tagging falla** | Family fallback (moda por family) | Ninguna carrera sin riasec_profile |
| **Embeddings generación falla** | RAG desactivado, /chat continúa | Top-5 sin detalles extendidos |
| **DynamoDB no respondiendo** | Fallback a listado global universidades | /universities retorna datos cacheados o vacío |
| **JWT validación falla** | HTTP 401 inmediato | No hay fallback, re-autenticate requerido |

---

## Configuración de Logging (Actualizado)

```python
import logging
import json
from datetime import datetime
import boto3

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_obj = {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "level": record.levelname,
            "component": record.name,
            "message": record.getMessage(),
            "session_id": getattr(record, "session_id", None),
            "function": record.funcName,
            "line": record.lineno,
            "extra": getattr(record, "extra_data", {})
        }
        # Nunca loguear tokens completos, claves AWS
        if "token" in str(log_obj).lower():
            log_obj["warning"] = "token_field_present_but_hidden"
        return json.dumps(log_obj)

# Configurar logging
handler = logging.StreamHandler()  # stdout (Docker)
handler.setFormatter(JSONFormatter())

logger = logging.getLogger("careermatch")
logger.addHandler(handler)
logger.setLevel(logging.INFO)  # Configurable por LOG_LEVEL env var

# Ejemplo de uso con contexto
logger.info("Ranking generated", extra={
    "extra_data": {
        "session_id": session_id,
        "top_5_count": 5,
        "confidence_score": 0.85
    }
})
```

**Destino:** 
- **CloudWatch Logs** (producción): Logs automáticos vía Fargate/Lambda
- **stdout** (desarrollo): Visible en `docker logs`
- **Archivo local** (opcional): Para persistencia local

---

## Docker & Containerización

### Dockerfile (Demo — un solo servicio FastAPI)

```dockerfile
FROM python:3.10-slim

WORKDIR /app

# Instalar uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# Instalar dependencias del sistema
RUN apt-get update && apt-get install -y \
    gcc \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Copiar archivos de dependencias
COPY pyproject.toml uv.lock ./
RUN uv sync --no-dev

# Copiar código
COPY backend/ ./backend/
COPY data/ ./data/

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Entrypoint
CMD ["uv", "run", "uvicorn", "backend.app:app", "--host", "0.0.0.0", "--port", "8000"]
```

### docker-compose.yml (Demo — un solo servicio FastAPI)

```yaml
version: '3.9'

services:
  # Backend único (demo): sirve /chat, /feedback, /universities
  backend:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      AWS_REGION: us-east-1
      LLM_PROVIDER: bedrock
      AURORA_ENDPOINT: postgres_db:5432
      ENVIRONMENT: development
    depends_on:
      - postgres_db
    volumes:
      - ./backend:/app/backend
      - ./data:/app/data
    networks:
      - careermatch_net

  # PostgreSQL local (simular Aurora)
  postgres_db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: careermatch
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - careermatch_net

volumes:
  postgres_data:

networks:
  careermatch_net:
    driver: bridge
```

**Ejecución local (demo):**

```bash
docker-compose up -d
# Backend disponible en http://localhost:8000
# PostgreSQL en localhost:5432
```

> **Producción:** En lugar de docker-compose, se usarán servicios AWS administrados (Aurora, DynamoDB, ECS Fargate, Lambda). Ver `infra/` y `docs/DEPLOYMENT.md` para la configuración de producción.

---

## Propiedades de Corrección (Revisadas)

### Propiedad 1: Determinismo de Ranking

**Invariante:** Mismo input (profile, features.csv, weights, filters) → mismo ranking Top-5 en 100% ejecuciones.

**Validación:**
- Test de determinismo en `tests/test_scoring_determinism.py`
- Ejecuta `rank_and_filter()` 100 veces con input idéntico
- Verifica que resultado es idéntico byte-a-byte

```python
def test_determinism():
    for _ in range(100):
        result = scoring_engine.rank_and_filter(profile, features_df)
        assert result == first_result  # Comparación completa
```

---

### Propiedad 6: Validez de Pesos

**Invariante:** Pesos siempre cumplen: cada wᵢ ∈ [0,1] AND Σwᵢ ∈ [0.99, 1.01]

**Validación:**
- Test en `tests/test_llm_weights_validity.py`
- Genera 1000 pesos aleatorios vía `calculate_weights_from_ranking()`
- Verifica invariante en todos

```python
def test_weights_validity():
    for _ in range(1000):
        weights = calculate_weights_from_ranking(random_order)
        assert all(0 <= w <= 1 for w in weights.values())
        assert 0.99 <= sum(weights.values()) <= 1.01
```

---

### Aislamiento de Sesión

**Invariante:** Usuario A NO accede datos de usuario B, incluso si B es atacante.

**Validación:**
- Test en `tests/test_session_isolation.py`
- Crea 2 sesiones, intenta acceso cruzado
- Verifica HTTP 403 o datos vacíos

```python
def test_isolation():
    session_a = auth_service.create_session()
    session_b = auth_service.create_session()
    
    # Intentar acceso cruzado
    result = feedback_storage.get_feedback(session_b['session_id'], 
                                            session_a['session_id'])
    assert result is None or len(result) == 0
```

---

### Propiedad 4: Reproducibilidad

**Invariante:** Ranking recalculable offline con snapshot + config + prompt version.

**Validación:**
- Test en `tests/test_scoring_reproducibility.py`
- Guarda snapshot en turno 1
- Recalcula ranking en turno 2 con mismo snapshot
- Verifica ranking idéntico

---

### Propiedad 5: Confianza Monótona

**Invariante:** Agregar información → confidence nunca disminuye.

**Validación:**
- Test en `tests/test_confidence_monotonicity.py`
- Secuencia de perfiles: vacío → +riasec → +pesos → +filtros
- Verifica: conf[i] >= conf[i-1]

---

## Estructura de Proyecto (AWS Versión)

```
careermatch-peru/
│
├── README.md
│   └── Setup, variables AWS, estructura
│
├── pyproject.toml
│   └── Gestión de dependencias con uv
├── uv.lock
│   └── Lock file generado automáticamente por uv
│
├── .env.example
│   └── Template AWS (AURORA_ENDPOINT, JWT_SECRET, etc.)
│
├── .gitignore
│   └── *.pyc, .env, data/, snapshots/, venv/, __pycache__/
│
├── Dockerfile
│   └── Image para Fargate (Python 3.10, FastAPI, Uvicorn)
│
├── docker-compose.yml
│   └── Local: backend Fargate + PostgreSQL local
│
├── backend/
│   ├── __init__.py
│   ├── app.py                   # FastAPI Fargate entrypoint
│   ├── auth.py                  # JWT sin estado
│   ├── session.py               # SessionManager (in-memory TTL)
│   ├── llm_service.py           # Bedrock Claude
│   ├── scoring.py               # ScoringEngine determinístico
│   ├── rag.py                   # pgvector Aurora
│   ├── embedding.py             # Bedrock Titan Embeddings
│   ├── feedback.py              # Aurora PostgreSQL persistence
│   ├── orchestration.py         # Orchestration (Bloques A/B/C)
│   ├── models.py                # Pydantic StudentProfile, etc.
│   ├── config.py                # Config centralizada (AWS vars)
│   └── utils.py                 # Logging JSON, helpers AWS
│
├── backend_lambda/              # Handlers Lambda (PRODUCCIÓN — fuera de alcance demo)
│   ├── session_handler.py       # POST /session/create
│   ├── feedback_handler.py      # POST /feedback
│   └── universities_handler.py  # GET /universities
│                                # ⚠ Demo: estos endpoints se sirven desde backend/app.py
│
├── data_pipeline/
│   ├── __init__.py
│   ├── ingestion.py             # Selenium Ponte en Carrera
│   ├── data_clean.py            # Limpieza pandas
│   ├── feature_engineering.py   # Imputación + normalización
│   ├── riasec_tagging.py        # Bedrock few-shot + validation
│   ├── config.py                # Config AWS S3, Bedrock
│   └── utils.py                 # Helpers pipeline
│
├── frontend/
│   ├── index.html
│   ├── styles.css
│   ├── app.js                   # CareerMatchApp (clase JS)
│   ├── components/
│   │   ├── chat.js
│   │   ├── ranking.js
│   │   ├── feedback.js
│   │   └── rag_panel.js
│   └── utils/
│       ├── api.js               # Wrappers fetch
│       ├── auth.js              # localStorage tokens
│       └── storage.js           # Helpers
│
├── data/
│   ├── .gitkeep
│   ├── raw.xlsx                 # No versionar
│   ├── filtered.csv             # No versionar
│   ├── features.csv             # No versionar
│   ├── feature_config.json      # No versionar
│   └── AFINIDAD_METHOD.md
│
├── snapshots/
│   ├── raw/                     # No versionar
│   ├── features/                # No versionar
│   └── configs/                 # No versionar
│
├── tests/
│   ├── conftest.py
│   ├── test_auth.py
│   ├── test_llm_service.py
│   ├── test_scoring_engine.py
│   ├── test_feedback_storage.py
│   ├── test_orchestration.py
│   ├── test_scoring_determinism.py
│   ├── test_llm_weights_validity.py
│   ├── test_session_isolation.py
│   ├── test_scoring_reproducibility.py
│   ├── test_confidence_monotonicity.py
│   ├── integration/
│   │   ├── test_chat_flow.py
│   │   ├── test_rag_flow.py
│   │   └── test_session_persistence.py
│   └── fixtures/
│       └── sample_features.csv
│
├── docs/
│   ├── ARCHITECTURE.md          # AWS: Bedrock, Aurora, DynamoDB, Lambda, Fargate
│   ├── API.md                   # REST endpoints (/chat, /feedback, /rag)
│   ├── DEVELOPMENT.md           # Setup local, convenciones
│   ├── DEPLOYMENT.md            # AWS SAM, Fargate, CloudFormation
│   ├── DECISIONS.md             # Log de decisiones (AWS vs. local)
│   └── AWS_SETUP.md             # Guía AWS (Bedrock, Aurora, IAM, etc.)
│
├── scripts/
│   ├── run_pipeline.py          # Master pipeline script
│   ├── setup_aurora.py          # Crear tablas Aurora
│   ├── setup_dynamodb.py        # Crear tabla universities
│   ├── setup_bedrock.py         # Verificar modelo Bedrock disponible
│   └── test_coverage.sh         # Reporte cobertura
│
├── infra/                       # Infrastructure as Code (PRODUCCIÓN — fuera de alcance demo)
│   ├── cloudformation/
│   │   ├── aurora.yaml          # Aurora PostgreSQL stack
│   │   ├── dynamodb.yaml        # DynamoDB table
│   │   ├── lambda.yaml          # Lambda functions
│   │   └── fargate.yaml         # Fargate service
│   └── terraform/               # Alternativa a CloudFormation
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│                                # ⚠ Demo: se usa docker-compose + PostgreSQL local
│
└── .github/
    └── workflows/               # GitHub Actions (opcional)
        ├── ci.yml               # Tests, cobertura
        └── deploy.yml           # Deploy a AWS
```

---

## Checklist de Implementación

### Fase 1: Configuración AWS

- [ ] Crear AWS Account (o usar existente)
- [ ] Crear IAM user con permisos: Bedrock, Aurora, DynamoDB, Lambda, Fargate, S3, CloudWatch
- [ ] Descargar AWS credentials
- [ ] Instalar AWS CLI y configurar profile
- [ ] Verificar acceso a Bedrock: `aws bedrock list-foundation-models`

### Fase 2: Recursos AWS

- [ ] Crear Aurora PostgreSQL cluster (1x instance `db.t3.medium` para demo)
- [ ] Crear DynamoDB tabla `universities`
- [ ] Crear S3 bucket para snapshots (opcional)
- [ ] Crear CloudWatch log group para Fargate
- [ ] Crear ECR repository para imagen Docker
> ⚠ Nota: Para la demo local, estos recursos no son necesarios — se usa PostgreSQL local (docker-compose) en lugar de Aurora, y stubs para DynamoDB. Los recursos AWS son necesarios solo para despliegue de producción.

### Fase 3: Desarrollo Local

- [ ] Clonar repositorio
- [ ] Instalar Python 3.10+ y `uv` (ver [astral.sh/uv](https://docs.astral.sh/uv/#installation))
- [ ] `uv sync`
- [ ] Copiar `.env.example` → `.env` y llenar variables AWS
- [ ] `docker-compose up -d` (backend + PostgreSQL local para testing)

### Fase 4: Backend

- [ ] Implementar `backend/app.py` (FastAPI endpoints)
- [ ] Implementar `backend/llm_service.py` (Bedrock Claude)
- [ ] Implementar `backend/scoring.py` (determinístico)
- [ ] Implementar `backend/feedback.py` (Aurora)
- [ ] Tests unitarios: `uv run pytest tests/test_*.py -v`

### Fase 5: Pipeline de Datos

- [ ] Implementar `data_pipeline/ingestion.py` (Selenium)
- [ ] Implementar `data_pipeline/feature_engineering.py` (normalización)
- [ ] Implementar `data_pipeline/riasec_tagging.py` (Bedrock few-shot)
- [ ] Generar `data/features.csv` (snapshot 1)
- [ ] Tests: `uv run pytest tests/test_pipeline_*.py -v`

### Fase 6: Frontend

- [ ] Implementar `frontend/app.js` (CareerMatchApp)
- [ ] Testing manual: `/chat` → `/feedback` → `/rag`
- [ ] Responsividad en móvil

### Fase 7: Despliegue AWS (PRODUCCIÓN — fuera de alcance demo)

> ⚠ Para la demo, el despliegue es simplemente `docker-compose up -d`. Los pasos siguientes son para la arquitectura de producción (Lambda + Fargate + API Gateway).

- [ ] Build Docker image: `docker build -t careermatch:latest .`
- [ ] Push a ECR: `aws ecr get-login-password | docker login ...`
- [ ] Crear Fargate service (ECS) con image ECR
- [ ] Configurar ALB + VPC Link (Lambda → Fargate)
- [ ] Crear Lambda functions para `/session/create`, `/feedback`, `/universities`
- [ ] Configurar API Gateway (expone `/chat` vía Fargate, otros vía Lambda)

### Fase 8: Testing & Monitoring

- [ ] Verificar cobertura: `uv run pytest --cov --cov-report=html`
- [ ] Setup CloudWatch alarms (Bedrock throttling, Aurora CPU, etc.)
- [ ] Tests de carga (optional): verificar latencias
- [ ] Verificar properties de corrección: determinismo, isolation, etc.

---

**Fin de la Sección de Stack Tecnológico (AWS)**
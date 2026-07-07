# Plan de Implementación: CareerMatch Perú — Agente de Orientación Vocacional (AWS)

## Descripción General

La implementación avanza en cinco fases estrictamente incrementales sobre AWS, cada una dejando el sistema en un estado funcional, integrado y validado: primero se construye el pipeline de datos (`ingestion`, `data_clean`, `feature_engineering`) y el `ScoringEngine` determinístico como fundación aislada sin dependencias externas; luego se implementa el flujo conversacional completo (Bloques A/B/C, interpretación LLM vía Bedrock, sesión en memoria Fargate) como servicio funcional de punta a punta con fixtures locales; después se integra el backend híbrido real (Lambda stateless para `/session/create` y `/feedback`, Aurora PostgreSQL para persistencia, DynamoDB para georreferencia, JWT sin estado); luego se añaden capacidades secundarias opcionales (RAG con pgvector, embeddings Bedrock Titan, visualización de mapa); y finalmente se cierra con etiquetado RIASEC a escala completa (6,208 carreras vía Bedrock batch), tests de integración con fixtures reales, validación de propiedades de corrección y documentación final. Cada fase termina con un checkpoint explícito que verifica progreso integrado e inspecciona antes de continuar.

## Tareas

- [ ] 1. Setup del proyecto
  - [ ] 1.1 Crear estructura de directorios: `data_pipeline/`, `backend/`, `backend/lambda/`, `backend/persistence/`, `frontend/`, `infra/`, `tests/`, `tests/integration/`, `tests/fixtures/`, `data/`, `snapshots/`, `snapshots/raw/`, `snapshots/features/`, `snapshots/configs/`, `docs/`, `scripts/`, `notebooks/`.
  - [ ] 1.2 Inicializar proyecto con `uv` y agregar dependencias especificadas en `design.md` § Stack Tecnológico:
    - Ejecutar `uv init` para crear `pyproject.toml` base.
    - Agregar dependencias con `uv add`:
      ```bash
      uv add "boto3>=1.28.0" "fastapi>=0.104.0" "uvicorn>=0.24.0" "pydantic>=2.0.0" "psycopg2-binary>=2.9.0" "pgvector>=0.1.8" "sqlalchemy>=2.0.0" "PyJWT>=2.8.0" "cryptography>=41.0.0" "python-dotenv>=1.0.0" "pandas>=2.0.0" "numpy>=1.24.0" "openpyxl>=3.10.0" "selenium>=4.15.0" "requests>=2.31.0" "moto>=4.2.0"
      uv add --dev "pytest>=7.0.0" "pytest-asyncio>=0.21.0" "pytest-cov>=4.1.0"
      ```
    - `uv sync` genera `uv.lock` automáticamente.
  - [ ] 1.3 Crear `.env.example` con variables de entorno: `AWS_REGION=us-east-1`, `LLM_PROVIDER=bedrock`, `LLM_MODEL=anthropic.claude-3-5-sonnet-20241022`, `EMBEDDING_PROVIDER=bedrock`, `EMBEDDING_MODEL=amazon.titan-embed-text-v2`, `AURORA_ENDPOINT=...`, `AURORA_PORT=5432`, `AURORA_USER=postgres`, `AURORA_PASSWORD=...`, `AURORA_DATABASE=careermatch`, `DYNAMODB_TABLE_UNIVERSITIES=universities`, `JWT_SECRET=...`, `JWT_ALGORITHM=HS256`, `SESSION_TIMEOUT_MINUTES=480`, `LOG_LEVEL=INFO`, `ENVIRONMENT=development`.
  - [ ] 1.4 Crear `pytest.ini` con configuración: `testpaths = tests`, `python_files = test_*.py`, `asyncio_mode = auto`, `addopts = --tb=short -v`, markers para `unit`, `integration`, `property`. Configurar `.coveragerc` con `source = backend, data_pipeline`, umbrales mínimos 60%.
  - [ ] 1.5 Crear `README.md` inicial con: descripción del proyecto, instrucciones de instalación (`uv sync`), guía de variables de entorno, estructura de directorios, fases de desarrollo, y comandos para ejecutar tests (`uv run pytest tests/ --cov`).
  - [ ] 1.6 Crear `Dockerfile` para servicio Fargate: imagen `python:3.10-slim`, instalar `uv` vía `COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/`, instalar dependencias de sistema (`gcc`, `postgresql-client`), copiar `pyproject.toml` y `uv.lock`, ejecutar `uv sync --no-dev`, exponer puerto 8000, healthcheck para `/health`, entrypoint `uv run uvicorn backend.app:app --host 0.0.0.0 --port 8000`.
  - [ ] 1.7 Crear `docker-compose.yml` para desarrollo local: servicio `backend` (imagen from Dockerfile, ports 8000:8000, env vars, volumes para código y datos), servicio `postgres_db` (imagen postgres:15-alpine, ports 5432:5432, env POSTGRES_USER/PASSWORD/DB), volumen `postgres_data`, red `careermatch_net`.

---

## Fase 1 — Fundación: Pipeline de Datos y Motor de Scoring (Alta): Datos Reproducibles y Ranking Determinístico sin Dependencias Externas

- [ ] 2. Implementar `data_pipeline/ingestion.py` — Descarga Automatizada
  - [ ] 2.1 Implementar clase `DataIngestion` con método `download(self) -> str`:
    - Inicializa cliente Selenium Chrome con opciones headless.
    - Navega a MINEDU_URL (variable entorno, default `https://ponte.minedu.gob.pe`).
    - Ejecuta búsqueda: espera elemento `btnBuscar` (máximo 30s) y hace clic.
    - Espera 5s a que descarga se inicie.
    - Ejecuta descarga: espera elemento `descargarDondeEstudioExcel` (máximo 30s) y hace clic.
    - Espera a que descarga complete (máximo 60s total).
    - Si descarga completa: copia archivo a `data/raw.xlsx`, crea snapshot en `snapshots/raw/raw_{timestamp}.xlsx`, retorna ruta a `data/raw.xlsx`.
    - Si timeout o error: loguea `WARNING` y retorna ruta a última versión conocida en `snapshots/raw/` (fallback).
    - Cierra navegador al finalizar.
    _Requerimientos: 10.3_
  - [ ] 2.2 Implementar clase `DataCleaner` con método `clean(self) -> pd.DataFrame`:
    - Constructor: lee archivo Excel con `pd.read_excel(raw_file, header=6)`.
    - Método `standardize_columns()`: renombra columnas a snake_case (mapeo exacto según catálogo MINEDU).
    - Método `remove_empty_rows()`: elimina filas donde `career` o `institution` es nulo; loguea cantidad eliminada.
    - Método `convert_types()`: convierte columnas numéricas (`duration_years`, `monthly_income`, `annual_cost`, `admission_rate`) usando `pd.to_numeric(..., errors='coerce')`.
    - Método `clean()`: ejecuta pipeline: `standardize_columns() → remove_empty_rows() → convert_types()`.
    - Método `save(output_path)`: guarda CSV con encoding `utf-8-sig`.
    _Requerimientos: 6.1_
  - [ ] 2.3 Tests: `test_data_clean.py` — fixture con 20 filas (15 válidas, 5 con valores faltantes críticos):
    - Verifica que `remove_empty_rows()` descarta exactamente 5 filas sin `career` o `institution`.
    - Verifica que `convert_types()` convierte correctamente `duration_years` a float.
    - Verifica que `clean()` retorna DataFrame con 15 filas, estructura correcta, sin valores `NaN` en columnas críticas.

- [ ] 3. Implementar `data_pipeline/feature_engineering.py` — Imputación y Normalización
  - [ ] 3.1 Implementar función `normalize_variable(series: pd.Series, log_transform: bool, invert: bool) -> pd.Series`:
    - Si `log_transform=True`: aplica `np.log1p(series)`.
    - Calcula `min_val`, `max_val` de la serie transformada.
    - Si `min_val == max_val`: retorna Serie con todos los valores = 1.0 (caso degenerado).
    - Si no: aplica `(series - min_val) / (max_val - min_val)` (MinMax).
    - Si `invert=True`: retorna `1 - resultado_normalizado`.
    - Retorna Serie normalizada en `[0, 1]`.
    _Requerimientos: 6.1_
  - [ ] 3.2 Implementar clase `FeatureEngineer` con método `run_pipeline(self) -> pd.DataFrame`:
    - Constructor: carga `cleaned_csv`, intenta cargar `feature_config.json` (si no existe, crea defaults con fallbacks: `duration_years_fallback=4.0`, `monthly_income_fallback=2500.0`, `annual_cost_fallback=1000.0`, `admission_rate_fallback=30.0`).
    - Método `create_imputation_flags()`: crea flags booleanos para columnas que serán imputadas:
      - `duration_imputed_flag = (duration_years <= 0) OR (duration_years > 10) OR (isna)`.
      - `income_imputed_flag = (monthly_income <= 0) OR (isna)`.
      - Análogo para `annual_cost` y `admission_rate`.
    - Método `hierarchical_imputation()`: para cada variable inválida:
      - Nivel 1: intenta mediana por `(career_family, management_type)`.
      - Nivel 2: intenta mediana por `career_family`.
      - Nivel 3: usa fallback desde config.
      - Crea columna `{variable}_imputed` con valores imputados.
    - Método `validate_ranges()`: reajusta valores tras imputación:
      - `duration_imputed` → clamp a `[3, 7]`.
      - `admission_rate_imputed` → clamp a `[0, 90]`.
      - Log warnings si se hace reajuste.
    - Método `normalize_features()`:
      - `income_norm = normalize_variable(monthly_income_imputed, log=True, invert=False)` (mayor ingreso = mejor).
      - `cost_norm = normalize_variable(annual_cost_imputed, log=True, invert=True)` (invertir: mayor = más barato).
      - `admission_norm = normalize_variable(admission_rate_imputed, log=False, invert=False)` (mayor tasa = más fácil acceso).
      - `duration_norm = normalize_variable(duration_years_imputed, log=False, invert=True)` (invertir: mayor = más corta).
      - **Nota**: todas las normas quedan orientadas "mayor = mejor para el estudiante". El scoring NO debe volver a invertir.
    - Método `save_features(output_path)`: guarda CSV con encoding `utf-8-sig`.
    - Método `save_config()`: guarda `feature_config.json`.
    - Método `create_snapshot()`: genera snapshots versionados en `snapshots/features/` y `snapshots/configs/` con timestamp.
    - Método `run_pipeline()`: ejecuta secuencia completa.
    _Requerimientos: 6.1_
  - [ ] 3.3 Tests: `test_feature_engineering.py` — fixture de 20 filas:
    - Verifica que `normalize_variable` retorna valores en `[0, 1]` para todos los casos (log, invert, ambos, ninguno).
    - Verifica que `invert=True` invierte correctamente el orden: mayor input → menor output.
    - Verifica que `run_pipeline` genera columnas `income_norm`, `cost_norm`, `admission_norm`, `duration_norm`, todas en `[0, 1]`.
    - Verifica que fixtures con valores extremos/faltantes se imputan correctamente sin errores.

- [ ] 4. Implementar `data_pipeline/riasec_tagging.py` — Etiquetado RIASEC (Bedrock + Validación Muestral)
  - [ ] 4.1 Implementar función `tag_careers_with_bedrock(catalog_df: pd.DataFrame, seed_examples: pd.DataFrame) -> pd.DataFrame`:
    - Inicializa cliente `boto3.client('bedrock-runtime', region_name=AWS_REGION)`.
    - Para cada carrera SIN `riasec_profile`:
      - Construye prompt few-shot con 10 carreras semilla (nombre, family, descripción → código RIASEC esperado).
      - Agrega carrera objetivo (nombre, family, descripción).
      - Invoca Bedrock: `invoke_model(modelId='anthropic.claude-3-5-sonnet-20241022', body=json.dumps(prompt_dict))`.
      - Parsea respuesta JSON: `{"riasec_profile": "XXX", "confidence": float}`.
      - Si JSON válido: asigna `riasec_profile`, `riasec_source='llm_tagged'`.
      - Si JSON inválido: reintenta máximo 3 veces.
      - Si falla tras 3 reintentos: asigna `riasec_source='pending'`, deja `riasec_profile=None` (será fallback luego).
    - Loguea cada intento y resultado para auditoría.
    - Retorna DataFrame con columnas `riasec_profile` y `riasec_source` actualizadas.
    _Requerimientos: 6.2_
  - [ ] 4.2 Implementar función `validate_sample(df: pd.DataFrame, sample_size: int = 300, seed: int = 42) -> None`:
    - Filtra filas con `riasec_source == 'llm_tagged'`.
    - Extrae muestra aleatoria determinística: `df.sample(n=sample_size, random_state=seed)`.
    - Exporta a `data/riasec_validation_sample.csv` con columnas: `id, career, riasec_profile, riasec_source, revisado_por, correcto, notas`.
    - Loguea path del archivo exportado para revisión humana.
    _Requerimientos: 6.3_
  - [ ] 4.3 Implementar función `apply_family_fallback(df: pd.DataFrame) -> pd.DataFrame`:
    - Para cada fila con `riasec_source == 'pending'`:
      - Calcula moda de `riasec_profile` por `career_family` (excluyendo filas `pending`).
      - Asigna moda a `riasec_profile`, `riasec_source='family_fallback'`.
    - Verifica que al finalizar NO hay filas con `riasec_profile == None` (si las hay, raise `ValueError`).
    - Loguea cantidad de filas fallback aplicadas.
    - Retorna DataFrame con garantía: 100% filas tienen `riasec_profile` no-nulo.
    _Requerimientos: 6.4, 10.3_
  - [ ] 4.4 Tests: `test_riasec_tagging.py`:
    - Mockea cliente Bedrock: respuesta válida `{"riasec_profile": "RIA", "confidence": 0.95}`, respuesta inválida (JSON malformado), timeout.
    - Fixture: 10 carreras (5 con `riasec_profile` semilla, 5 sin). Verifica:
      - `tag_careers_with_bedrock` retorna DataFrame con 5 carreras taggeadas (`llm_tagged`) y 5 sin cambios.
      - `apply_family_fallback` rellena las 5 pendientes con moda de su familia.
      - Al finalizar: 100% filas tienen `riasec_profile` no-nulo.
    - Verifica que intenta 3 veces antes de marcar como `pending`.

- [ ] 5. Implementar `backend/models.py` — Modelos Pydantic/Dataclasses
  - [ ] 5.1 Implementar dataclass `StudentProfile`:
    - Bloque A: `riasec_scores: dict[str, int]` (valores [1,10]), `riasec_code: str | None` (3 letras o None).
    - Bloque B: `w_afinidad: float`, `w_ingreso: float`, `w_costo: float`, `w_admision: float`, `w_duracion: float` (todos [0,1]).
    - Bloque C: `location: str | None` (región/ubicación del estudiante, se mapea a columna `location` en features.csv), `management_type: str | None` ('Pública' | 'Privada', se mapea a columna `management_type` en CSV), `presupuesto_max: float | None`, `modalidad: str | None`.
    - NOTA: `institution_type` (Universidad/Instituto) es una columna separada en features.csv y NO es el filtro management_type del Figma.
    - Contexto: `interests: list[str]`, `strengths: list[str]`, `preferred_fields: list[str]`, `dislikes: list[str]`.
    - Estado: `confidence_score: float` ([0,1]), `confidence_reasoning: str`.
    - Defaults: si Bloque B no se prioriza, cada peso = 0.2; si usuario dice "no me importa X", peso = 0 (re-normalizar después).
    _Requerimientos: 1.1, 1.3, 2.3_
  - [ ] 5.2 Implementar dataclass `RankingItem`:
    - `rank: int`, `id: str`, `career: str`, `institution: str`.
    - `concordancia_score: float` ([0,1]).
    - `scores_by_criterion: dict` con keys `{afinidad, ingreso, costo, admision, duracion}` (valores [0,1]).
    - `datos_verificables: dict` con keys `{monthly_income_imputed, annual_cost_imputed, admission_rate_imputed, duration_years_imputed}`.
    - `explicacion: str`.
    _Requerimientos: 5.4_
  - [ ] 5.3 Implementar dataclass `SessionContext`:
    - `session_id: str`, `profile: StudentProfile`, `conversation_history: list[dict]` (cada turno: `{role, content, timestamp}`).
    - `turn_count: int` (máx 4), `dialogue_state: str` ('gathering_info' | 'ready_for_recommendation').
    - `created_at: float`, `ttl_seconds: int = 1800`.
    _Requerimientos: 2.1, 8.2_
  - [ ] 5.4 Implementar método `StudentProfile.validate_score_ranges()`:
    - Verifica cada `riasec_score` en `[1, 10]`.
    - Lanza `ValueError` si alguno está fuera de rango.
    _Requerimientos: 1.1_
  - [ ] 5.5 Tests: `test_models.py` — instancia cada dataclass:
    - Valores válidos: verifica que se crean sin error.
    - Valores inválidos (ej riasec_score=0 o 11): verifica `ValueError`.
    - Verifica defaults (pesos=0.2, ttl_seconds=1800).

- [ ] 6. Implementar `backend/scoring.py` — ScoringEngine (Determinístico)
  - [ ] 6.1 Implementar función `calculate_riasec_code(riasec_scores: dict[str, int]) -> str`:
    - Ordena 6 scores descendente por valor.
    - Empate resuelto por orden alfabético de la letra (determinismo).
    - Retorna string de 3 letras (top 3).
    - Ejemplo: `{R:8, I:9, A:7, S:6, E:5, C:4}` → `"IRA"`.
    _Requerimientos: 1.2_
  - [ ] 6.2 Implementar función `calculate_affinity(student_code: str, career_profile: str) -> float`:
    - Iterar sobre cada letra en `career_profile` (3 letras).
    - Para cada letra, buscar posición en `student_code`:
      - Posición 0 (index 0): +3 puntos.
      - Posición 1 (index 1): +2 puntos.
      - Posición 2 (index 2): +1 punto.
      - No encontrada: 0 puntos.
    - `score_crudo = suma de puntos`.
    - `afinidad_norm = score_crudo / 6.0` (máximo posible: 3+2+1 = 6).
    - Retorna valor en `[0, 1]`.
    _Requerimientos: 5.1_
  - [ ] 6.3 Implementar función `calculate_weights_from_ranking(order: list[str]) -> dict[str, float]`:
    - Input: `order = ["afinidad", "ingreso", "costo", "admision", "duracion"]` (en orden de prioridad).
    - Asigna puntaje: 5, 4, 3, 2, 1 según posición en `order`.
    - Calcula suma (15).
    - Retorna dict con pesos normalizados: cada peso = puntaje / 15.
    - Ejemplo: order[0]="afinidad" → w_afinidad = 5/15 ≈ 0.333.
    _Requerimientos: 2.1_
  - [ ] 6.4 Implementar función `validate_weights(weights: dict) -> dict`:
    - Calcula suma de pesos.
    - Si suma no está en `[0.99, 1.01]`: re-normaliza dividiendo cada peso entre suma.
    - Soporta pesos = 0 explícitos (re-normaliza el resto).
    - Loguea warning si se hace ajuste.
    - Retorna dict con pesos válidos.
    _Requerimientos: 2.2, 2.3_
  - [ ] 6.5 Implementar función `calculate_score(weights: dict, afinidad_norm: float, ingreso_norm: float, costo_norm: float, admission_norm: float, duracion_norm: float) -> float`:
    - Todas las normas ya están orientadas "mayor = mejor" por el pipeline de datos; los parámetros reciben los valores desde columnas `income_norm`, `cost_norm`, `duration_norm`:
      - `ingreso_norm` recibe `row["income_norm"]` (mayor ingreso = mejor).
      - `costo_norm` recibe `row["cost_norm"]` (mayor = más barato, ya invertido en pipeline).
      - `admission_norm` recibe `row["admission_norm"]` (mayor = más fácil acceso).
      - `duracion_norm` recibe `row["duration_norm"]` (mayor = más corta, ya invertido en pipeline).
    - **NO aplicar inversiones adicionales**.
    - Fórmula: `score = w_afinidad * afinidad_norm + w_ingreso * ingreso_norm + w_costo * costo_norm + w_admision * admission_norm + w_duracion * duracion_norm`.
    - Clamp a `[0, 1]`.
    - Retorna float.
    _Requerimientos: 5.1_
  - [ ] 6.6 Implementar clase `ScoringEngine`:
    - Constructor: carga `features.csv` en DataFrame, carga `feature_config.json`.
    - Valida schema: debe tener columnas `id, career, institution, riasec_profile, income_norm, cost_norm, admission_norm, duration_norm, location, management_type, monthly_income_imputed, annual_cost_imputed, admission_rate_imputed, duration_years_imputed`.
    - Si archivo no existe: raise `FileNotFoundError`.
    - Si schema incorrecto: raise `ValueError`.
  - [ ] 6.7 Implementar método `rank_and_filter(self, profile: StudentProfile, features_df: pd.DataFrame) -> list[RankingItem]`:
    - Paso 1: Calcula `riasec_code` desde `profile.riasec_scores`.
    - Paso 2: Aplica filtros duros (ignorar los con valor None, que significa "me da igual"):
      - Si `profile.location` es not None: retener solo filas donde `location == profile.location`.
      - Si `profile.management_type` es not None: retener solo filas donde `management_type == profile.management_type` (usar capitalización: 'Pública'/'Privada' vs. perfil guarda 'publica'/'privada').
      - Si `profile.presupuesto_max` es not None: retener solo filas donde `annual_cost_imputed <= profile.presupuesto_max`.
      - Si `profile.modalidad` es not None: retener solo filas donde `modalidad == profile.modalidad`.
    - Paso 3: Para cada fila restante:
      - Calcula afinidad: `calculate_affinity(riasec_code, row['riasec_profile'])`.
      - Si `riasec_profile` es None: afinidad = 0.5 (fallback), loguea warning.
      - Calcula score: `calculate_score(weights, afinidad, row['income_norm'], ...)`.
      - Crea dict con `{rank, id, career, institution, concordancia_score, scores_by_criterion, datos_verificables}`.
    - Paso 4: Ordena descendente por `concordancia_score`. Desempate: orden alfabético por `institution`.
    - Paso 5: Asigna ranks (1, 2, 3, ...).
    - Paso 6: Retorna lista completa (sin truncar; truncado a Top-5 es responsabilidad del llamador).
    _Requerimientos: 3.1, 3.2, 5.2, 5.3, 5.4, 10.1_
  - [ ] 6.8 Tests: `test_scoring.py` — fixture de 5 carreras:
    - Verifica `calculate_riasec_code` con ejemplo: `{R:8, I:9, A:7, S:6, E:5, C:4}` → `"IRA"`.
    - Verifica `calculate_affinity`: `student_code="RIA", career_profile="RIA"` → `6/6 = 1.0`.
    - Verifica `calculate_weights_from_ranking`: orden `["afinidad", "ingreso", ...]` → pesos suman 1.0.
    - Verifica `calculate_score` con valores conocidos.
    - Verifica desempate: dos carreras con scores `0.85` y `0.8499` (diff < 0.001) se ordenan alfabético por institution.
  - [ ] 6.9 Tests basados en propiedades: **Propiedad 1 — Determinismo del Ranking**. **Valida: Requerimiento 5 criterio 5.2**. Estrategia:
    - Usar `hypothesis` para generar 50 conjuntos aleatorios: `weights` válidos (suma ≈ 1.0), `features_df` (hasta 200 filas con `*_norm` en [0,1]).
    - Para cada conjunto: ejecutar `rank_and_filter` dos veces con el mismo input.
    - Invariante: ambas ejecuciones retornan el mismo orden de carreras (mismo `id` en posiciones idénticas).
  - [ ] 6.10 Tests basados en propiedades: **Propiedad 6 — Pesos Válidos**. **Valida: Requerimiento 2 criterio 2.1**. Estrategia:
    - Usar `hypothesis` para generar 100 rankings aleatorios (arrays de 5 elementos únicos).
    - Para cada ranking: ejecutar `calculate_weights_from_ranking` y `validate_weights`.
    - Invariante: pesos resultantes siempre cumplen: cada wᵢ ∈ [0,1] AND suma ∈ [0.99, 1.01].

- [ ] 7. Checkpoint Fase 1 — Validación de Fundación
  - [ ] 7.1 Ejecutar suite `uv run pytest data_pipeline/ tests/test_scoring.py --cov` con fixture reducido (50 carreras, 10 semilla RIASEC).
    - Verificar que `DataIngestion.download()` retorna path válido (o fallback si portal no disponible).
    - Verificar que `DataCleaner.clean()` retorna DataFrame válido sin filas incompletas.
    - Verificar que `FeatureEngineer.run_pipeline()` genera `features.csv` con columnas `income_norm`, `cost_norm`, `admission_norm`, `duration_norm` en `[0,1]`.
    - Verificar que `tag_careers_with_bedrock` (mockeado) tagea correctamente y `apply_family_fallback` elimina `pending`.
    - Verificar que `ScoringEngine.rank_and_filter()` produce Top-5 con scores en `[0,1]`, determinístico, desempatado correctamente.
    - Cobertura mínima 60% en `data_pipeline` y `backend/scoring.py`.
    - Ejecutar test de determinismo (Propiedad 1) y pesos válidos (Propiedad 6): ambos deben pasar.

---

## Fase 2 — Flujo Conversacional Principal (Alta): Agente End-to-End sobre Fixtures, sin Persistencia Real

- [ ] 8. Implementar `backend/llm_service.py` — LLM_Layer (Amazon Bedrock Claude)
  - [ ] 8.1 Implementar clase `LLMService`:
    - Constructor: `__init__(self, provider: str = "bedrock", model: str = "anthropic.claude-3-5-sonnet-20241022", region: str = "us-east-1")`.
    - Inicializa `self.client = boto3.client('bedrock-runtime', region_name=region)`.
    - Almacena `self.model_id = model`.
    - Loguea inicialización.
    _Requerimientos: 7.1_
  - [ ] 8.2 Implementar método `interpret_bloque_a(self, message: str, history: list[dict]) -> dict`:
    - Construye system prompt con mínimo 3 ejemplos few-shot de perfiles RIASEC.
    - Prompt incluye: mensaje usuario actual, histórico conversacional (últimos 3 turnos).
    - Invoca Bedrock: `self.client.invoke_model(modelId=self.model_id, body=json.dumps({...}))`.
    - Parsea respuesta: espera JSON `{"riasec_scores": {"R": int, ...}, "missing_letters": [str], "confidence": float}`.
    - Si JSON válido: retorna dict con `riasec_scores`, `missing_letters`.
    - Si JSON inválido: reintenta máximo 3 veces.
    - Si falla tras 3 intentos: retorna `{"error": "fallback_templated", "missing_letters": ["R", "I", "A", "S", "E", "C"]}` (plantilla genérica).
    - Loguea cada intento.
    - Timeout máximo 5s; si excede: loguea `TimeoutError`, retorna fallback.
    _Requerimientos: 1.1, 7.1_
  - [ ] 8.3 Implementar método `interpret_bloque_b(self, message: str, history: list[dict]) -> dict`:
    - System prompt detecciona si usuario da orden explícita de prioridades o ajustes de pesos.
    - Retorna JSON: `{"priority_order": list[str] | None, "explicit_weights": dict | None, "confidence": float}`.
    - `priority_order` es permutación de 5 criterios si usuario prioriza claramente.
    - `explicit_weights` es dict si usuario dice "pesos explícitos" (raro en demo).
    - Si ambos None: usuario aún no prioriza claramente.
    - Idem manejo de fallos y timeouts que Bloque A.
    _Requerimientos: 2.1, 7.1_
  - [ ] 8.4 Implementar método `interpret_bloque_c(self, message: str, history: list[dict]) -> dict`:
    - Detecta menciones de región (normaliza: "Lima" → "Lima", "arequipa" → "Arequipa"), tipo institución, presupuesto, modalidad.
    - Retorna JSON: `{"location": str | None, "management_type": "publica" | "privada" | None, "presupuesto_max": float | None, "modalidad": "presencial" | "virtual" | None, "confidence": float}`.
    - NOTA: el LLM devuelve estos nombres de campo; el Orchestration mapea a StudentProfile.location y StudentProfile.management_type directamente.
    - Si usuario dice "me da igual" o no menciona: valor = None.
    - Manejo de fallos y timeouts idem Bloques A/B.
    _Requerimientos: 3.1, 3.2, 7.1_
  - [ ] 8.5 Implementar método `_parse_and_validate_json(self, raw_response: str) -> dict`:
    - Intenta `json.loads(raw_response)`.
    - Si `JSONDecodeError`: reintenta máximo 3 veces.
    - Si falla tras 3 intentos: retorna `{"error": "fallback_templated"}` (señal para usar plantilla).
    - Loguea cada intento fallido como warning.
    _Requerimientos: 7.2_
  - [ ] 8.6 Implementar método `generate_follow_up(self, profile: StudentProfile, missing_info: list[str], history: list[dict]) -> str`:
    - Construye pregunta abierta basada en `missing_info` (ej si falta `w_ingreso`: pregunta sobre prioridad de ingresos).
    - Verifica que no repite lo ya mencionado en `history` (filtra últimos 5 turnos).
    - Responde vía Bedrock con few-shot de preguntas naturales.
    - Si LLM falla: retorna plantilla genérica por tipo de información faltante (ej "¿Hay algo más que quieras que considere?").
    - Retorna string en español peruano, conversacional.
    _Requerimientos: 7.2, 7.3_
  - [ ] 8.7 Implementar método `generate_explanation(self, ranking_item: RankingItem, profile: StudentProfile) -> str`:
    - Construye prompt: ranking_item (career, institution, scores, datos verificables), profile (interests, pesos).
    - Invoca Bedrock con few-shot de explicaciones reales.
    - Retorna string 3–5 oraciones que:
      - Menciona qué criterio coincide más con el perfil.
      - Cita al menos 1 dato verificable (ingresos, costo, tasa admisión).
      - Conecta beneficio a usuario ("es ideal para ti porque...").
    - Si LLM falla: retorna plantilla fallback: "Recomendamos [carrera] en [universidad] porque alinea bien con tus intereses ([criterios]). Datos: ingresos S/. [X]/mes, costo S/. [Y]/año, admisión [Z]%."
    - Nunca inventa datos; todos vienen de `ranking_item.datos_verificables`.
    _Requerimientos: 7.3_
  - [ ] 8.8 Tests: `test_llm_service.py`:
    - Mockea cliente Bedrock: respuesta válida, respuesta inválida x3 (JSON malformado), timeout simulado.
    - Verifica `interpret_bloque_a` con mensaje de ejemplo: extrae correctamente `riasec_scores`.
    - Verifica fallback templado cuando Bedrock falla 3 veces.
    - Verifica que `generate_follow_up` no repite información del historial.

- [ ] 9. Implementar `backend/session.py` — SessionManager
  - [ ] 9.1 Implementar clase `SessionManager`:
    - Constructor: inicializa `self._sessions: dict[str, SessionContext]` vacío.
  - [ ] 9.2 Implementar método `create_context(self, session_id: str) -> SessionContext`:
    - Crea contexto nuevo: `profile=None`, `conversation_history=[]`, `turn_count=0`, `dialogue_state='gathering_info'`, `created_at=time.time()`, `ttl_seconds=1800`.
    - Almacena en `self._sessions[session_id]`.
    - Retorna contexto.
    _Requerimientos: 8.2_
  - [ ] 9.3 Implementar método `get_context(self, session_id: str) -> SessionContext | None`:
    - Si `session_id` no existe: retorna None.
    - Si existe: verifica TTL: `time.time() - context.created_at > context.ttl_seconds`.
    - Si expirado: elimina, retorna None (limpieza lazy).
    - Si válido: retorna copia (para evitar mutaciones accidentales).
    _Requerimientos: 8.2, 10.2_
  - [ ] 9.4 Implementar método `update_context(self, session_id: str, new_data: dict) -> SessionContext`:
    - Recupera contexto existente.
    - Merge incremental: actualiza campos de `profile` con nuevos valores, sin sobreescribir con None.
    - Si `new_data` contiene `riasec_scores`: actualiza, calcula `riasec_code` vía `calculate_riasec_code`.
    - Si `new_data` contiene pesos: actualiza, valida vía `validate_weights`.
    - Actualiza `last_activity` a `time.time()`.
    - Loguea auditoría: "Updated session {session_id}: fields {list de campos actualizados}".
    - Retorna contexto actualizado.
    _Requerimientos: 10.2_
  - [ ] 9.5 Tests: `test_session_manager.py`:
    - Verifica creación, recuperación, expiración por TTL (simular `created_at` en pasado).
    - Verifica que `update_context` hace merge sin perder campos previos.
    - Verifica que campos None no sobreescriben valores previos.

- [ ] 10. Implementar `backend/orchestration.py` — Orchestration_Layer
  - [ ] 10.1 Implementar función `calculate_confidence(profile: StudentProfile) -> float`:
    - Cuenta campos completos:
      - `riasec_completos = 1` si los 6 keys en `riasec_scores` están presentes y todos != None, `0` si no.
      - `pesos_completos = count({w_afinidad, w_ingreso, w_costo, w_admision, w_duracion} != None)` (máx 5).
    - Fórmula: `confidence = (riasec_completos * 6 + pesos_completos) / 11.0`.
    - Clamp a `[0, 1]`.
    - Retorna float.
    _Requerimientos: 4.1_
  - [ ] 10.2 Implementar clase `Orchestration`:
    - Constructor: inyecta `auth_service`, `session_manager`, `llm_service`, `scoring_engine`, `rag_module` (None si no disponible), `feedback_storage`.
    - Almacena referencias a estos servicios.
  - [ ] 10.3 Implementar método `handle_turn(self, session_id: str, message: str) -> dict`:
    - Paso 1: Recupera contexto vía `session_manager.get_context(session_id)`. Si no existe, crea uno nuevo.
    - Paso 2: Agrega `message` al `conversation_history` con rol 'user', timestamp.
    - Paso 3: Determina bloque activo:
      - Si `riasec_scores` incompleto: Bloque A.
      - Elif pesos no priorizados (todos None): Bloque B.
      - Elif filtros no definidos: Bloque C.
      - Else: proceder a ranking.
    - Paso 4: Interpreta bloque activo vía `llm_service.interpret_bloque_*()`.
    - Paso 5: Actualiza contexto vía `session_manager.update_context()` con datos interpretados.
    - Paso 6: Recalcula `profile.confidence_score = calculate_confidence(profile)`.
    - Paso 7: Evalúa avance:
      - Si `confidence >= 0.70` O `turn_count >= 4`: proceder a ranking (paso 8).
      - Si no: generar pregunta seguimiento (paso 10).
    - Paso 8: RANKING:
      - Incrementa `turn_count`.
      - Marca `dialogue_state = 'ready_for_recommendation'`.
      - Si `turn_count >= 4`: loguea `{"graceful_degradation": true}` (información incompleta pero procede).
      - Invoca `scoring_engine.rank_and_filter(profile, features_df)` → lista completa.
      - Trunca a Top-5 vía `ranked_list[:5]`.
      - Para cada item: invoca `llm_service.generate_explanation(item, profile)` → agrega `item['explicacion']`.
      - Persiste en `feedback_storage.save_ranking()` (Fase 3 integra esto).
      - Retorna `{"status": "success", "ranking": {"status": "ready", "top_5": items}, "bot_message": "Aquí están mis 5 mejores recomendaciones...", "requires_follow_up": false}`.
    - Paso 9: REPREGUNTA:
      - Incrementa `turn_count`.
      - Identifica `missing_info = [campos null en profile]`.
      - Invoca `llm_service.generate_follow_up(profile, missing_info, history)`.
      - Agrega bot_message al `conversation_history`.
      - Retorna `{"status": "success", "ranking": {"status": "awaiting_info"}, "bot_message": str_pregunta, "requires_follow_up": true}`.
    - Manejo de errores:
      - Si `llm_service` falla (3 intentos): loguea, continúa con fallback.
      - Si `scoring_engine.rank_and_filter` falla (FileNotFoundError): retorna HTTP 500.
      - Si `scoring_engine` retorna lista vacía (filtros eliminan todo): retorna error amigable.
    _Requerimientos: 1.1, 2.1, 3.1, 4.1, 4.2, 4.3_
  - [ ] 10.4 Tests: `test_orchestrator.py`:
    - Simula conversación de 2 turnos: Turno 1 (riasec parcial) → Turno 2 (pesos) → confidence >= 0.70 → ranking.
    - Simula conversación de 4 turnos SIN completar información → fuerza ranking en turno 4 con `graceful_degradation=true`.
    - Verifica que `conversation_history` se va llenando correctamente.
  - [ ] 10.5 Tests basados en propiedades: **Propiedad 5 — Confidence Monótono**. **Valida: Requerimiento 4 criterio 4.1**. Estrategia:
    - Generar secuencias aleatorias de actualizaciones: `profile` comienza vacío → agrega riasec → agrega pesos → agrega filtros.
    - Invariante: `confidence_score` nunca disminuye.
    - Verificar con `hypothesis`: generar 100 secuencias aleatorias.

- [ ] 11. Implementar `backend/app.py` — FastAPI Entrypoint (Fargate)
  - [ ] 11.1 Implementar fixture de FastAPI:
    - `app = FastAPI(title="CareerMatch Perú Agent API")`.
    - Variable global `orchestration = None` (inicializada en startup).
    - Función para extraer token: `get_session_token(request: Request) -> str` desde header `Authorization: Bearer {token}` o query param `session_token`.
  - [ ] 11.2 Implementar endpoint `POST /chat`:
    - Recibe JSON: `{message: str, metadata: dict | None}`.
    - Header requerido: `Authorization: Bearer {token}`.
    - Valida token vía `auth_service.validate_token()` (Fase 3; ahora: stub que retorna session_id).
    - Invoca `orchestration.handle_turn(session_id, message, metadata)`.
    - Retorna JSON response (status, ranking, bot_message, etc.).
    - HTTP 401 si token inválido.
    - HTTP 500 si error interno (loguea sin PII).
    _Requerimientos: 8.2_
  - [ ] 11.3 Implementar endpoint `POST /feedback`:
    - Recibe JSON: `{ranking_id: str, validation_score: int [1,5], selected_career: str | None, notes: str | None}`.
    - Valida `validation_score ∈ [1, 5]`: si no, HTTP 422.
    - Invoca `feedback_storage.update_validation()` (Fase 3; ahora: stub).
    - Retorna `{status: "success", message: "..."}`.
    _Requerimientos: 8.1, 9.1_
  - [ ] 11.4 Implementar endpoint `POST /rag` (stub para Fase 4):
    - Recibe JSON: `{career: str, question: str}`.
    - Retorna `{status: "no_documents", answer: "", sources: []}` (no implementado aún).
  - [ ] 11.5 Implementar event handler `@app.on_event("startup")`:
    - Inicializa componentes:
      - `auth_service = AuthService()` (stub).
      - `session_manager = SessionManager()`.
      - `llm_service = LLMService(provider=os.getenv("LLM_PROVIDER"), model=os.getenv("LLM_MODEL"))`.
      - `scoring_engine = ScoringEngine("data/features.csv", "data/feature_config.json")`.
      - `feedback_storage = FeedbackStorage(db_url=os.getenv("AURORA_ENDPOINT"))` (stub).
      - `rag_module = None` (implementar en Fase 4).
      - `orchestration = Orchestration(...)` (inyecta dependencias).
    - Loguea "Application initialized".
  - [ ] 11.6 Implementar endpoint `GET /health`:
    - Retorna `{status: "ok"}`.
    - Usado para health check de Fargate.
  - [ ] 11.7 Tests de integración: `test_app_chat_flow.py`:
    - Usa `TestClient` de FastAPI.
    - Fixture: cargar `tests/fixtures/features_50_careers.csv` (50 carreras, 10 con RIASEC semilla).
    - Simula `/chat` request: `{message: "Me gustan matemáticas"}`.
    - Verifica response JSON structure.
    - Simula secuencia de 2+ mensajes hasta obtener ranking.
    - Verifica Top-5 retornado, scores en `[0,1]`, explicaciones presentes.

- [ ] 12. Checkpoint Fase 2 — Validación de Flujo Conversacional
  - [ ] 12.1 Levantar servicio: `docker-compose up -d` (backend + postgres local).
  - [ ] 12.2 Ejecutar conversación completa vía curl/httpie:
    ```bash
    curl -X POST http://localhost:8000/chat \
      -H "Authorization: Bearer token_xyz" \
      -H "Content-Type: application/json" \
      -d '{"message": "Me interesan matemáticas y análisis de datos"}'
    ```
    - Mensaje 1: respuesta con pregunta (requires_follow_up=true).
    - Mensaje 2: respuesta con pregunta.
    - Mensaje 3 (o antes): respuesta con Top-5 ranking (status=ready).
  - [ ] 12.3 Verificar que Top-5 tiene:
    - 5 items, ranks 1–5.
    - `concordancia_score` en `[0,1]`.
    - `scores_by_criterion` con 5 criterios.
    - `datos_verificables` con claves `monthly_income_imputed`, `annual_cost_imputed`, `admission_rate_imputed`, `duration_years_imputed`.
    - `explicacion` non-empty.
  - [ ] 12.4 Ejecutar suite: `uv run pytest tests/test_*.py tests/integration/ --cov`.
    - Cobertura ≥ 60% en `backend/llm_service.py`, `backend/session.py`, `backend/orchestration.py`, `backend/app.py`.
    - Todos los tests de propiedades (1, 5, 6) deben pasar.

---

## Fase 3 — Backend Híbrido y Persistencia (Alta): Lambda, Aurora, DynamoDB, JWT Integrados

- [ ] 13. Configurar Infraestructura AWS Base
  - [ ] 13.1 Crear tabla Aurora PostgreSQL vía AWS Console o AWS CLI:
    ```bash
    aws rds create-db-cluster \
      --db-cluster-identifier careermatch-cluster \
      --engine aurora-postgresql \
      --master-username postgres \
      --master-user-password {secure_password} \
      --region us-east-1
    ```
    - Instance type: `db.t3.medium` (demo).
    - Habilitar `pgvector` extension.
    - Anotar endpoint en `.env`: `AURORA_ENDPOINT=...amazonaws.com`.
  - [ ] 13.2 Crear tabla DynamoDB `universities` vía AWS Console o Terraform:
    - PK: `institution_id` (String).
    - Atributos: `name`, `location` (antes region), `management_type` (Pública/Privada, antes tipo_institucion), `latitude`, `longitude`, `careers_ids` (StringSet).
    - GSI: `location-index` (PK: `location`, SK: `institution_id`).
  - [ ] 13.3 Crear repositorio ECR: `careermatch-repo` en región `us-east-1`.

- [ ] 14. Implementar `infra/db_schema.sql` — Esquema Aurora PostgreSQL
  - [ ] 14.1 Crear tabla `career_chunks` (para RAG, pgvector):
    ```sql
    CREATE EXTENSION IF NOT EXISTS vector;
    CREATE TABLE career_chunks (
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
    _Requerimientos: 9.2_
  - [ ] 14.2 Crear tabla `feedback`:
    ```sql
    CREATE TABLE feedback (
        id BIGSERIAL PRIMARY KEY,
        session_id UUID NOT NULL,
        career_id TEXT NOT NULL,
        validation_score SMALLINT CHECK (validation_score BETWEEN 1 AND 5),
        created_at TIMESTAMPTZ DEFAULT now(),
        INDEX idx_session ON feedback(session_id)
    );
    ```
    _Requerimientos: 9.1, 10.2_
  - [ ] 14.3 Crear tabla `rankings`:
    ```sql
    CREATE TABLE rankings (
        id BIGSERIAL PRIMARY KEY,
        session_id UUID NOT NULL,
        ranking_json JSONB NOT NULL,
        reproducibility_metadata JSONB,
        created_at TIMESTAMPTZ DEFAULT now(),
        INDEX idx_session ON rankings(session_id)
    );
    ```
    _Requerimientos: 9.1, 10.2_

- [ ] 15. Implementar `backend/persistence/feedback_storage.py` — FeedbackStorage (Aurora)
  - [ ] 15.1 Implementar clase `FeedbackStorage`:
    - Constructor: inicializa conexión PostgreSQL vía `psycopg2.connect(dsn=AURORA_CONN_STRING)`.
    - Valida conexión; si falla: loguea `ERROR` y almacena fallbacks en queue local (in-memory).
  - [ ] 15.2 Implementar método `save_ranking(self, session_id: str, ranking_json: dict, metadata: dict) -> str`:
    - Serializa `ranking_json` y `metadata` a JSON strings.
    - Inserta en tabla `rankings`: `INSERT INTO rankings (session_id, ranking_json, reproducibility_metadata, created_at) VALUES (...)`.
    - Genera y retorna `ranking_id` (uuid4).
    - Si DB no disponible: almacena en queue local, loguea warning, retorna provisional ranking_id.
    _Requerimientos: 9.1_
  - [ ] 15.3 Implementar método `save_feedback(self, session_id: str, ranking_id: str, validation_score: int, selected_career: str | None, notes: str | None) -> None`:
    - Valida `validation_score ∈ [1, 5]`: si no, lanza `ValueError`.
    - Inserta en tabla `feedback`: `INSERT INTO feedback (session_id, career_id, validation_score, created_at) VALUES (...)` (or updateranking_json si es update).
    - Si DB no disponible: queue local.
    _Requerimientos: 9.1_
  - [ ] 15.4 Implementar método `get_feedback_by_session(self, session_id: str) -> list[dict]`:
    - Consulta: `SELECT * FROM feedback WHERE session_id = %s`.
    - Filtra siempre por `session_id` (aislamiento garantizado).
    - Retorna lista de registros.
    _Requerimientos: 10.2_
  - [ ] 15.5 Implementar método `sync_queue_to_db(self)`:
    - Intenta sincronizar queue local a DB.
    - Si DB ahora disponible: persiste registros pendientes.
    - Loguea éxito/fallo.
  - [ ] 15.6 Tests: `test_feedback_storage.py`:
    - Usa `testcontainers` para Postgres local o `postgres:15-alpine` en docker-compose.
    - Verifica inserción válida de ranking y feedback.
    - Verifica `ValueError` si `validation_score` fuera de rango.
    - Verifica que `get_feedback_by_session` no retorna registros de otra sesión.
    - Verifica que DB error dispara fallback queue local.

- [ ] 16. Implementar `backend/lambda/auth_service.py` — Auth_Service (JWT)
  - [ ] 16.1 Implementar clase `AuthService`:
    - Constructor: almacena `self.secret = os.getenv("JWT_SECRET")`, `self.algorithm = os.getenv("JWT_ALGORITHM", "HS256")`, `self.timeout_minutes = int(os.getenv("SESSION_TIMEOUT_MINUTES", 480))`.
    - Almacena tokens activos en dict en-memoria (Fase 3 demo): `self._active_tokens = {}`.
  - [ ] 16.2 Implementar método `create_session(self) -> dict`:
    - Genera `session_id = str(uuid.uuid4())`.
    - Crea JWT con payload `{session_id, exp: now + timeout_minutes, iat: now}`.
    - Firma con `JWT_SECRET` y `HS256`.
    - Almacena en `self._active_tokens[session_token] = session_id` (para validate rápido).
    - Retorna `{session_id, session_token}`.
    _Requerimientos: 8.3_
  - [ ] 16.3 Implementar método `validate_token(self, session_token: str) -> str | None`:
    - Intenta decodificar JWT con `jwt.decode(session_token, self.secret, algorithms=[self.algorithm])`.
    - Si válido: extrae `session_id` del payload, retorna.
    - Si inválido (firma, expiración): loguea, retorna None.
    - Lanza `InvalidTokenError` (custom exception) si token malformado.
    _Requerimientos: 8.3_
  - [ ] 16.4 Implementar método `revoke_token(self, session_token: str) -> None`:
    - Marca token como revocado (elimina de `_active_tokens`).
    - Loguea auditoría.
  - [ ] 16.5 Tests: `test_auth_service.py`:
    - Verifica token válido: decodifica correctamente.
    - Verifica token expirado: `validate_token` retorna None (no exception).
    - Verifica token con firma incorrecta: `InvalidTokenError`.
    - Verifica revocación: token revocado → None en siguiente validación.

- [ ] 17. Implementar `backend/lambda/session_handler.py` y `feedback_handler.py` — Lambda Handlers
  - [ ] 17.1 Implementar `session_handler.lambda_handler(event, context)`:
    - Evento API Gateway: `{httpMethod: "POST", path: "/session/create", ...}`.
    - Invoca `auth_service.create_session()`.
    - Retorna response: `{statusCode: 200, body: json.dumps({session_id, session_token})}`.
    _Requerimientos: 8.1_
  - [ ] 17.2 Implementar `feedback_handler.lambda_handler(event, context)`:
    - Evento API Gateway: `{httpMethod: "POST", path: "/feedback", body: json_string}`.
    - Parsea body: `{ranking_id, validation_score, selected_career, notes}`.
    - Valida `validation_score ∈ [1, 5]`: si no, retorna `{statusCode: 422, body: "..."}`.
    - Extrae `session_id` desde JWT en header (valida vía `auth_service.validate_token`).
    - Invoca `feedback_storage.save_feedback(...)`.
    - Retorna `{statusCode: 200, body: json.dumps({status: "success"})}`.
    _Requerimientos: 8.1, 9.1_
  - [ ] 17.3 Tests: `test_lambda_handlers.py`:
    - Mockea eventos API Gateway.
    - Verifica `/session/create` retorna token válido.
    - Verifica `/feedback` rechaza score fuera de rango (422).
    - Verifica `/feedback` rechaza token inválido (401).

- [ ] 18. Integrar JWT en `/chat` Endpoint
  - [ ] 18.1 Modificar `backend/app.py` endpoint `POST /chat`:
    - Extrae token del header `Authorization: Bearer {token}`.
    - Invoca `auth_service.validate_token(token)` → obtiene `session_id`.
    - Si token inválido: retorna HTTP 401.
    - Invoca `orchestration.handle_turn(session_id, message, metadata)` (como antes).
    _Requerimientos: 8.3_
  - [ ] 18.2 Modificar `backend/orchestration.py` método `handle_turn`:
    - Al generar ranking: invoca `feedback_storage.save_ranking(session_id, ranking_json, metadata)`.
    - Guarda `ranking_id` en contexto y retorna en respuesta JSON (para futura validación).
    _Requerimientos: 9.1_
  - [ ] 18.3 Tests de integración: `test_auth_persistence_integration.py`:
    - Flujo: `/session/create` → obtiene token.
    - `/chat` con token válido → genera ranking → persiste en Aurora.
    - `/feedback` con ranking_id → guarda validación.
    - Verifica datos persistidos en Aurora.

- [ ] 19. Implementar Georreferencia — DynamoDB Universities
  - [ ] 19.1 Crear tabla DynamoDB `universities` (vía AWS Console o Terraform):
    - PK: `institution_id` (S).
    - Atributos: `name` (S), `location` (S, antes region), `management_type` (S, Pública/Privada, antes tipo_institucion), `latitude` (N), `longitude` (N), `careers_ids` (SS).
    - GSI: `location-index` con PK=`location`, SK=`institution_id`.
    _Requerimientos: 9.3_
  - [ ] 19.2 Implementar `backend/lambda/universities_handler.py`:
    - Endpoint `GET /universities?location=Lima` (querystring optional).
    - Inicializa cliente DynamoDB: `dynamodb = boto3.resource('dynamodb', region_name=AWS_REGION)`.
    - Si `location` en params: consulta GSI `location-index` → `table.query(IndexName='location-index', KeyConditionExpression='location = :location', ExpressionAttributeValues={':location': location})`.
    - Si no: `table.scan()` (retorna todas).
    - Retorna `{statusCode: 200, body: json.dumps(items)}`.
    _Requerimientos: 9.3_
- [ ] 19.3 Tests: `test_universities_handler.py`:
    - Mockea DynamoDB con `moto`.
    - Inserta 3 universidades (2 Lima, 1 Arequipa).
    - Verifica listado completo retorna 3 items.
    - Verifica filtrado por location retorna 2 items (Lima).

- [ ] 20. Checkpoint Fase 3 — Validación de Backend Híbrido e Integración
  - [ ] 20.1 Ejecutar flujo completo contra recursos mockeados:
    - Postgres local (docker-compose) en lugar de Aurora.
    - DynamoDB mockeado con `moto`.
    - `/session/create` → obtiene token JWT válido.
    - `/chat` con token → conversa 2–4 turnos → ranking generado.
    - `/feedback` con ranking_id y score válido → persistido en Postgres.
    - `/universities` → retorna lista, filtrado por location.
  - [ ] 20.2 Verificar aislamiento por `session_id`:
    - Crear 2 sesiones distintas.
    - Session A: feedback sobre carrera X.
    - Session B: consultar feedback → no ve datos de Session A.
  - [ ] 20.3 Ejecutar suite: `uv run pytest tests/ --cov`.
    - Cobertura ≥ 60% en `backend/persistence/`, `backend/lambda/`, `backend/auth.py`.
    - Propiedades de corrección (3 — Aislamiento, 4 — Reproducibilidad) deben pasar.

---

## Fase 4 — Capacidades Secundarias (Media): RAG Opcional, Embeddings, Georreferencia Visual

- [ ] 21. Implementar `backend/embedding.py` — EmbeddingService (Bedrock Titan)
  - [ ] 21.1 Implementar clase `EmbeddingService`:
    - Constructor: `__init__(self, provider: str = "bedrock", model: str = "amazon.titan-embed-text-v2", region: str = "us-east-1")`.
    - Inicializa `self.client = boto3.client('bedrock-runtime', region_name=region)`.
    - Almacena `self.model_id = model`.
    _Requerimientos: 9.2_
  - [ ] 21.2 Implementar método `embed(self, text: str) -> list[float] | None`:
    - Construye body: `{"inputText": text}`.
    - Invoca Bedrock: `self.client.invoke_model(modelId=self.model_id, body=json.dumps(body))`.
    - Parsea respuesta JSON: espera `{"embedding": list[float]}`.
    - Verifica dimensión (1024 para Titan).
    - Timeout máximo 5s.
    - Si falla: loguea error, retorna None (no bloquea flujo principal).
    _Requerimientos: 10.3_
  - [ ] 21.3 Implementar método `embed_batch(self, texts: list[str]) -> list[list[float]] | None`:
    - Opcional: versión optimizada para múltiples textos.
    - Si no implementado: llamar `embed()` iterativamente.
  - [ ] 21.4 Implementar método `similarity(self, vec1: list[float], vec2: list[float]) -> float`:
    - Calcula similitud coseno entre dos vectores.
    - Verifica dimensiones coincidan.
    - Retorna float en `[0, 1]`.
  - [ ] 21.5 Tests: `test_embedding_service.py`:
    - Mockea cliente Bedrock.
    - Verifica `embed()` retorna vector de 1024 dimensiones.
    - Verifica timeout → retorna None.
    - Verifica `similarity()` entre dos vectores conocidos.

- [ ] 22. Implementar `backend/rag.py` — RAGModule (pgvector/Aurora)
  - [ ] 22.1 Implementar clase `RAGModule`:
    - Constructor: `__init__(self, db_url: str, embedding_service: EmbeddingService)`.
    - Inicializa conexión PostgreSQL con `psycopg2.connect(dsn=db_url)`.
    - Almacena `embedding_service`.
  - [ ] 22.2 Implementar método `is_career_available(self, career: str) -> bool`:
    - Consulta: `SELECT COUNT(*) FROM career_chunks WHERE career_id ILIKE %s`.
    - Retorna True si count > 0, False si no.
    _Requerimientos: 10.3_
  - [ ] 22.3 Implementar método `query(self, career: str, question: str, top_k: int = 3) -> dict`:
    - Paso 1: Verifica si carrera disponible vía `is_career_available()`. Si no: retorna `{status: "no_documents", answer: "", sources: []}`.
    - Paso 2: Genera embedding de `question` vía `embedding_service.embed(question)`.
    - Si embedding falla: retorna `{status: "error", ...}`.
    - Paso 3: Ejecuta similarity search en Aurora:
      ```sql
      SELECT content, metadata FROM career_chunks 
      WHERE career_id = %s 
      ORDER BY embedding <=> %s LIMIT %k
      ```
    - Paso 4: Pasa chunks recuperados + pregunta a LLM (`llm_service.generate_explanation`) para generar respuesta.
    - Paso 5: Extrae y retorna fuentes desde `metadata`.
    - Timeout total máximo 3s.
    - Retorna `{status: "success", answer: str, sources: list[str], confidence: float}`.
    _Requerimientos: 10.3_
  - [ ] 22.4 Implementar método `index_career_chunks(self, career: str, documents: list[str], metadata: dict | None = None) -> None`:
    - Divide cada documento en chunks (~500 caracteres, 50 char overlap).
    - Genera embeddings para cada chunk vía `embedding_service.embed_batch()`.
    - Inserta en tabla `career_chunks`:
      ```sql
      INSERT INTO career_chunks (career_id, content, embedding, metadata) 
      VALUES (%s, %s, %s, %s)
      ```
    - Loguea cantidad de chunks indexados.
    _Requerimientos: 9.2_
  - [ ] 22.5 Implementar método `get_available_careers(self) -> list[str]`:
    - Consulta: `SELECT DISTINCT career_id FROM career_chunks`.
    - Retorna lista de nombres.
  - [ ] 22.6 Tests: `test_rag.py`:
    - Usa Postgres local con pgvector.
    - Indexa 3 chunks de ejemplo para carrera "Estadística".
    - Verifica `query("Estadística", "¿Qué cursos lleva?")` retorna respuesta con sources.
    - Verifica `query()` con carrera sin RAG retorna `status="no_documents"`.
    - Verifica embedding failure → `status="error"`.

- [ ] 23. Integrar RAGModule en Orchestrator (Opcional para MVP)
  - [ ] 23.1 Modificar `backend/orchestration.py` método `handle_turn()`:
    - Después de generar ranking: detecta si usuario pregunta detalles sobre una carrera del Top-5.
    - Si sí: invoca `rag_module.query(career, question)`.
    - Agrega respuesta RAG al `bot_message` o retorna campo separado `rag_response`.
    - Si RAG no disponible (module=None) o carrera sin RAG: continúa sin error (graceful degradation).
    _Requerimientos: 10.3_
  - [ ] 23.2 Tests: `test_orchestrator_rag_integration.py`:
    - Simula pregunta de detalle sobre carrera del Top-5 con RAG disponible.
    - Verifica que respuesta incluye información de RAG.
    - Simula con RAG fallando: verifica que flujo continúa sin error.

- [ ] 24. (Opcional) Implementar Visualización de Mapa en Frontend
  - [ ] (Opcional) 24.1 Agregar en `frontend/index.html`:
    - Div para mapa: `<div id="map"></div>`.
    - Script para Leaflet: cargar biblioteca (CDN o npm).
  - [ ] (Opcional) 24.2 Agregar en `frontend/app.js`:
    - Función `displayMap(top5_items, location)` (recibe `profile.location`).
    - Consumir `/universities?location={location}` vía `fetch()`.
    - Pintar marcadores de universidades en mapa (Leaflet markers).
    - Llamar después de generar ranking.
  - [ ] (Opcional) 24.3 Tests: verificación manual en navegador (no automatizados).

---

## Fase 5 — Batch a Escala Completa, Robustez y Pulido Final (Alta)

- [ ] 25. Implementar `data_pipeline/riasec_tagging.py` (Ampliación) — Pipeline Batch Completo
  - [ ] 25.1 Implementar script `data_pipeline/pipeline_runner.py`:
    - Función `run_full_pipeline() -> str`:
      - Paso 1: `download_ponte_en_carrera()` → `data/raw.xlsx`.
      - Paso 2: `clean_and_validate()` → `data/filtered.csv`.
      - Paso 3: `generate_features()` → `data/features.csv`, `data/feature_config.json`.
      - Paso 4: `tag_careers_with_bedrock()` con las 6,208 carreras (mockeado en demo, real en producción).
      - Paso 5: `validate_sample()` → `data/riasec_validation_sample.csv` para revisión humana.
      - Paso 6: `apply_family_fallback()` → garantiza 100% carreras con RIASEC.
      - Paso 7: Genera snapshot versionado: `snapshots/features/features_{timestamp}.csv`.
      - Retorna ruta del snapshot.
    - Loguea cada paso.
    _Requerimientos: 6.1, 6.2, 6.4_
  - [ ] 25.2 Documentar trigger EventBridge (cron):
    - Cron expression: `cron(0 2 * * ? *)` (diario a las 2 AM UTC).
    - Target: AWS Batch job que invoca `uv run python -m data_pipeline.pipeline_runner`.
    - Resultado: actualización automática de `data/features.csv` cada día.
    _Requerimientos: 6.2_
  - [ ] 25.3 Tests: `test_pipeline_runner.py`:
    - Ejecuta `run_full_pipeline()` sobre fixture de 50 carreras (mockeando Bedrock).
    - Verifica que snapshot final tiene 50 carreras, todas con `riasec_profile` non-null.
    - Verifica que `features.csv` tiene columnas `income_norm`, `cost_norm`, `admission_norm`, `duration_norm` en `[0,1]`.

- [ ] 26. Tests de Integración End-to-End con Fixtures Reales
  - [ ] 26.1 Crear fixtures en `tests/fixtures/`:
    - `catalog_50_careers.csv`: 50 carreras con datos verificables (ingresos, costos, admisión, duración).
    - `seed_riasec_10.csv`: 10 carreras con RIASEC etiquetado manualmente (semilla).
  - [ ] 26.2 Test: `test_integration_full_flow.py`:
    - Setup: Carga fixtures en `data/`.
    - Genera `features.csv` vía `run_full_pipeline()`.
    - Crea `ScoringEngine` con fixtures.
    - Simula 4 turnos de conversación:
      - Turno 1: usuario menciona intereses.
      - Turno 2: usuario prioriza criterios.
      - Turno 3–4: usuario especifica filtros.
    - Genera ranking.
    - Persiste en Aurora (test DB).
    - Consulta `/universities` filtrado.
    - Verifica:
      - Top-5 generado correctamente.
      - Aislamiento por `session_id`.
      - Datos persistidos en Aurora.
      - Datos recuperables desde Aurora.
    - Módulos implicados: `data_pipeline`, `backend/llm_service`, `backend/scoring`, `backend/orchestration`, `backend/persistence`, `backend/lambda`.
    _Requerimientos: 5.4, 7.1–7.3, 8.1–8.3, 9.1, 10.2_

- [ ] 27. Validación de Restricciones No Funcionales
  - [ ] 27.1 Tests basados en propiedades: **Propiedad 4 — Reproducibilidad**. **Valida: Requerimiento 9 criterio 9.1**. Estrategia:
    - Fijar snapshot `features.csv` (timestamp conocido).
    - Fijar config `feature_config.json` (versión conocida).
    - Fijar perfil usuario y pesos.
    - Ejecutar `rank_and_filter()` en dos procesos distintos (en paralelo).
    - Invariante: `ranking_json` producido es idéntico byte-a-byte (serialización determinística).
    - Ejecutar 10 veces para verificar.
  - [ ] 27.2 Test de benchmark: `test_scoring_performance.py`:
    - Genera dataset simulado de 6,208 filas (carreras × universidades).
    - Mide tiempo de `rank_and_filter()` (incluye scoring completo, filtrado, desempate, truncado a Top-5).
    - Verificar: `elapsed_time <= 1.0 segundo`.
    _Requerimientos: 10.1_
  - [ ] (Opcional) 27.3 Benchmark de latencia end-to-end `/chat`:
    - Mide tiempo desde request POST `/chat` hasta response JSON con Top-5.
    - Objetivo referencial: ≤ 5 segundos (incluye LLM, scoring, persistencia).
    - No es bloqueante para MVP de demo.

- [ ] 28. Documentación Final y README
  - [ ] 28.1 Actualizar `README.md`:
    - Descripción clara del proyecto.
    - Arquitectura híbrida AWS:
      - Lambda: `/session/create`, `/feedback`, `/universities` (stateless).
      - Fargate: `/chat` (stateful, sesiones en memoria).
      - Aurora PostgreSQL: persistencia (feedback, rankings, pgvector).
      - DynamoDB: georreferencia (universities).
      - Bedrock: LLM (Claude) + Embeddings (Titan).
    - Instalación local: `uv sync`, `docker-compose up`.
    - Variables de entorno: copiar `.env.example` → `.env`, llenar valores AWS.
    - Comandos útiles:
      ```bash
      uv run pytest tests/ --cov  # Ejecutar tests con cobertura
      uv run python -m data_pipeline.pipeline_runner  # Ejecutar pipeline batch local
      docker-compose up  # Levantar backend + Postgres local
      ```
    - Fases de desarrollo (link a `tasks.md`).
    - Estructura de directorios.
  - [ ] 28.2 Documentar política de logging seguro en `README.md`:
    - NUNCA loguear: tokens completos, API keys (AWS), claves JWT, credenciales DB, PII de usuarios (nombres reales completos).
    - SÍ loguear: `session_id`, timestamps, nombres de carreras (públicos), scores, pesos, eventos de flujo.
    - Ejemplo: `"Ranking generated: session_id={session_id}, career_count=5, confidence_score=0.85"`.
    - Configuración centralizada en `backend/utils.py` (función `setup_logging()`).
    _Requerimientos: 10.4_
  - [ ] 28.3 Crear `docs/ARCHITECTURE.md`:
    - Diagrama de componentes (ASCII o Mermaid).
    - Flujo de datos end-to-end.
    - Justificación de decisiones AWS (Bedrock vs OpenAI, Aurora vs DynamoDB, etc.).
    - Links a design.md y requirements.
  - [ ] 28.4 Crear `docs/DEPLOYMENT.md`:
    - Guía de despliegue a AWS:
      - Crear Aurora cluster.
      - Crear DynamoDB table.
      - Crear ECR repository.
      - Build + push Docker image.
      - Deploy Lambda functions (via SAM o Terraform).
      - Deploy Fargate service (via ECS/Terraform).
      - Configurar API Gateway.
    - Checklist de verificación post-deploy.

- [ ] 29. Checkpoint Final — Validación Integral del Sistema
  - [ ] 29.1 Ejecutar suite completa:
    ```bash
    uv run pytest tests/ --cov --cov-report=html
    ```
    - Cobertura ≥ 60% en `backend/`, `data_pipeline/`.
    - Todos los tests pasan (unit + integration).
    - Propiedades de corrección (1, 4, 5, 6) tienen tests asociados y pasan.
  - [ ] 29.2 Verificar que 4 propiedades de corrección están implementadas:
    - **Propiedad 1 — Determinismo del Ranking**: test en `test_scoring_determinism.py` ✓.
    - **Propiedad 4 — Reproducibilidad**: test en `test_scoring_reproducibility.py` ✓.
    - **Propiedad 5 — Confidence Monótono**: test en `test_confidence_monotonicity.py` ✓.
    - **Propiedad 6 — Pesos Válidos**: test en `test_llm_weights_validity.py` ✓.
  - [ ] 29.3 Ejecutar test de integración end-to-end (`test_integration_full_flow.py`):
    - Fixture de 50 carreras.
    - Conversa completa (4 turnos) → ranking → persistencia → consulta.
    - Aislamiento verificado.
    - Todos los módulos integrados y funcionando.
  - [ ] 29.4 Ejecutar benchmark de performance (`test_scoring_performance.py`):
    - 6,208 carreras → score ≤ 1 segundo ✓.
  - [ ] 29.5 Verificar README, docs, y estructura de proyecto:
    - `README.md` actualizado con instrucciones claras.
    - `docs/ARCHITECTURE.md` documenta decisiones.
    - `docs/DEPLOYMENT.md` lista pasos de deploy.
    - `.env.example` con todas las variables necesarias.
    - `pyproject.toml` y `uv.lock` actualizados.
    - `Dockerfile` y `docker-compose.yml` funcionales.
    - Todos los archivos en sus ubicaciones correctas.
  - [ ] 29.6 Documento de seguimiento:
    - Generar reporte de cobertura: `coverage html`, revisar `htmlcov/index.html`.
    - Listar todos los requerimientos cubiertos (R1–R10, C1–C7).
    - Documentar cualquier desviación o trabajo pendiente.
    - Firmar off como "Fase 5 completa — Sistema listo para demo".

---

## Notas

- El prefijo `(Opcional)` en una subtarea indica que es trabajo no bloqueante para el progreso funcional del MVP — no es requerido para demostración o cierre de fase. Los **tests de corrección y de integración NO son opcionales** salvo que se marquen explícitamente; son obligatorios y validan que el sistema cumple con especificaciones críticas.

- Cada subtarea de implementación termina con `_Requerimientos: X.Y_` (ej `_Requerimientos: 1.2, 4.1_`), referenciando criterios en `design.md` § 3 (Requisitos). Este mapeo asegura trazabilidad: cualquier criterio de requerimiento aparece en al menos una subtarea del plan. Los checkpoints y tareas organizativas pueden no tener trazabilidad directa.

- **Property-based testing** usa librería `hypothesis`:
  - **Propiedad 1**: Generador de `(weights, features_df)` válidos; invariante: ejecutar 2 veces → mismo ranking.
  - **Propiedad 5**: Generador de secuencias de actualizaciones a `StudentProfile`; invariante: confidence nunca disminuye.
  - **Propiedad 6**: Generador de rankings aleatorios (5 elementos); invariante: pesos suma ∈ [0.99, 1.01].
  - **Propiedad 4**: Snapshot + config fijos; ejecutar en 2 procesos; invariante: `ranking_json` idéntico byte-a-byte.

- **Restricciones transversales** (heredadas de `design.md` § Restricciones Transversales):
  - Package manager: `uv` (NO Poetry, NO Conda, NO pip puro).
  - Framework Fargate: `FastAPI` + `Uvicorn` (NO Flask, NO Django).
  - Framework Lambda: `boto3` puro (NO frameworks adicionales como `zappa`).
  - Credenciales: ÚNICAMENTE vía variables de entorno o AWS Secrets Manager (NUNCA hardcoded, NUNCA en código).
  - LLM: ÚNICAMENTE Bedrock/Claude (NO Gemini, NO OpenAI en este ciclo).
  - Embeddings: ÚNICAMENTE Bedrock Titan (NO sentence-transformers local, NO OpenAI).
  - Vector DB: ÚNICAMENTE pgvector en Aurora (NO FAISS local, NO Pinecone, NO Chroma).

- **Sesión conversacional**: Vive **en memoria dentro del proceso Fargate** con TTL simple (1800s), decisión explícita para demo. NO introduce Redis ni DynamoDB para sesiones. JWT es stateless para que Lambda y Fargate NO dependan de almacén compartido.

- **Etiquetado RIASEC** (6,208 carreras): Corre como **job batch** (AWS Batch o Fargate task, NO Lambda) por duración de llamadas Bedrock (potencialmente horas). Pipeline automatizado genera snapshot versionado. Validación muestral (300 carreras) es paso humano externo, pero export CSV es responsabilidad pipeline (tarea 25.2).

- **Fases son incrementales e integradas**: Cada una termina con checkpoint que verifica estado funcional. Fase N depende de trabajo de Fase N-1 pero NO rompe su funcionalidad. Puntos de integración y reemplazos están explícitos (ej Tarea 18.1: "Modificar endpoint").

- **Granularidad de tareas**: Nivel superior = módulo/archivo/integración/test. Nivel 2 = función pública o grupo cohesivo. NO hay nivel 3 (sub-subtasks). Si una subtarea > 12–15 líneas detalle, dividir en varias subtasks.

- **Autosuficiencia**: Cada subtarea incluye inline suficiente detalle (firmas, parámetros, defaults, thresholds, errores, fallbacks) para implementar sin consultar constantemente `design.md`. Cuando aplique: código pseudocode, SQL, ejemplos, comandos exactos.

- **Trazabilidad obligatoria**: Cada subtarea de implementación tiene `_Requerimientos: X.Y_`. Cada criterio (R1–R10, C1–C7) aparece en al menos una. Tests tienen referencias explícitas a propiedades (negrita) o requerimientos.

- **Orden de tareas respeta dependencias reales**: Data pipeline (Fase 1) antes de LLM (Fase 2). Scoring determinístico antes de Orchestration. LLM antes de Fargate endpoint. Auth y persistencia (Fase 3) antes de Lambda handlers. RAG (Fase 4, opcional) después de todo funcional. Batch (Fase 5) último.

# Plan de Implementación: CareerMatch Perú — Matching Context (Scoring + Data Pipeline)

## Descripción General

**Alcance de este documento: solo Matching Context (scoring determinístico en Python 3.12 Lambda) + Data Pipeline.** Los contextos Identity, Assessment, Career y AI Advisor se implementan en otros repositorios/stacks (ver `design.md` § 2.2). Este plan NO cubre el backend completo.

La implementación avanza en cinco fases incrementales sobre AWS, cada una dejando el Matching Context en un estado funcional e integrado: primero se define el schema `CareerOffering` en Aurora (schema `career.career_offerings`, el aggregate carrera×universidad con métricas) y se construye el pipeline de datos que lo puebla (`data_loading` desde snapshot existente, `data_clean`, `feature_engineering`, escritura batch a Aurora). Luego se implementa el `ScoringEngine` determinístico leyendo desde Aurora; después se agregan persistencia de feedback (Aurora PostgreSQL) y capacidades secundarias opcionales (RAG con pgvector, embeddings Bedrock Titan). Finalmente se cierra con etiquetado RIASEC a escala completa (554 carreras únicas vía Bedrock batch + join a las 6,208 filas), tests de integración con fixtures reales, validación de propiedades de corrección y documentación final. Cada fase termina con un checkpoint explícito que verifica progreso integrado e inspecciona antes de continuar.

## Tareas

- [ ] 1. Setup del proyecto
  - [ ] 1.1 Crear estructura de directorios: `data_pipeline/`, `backend/`, `backend/lambda/`, `backend/persistence/`, `frontend/`, `infra/`, `tests/`, `tests/integration/`, `tests/fixtures/`, `data/`, `snapshots/`, `snapshots/raw/`, `snapshots/features/`, `snapshots/configs/`, `docs/`, `scripts/`, `notebooks/`.
  - [ ] 1.2 Inicializar proyecto con `uv` y agregar dependencias especificadas en `design.md` § Stack Tecnológico:
    - Ejecutar `uv init` para crear `pyproject.toml` base.
    - Agregar dependencias con `uv add`:
      ```bash
      uv add "boto3>=1.28.0" "fastapi>=0.104.0" "uvicorn>=0.24.0" "pydantic>=2.0.0" "psycopg2-binary>=2.9.0" "pgvector>=0.1.8" "sqlalchemy>=2.0.0" "PyJWT>=2.8.0" "cryptography>=41.0.0" "python-dotenv>=1.0.0" "pandas>=2.0.0" "numpy>=1.24.0" "openpyxl>=3.10.0" "selenium>=4.15.0" "requests>=2.31.0" "moto>=4.2.0"
      # NOTA: PyJWT y cryptography son opcionales para MVP (solo necesarios si se implementa JWT, tarea 16.O). Se incluyen por si el equipo decide activarlo.
      uv add --dev "pytest>=7.0.0" "pytest-asyncio>=0.21.0" "pytest-cov>=4.1.0"
      ```
    - `uv sync` genera `uv.lock` automáticamente.
  - [ ] 1.3 Crear `.env.example` con variables de entorno: `AWS_REGION=us-east-1`, `LLM_PROVIDER=bedrock`, `LLM_MODEL=anthropic.claude-3-5-sonnet-20241022`, `EMBEDDING_PROVIDER=bedrock`, `EMBEDDING_MODEL=amazon.titan-embed-text-v2`, `AURORA_ENDPOINT=...`, `AURORA_PORT=5432`, `AURORA_USER=postgres`, `AURORA_PASSWORD=...`, `AURORA_DATABASE=careermatch`, `SCHEMA_NAME=career`, `LOG_LEVEL=INFO`, `ENVIRONMENT=development`, `JWT_SECRET=...`, `JWT_ALGORITHM=HS256`.
    - NOTA: `JWT_SECRET` y `JWT_ALGORITHM` son obligatorios y deben coincidir con los que usa Identity Context (Secrets Manager compartido, no un secreto propio del servicio de scoring/chat).
  - [ ] 1.4 Crear `pytest.ini` con configuración: `testpaths = tests`, `python_files = test_*.py`, `asyncio_mode = auto`, `addopts = --tb=short -v`, markers para `unit`, `integration`, `property`. Configurar `.coveragerc` con `source = backend, data_pipeline`, umbrales mínimos 60%.
  - [ ] 1.5 Crear `README.md` inicial con: descripción del proyecto, instrucciones de instalación (`uv sync`), guía de variables de entorno, estructura de directorios, fases de desarrollo, y comandos para ejecutar tests (`uv run pytest tests/ --cov`).
  - [ ] 1.6 Crear `Dockerfile` para desarrollo local del Matching Context: imagen `python:3.10-slim`, instalar `uv` vía `COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/`, instalar dependencias de sistema (`gcc`, `postgresql-client`), copiar `pyproject.toml` y `uv.lock`, ejecutar `uv sync --no-dev`, exponer puerto 8000, healthcheck para `/health`, entrypoint `uv run uvicorn backend.app:app --host 0.0.0.0 --port 8000`. NOTA: Este Dockerfile es solo para desarrollo local del Matching Context, NO para el backend completo.
  - [ ] 1.7 Crear `docker-compose.yml` para desarrollo local: servicio `backend` (imagen from Dockerfile, ports 8000:8000, env vars, volumes para código y datos), servicio `postgres_db` (imagen postgres:15-alpine, ports 5432:5432, env POSTGRES_USER/PASSWORD/DB), volumen `postgres_data`, red `careermatch_net`.

---

## Fase 1 — Fundación: Pipeline de Datos y Motor de Scoring (Alta): Datos Reproducibles y Ranking Determinístico sin Dependencias Externas

- [ ] 2. Implementar `data_pipeline/data_loading.py` — Carga desde Snapshot / raw.xlsx
  - [ ] 2.1 Implementar clase `DataLoader` con método `load(self) -> str`:
    - Verifica existencia de `data/raw.xlsx` como fuente primaria.
    - Si no existe: busca el archivo más reciente en `snapshots/raw/` ordenado por timestamp.
    - Si hay snapshot disponible: copia a `data/raw.xlsx` (restauración) y retorna ruta.
    - Si no hay ningún snapshot: loguea `WARNING` e intenta ejecutar `ingestion.py` como fallback (si el script existe).
    - Retorna ruta a `data/raw.xlsx` lista para ser consumida por `DataCleaner`.
    - NOTA: El Selenium scraping es opcional para la demo; los snapshots existentes contienen datos suficientes.
    _Requerimientos: 10.3_
  - [ ] (Opcional) 2.O Implementar `data_pipeline/ingestion.py` — Job de Refresco Selenium
    - NOTA: Esta tarea es **opcional y no bloqueante** para la demo. Los scripts Selenium + snapshots versionados ya existen y funcionan (confirma que la simplificación es de bajo riesgo).
    - Implementar clase `DataIngestion` con método `download(self) -> str` que orquesta Selenium headless para descargar el Excel de Ponte en Carrera.
    - Si descarga completa: copia a `data/raw.xlsx`, crea snapshot versionado en `snapshots/raw/raw_{timestamp}.xlsx`.
    - Si timeout o error: loguea `WARNING` y retorna ruta a último snapshot conocido (fallback).
    - Puede ejecutarse como job cron/EventBridge programado (semanal, mensual), no como paso bloqueante del pipeline.
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
    - Constructor: carga `cleaned_csv`, intenta cargar `feature_config.json` (si no existe, crea defaults con fallbacks: `duration_years_fallback_instituto=4.0`, `duration_years_fallback_universidad=5.0`, `monthly_income_fallback=2500.0`, `annual_cost_fallback=1000.0`, `admission_rate_fallback=30.0`).
    - Método `create_imputation_flags()`: crea flags booleanos para columnas que serán imputadas:
      - `duration_imputed_flag = (duration_years <= 0) OR (duration_years > 10) OR (isna)`.
      - `income_imputed_flag = (monthly_income <= 0) OR (isna)`.
      - Análogo para `annual_cost` y `admission_rate`.
    - Método `hierarchical_imputation()`: para cada variable inválida:
      - Nivel 1: intenta mediana por `(career_family, management_type)`.
      - Nivel 2: intenta mediana por `career_family`.
      - Nivel 3: usa fallback desde config (para duración: 4 años si Instituto, 5 años si Universidad).
      - Crea columna `{variable}_imputed` con valores imputados.
    - Método `validate_ranges()`: reajusta valores tras imputación:
      - `admission_rate_imputed` → clamp a `[0, 90]`.
      - **Nota**: `duration_years_imputed` no tiene clamp final; el pipeline real no hace clip a `[3,7]`. La imputación jerárquica + fallback por tipo de institución es suficiente.
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
  - [ ] 4.1 Implementar función `tag_careers_with_bedrock(catalog_unique: pd.DataFrame, seed_examples: pd.DataFrame) -> pd.DataFrame`:
    - El tagging opera sobre **carreras únicas** (554 distinct `career`), NO sobre las 6.208 filas de `career.career_offerings`.
    - Input `catalog_unique`: DataFrame deduplicado por `career` (1 fila por carrera única), con columnas `id, career, career_family, description`.
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
    - Retorna DataFrame con columnas `id, career, riasec_profile, riasec_source` (554 filas, 1 por carrera única).
    - **Post-condición**: Quien invoca debe hacer `features_df = features_df.merge(riasec_tags, on='career', how='left')` para propagar `riasec_profile` a las 6.208 filas.
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
    - Fixture: 10 carreras únicas (5 con `riasec_profile` semilla, 5 sin). Verifica:
      - `tag_careers_with_bedrock` recibe DataFrame deduplicado (1 fila por `career`).
      - Retorna DataFrame con 5 carreras taggeadas (`llm_tagged`) y 5 sin cambios.
      - `apply_family_fallback` rellena las 5 pendientes con moda de su familia.
      - Al finalizar: 100% filas tienen `riasec_profile` no-nulo.
    - Verifica que intenta 3 veces antes de marcar como `pending`.
    - Verifica que el merge posterior (`features_df.merge(riasec_tags, on='career')`) propaga correctamente el RIASEC a todas las filas de cada carrera.

- [ ] 5. Implementar `backend/models.py` — Modelos Pydantic/Dataclasses
  - [ ] 5.1 Implementar dataclass `StudentProfile`:
    - Bloque A: `riasec_scores: dict[str, int]` (valores [1,10]), `riasec_code: str | None` (3 letras o None).
    - Bloque B: `w_afinidad: float`, `w_ingreso: float`, `w_costo: float`, `w_admision: float`, `w_duracion: float` (todos [0,1]).
    - Bloque C: `location: str | None` (región/ubicación del estudiante, se mapea a columna `location` en `career.career_offerings`), `management_type: str | None` ('Pública' | 'Privada', se mapea a columna `management_type` en carrera), `presupuesto_max: float | None`, `modalidad: str | None`.
    - NOTA: `institution_type` (Universidad/Instituto) es una columna separada en `career.career_offerings` y NO es el filtro management_type del Figma.
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

- [ ] 5.6 Definir schema `CareerOffering` en Aurora:
    - Crear archivo `infra/schema_career_offerings.sql` con DDL del aggregate:
      ```sql
      CREATE SCHEMA IF NOT EXISTS career;

      CREATE TABLE career.career_offerings (
          id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
          career_id TEXT NOT NULL,
          career TEXT NOT NULL,
          institution TEXT NOT NULL,
          career_family TEXT,
          riasec_profile VARCHAR(3),
          riasec_source VARCHAR(20),
          monthly_income DECIMAL(10,2),
          annual_cost DECIMAL(10,2),
          admission_rate DECIMAL(5,2),
          duration_years DECIMAL(3,1),
          monthly_income_imputed DECIMAL(10,2),
          annual_cost_imputed DECIMAL(10,2),
          admission_rate_imputed DECIMAL(5,2),
          duration_years_imputed DECIMAL(3,1),
          income_norm DECIMAL(5,4),
          cost_norm DECIMAL(5,4),
          admission_norm DECIMAL(5,4),
          duration_norm DECIMAL(5,4),
          location TEXT,
          management_type TEXT,
          institution_type TEXT,
          modalidad TEXT,
          income_imputed_flag BOOLEAN DEFAULT FALSE,
          cost_imputed_flag BOOLEAN DEFAULT FALSE,
          admission_imputed_flag BOOLEAN DEFAULT FALSE,
          duration_imputed_flag BOOLEAN DEFAULT FALSE,
          snapshot_id TEXT,
          version INTEGER DEFAULT 1,
          created_at TIMESTAMPTZ DEFAULT now(),
          updated_at TIMESTAMPTZ DEFAULT now(),
          INDEX idx_career ON career_offerings(career),
          INDEX idx_institution ON career_offerings(institution),
          INDEX idx_location ON career_offerings(location)
      );
      ```
    - Este schema es propiedad de Career Context. Matching Context tiene acceso de LECTURA.
    - NOTA: El pipeline batch (data_pipeline) escribe aquí directamente (INSERT/UPDATE batch).
    _Requerimientos: 6.1_
  - [ ] 5.7 Crear migración inicial del schema:
    - Script SQL ejecutable: `infra/migrations/V001__create_career_offerings.sql`.
    - Debe ser idempotente: `CREATE SCHEMA IF NOT EXISTS`, `CREATE TABLE IF NOT EXISTS`.
    - Incluir en la configuración de infraestructura (ejecutar contra Aurora antes del deploy).
    _Requerimientos: 6.1_

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
    - Todas las normas ya están orientadas "mayor = mejor" por el pipeline de datos; los parámetros reciben los valores desde columnas `income_norm`, `cost_norm`, `admission_norm`, `duration_norm`:
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
    - Constructor: recibe `db_connection_string: str` (URI Aurora) y `config_json_path: str`.
    - Conecta a Aurora PostgreSQL vía `psycopg2.connect(dsn=db_connection_string)`.
    - Consulta schema `career`, tabla `career_offerings`:
      ```sql
      SELECT id, career, institution, riasec_profile, location,
             management_type, income_norm, cost_norm, admission_norm,
             duration_norm, monthly_income_imputed, annual_cost_imputed,
             admission_rate_imputed, duration_years_imputed
      FROM career.career_offerings
      ```
    - Lee resultado en Pandas DataFrame con `pd.read_sql()`.
    - Valida schema: debe tener columnas `id, career, institution, riasec_profile, income_norm, cost_norm, admission_norm, duration_norm, location, management_type, monthly_income_imputed, annual_cost_imputed, admission_rate_imputed, duration_years_imputed`.
    - Si la tabla no existe o está vacía: raise `ValueError("career_offerings vacía o no existe")`.
    - Fallback desarrollo local: si `AURORA_ENDPOINT` no está configurado, carga `features.csv` local.
    _Requerimientos: 6.1_
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
    - Verificar que `DataLoader.load()` retorna path válido desde snapshot existente (o fallback si no hay snapshots).
    - Verificar que `DataCleaner.clean()` retorna DataFrame válido sin filas incompletas.
    - Verificar que `FeatureEngineer.run_pipeline()` genera DataFrame con columnas `income_norm`, `cost_norm`, `admission_norm`, `duration_norm` en `[0,1]` (listo para escribir en Aurora).
    - Verificar que `tag_careers_with_bedrock` opera sobre carreras únicas (mockeado) y el join subsiguiente propaga correctamente `riasec_profile` a features_df.
    - Verificar que `ScoringEngine.rank_and_filter()` (cargando desde Aurora o fallback CSV) produce Top-5 con scores en `[0,1]`, determinístico, desempatado correctamente.
    - Cobertura mínima 60% en `data_pipeline` y `backend/scoring.py`.
    - Ejecutar test de determinismo (Propiedad 1) y pesos válidos (Propiedad 6): ambos deben pasar.

---

## Fase 2 — Flujo de Scoring Local (Alta): Matching Context sobre Fixtures, sin Persistencia Real

> ❄️ **Tareas 8.x y 10.1 congeladas:** La tarea 8 (LLM_Layer con `interpret_bloque_a/b/c`, `generate_follow_up`) y la tarea 10.1 (`calculate_confidence`) asumen captura conversacional de Bloques A/B/C vía LLM. El Assessment Context real es un **cuestionario estructurado** (`POST /v1/assessments`, `POST /v1/assessments/{id}/responses`). Estas tareas quedan **congeladas** hasta la decisión de producto (Opción A: eliminar, Opción B: coexistencia con mapeo campo→campo). Ver `design.md` §1 y §3 para detalle. La tarea 8.7 (`generate_explanation`) sigue vigente para el AI Advisor.

- [ ] ❄️ 8. Implementar `backend/llm_service.py` — LLM_Layer (Amazon Bedrock Claude) [CONGELADA]
  - [ ] ❄️ 8.1 Implementar clase `LLMService` [CONGELADA]:
    - (Congelada — ver nota de fase)
  - [ ] ❄️ 8.2 Implementar método `interpret_bloque_a` [CONGELADA]
  - [ ] ❄️ 8.3 Implementar método `interpret_bloque_b` [CONGELADA]
  - [ ] ❄️ 8.4 Implementar método `interpret_bloque_c` [CONGELADA]
  - [ ] ❄️ 8.5 Implementar método `_parse_and_validate_json` [CONGELADA]
  - [ ] ❄️ 8.6 Implementar método `generate_follow_up` [CONGELADA]
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
  - [ ] ❄️ 8.8 Tests: `test_llm_service.py` [CONGELADA]

- [ ] 9. Implementar `backend/session.py` — SessionManager (solo desarrollo local)
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

- [ ] 10. Implementar `backend/lambda/matching_handler.py` — ScoringEngine como Lambda Reactivo
  - [ ] ❄️ 10.1 Implementar función `calculate_confidence(profile: StudentProfile) -> float` [CONGELADA]:
    - La confianza se calcula en Assessment Context según completitud del cuestionario estructurado, no en Matching Context.
    - Congelada hasta decisión de producto (Opción A: eliminar, Opción B: documentar mapeo).
  - [ ] 10.2 Definir schema del evento `AssessmentCompleted`:
    - Crear dataclass `AssessmentCompletedEvent`:
      ```python
      @dataclass
      class AssessmentCompletedEvent:
          userId: str
          sessionId: str
          riasec_scores: dict[str, int]  # {"R": 8, "I": 9, ...}
          weights: dict[str, float]       # {"w_afinidad": 0.3, ...}
          filters: dict                   # {"location": "Lima", "management_type": "publica", ...}
          confidence: float               # [0, 1]
          timestamp: str                  # ISO 8601
      ```
    - El evento llega desde EventBridge con `detail-type: "AssessmentCompleted"`.
    - Implementar parser que extrae `detail` del evento y construye `AssessmentCompletedEvent`.
    _Requerimientos: 8.2_
  - [ ] 10.3 Implementar handler `matching_handler.lambda_handler(event, context)`:
    - Firma estándar Lambda: `def lambda_handler(event: dict, context: object) -> dict`.
    - Paso 1: Validar que `event["detail-type"] == "AssessmentCompleted"`. Si no: retorna `{"status": "ignored"}`.
    - Paso 2: Parsear `event["detail"]` → `AssessmentCompletedEvent` (reusa 10.2).
    - Paso 3: Cargar `career_offerings` desde Aurora (o `features.csv` local en desarrollo).
    - Paso 4: Crear `StudentProfile` desde los campos del evento:
      - `riasec_scores` desde `event.riasec_scores`
      - Pesos desde `event.weights`
      - Filtros desde `event.filters`
    - Paso 5: Calcular `riasec_code = calculate_riasec_code(profile.riasec_scores)`.
    - Paso 6: Invocar `scoring_engine.rank_and_filter(profile, features_df)`.
    - Paso 7: Extraer Top-5: `top_5 = scoring_engine.get_top_n(ranked, 5)`.
    - Paso 8: Generar `ranking_id = str(uuid.uuid4())`.
    - Paso 9: Persistir ranking en Aurora PostgreSQL (tabla `rankings`).
    - Paso 10: Emitir `RecommendationGenerated` a EventBridge:
      ```python
      client = boto3.client("events")
      client.put_events(Entries=[{
          "Source": "careermatch.matching",
          "DetailType": "RecommendationGenerated",
          "Detail": json.dumps({
              "userId": event.userId,
              "sessionId": event.sessionId,
              "rankingId": ranking_id,
              "top_5": top_5,
              "snapshot_id": os.getenv("SNAPSHOT_ID", "latest"),
              "timestamp": datetime.utcnow().isoformat()
          }),
          "EventBusName": os.getenv("EVENT_BUS_NAME", "spark-match-events")
      }])
      ```
    - Paso 11: Retornar `{"statusCode": 200, "body": json.dumps({"rankingId": ranking_id})}`.
    - Manejo de errores:
      - Si Aurora no disponible: loguea, retorna `{"statusCode": 500}`.
      - Si Aurora caído: queue local + reintento.
      - Si parse inválido: loguea, retorna `{"statusCode": 400}`.
    - Timeout Lambda: 30 segundos (suficiente para 6,208 combinaciones).
    _Requerimientos: 1.1, 2.1, 3.1, 4.1, 4.2, 4.3, 8.2_
  - [ ] 10.4 Definir schema del evento `RecommendationGenerated`:
    - Crear dataclass `RecommendationGeneratedEvent`:
      ```python
      @dataclass
      class RecommendationGeneratedEvent:
          userId: str
          sessionId: str
          rankingId: str
          top_5: list[dict]  # Lista de RankingItem serializados
          snapshot_id: str
          timestamp: str
      ```
    - Este evento es consumido por el AI Advisor para generar explicaciones.
    _Requerimientos: 8.2_
  - [ ] 10.5 Implementar `backend/lambda/feedback_handler.py` — Feedback Lambda Handler:
    - Firma: `def lambda_handler(event: dict, context: object) -> dict`.
    - Activado por API Gateway v2 (ruta `POST /feedback`).
    - Paso 1: Extraer JWT de `event["headers"]["Authorization"]`.
    - Paso 2: Validar JWT con `PyJWT.decode(jwt, secret, algorithms=["HS256"])` → `userId`.
      - Si inválido: retorna `{"statusCode": 401, "body": "Unauthorized"}`.
    - Paso 3: Parsear body: `{rankingId, validationScore, selectedCareer, notes}`.
    - Paso 4: Validar `validationScore ∈ [1, 5]`: si no, `{"statusCode": 422}`.
    - Paso 5: Verificar que `rankingId` pertenece a `userId` (security).
    - Paso 6: Persistir feedback en Aurora (tabla `feedback`).
    - Paso 7: Retornar `{"statusCode": 200, "body": json.dumps({"status": "success"})}`.
    _Requerimientos: 8.1, 9.1_
  - [ ] 10.6 Tests: `test_matching_handler.py`:
    - Mockea evento EventBridge con `AssessmentCompleted` válido.
    - Mockea `ScoringEngine`, `boto3.client("events")` y Aurora.
    - Verifica que el handler parsea el evento, ejecuta scoring y emite `RecommendationGenerated`.
    - Verifica error handling: evento con `detail-type` incorrecto → `status="ignored"`.
    - Verifica error handling: Aurora no disponible → `statusCode=500`.
  - [ ] 10.7 Tests: `test_feedback_handler.py`:
    - Mockea API Gateway event con JWT válido.
    - Mockea `PyJWT.decode` y Aurora.
    - Verifica validación: score fuera de rango → 422.
    - Verifica validación: JWT inválido → 401.
    - Verifica flujo exitoso → 200 con `{"status": "success"}`.
  - [ ] 10.8 Tests basados en propiedades: **Propiedad 5 — Confidence Monótono**. **Valida: Requerimiento 4 criterio 4.1**. Estrategia:
    - Generar secuencias aleatorias de actualizaciones: `profile` comienza vacío → agrega riasec → agrega pesos.
    - Invariante: `confidence_score` nunca disminuye.
    - Verificar con `hypothesis`: generar 100 secuencias aleatorias.

- [ ] 11. Implementar `backend/app.py` — FastAPI Entrypoint (desarrollo local Matching Context)
  - [ ] 11.1 Implementar fixture de FastAPI:
    - `app = FastAPI(title="CareerMatch Perú — Matching Context (local dev)")`.
    - Variable global `scoring_engine = None` (inicializada en startup).
    - NOTA: Este entrypoint es solo para desarrollo local. En producción, los handlers son Lambdas individuales.
  - [ ] 11.2 Implementar endpoint `POST /v1/matching/score`:
    - Recibe JSON: `{riasec_scores: dict, weights: dict, filters: dict, userId: str}`.
    - Construye `StudentProfile` desde el request.
    - Invoca `scoring_engine.rank_and_filter(profile, features_df)`.
    - Retorna `{status: "success", top_5: list[RankingItem]}`.
    - HTTP 500 si error interno (loguea sin PII).
    - NOTA: Este endpoint simula localmente lo que el Lambda `matching_handler` hace en producción.
    _Requerimientos: 8.2_
  - [ ] 11.3 Implementar endpoint `GET /health`:
    - Retorna `{status: "ok"}`.
    - Usado para health check del servicio local.
  - [ ] 11.4 Implementar event handler `@app.on_event("startup")`:
    - Inicializa componentes:
      - `scoring_engine = ScoringEngine(os.getenv("AURORA_ENDPOINT", "psycopg://localhost:5432/careermatch"), "data/feature_config.json")`.
    - Loguea "Matching Context local iniciado correctamente".
  - [ ] 11.5 Tests de integración: `test_app_score_flow.py`:
    - Usa `TestClient` de FastAPI.
    - Fixture: cargar `tests/fixtures/features_50_careers.csv` o tabla Postgres local con 50 registros.
    - Simula `POST /v1/matching/score` con perfil completo.
    - Verifica response: `status: "success"`, `top_5` con 5 items.
    - Verifica scores en `[0,1]`, ranks 1–5.

- [ ] 12. Checkpoint Fase 2 — Validación de Scoring Local (Matching Context)
  - [ ] 12.1 Levantar servicio: `docker-compose up -d` (backend + postgres local).
  - [ ] 12.2 Ejecutar scoring local vía curl:
    ```bash
    curl -X POST http://localhost:8000/v1/matching/score \
      -H "Content-Type: application/json" \
      -d '{
        "riasec_scores": {"R": 8, "I": 9, "A": 7, "S": 6, "E": 5, "C": 4},
        "weights": {"w_afinidad": 0.3, "w_ingreso": 0.25, "w_costo": 0.2, "w_admision": 0.15, "w_duracion": 0.1},
        "filters": {"location": null, "management_type": null, "presupuesto_max": null, "modalidad": null},
        "userId": "test-user"
      }'
    ```
    - Respuesta esperada: `{"status": "success", "top_5": [...]}`.
  - [ ] 12.3 Verificar que Top-5 tiene:
    - 5 items, ranks 1–5.
    - `concordancia_score` en `[0,1]`.
    - `scores_by_criterion` con 5 criterios.
    - `datos_verificables` con claves `monthly_income_imputed`, `annual_cost_imputed`, `admission_rate_imputed`, `duration_years_imputed`.
  - [ ] 12.4 Ejecutar suite: `uv run pytest tests/test_*.py tests/integration/ --cov`.
    - Cobertura ≥ 60% en `backend/scoring.py`, `backend/lambda/matching_handler.py`, `backend/app.py`.
    - Todos los tests de propiedades (1, 5, 6) deben pasar.

---

## Fase 3 — Persistencia y Matching Context Completo (Alta): Aurora, Infraestructura Compartida

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
  - [ ] 13.2 (Omitido — universities viven en AURORA schema `career.career_offerings`)
    - Las universidades no tienen tabla DynamoDB separada. Toda la información de
      carrera × universidad (incluyendo location, management_type, y georreferencia)
      reside en `career.career_offerings` (Aurora PostgreSQL, schema `career`).
    - El campo `location` en `career.career_offerings` sirve para filtrado y
      georreferencia. No hay tabla DynamoDB `universities`.
  - [ ] 13.3 Consumir infraestructura compartida existente en lugar de crear desde cero:
    - Leer SSM Parameters existentes: `AURORA_ENDPOINT`, `EVENT_BUS_NAME` desde AWS Systems Manager Parameter Store.
    - (DYNAMODB_TABLE_UNIVERSITIES ya no existe — DynamoDB es solo para Notifications).
    - Leer `JWT_SECRET` desde Secrets Manager (compartido con Identity Context).
    - No crear EventBridge bus propio; usar `spark-match-events` existente.
    - Configurar permisos Lambda para publicar en EventBridge existente.
    - NOTA: No crear Aurora ni EventBridge desde este plan. Consumir los ya existentes. DynamoDB se usa exclusivamente para Notifications (fuera del alcance de Matching Context).

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
    user_id TEXT NOT NULL,
    career_id TEXT NOT NULL,
    validation_score SMALLINT CHECK (validation_score BETWEEN 1 AND 5),
    created_at TIMESTAMPTZ DEFAULT now(),
    INDEX idx_user ON feedback(user_id)
    );
    ```
    _Requerimientos: 9.1, 10.2_
  - [ ] 14.3 Crear tabla `rankings`:
    ```sql
    CREATE TABLE rankings (
        id BIGSERIAL PRIMARY KEY,
    user_id TEXT NOT NULL,
    ranking_json JSONB NOT NULL,
    reproducibility_metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT now(),
    INDEX idx_user ON rankings(user_id)
    );
    ```
    _Requerimientos: 9.1, 10.2_

- [ ] 15. Implementar `backend/persistence/feedback_storage.py` — FeedbackStorage (Aurora)
  - [ ] 15.1 Implementar clase `FeedbackStorage`:
    - Constructor: inicializa conexión PostgreSQL vía `psycopg2.connect(dsn=AURORA_CONN_STRING)`.
    - Valida conexión; si falla: loguea `ERROR` y almacena fallbacks en queue local (in-memory).
  - [ ] 15.2 Implementar método `save_ranking(self, user_id: str, ranking_json: dict, metadata: dict) -> str`:
    - Serializa `ranking_json` y `metadata` a JSON strings.
    - Inserta en tabla `rankings`: `INSERT INTO rankings (user_id, ranking_json, reproducibility_metadata, created_at) VALUES (...)`.
    - Genera y retorna `ranking_id` (uuid4).
    - Si DB no disponible: almacena en queue local, loguea warning, retorna provisional ranking_id.
    _Requerimientos: 9.1_
  - [ ] 15.3 Implementar método `save_feedback(self, user_id: str, ranking_id: str, validation_score: int, selected_career: str | None, notes: str | None) -> None`:
    - Valida `validation_score ∈ [1, 5]`: si no, lanza `ValueError`.
    - Inserta en tabla `feedback`: `INSERT INTO feedback (user_id, career_id, validation_score, created_at) VALUES (...)` (or updateranking_json si es update).
    - Si DB no disponible: queue local.
    _Requerimientos: 9.1_
  - [ ] 15.4 Implementar método `get_feedback_by_user(self, user_id: str) -> list[dict]`:
    - Consulta: `SELECT * FROM feedback WHERE user_id = %s`.
    - Filtra siempre por `user_id` (aislamiento garantizado por JWT sub).
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

- [ ] 16. Implementar `backend/auth.py` — Auth Service (JWT desde Identity Context)
  - [ ] 16.1 Implementar función `validate_jwt(token: str) -> dict | None`:
    - Decodifica JWT usando `PyJWT.decode(token, secret, algorithms=["HS256"])`.
    - `secret` se obtiene desde AWS Secrets Manager (compartido con Identity Context).
    - Si válido: retorna payload decodificado (contiene `sub` = userId, `exp`, etc.).
    - Si expirado (`ExpiredSignatureError`): retorna None.
    - Si firma incorrecta (`InvalidSignatureError`): retorna None.
    - Si cualquier otro error: loguea warning, retorna None.
    - Loguea auditoría: `"JWT validated: userId={sub}"`.
    _Requerimientos: 8.3_
  - [ ] 16.2 Implementar clase `JWTValidator`:
    - Constructor: lee `JWT_SECRET` desde Secrets Manager (SSM Parameter `JWT_SECRET`).
    - Almacena en `self.secret` y `self.algorithm = os.getenv("JWT_ALGORITHM", "HS256")`.
    - Método `validate(self, token: str) -> dict | None`: envuelve `validate_jwt()`.
    - NOTA: El Matching Context NO emite JWT — solo valida. La emisión es responsabilidad del Identity Context (TypeScript/Lambda, endpoints `/v1/auth/login` y `/v1/auth/register`).
    - Si el secreto no está disponible en startup (Secrets Manager inaccesible), el servicio no arranca (dependencia obligatoria).
    _Requerimientos: 8.3_
  - [ ] 16.3 Tests: `test_auth_service.py`:
    - Mockea `PyJWT.decode` con token válido → retorna payload con `sub: "user-123"`.
    - Mockea `PyJWT.decode` con `ExpiredSignatureError` → retorna None.
    - Mockea `PyJWT.decode` con `InvalidSignatureError` → retorna None.
    - Verifica que `validate_jwt` loguea auditoría en caso exitoso.
    - NOTA: Google OAuth puede integrarse como extensión futura si se requiere identidad de usuario.

- [ ] 17. Integrar validación JWT desde Identity Context
  - [ ] 17.1 Implementar validación JWT en Lambda handlers:
    - En cada Lambda handler (`matching_handler`, `feedback_handler`), el JWT llega en el header `Authorization`.
    - Extraer token: `event["headers"]["Authorization"].removeprefix("Bearer ")`.
    - Validar con `JWTValidator.validate(token)`.
    - Si inválido: retornar `{"statusCode": 401, "body": "Unauthorized"}` inmediatamente.
    - Si válido: extraer `userId` de `payload["sub"]`.
    - NOTA: El endpoint `/v1/auth/login` y `/v1/auth/register` viven en el Identity Context (TypeScript/Lambda). El Matching Context solo consume/valida JWT; no emite ni gestiona sesiones.
    _Requerimientos: 8.1, 8.3_
  - [ ] 17.2 Implementar `feedback_handler.lambda_handler(event, context)` (ya cubierto en 10.5):
    - Activado por API Gateway v2 (ruta `POST /feedback`).
    - Valida JWT → extrae `userId`.
    - Parsea body → persiste feedback en Aurora.
    - Retorna 200/401/422 según corresponda.
    _Requerimientos: 8.1, 9.1_
  - [ ] 17.3 Tests: `test_lambda_handlers.py`:
    - Mockea eventos API Gateway con JWT válido e inválido.
    - Verifica `/feedback` con JWT válido → 200.
    - Verifica `/feedback` con JWT inválido → 401.
    - Verifica `/feedback` con score fuera de rango → 422.

- [ ] 18. Integrar JWT como mecanismo único de autenticación en Lambdas
  - [ ] 18.1 Integrar `JWTValidator` en `matching_handler`:
    - El `matching_handler` es invocado por EventBridge (no por API Gateway), por lo que NO recibe JWT en el evento.
    - El `AssessmentCompleted` ya contiene `userId` validado por Assessment Context.
    - Se confía en el `userId` del evento (Assessment Context ya validó JWT antes de emitir).
    - Validar que `event["detail"]["userId"]` no sea nulo o vacío.
    _Requerimientos: 8.3_
  - [ ] 18.2 Integrar `JWTValidator` en `feedback_handler`:
    - El `feedback_handler` es invocado por API Gateway, recibe JWT en header `Authorization`.
    - Extraer token, validar, extraer `userId`.
    - Verificar que el `rankingId` del body pertenece al `userId` del token (security: no validar feedback de otro usuario).
    - Si `userId` no coincide: retorna 403.
    _Requerimientos: 8.3, 9.1_
  - [ ] 18.3 Tests de integración: `test_auth_persistence_integration.py`:
    - Flujo: mockear `JWTValidator.validate()` → retorna payload con `userId`.
    - Simular evento `AssessmentCompleted` → `matching_handler` genera ranking.
    - Simular `POST /feedback` con JWT válido → persiste validación.
    - Verifica datos persistidos en Aurora (mockeado).

- [ ] 19. Implementar Georreferencia — Universidades desde Aurora `career.career_offerings`
  - [ ] 19.1 (Omitido — DynamoDB `universities` NO existe)
    - Las universidades residen en `career.career_offerings` (Aurora, schema `career`),
      no en DynamoDB. DynamoDB se usa exclusivamente para Notifications (fuera del alcance
      del Matching Context).
    - Para consultar universidades, filtrar sobre `career.career_offerings.location`
      directamente desde Aurora.
  - [ ] 19.2 Implementar clase `UniversityQuery` en `backend/persistence/university_query.py`:
    - Constructor: `__init__(self, db_connection_string: str)` — conexión a Aurora.
    - Método `get_universities_by_location(self, location: str | None) -> list[dict]`:
      ```sql
      SELECT DISTINCT institution, location, management_type
      FROM career.career_offering
      WHERE ($1::text IS NULL OR location = $1)
      ORDER BY institution
      ```
    - Retorna lista de dicts con `{institution, location, management_type}`.
    - Método `get_all_locations(self) -> list[str]`:
      ```sql
      SELECT DISTINCT location FROM career.career_offerings ORDER BY location
      ```
    - Si `location=None` en `get_universities_by_location`: retorna todas (sin filtro).
    _Requerimientos: 9.3_
  - [ ] 19.3 Tests: `test_university_query.py`:
    - Usa Postgres local con tabla `career.career_offerings` poblada con 3+ registros.
    - Verifica `get_universities_by_location("Lima")` retorna solo universidades con location="Lima".
    - Verifica `get_universities_by_location(None)` retorna todas.
    - Verifica `get_all_locations()` retorna lista única de locations.

- [ ] 20. Checkpoint Fase 3 — Validación de Matching Context con Persistencia
  - [ ] 20.1 Ejecutar flujo completo contra recursos mockeados:
    - Postgres local (docker-compose) en lugar de Aurora.
    - Simular evento `AssessmentCompleted` con perfil completo → invoca `matching_handler.lambda_handler()`.
    - Verificar que emite `RecommendationGenerated` (mockear EventBridge).
    - Simular `POST /feedback` con JWT mockeado → persistido en Postgres.
    - Consultar `UniversityQuery.get_universities_by_location()` contra Postgres local → retorna lista, filtrado por location.
  - [ ] 20.2 Verificar aislamiento por `userId`:
    - Simular 2 eventos con distintos `userId`.
    - Feedback del usuario A → consultar feedback del usuario A → solo ve sus datos.
    - Feedbacks de ambos usuarios en DB pero query filtrado por `userId`.
  - [ ] 20.3 Ejecutar suite: `uv run pytest tests/ --cov`.
    - Cobertura ≥ 60% en `backend/persistence/`, `backend/auth.py`, `backend/lambda/`.
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

- [ ] 23. Integrar RAG en AI Advisor (fuera de alcance de Matching Context)
  - [ ] (Opcional — pospuesto) 23.1 El RAG query se integra en el AI Advisor (repo `08-deep-agent`), no en el Matching Context:
    - AI Advisor recibe el ranking desde `RecommendationGenerated`.
    - Si el usuario pregunta detalles sobre una carrera del Top-5, AI Advisor invoca RAG contra Aurora (pgvector).
    - Matching Context no expone endpoint RAG — eso es responsabilidad del AI Advisor.
    - Si RAG no disponible: AI Advisor retorna respuesta sin información adicional (graceful degradation).
    _Requerimientos: 10.3_
  - [ ] (Opcional — pospuesto) 23.2 Tests: `test_rag_integration.py` (en repo AI Advisor):
    - Simula pregunta de detalle sobre carrera del Top-5 con RAG disponible.
    - Verifica que respuesta incluye información de RAG.
    - Simula con RAG fallando: verifica que flujo continúa sin error.

- [ ] 24. (Opcional) Implementar Visualización de Mapa en Frontend
  - [ ] (Opcional) 24.1 Agregar en `frontend/index.html`:
    - Div para mapa: `<div id="map"></div>`.
    - Script para Leaflet: cargar biblioteca (CDN o npm).
  - [ ] (Opcional) 24.2 Agregar en `frontend/app.js`:
    - Función `displayMap(top5_items, location)` (recibe `profile.location`).
    - Consumir endpoint `/universities?location={location}` (responde desde Aurora `career.career_offerings`) vía `fetch()`.
    - Pintar marcadores de universidades en mapa (Leaflet markers).
    - Llamar después de generar ranking.
  - [ ] (Opcional) 24.3 Tests: verificación manual en navegador (no automatizados).

---

## Fase 5 — Batch a Escala Completa, Robustez y Pulido Final (Alta)

- [ ] 25. Implementar `data_pipeline/riasec_tagging.py` (Ampliación) — Pipeline Batch Completo
  - [ ] 25.1 Implementar script `data_pipeline/pipeline_runner.py`:
    - Función `run_full_pipeline(db_connection_string: str) -> str`:
      - Paso 1: `download_ponte_en_carrera()` → `data/raw.xlsx`.
      - Paso 2: `clean_and_validate()` → `data/clean_data.csv` (intermedio).
      - Paso 3: `generate_features()` → genera DataFrame con features normalizadas.
      - Paso 4: Extraer carreras únicas (`unique_careers = features_df[['id','career','career_family']].drop_duplicates(subset='career')`) → 554 filas.
      - Paso 4b: `tag_careers_with_bedrock(unique_careers)` — etiqueta solo las 554 carreras únicas (mockeado en demo, real en producción).
      - Paso 4c: `features_df = features_df.merge(riasec_tags, on='career', how='left')` — propaga etiquetas a las 6.208 filas.
      - Paso 5: `validate_sample()` → `data/riasec_validation_sample.csv` para revisión humana.
      - Paso 6: `apply_family_fallback()` → garantiza 100% carreras con RIASEC.
      - **Paso 7**: Escribir batch a Aurora PostgreSQL (schema `career`, tabla `career_offerings`):
        - Conectar vía `psycopg2.connect(dsn=db_connection_string)`.
        - Reemplazar snapshot completo: `DELETE FROM career.career_offerings;`.
        - Insert batch con `executemany()` o `COPY` (≈ 6.208 filas):
          ```python
          rows = features_df.to_dict('records')
          with conn.cursor() as cur:
              for row in rows:
                  cur.execute("""
                      INSERT INTO career.career_offerings
                      (career_id, career, institution, career_family, riasec_profile,
                       riasec_source, monthly_income_imputed, annual_cost_imputed,
                       admission_rate_imputed, duration_years_imputed,
                       income_norm, cost_norm, admission_norm, duration_norm,
                       location, management_type, institution_type, modalidad,
                       snapshot_id)
                      VALUES (...)
                  """, row)
              conn.commit()
          ```
      - **Paso 8**: Generar snapshot versionado local: `snapshots/features/features_{timestamp}.csv`
        (solo como respaldo, no como fuente primaria).
      - Retorna ruta del snapshot local.
    - Loguea cada paso.
    _Requerimientos: 6.1, 6.2, 6.4_
  - [ ] 25.2 Documentar trigger EventBridge (cron):
    - Cron expression: `cron(0 2 * * ? *)` (diario a las 2 AM UTC).
    - Target: AWS Batch job que invoca `uv run python -m data_pipeline.pipeline_runner`.
    - Resultado: actualización automática de `career.career_offerings` en Aurora cada día.
    _Requerimientos: 6.2_
  - [ ] 25.3 Tests: `test_pipeline_runner.py`:
    - Ejecuta `run_full_pipeline()` sobre fixture de 50 carreras (mockeando Bedrock, operando sobre carreras únicas).
    - Verifica que el tagging se aplicó sobre carreras únicas (no más de 50 filas etiquetadas).
    - Verifica que snapshot final tiene `riasec_profile` non-null en todas las filas (join propagó correctamente).
    - Verifica que los datos en Aurora (mockeado) tienen columnas `income_norm`, `cost_norm`, `admission_norm`, `duration_norm` en `[0,1]`.

- [ ] 26. Tests de Integración End-to-End con Fixtures Reales
  - [ ] 26.1 Crear fixtures en `tests/fixtures/`:
    - `catalog_50_careers.csv`: 50 carreras con datos verificables (ingresos, costos, admisión, duración).
    - `seed_riasec_10.csv`: 10 carreras con RIASEC etiquetado manualmente (semilla).
  - [ ] 26.2 Test: `test_integration_full_flow.py`:
    - Setup: Poblar Postgres local con fixtures de `career.career_offerings`.
    - Ejecuta `run_full_pipeline()` que escribe en Aurora `career.career_offerings` (mockeado o en Postgres local).
    - Crea `ScoringEngine` con fixtures.
    - Simula un evento `AssessmentCompleted` con perfil completo (riasec_scores, weights, filters).
    - Invoca `matching_handler.lambda_handler(event, context)`.
    - Verifica que emite `RecommendationGenerated` (mockear `boto3.client("events")`).
    - Verifica:
      - Top-5 generado correctamente.
      - Aislamiento por `userId` (event tiene userId).
      - Datos persistidos en Aurora (mockeado).
    - Módulos implicados: `data_pipeline`, `backend/scoring`, `backend/lambda/matching_handler`, `backend/persistence`.
    _Requerimientos: 5.4, 8.1, 8.2, 8.3, 9.1, 10.2_

- [ ] 27. Validación de Restricciones No Funcionales
  - [ ] 27.1 Tests basados en propiedades: **Propiedad 4 — Reproducibilidad**. **Valida: Requerimiento 9 criterio 9.1**. Estrategia:
    - Fijar snapshot `features.csv` o snapshot de `career.career_offerings` (timestamp conocido).
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
    - Descripción clara del proyecto (solo Matching Context + Data Pipeline).
    - Arquitectura real:
      - API Gateway v2 (HTTP) enruta a Lambdas por contexto (Identity/Assessment/Career en TypeScript, Matching en Python).
      - EventBridge (`spark-match-events`) para handlers async.
      - AI Advisor: servicio Python separado (repo `08-deep-agent`) para chat+RAG.
      - Matching Context (este repo): scoring determinístico en Python 3.12 Lambda.
    - Aurora PostgreSQL: persistencia (feedback, rankings, pgvector) + `career.career_offerings` (carreras × universidades).
    - DynamoDB: solo Notifications (fuera del alcance de Matching Context).
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
    - SÍ loguear: `user_id`, timestamps, nombres de carreras (públicos), scores, pesos, eventos de flujo.
    - Ejemplo: `"Ranking generated: user_id={user_id}, career_count=5, confidence_score=0.85"`.
    - Configuración centralizada en `backend/utils.py` (función `setup_logging()`).
    _Requerimientos: 10.4_
  - [ ] 28.3 Crear `docs/ARCHITECTURE.md`:
    - Diagrama de componentes (ASCII o Mermaid).
    - Flujo de datos end-to-end.
    - Justificación de decisiones AWS (Bedrock vs OpenAI, Aurora vs DynamoDB, etc.).
    - Links a design.md y requirements.
  - [ ] 28.4 Crear `docs/DEPLOYMENT.md`:
    - Guía de despliegue del Matching Context en AWS:
      - Consumir Aurora cluster existente (no crear).
      - Consumir DynamoDB table existente (no crear).
      - Configurar Lambda (Python 3.12) para Matching Context.
      - Conectar a EventBridge bus existente (`spark-match-events`).
      - Leer SSM Parameters y Secrets Manager existentes.
      - Build + push Docker image (solo para desarrollo local).
      - Configurar API Gateway v2 routes hacia Lambdas por contexto.
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
    - Simular evento `AssessmentCompleted` → `matching_handler` genera ranking → emite `RecommendationGenerated`.
    - Aislamiento por `userId` verificado.
    - Todos los módulos del Matching Context integrados y funcionando.
  - [ ] 29.4 Ejecutar benchmark de performance (`test_scoring_performance.py`):
    - 6,208 carreras → score ≤ 1 segundo ✓.
  - [ ] 29.5 Verificar README, docs, y estructura de proyecto:
    - `README.md` actualizado con instrucciones claras.
    - `docs/ARCHITECTURE.md` documenta decisiones (incluyendo EDA).
    - `docs/DEPLOYMENT.md` lista pasos de deploy (consumir infra existente).
    - `.env.example` con todas las variables necesarias.
    - `pyproject.toml` y `uv.lock` actualizados.
    - `Dockerfile` y `docker-compose.yml` funcionales (solo desarrollo local).
    - Todos los archivos en sus ubicaciones correctas.
  - [ ] 29.6 Documento de seguimiento:
    - Generar reporte de cobertura: `coverage html`, revisar `htmlcov/index.html`.
    - Listar todos los requerimientos cubiertos (R1–R10, C1–C7).
    - Documentar cualquier desviación o trabajo pendiente.
    - Firmar off como "Fase 5 completa — Matching Context listo para integración con otros contextos".

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
  - Framework servicio local (desarrollo Matching Context): `FastAPI` + `Uvicorn` (NO Flask, NO Django).
  - Framework Lambda (producción): `boto3` puro (NO frameworks adicionales como `zappa`).
  - Credenciales: ÚNICAMENTE vía variables de entorno o AWS Secrets Manager (NUNCA hardcoded, NUNCA en código).
  - LLM: ÚNICAMENTE Bedrock/Claude (NO Gemini, NO OpenAI en este ciclo).
  - Embeddings: ÚNICAMENTE Bedrock Titan (NO sentence-transformers local, NO OpenAI).
  - Vector DB: ÚNICAMENTE pgvector en Aurora (NO FAISS local, NO Pinecone, NO Chroma).

- **Sesión conversacional**: La sesión conversacional vive en el AI Advisor (repo `08-deep-agent`), no en el Matching Context. Para Matching Context, el JWT del Identity Context se valida en cada invocación Lambda. El Matching Context NO tiene estado de sesión en memoria — toda la información llega en el evento `AssessmentCompleted`.

- **Autenticación JWT (2.2)**: El JWT es emitido por el Identity Context (TypeScript/Lambda con endpoints `register`/`login`/`users/me`). No existe `session_id` anónimo. El Matching Context valida JWT con `PyJWT.decode()` usando el secreto compartido desde Secrets Manager. El Identity Context es una dependencia obligatoria del sistema; sin él, ni los Lambdas ni los servicios locales pueden operar. Google OAuth es una extensión futura posible.

- **Coreografía de eventos**: No existe clase `Orchestration` monolítica. La coordinación entre contextos es vía eventos EventBridge: Assessment emite `AssessmentCompleted`, Matching consume y emite `RecommendationGenerated`, AI Advisor consume. Cada contexto es autónomo y reacciona a eventos de forma asíncrona.

- **Matching Context es solo una pieza (2.3)**: Este plan cubre únicamente Matching Context (scoring determinístico Python 3.12 Lambda) + Data Pipeline. Identity, Assessment, Career y AI Advisor se implementan en otros repositorios. El `Dockerfile`/`docker-compose.yml` son solo para desarrollo local del Matching Context, no para el backend completo.

- **Etiquetado RIASEC** (6,208 carreras): Corre como **job batch** (AWS Batch, NO Lambda) por duración de llamadas Bedrock (potencialmente horas). Pipeline automatizado genera snapshot versionado. Validación muestral (300 carreras) es paso humano externo, pero export CSV es responsabilidad pipeline (tarea 25.2).

- **Fases son incrementales e integradas**: Cada una termina con checkpoint que verifica estado funcional. Fase N depende de trabajo de Fase N-1 pero NO rompe su funcionalidad. Puntos de integración y reemplazos están explícitos (ej Tarea 18.1: "Modificar endpoint").

- **Granularidad de tareas**: Nivel superior = módulo/archivo/integración/test. Nivel 2 = función pública o grupo cohesivo. NO hay nivel 3 (sub-subtasks). Si una subtarea > 12–15 líneas detalle, dividir en varias subtasks.

- **Autosuficiencia**: Cada subtarea incluye inline suficiente detalle (firmas, parámetros, defaults, thresholds, errores, fallbacks) para implementar sin consultar constantemente `design.md`. Cuando aplique: código pseudocode, SQL, ejemplos, comandos exactos.

- **Trazabilidad obligatoria**: Cada subtarea de implementación tiene `_Requerimientos: X.Y_`. Cada criterio (R1–R10, C1–C7) aparece en al menos una. Tests tienen referencias explícitas a propiedades (negrita) o requerimientos.

- **Orden de tareas respeta dependencias reales**: Data pipeline (Fase 1) antes del Scoring local (Fase 2). Scoring determinístico antes de persistencia. Auth y persistencia (Fase 3) antes de RAG (Fase 4, opcional). Batch (Fase 5) último.

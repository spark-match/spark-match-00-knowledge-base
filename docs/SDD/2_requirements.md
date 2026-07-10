---
title: "Requirements Specification — CareerMatch Perú"
author: "@FabiTaparaQuispe"
date: 2026-07-04
updated: 2026-07-09
tags:
  - area/architecture
  - topic/product/requirements
  - status/draft
audience:
  - product-owners
  - tech-leads
  - members
status: draft
related:
  - "[[MOC-architecture]]"
  - "[[docs/SDD/1_PRD]]"
  - "[[docs/SDD/3_design]]"
---

# requirements.md

## Introducción

CareerMatch Perú es una plataforma de orientación vocacional basada en inteligencia artificial generativa que ayuda a estudiantes de educación secundaria en Perú a tomar decisiones informadas sobre la elección de carrera universitaria. El sistema interpreta preferencias expresadas en lenguaje natural mediante un asistente conversacional, mantiene diálogos iterativos para capturar información incompleta, y genera recomendaciones personalizadas de combinaciones carrera–universidad sustentadas en datos oficiales del Ministerio de Educación. La solución proporciona transparencia y auditabilidad mediante un motor de evaluación multi-criterio desacoplado, almacena feedback de usuarios para construir una base de datos histórica de entrenamiento futuro, y permite acceso conversacional a detalles de carreras mediante un módulo RAG opcional. CareerMatch Perú es valioso porque reduce la brecha de información en orientación vocacional, es accesible en contextos de recursos limitados, mantiene datos verificables, y construye un activo estratégico (feedback histórico) que habilita modelos propios independientes a largo plazo.

---

## Glosario

### Producto e Interfaz

**CareerMatch Perú**  
Plataforma de orientación vocacional basada en IA generativa que interpreta preferencias de estudiantes y genera recomendaciones personalizadas de carreras universitarias usando datos oficiales del MINEDU.

**Web_Interface**  
Interfaz conversacional accesible a través de navegador web (responsive, sin app nativa) que permite al usuario expresar consultas en lenguaje natural, visualizar ranking de recomendaciones y proporcionar validación de feedback.

### Componentes, Módulos y Subsistemas

**Auth_Service**  
Módulo responsable de autenticación, autorización y gestión de sesiones de usuario. Crea, valida y persiste tokens de sesión, aisla datos entre usuarios y revoca sesiones al cierre.

**Data_Pipeline**  
Módulo que descarga datos de Ponte en Carrera, aplica limpieza, imputación jerárquica y normalización de variables para generar features normalizados listos para evaluación. Compuesto por tres etapas: `ingestion.py`, `data_clean.py`, `feature_engineering.py`.

**LLM_Layer**  
Módulo que interpreta preferencias del usuario en lenguaje natural, detecta información incompleta, genera preguntas conversacionales de seguimiento, produce explicaciones personalizadas y opcionalmente genera pesos dinámicos. Utiliza modelos fundacionales (Gemini API, GPT-4o-mini u equivalentes).

**Scoring_Engine**  
Módulo que calcula indicadores de concordancia multi-criterio para cada combinación carrera–universidad, ordena opciones en ranking consistente y facilita exploración de alternativas. No es definitivo; es propuesta inicial evaluable susceptible de evolución basada en feedback histórico.

**RAG_Module**  
Módulo opcional que permite consultas conversacionales sobre detalles de carreras recomendadas (cursos, descripción, perfil profesional, etc.) mediante recuperación-aumento-generación sobre documentos oficiales embeddeados. Inicialmente cubre 3–5 carreras piloto.

**Feedback_Storage**  
Módulo que persiste consultas, perfiles interpretados, rankings generados, explicaciones, validaciones Likert y metadatos de sesión en base de datos, construyendo históricamente una base de datos de entrenamiento para modelos futuros.

**Session_Manager**  
Módulo que mantiene contexto de conversación multi-turno dentro de una sesión, actualiza perfil interpretado con nueva información, recalcula confidence para recomendación y gestiona transiciones entre estados de diálogo.

**Orchestration_Layer**  
Módulo que coordina flujo entre **Auth_Service**, **LLM_Layer**, **Scoring_Engine**, **RAG_Module** y **Feedback_Storage**. Maneja endpoints REST, validación de inputs, gestión de errores y logging de auditoría.

**Embedding_Service**  
Servicio agnóstico que genera embeddings de texto para soporte a **RAG_Module** y cálculo de afinidad. Puede utilizar modelos de APIs (OpenAI, Google, etc.) o locales (Hugging Face, etc.).

**Vector_Database**  
Almacenamiento agnóstico de vectores embeddeados de documentos de carreras y chunks de planes curriculares para soporte a **RAG_Module**. Puede ser FAISS (local), Pinecone, Chroma, Weaviate o equivalente.

### Términos Técnicos

**Afinidad**  
Medida cuantitativa [0,1] de alineación entre los intereses interpretados de un estudiante y el perfil académico de una carrera. Método de cálculo a definir: embeddings + similitud coseno, tabla de mapeo semántico, score directo de LLM u otro.

**Indicador de Concordancia**  
Puntaje numérico [0,1] que expresa qué tan bien una combinación carrera–universidad se alinea con el perfil y restricciones del estudiante. Derivado de evaluación multi-criterio (afinidad, ingresos, costo, facilidad).

**Pesos Dinámicos**  
Coeficientes {w₁, w₂, w₃, w₄} generados por el **LLM_Layer** en cada sesión basados en preferencias expresadas por el usuario. Síntesis: wᵢ ∈ [0,1], Σwᵢ = 1.0. Alternativamente, pueden ser fijos o explorables manualmente.

**Confidence Score**  
Medida [0,1] que indica cuánta información tiene el sistema para generar recomendación confiable. Si < 0.70, el **Session_Manager** dispara pregunta conversacional de seguimiento.

**Graceful Degradation**  
Comportamiento de fallback donde el sistema genera recomendación con información disponible sin forzar al usuario a responder completamente. Máximo 3–4 turnos de seguimiento antes de recomendar.

**Reproducibilidad**  
Capacidad de obtener exactamente los mismos resultados en una consulta anterior usando snapshots versionados de datos, configuración y prompts. Todas las decisiones son trazables y auditables.

**Versionado**  
Práctica de mantener snapshots históricos con timestamp de datasets crudos, features normalizados, configuración de imputación y prompts (v1, v2, v3) para trazabilidad completa.

**Feedback Loop**  
Ciclo de captura de validaciones de usuario (escala Likert 1–5) sobre recomendaciones, almacenamiento histórico, y análisis para mejorar futuras recomendaciones o entrenar modelos propios.

**RAG (Retrieval-Augmented Generation)**  
Técnica que recupera fragmentos relevantes de documentos embeddeados y los proporciona al LLM para generar respuestas contextuales y fundamentadas sobre detalles de carreras específicas.

**Sesión de Usuario**  
Período de interacción autenticada de un usuario con la plataforma, identificado por `session_id`. Contiene contexto conversacional, perfil interpretado, confianza acumulada y feedback generado. Aislada de otras sesiones de usuario.

**Token de Sesión**  
Identificador criptográfico opaco que valida autenticidad y autorización de una sesión. Vinculado a usuario único. Almacenado como `session_token`.

---

## Requerimientos

### Requerimiento 1: Autenticación y Creación de Sesión de Usuario

**User Story:** Como estudiante o evaluador, quiero iniciar sesión en la plataforma para acceder a mis consultas de orientación vocacional sin que mis datos se mezclen con otros usuarios.

#### Criterios de Aceptación

1. WHEN el usuario accede a `Web_Interface` por primera vez, THE **Auth_Service** SHALL generar un `session_id` único (UUID v4 mínimo) y un `session_token` criptográficamente seguro.
2. THE `session_id` SHALL ser almacenado en el navegador (local storage o cookie HTTP-only con flag Secure y SameSite).
3. THE `session_token` SHALL ser validado en cada solicitud API para verificar autenticidad; IF token es inválido o expirado, THE **Orchestration_Layer** SHALL retornar código HTTP 401 Unauthorized.
4. THE sesión SHALL tener duración máxima de 8 horas de inactividad; IF no hay actividad en este período, THE **Auth_Service** SHALL revocar automáticamente el token y requerir reinicio de sesión.
5. WHEN el usuario cierra la sesión explícitamente o el navegador es cerrado, THE **Auth_Service** SHALL invalidar el `session_token` y limpiar datos de sesión de memoria.
6. THE **Auth_Service** SHALL mantener aislamiento de datos: consultas, perfiles interpretados, rankings y feedback de un usuario SHALL no ser accesibles por otro usuario, incluso si comprometiera su token.
7. THE `session_id` SHALL ser incluido en todos los logs y registros de auditoría para trazabilidad de acciones por usuario.

---

### Requerimiento 2: Gestión de Sesión Multi-Turno Conversacional

**User Story:** Como usuario, quiero mantener una conversación con múltiples turnos dentro de la misma sesión, donde el sistema recuerda información previa y actualiza su comprensión de mis preferencias.

#### Criterios de Aceptación

1. THE **Session_Manager** SHALL mantener un `session_context` que incluya: histórico de turnos de conversación, perfil interpretado acumulado, confidence score actual, y estado de diálogo (aguardando entrada, procesando, listo para recomendar).
2. WHEN el usuario envía un nuevo mensaje en el mismo `session_id`, THE **Session_Manager** SHALL recuperar el contexto previo y concatenarlo con el nuevo mensaje antes de pasar al **LLM_Layer**.
3. THE perfil interpretado SHALL ser actualizado incrementalmente: nueva información no anula previa, sino que refina o completa (ej: si usuario primero dice "matemáticas" y luego "También me gusta diseño", ambos intereses son registrados).
4. THE `confidence_score` SHALL ser recalculado en cada turno basado en completitud de información necesaria para recomendación (intereses presentes, restricciones financieras presentes, preferencia geográfica explícita o flexible, etc.).
5. IF confidence_score < 0.70 AFTER nuevo turno, THE **LLM_Layer** SHALL generar pregunta conversacional contextual (no invasiva, abierta) dirigida a obtener información faltante.
6. IF confidence_score >= 0.70, THE sistema SHALL proceder a fase de evaluación (Requerimiento 7).
7. THE histórico de conversación SHALL persistir en **Feedback_Storage** asociado al `session_id` para auditoría y análisis posterior.
8. WHEN usuario inicia nueva consulta independiente dentro de la misma sesión (ej: "Ahora consideremos otro perfil"), THE **Session_Manager** SHALL permitir reset parcial del contexto si usuario lo especifica, manteniendo session_token vigente.

---

### Requerimiento 3: Interpretación de Preferencias en Lenguaje Natural

**User Story:** Como estudiante, quiero expresar mis preferencias y restricciones en lenguaje natural sin necesidad de llenar formularios, y quiero que el sistema entienda mis verdaderas intenciones.

#### Criterios de Aceptación

1. THE **LLM_Layer** SHALL aceptar consultas en español peruano (dialectal, informal) y generar una interpretación estructurada en formato JSON.
2. THE JSON de salida SHALL incluir al mínimo los campos: `interests` (array de strings), `salary_priority` (float 0–1), `cost_sensitivity` (float 0–1), `admission_tolerance` (float 0–1), `geographic_preference` (string o null), `institution_type_preference` (string: "pública", "privada", "cualquiera", o null).
3. THE `interests` array SHALL contener entre 1 y 5 carreras o campos de conocimiento interpretados (ej: ["matemáticas", "análisis de datos", "estadística"]).
4. THE valores `salary_priority`, `cost_sensitivity`, `admission_tolerance` SHALL ser válidos: números en rango [0,1] donde mayor valor = mayor importancia/sensibilidad.
5. IF el LLM genera JSON inválido o con campos faltantes, THE **Orchestration_Layer** SHALL registrar error y reintentarr con schema validation; IF falla 3 veces consecutivas, THE sistema SHALL retornar mensaje de error amigable al usuario sugiriendo reformular.
6. THE **LLM_Layer** SHALL usar few-shot prompting con mínimo 3 ejemplos de perfiles diferentes (estudiante enfocado en salario, estudiante con restricción presupuestaria, estudiante con múltiples intereses, etc.) incluidos en el system prompt v1.
7. WHERE información es ambigua o incompleta, THE **LLM_Layer** SHALL generar interpretación conservadora documentada en campo `confidence_reasoning` del JSON.

---

### Requerimiento 4: Detección de Información Incompleta y Diálogo Iterativo

**User Story:** Como sistema, quiero identificar cuando falta información importante para buena recomendación y pedir información adicional de forma natural y no invasiva.

#### Criterios de Aceptación

1. THE **LLM_Layer** SHALL calcular `missing_info` array que enumere campos del perfil que tienen valor null o especificación incompleta (ej: `["geographic_preference", "institution_type_preference"]`).
2. THE `confidence_score` SHALL ser función de completitud: calcularse como porcentaje de campos no-null en perfil estructurado. Formula: `confidence = (numero_de_campos_llenos / numero_total_campos_requeridos)`, con mínimo de campos requeridos: 4 (interests, salary_priority, cost_sensitivity, admission_tolerance).
3. IF `missing_info` array no está vacío AND `confidence_score` < 0.70, THE **LLM_Layer** SHALL generar campo `suggested_follow_up` contiendo pregunta conversacional que cumpla: (a) es abierta (no SÍ/NO), (b) menciona por qué la información es útil, (c) no repite información ya obtenida, (d) es contextual al perfil parcial conocido.
4. THE pregunta SHALL ser redactada como una oración conversacional en español peruano, no como formulario.
5. Example correcto: "¿Hay alguna región del Perú donde prefieras estudiar, o tienes flexibilidad? Algunos estudiantes lo dejan abierto para más opciones."  
   Example incorrecto: "¿Qué región? (Respuesta obligatoria)"
6. THE sistema SHALL mantener un contador de turnos de seguimiento; IF contador alcanza 4 turnos, THE sistema SHALL generar recomendación con información disponible sin hacer más preguntas, incluso si confidence < 0.70, documentando en log que fue "graceful degradation".
7. IF usuario proporciona información contradictoria entre turnos (ej: primero "salario es crítico", luego "cualquier presupuesto"), THE **Session_Manager** SHALL privilegiar información más reciente pero registrar ambas en auditoría.

---

### Requerimiento 5: Descarga y Validación de Datos de Ponte en Carrera

**User Story:** Como Data_Pipeline, quiero obtener datos oficiales del MINEDU, validarlos y prepararlos para evaluación de recomendaciones.

#### Criterios de Aceptación

1. THE **Data_Pipeline** SHALL descargar archivo Excel oficial de Ponte en Carrera desde `https://ponteencarrera.minedu.gob.pe/pec-portal-web/` utilizando Selenium o equivalente (agnóstico de tecnología).
2. THE descarga SHALL generar un snapshot versionado con timestamp en formato `raw_YYYYMMDD_HHMMSS.xlsx` almacenado en carpeta `snapshots/raw/`.
3. THE archivo Excel original SHALL ser procesado desde header = 6 (estructura interna de Ponte en Carrera).
4. THE **Data_Pipeline** SHALL extraer las siguientes 14 columnas: `career_family`, `career`, `institution`, `location`, `institution_type`, `management_type`, `duration_years`, `annual_cost`, `admission_rate`, `monthly_income`, `scholarships_available`, `admitted`, `applicants`, `enrolled`.
5. WHEN durante lectura se detecten filas sin valores en `career` o `institution`, THE **Data_Pipeline** SHALL descartar esas filas y registrar cantidad descartada en log.
6. WHERE datos presentan tipos incorrectos (ej: `duration_years` es texto cuando debe ser numérico), THE **Data_Pipeline** SHALL intentar conversión automática; IF conversión falla, SHALL marcar registro con flag `data_quality_issue: true`.
7. THE resultado de limpieza SHALL ser exportado a `data/filtered.csv` en formato UTF-8 con BOM (encoding UTF-8-sig).

---

### Requerimiento 6: Imputación Jerárquica y Feature Engineering

**User Story:** Como Data_Pipeline, quiero completar valores faltantes de manera consistente, auditable y documentada, y generar variables normalizadas listas para evaluación.

#### Criterios de Aceptación

1. THE **Data_Pipeline** SHALL implementar imputación jerárquica en cascada para cada variable crítica (`duration_years`, `monthly_income`, `annual_cost`, `admission_rate`):
   - Nivel 1: Usar mediana de valores válidos agrupados por (`career_family` + `institution_type`).
   - Nivel 2: Si Nivel 1 no produce suficientes valores (< 3), usar mediana agrupada únicamente por `career_family`.
   - Nivel 3: Si Nivel 2 insuficiente, usar valor fallback configurado en `feature_config.json`.

2. THE fallback values configurados (no hardcodeados) SHALL ser:
   - `duration_institute_fallback`: 4 años
   - `duration_university_fallback`: 5 años
   - `income_fallback`: S/. 1,100
   - `cost_fallback`: S/. 0
   - `admission_fallback`: 0%

3. FOR cada variable imputada, THE **Data_Pipeline** SHALL crear flag booleano asociado: `duration_imputed_flag`, `monthly_income_imputed_flag`, `annual_cost_imputed_flag`, `admission_rate_imputed_flag`. Valor TRUE indica que registro fue imputado (facilita auditoría).

4. THE **Data_Pipeline** SHALL aplicar validación de rangos tras imputación:
   - `duration_years`: debe estar en rango [3, 7]; IF fuera de rango, marcar como imputado y reasignar mediana.
   - `admission_rate`: debe estar en rango [0%, 90%]; IF > 90%, clipear a 90%.
   - `monthly_income`: debe ser > 0; IF <= 0, marcar como imputado.
   - `annual_cost`: debe ser >= 0.

5. THE **Data_Pipeline** SHALL generar 4 variables de scoring normalizadas a [0,1] mediante estas transformaciones:
   - `income_score = MinMax(log1p(monthly_income_imputed))` donde `log1p` es logaritmo natural de (1 + x) para reducir outliers, y `MinMax` normaliza al rango [0,1].
   - `cost_score = 1 - MinMax(log1p(annual_cost_imputed))` (invertida: mayor score = menor costo real).
   - `duration_score = 1 - MinMax(duration_years_imputed)` (invertida: mayor score = menor duración).
   - `admission_score = MinMax(admission_rate_imputed)` (directo: mayor score = más fácil ingresar).

6. WHERE `MinMax(x)` es definida como: `(x - min(x)) / (max(x) - min(x))`. IF min(x) == max(x), retornar vector de unos.

7. THE resultado final SHALL ser exportado a `data/features.csv` con todas las columnas: originales + imputed + scores normalizados + flags de imputación.

8. THE configuración de imputación SHALL ser persistida en `data/feature_config.json` en formato JSON con timestamps.

9. THE **Data_Pipeline** SHALL generar snapshot de features con timestamp: `features_YYYYMMDD_HHMMSS.csv` en carpeta `snapshots/features/`.

10. IF pipeline se ejecuta exitosamente, THE sistema SHALL generar snapshot también de configuración: `feature_config_YYYYMMDD_HHMMSS.json` en carpeta `snapshots/configs/`.

---

### Requerimiento 7: Cálculo de Afinidad entre Estudiante y Carrera

**User Story:** Como Scoring_Engine, quiero calcular una medida cuantitativa [0,1] de qué tan alineada está cada carrera con los intereses interpretados del estudiante.

#### Criterios de Aceptación

1. THE **Scoring_Engine** SHALL calcular `afinidad_score` [0,1] para cada combinación (estudiante, carrera) donde estudiante es representado por array de `interests` del perfil interpretado, y carrera es representada por `career_family` + descripción de carrera.

2. THE método de cálculo de afinidad será uno de los siguientes (seleccionado durante sprint de Data, antes del 28 jun):
   - **Opción A (Embeddings + Similitud Coseno)**: generar embedding de cada interés estudiante, generar embedding de descripción carrera, calcular similitud coseno promediada.
   - **Opción B (Tabla de Mapeo Semántico)**: tabla estática {interés → career_families afines con scores fijos}; retorna promedio de scores aplicables.
   - **Opción C (Score Directo LLM)**: LLM genera afinidad [0,1] directamente para cada carrera vs. perfil.
   - **Opción D (Modelo Propio - Futuro)**: entrenar modelo supervisado con datos históricos de feedback (solo viable post-demo con suficientes casos).

3. THE afinidad resultante SHALL ser valor numérico en rango [0,1] inclusive.

4. IF método elegido es Opción A (Embeddings), THE sistema SHALL utilizar modelo de embeddings agnóstico (OpenAI API, Google VertexAI, Hugging Face, etc.); modelo específico será definido en documento de Engineering, no aquí.

5. IF método elegido es Opción B (Tabla), THE tabla SHALL estar versionada en archivo `career_affinity_mapping.json` en `data/` con estructura:
   ```json
   {
     "interés": {
       "career_family_1": score,
       "career_family_2": score,
       ...
     }
   }
   ```
   donde score ∈ [0, 1].

6. THE afinidad_score para registro es archivado en columna `affinity_score` en `data/features.csv`.

7. THE método de cálculo elegido SHALL ser documentado en `data/AFINIDAD_METHOD.md` con: descripción, ejemplo de cálculo paso a paso, y razón por la cual fue elegido.

---

### Requerimiento 8: Motor de Evaluación Multi-Criterio

**User Story:** Como Scoring_Engine, quiero calcular un indicador de concordancia para cada combinación carrera–universidad que combine múltiples criterios de una manera consistente, auditable y adaptable.

#### Criterios de Aceptación

1. THE **Scoring_Engine** SHALL calcular un `concordancia_score` [0,1] para cada combinación (carrera, universidad) usando la fórmula base:
   ```
   concordancia_score = w1 * afinidad + w2 * income_score + w3 * cost_score + w4 * admission_score
   ```
   donde:
   - `w1`, `w2`, `w3`, `w4` son pesos proporcionados por **LLM_Layer** (pesos dinámicos) O valores configurados manualmente.
   - afinidad, income_score, cost_score, admission_score son valores [0,1] del dataset features.csv.

2. THE suma de pesos SHALL satisfacer: `w1 + w2 + w3 + w4 = 1.0` (tolerancia ±0.01). IF suma fuera de rango, THE **Orchestration_Layer** SHALL registrar warning y normalizar pesos automáticamente dividiendo cada uno por la suma total.

3. WHERE pesos son dinámicos (generados por LLM), cada peso SHALL estar validado: `wᵢ ∈ [0, 1]`.

4. WHERE pesos son fijos (fallback si LLM falla), THE valores por defecto SHALL ser: `w1=0.40, w2=0.30, w3=0.20, w4=0.10`.

5. THE **Scoring_Engine** SHALL aplicar opcionalmente estrategias alternativas de combinación a ser evaluadas durante demo (no ambas simultáneamente en producción inicial, pero arquitectura debe permitir switch):
   - Promedio geométrico: `(afinidad^w1 * income^w2 * cost^w3 * admission^w4)^(1/sum(w))`
   - Mínimo de criterios: `min(afinidad, income, cost, admission)` (garantiza aceptabilidad en todos)
   - Ponderación no-lineal: `w1*afinidad² + w2*income² + ...` (penaliza desbalance)

6. THE estrategia elegida SHALL ser documentada en `SCORING_STRATEGY.md` junto con razón de elección, especialmente si se descubre durante evaluación que es superior a propuesta inicial.

7. WHEN concordancia_score es calculado, THE **Scoring_Engine** SHALL también retornar desglose por criterio: `{afinidad, income_score, cost_score, admission_score}` para auditoría y explicación al usuario.

8. THE **Scoring_Engine** SHALL mantener `concordancia_score` [0,1] para toda combinación válida (carrera, universidad) en dataset.

9. IF dataset tiene 1,500 combinaciones (producción), scoring debe completarse en ≤ 1 segundo.

---

### Requerimiento 9: Ranking y Selección de Top-N Recomendaciones

**User Story:** Como Scoring_Engine, quiero ordenar todas las combinaciones carrera–universidad por concordancia, aplicar filtros opcionales, y seleccionar las Top-N más relevantes para presentación al usuario.

#### Criterios de Aceptación

1. AFTER calcular concordancia_score para todas las combinaciones, THE **Scoring_Engine** SHALL ordenar resultados en orden descendente por concordancia_score.

2. THE **Scoring_Engine** SHALL aplicar filtros opcionales SI usuario los especificó en su sesión:
   - Filter geographic: retener únicamente combinaciones donde `location` coincide con `geographic_preference` del usuario (o todas si preference es null/"flexible").
   - Filter institution_type: retener únicamente combinaciones donde `institution_type` coincide con `institution_type_preference` (o todas si preference es null/"cualquiera").
   - Filter budget: retener únicamente combinaciones donde `annual_cost <= budget_max` especificado por usuario.

3. WHEN todos los filtros son aplicados, THE orden descendente por concordancia_score es preservado en el subconjunto resultante.

4. THE sistema SHALL retornar Top-3 combinaciones por defecto. IF usuario especifica N diferente en request, THE sistema SHALL retornar Top-N donde N ∈ [1, 10].

5. FOR cada combinación en Top-N, THE respuesta SHALL incluir: `{carrera, institution, location, concordancia_score, scores_por_criterio, datos_verificables}` donde datos_verificables incluyen: `monthly_income`, `annual_cost`, `admission_rate`, `duration_years`.

6. THE orden de resultado final SHALL ser determinístico: mismo input → mismo ranking en 100% de los casos. Esto SHALL ser validado en tests y auditoría.

7. IF Top-N contiene empates de concordancia_score (diferencia < 0.001), THE desempate SHALL ser por `institution` (alfabético ascendente) para determinismo.

---

### Requerimiento 10: Generación de Explicaciones Personalizadas

**User Story:** Como LLM_Layer, quiero redactar justificaciones claras y personalizadas que expliquen por qué cada recomendación es apropiada para el estudiante, incluyendo datos duros verificables.

#### Criterios de Aceptación

1. WHEN **Scoring_Engine** retorna Top-N, THE **LLM_Layer** SHALL generar una explicación para cada combinación en Top-N (típicamente 3).

2. FOR cada explicación, THE LLM SHALL recibir como input: `{ranking_position, carrera, universidad, concordancia_score, scores_por_criterio, datos_verificables, perfil_usuario}`.

3. THE explicación SHALL ser redactada en español peruano, consistir de 3-5 oraciones, e incluir:
   - (a) Mención de qué criterio o característica de la carrera coincide con el perfil del usuario.
   - (b) Al mínimo un dato verificable (ingresos, costo, o tasa de admisión).
   - (c) Conexión entre el criterio y el beneficio para el usuario.

4. Example correcto:  
   "Te recomendamos Estadística en la UNMSM porque tu afinidad con el análisis de datos es muy alta (85/100). Esta carrera ofrece uno de los mejores ingresos (S/. 3,200/mes) y es accesible financieramente (S/. 1,200/año). Aunque es selectiva (18% de admisión), es factible si tienes fortaleza en matemáticas."

5. THE explicación SHALL NO inventar datos no presentes en dataset; todos los números citados deben venir de datos_verificables.

6. THE explicación SHALL citar fuente implícitamente: "[dato de Ponte en Carrera]" o "[según datos oficiales del MINEDU]".

7. THE **LLM_Layer** SHALL usar system prompt v1 definido con ejemplos de explicaciones bien formadas (few-shot).

8. IF LLM falla a generar explicación coherente, THE **Orchestration_Layer** SHALL generar explicación templated:  
   "Recomendamos [carrera] en [universidad] porque alinea bien con tus intereses. Ingresos promedio: [valor]. Costo: [valor]. Tasa de admisión: [valor]."

---

### Requerimiento 11: Almacenamiento de Feedback de Usuario

**User Story:** Como Feedback_Storage, quiero capturar y persistir todas las decisiones de sistema y validaciones de usuario para construir una base de datos histórica de entrenamiento futuro.

#### Criterios de Aceptación

1. WHEN usuario valida una recomendación mediante escala Likert (1–5), THE **Feedback_Storage** SHALL almacenar en base de datos un registro con siguiente estructura JSON (como mínimo):

```json
{
  "session_id": "s_1234567890",
  "timestamp_query": "2026-07-15T14:32:00Z",
  
  "user_input": {
    "raw_query": "string con consulta original",
    "region": "string o null",
    "budget_max": "número o null",
    "institution_type_pref": "string o null"
  },
  
  "profile_interpreted": {
    "interests": ["array de strings"],
    "salary_priority": 0.8,
    "cost_sensitivity": 0.9,
    "admission_tolerance": 0.5,
    "confidence_score": 0.82
  },
  
  "weights_generated": {
    "affinity": 0.40,
    "salary": 0.30,
    "cost": 0.20,
    "admission": 0.10
  },
  
  "ranking_generated": [
    {
      "rank": 1,
      "career": "string",
      "institution": "string",
      "concordancia_score": 0.847,
      "scores_by_criterion": {
        "affinity": 0.85,
        "salary": 0.82,
        "cost": 0.90,
        "admission": 0.60
      }
    }
  ],
  
  "user_validation": {
    "validation_score": 4,
    "selected_career": "string (carrera elegida por usuario)",
    "timestamp_validation": "2026-07-15T14:35:00Z",
    "notes": "string (comentarios opcionales)"
  }
}
```

2. THE registro SHALL ser insertado en base de datos tan pronto como usuario completa la validación, sin retraso > 5 segundos.

3. WHERE usuario no completa validación en sesión (cierra navegador sin dar feedback), THE sistema SHALL almacenar registro incompleto marcado con `validation_status: "incomplete"` para análisis posterior.

4. THE **Feedback_Storage** SHALL implementar aislamiento por `session_id`: cada usuario accede únicamente a sus propios registros históricos.

5. WHEN consultas posteriores son analizadas, THE sistema SHALL permitir agregación estadística (promedio de validation_score por carrera, por región, etc.) SIN exponer datos individuales de usuarios.

6. THE base de datos SHALL ser agnóstica de tecnología: puede ser SQL relacional, document store, data warehouse, etc., mientras preserve structure anterior.

7. FOR demo (Julio 2026), almacenamiento en archivo JSON o SQLite local es aceptable. PARA producción (Fase 2+), base de datos escalable requerida.

8. THE **Feedback_Storage** SHALL generar logs de auditoría de todas las inserciones/actualizaciones con timestamp y usuario responsable.

---

### Requerimiento 12: Módulo RAG para Detalles de Carrera (Opcional)

**User Story:** Como usuario, quiero hacer preguntas sobre detalles específicos de una carrera recomendada (qué cursos lleva, a qué se dedica, perfiles profesionales, etc.) y obtener respuestas fundamentadas en documentos oficiales.

#### Criterios de Aceptación

1. THE **RAG_Module** es OPCIONAL en demo (Fase 0); solamente 3–5 carreras piloto tendrán RAG habilitado inicialmente.

2. WHEN usuario pregunta "¿Qué cursos lleva [carrera]?" O similar, IF carrera está en lista de carreras con RAG, THE **RAG_Module** SHALL:
   - Recuperar embedding de pregunta.
   - Buscar en **Vector_Database** los K=3 chunks más relevantes de documentos de esa carrera.
   - Pasar chunks + pregunta al LLM para generar respuesta coherente y fundamentada.

3. THE documento origen de cada carrera SHALL ser plan curricular oficial obtenido de universidad (PDF). Documento SHALL incluir: lista de cursos, descripción de carrera, perfil profesional, competencias.

4. WHEN documento es cargado inicialmente, THE sistema SHALL:
   - Extraer texto de PDF.
   - Dividir en chunks de ~500 caracteres con 50 caracteres de overlap.
   - Generar embedding de cada chunk usando **Embedding_Service**.
   - Almacenar en **Vector_Database** con metadata: {carrera, universidad, documento_id, chunk_id}.

5. THE búsqueda en **Vector_Database** SHALL retornar chunks ordenados por similitud coseno descendente.

6. THE LLM prompt para RAG SHALL ser: "Basándote en los siguientes fragmentos de documentos oficiales, responde la pregunta del usuario. Cita la fuente."

7. IF user pregunta sobre carrera SIN RAG habilitado, THE sistema SHALL retornar: "Información detallada no disponible aún para [carrera]. Aquí tienes acceso a los datos de Ponte en Carrera: [datos básicos]."

8. THE **RAG_Module** latencia máxima por query: ≤ 2 segundos (incluye retrieval + generación).

9. THE **Embedding_Service** y **Vector_Database** son agnósticos de tecnología: FAISS (local), Pinecone, Chroma, Weaviate, AWS OpenSearch, etc., son opciones válidas.

10. IF durante demo se identifica que RAG agrega valor significativo, SHALL ser priorizado para Fase 1 (post-demo) con cobertura expandida.

---

### Requerimiento 13: Interfaz Web Conversacional

**User Story:** Como estudiante, quiero interactuar con la plataforma a través de un navegador web que sea intuitivo, conversacional y responsivo, sin necesidad de descargar aplicaciones.

#### Criterios de Aceptación

1. THE **Web_Interface** SHALL ser accesible en navegador web moderno (Chrome, Firefox, Safari, Edge con soporte para ES2020 mínimo) en desktop y dispositivos móviles.

2. THE **Web_Interface** NO incluye aplicación nativa móvil. Si usuario accede desde teléfono, utiliza navegador web responsive.

3. THE **Web_Interface** es agnóstico de framework front-end: React, Vue, Angular, o servidor-rendered HTML + JS son opciones válidas, seleccionado durante desarrollo.

4. THE interfaz SHALL incluir:
   - Panel de chat conversacional donde usuario escribe consulta y ve respuestas del bot.
   - Sección de resultados (Top-3 recomendaciones) con visualización de datos: carrera, universidad, scores, datos verificables.
   - Sección de explicaciones personalizadas legibles.
   - Botones de validación (escala Likert 1–5) por cada recomendación.
   - [Opcional] Sección de detalles via RAG con input de pregunta.

5. WHEN usuario escribe en campo de chat y presiona Enter, THE solicitud SHALL ser enviada a `/chat` endpoint (Requerimiento 14) con `session_token` en header `Authorization: Bearer {token}`.

6. THE respuesta del servidor SHALL ser renderizada progresivamente: primero respuesta de chat, luego ranking (si aplica), luego explicaciones.

7. THE interfaz SHALL ser responsive: legible en pantallas ≥ 360px (mobile). Desktop target: ≥ 1024px.

8. THE interfaz SHALL seguir principios de accesibilidad WCAG 2.1 nivel AA (futuro, no crítico para demo): contraste adecuado, navegación por teclado, labels para inputs.

9. WHEN usuario hace click en "Validar recomendación", THE interfaz SHALL mostrar escala Likert 1–5 (con labels: "Poco útil" a "Muy útil") y enviar selección a endpoint `/feedback` (Requerimiento 15).

10. THE interfaz SHALL mantener `session_id` y `session_token` en local storage (si cookie no es viable). Token SHALL ser incluido en cada solicitud API.

11. WHEN `session_token` expira (401 Unauthorized), THE interfaz SHALL mostrar mensaje "Sesión expirada. Por favor, inicia de nuevo." y redirigir a inicio.

---

### Requerimiento 14: Endpoint REST /chat para Conversación

**User Story:** Como Orchestration_Layer, quiero manejar requests de chat del usuario, orquestar flujo LLM-Evaluación-Feedback, y retornar respuestas estructuradas.

#### Criterios de Aceptación

1. THE `/chat` endpoint SHALL aceptar requests POST en formato JSON con estructura:
   ```json
   {
     "message": "string (consulta del usuario)",
     "session_token": "string (criptográfico, opcionalmente en header Authorization)"
   }
   ```

2. WHEN request es recibida, THE **Orchestration_Layer** SHALL:
   - Validar `session_token` con **Auth_Service**; IF inválido, retornar HTTP 401.
   - Recuperar contexto de sesión con ese `session_id` desde **Session_Manager**.
   - Pasar mensaje + contexto previo al **LLM_Layer** para interpretación.
   - Si confidence >= 0.70, invocar **Scoring_Engine** para ranking.
   - Guardar todo en **Feedback_Storage** (sin validación usuario aún).

3. THE response SHALL ser JSON con estructura:
   ```json
   {
     "status": "success" | "error",
     "conversation": {
       "turn": 1,
       "bot_message": "string (respuesta del bot)",
       "requires_follow_up": false | true
     },
     "ranking": {
       "status": "ready" | "awaiting_info",
       "top_3": [
         {
           "rank": 1,
           "career": "string",
           "institution": "string",
           "concordancia_score": 0.847,
           "monthly_income": 3200,
           "annual_cost": 1200,
           "admission_rate": 18,
           "explanation": "string"
         }
       ]
     },
     "rag_available_for": ["Estadística", "..."]
   }
   ```

4. WHERE `bot_message` es respuesta de **LLM_Layer**: pregunta de seguimiento (si requires_follow_up=true) O "Listo, aquí están las recomendaciones" (si requires_follow_up=false).

5. IF error ocurre (LLM falla, scoring falla, etc.), THE response SHALL tener `"status": "error"` con `"error_message": "string descriptivo"` y será logeado para auditoría.

6. THE latencia máxima de `/chat` SHALL ser 5 segundos (demo); 3 segundos (producción).

7. THE endpoint SHALL ser agnóstico de framework: FastAPI, Express, Flask, Django, etc.

8. THE `session_id` es extraído del `session_token`; no es parámetro de request.

---

### Requerimiento 15: Endpoint REST /feedback para Validación de Usuario

**User Story:** Como Orchestration_Layer, quiero capturar la validación (escala Likert) del usuario sobre recomendaciones y guardarla para análisis futuro.

#### Criterios de Aceptación

1. THE `/feedback` endpoint SHALL aceptar requests POST en formato JSON:
   ```json
   {
     "session_token": "string",
     "ranking_id": "string (identificador único del ranking generado)",
     "validation_score": 4,
     "selected_career": "string (carrera elegida, opcional)",
     "notes": "string (comentarios libres, opcional)"
   }
   ```

2. WHEN request es recibida, THE **Orchestration_Layer** SHALL:
   - Validar `session_token`.
   - Recuperar registro previo desde **Feedback_Storage** usando `ranking_id`.
   - Actualizar registro con `user_validation` incluyendo score, timestamp y notas.
   - Guardar en **Feedback_Storage**.

3. THE response SHALL ser JSON:
   ```json
   {
     "status": "success" | "error",
     "message": "string"
   }
   ```

4. IF `session_token` inválido, retornar HTTP 401.

5. IF `ranking_id` no existe en histórico de sesión, retornar HTTP 400 Bad Request.

6. THE `validation_score` SHALL ser entero en rango [1, 5]. IF fuera de rango, retornar HTTP 422 Unprocessable Entity.

7. THE latencia máxima de `/feedback` SHALL ser 2 segundos.

---

### Requerimiento 16: Versionado de Datos y Configuración

**User Story:** Como Data_Pipeline, quiero mantener snapshots versionados de datos, configuración y prompts para garantizar reproducibilidad y auditoria.

#### Criterios de Aceptación

1. WHEN `ingestion.py` completa descarga exitosa, THE sistema SHALL generar snapshot: `snapshots/raw/raw_YYYYMMDD_HHMMSS.xlsx` (copia del archivo Excel descargado).

2. WHEN `feature_engineering.py` completa exitosamente, THE sistema SHALL generar:
   - Snapshot de features: `snapshots/features/features_YYYYMMDD_HHMMSS.csv` (copia de `data/features.csv`).
   - Snapshot de config: `snapshots/configs/feature_config_YYYYMMDD_HHMMSS.json` (copia de `data/feature_config.json`).

3. THE nombre de archivo snapshot DEBE incluir timestamp en formato `YYYYMMDD_HHMMSS` (UTC) para garantizar unicidad y ordenamiento cronológico.

4. THE archivo `feature_config.json` SHALL ser versión actualizable en `data/` (sin timestamp) para uso diario; snapshots ARE para histórico.

5. WHEN sistema genera recomendaciones, THE `ranking_id` incluido en respuesta SHALL ser `{session_id}_{timestamp}_{hash(ranking_top_3)}` para trazabilidad.

6. THE estructura de carpetas SHALL ser:
   ```
   data/
   ├── raw.xlsx                    (última descarga)
   ├── filtered.csv                (últimas features limpias)
   ├── features.csv                (últimas features normalizadas)
   ├── feature_config.json         (última configuración)
   ├── AFINIDAD_METHOD.md           (documentación de método elegido)
   ├── SCORING_STRATEGY.md          (documentación de estrategia elegida)
   └── career_affinity_mapping.json (si método B es elegido)
   
   snapshots/
   ├── raw/
   │   ├── raw_20260614_143022.xlsx
   │   ├── raw_20260615_091545.xlsx
   │   └── ...
   ├── features/
   │   ├── features_20260614_143022.csv
   │   ├── features_20260615_091545.csv
   │   └── ...
   └── configs/
       ├── feature_config_20260614_143022.json
       ├── feature_config_20260615_091545.json
       └── ...
   ```

7. THE sistema SHALL permitir auditor recuperar exactamente los datos y configuración usados para cualquier recomendación anterior simplemente conociendo `timestamp` del ranking.

---

### Requerimiento 17: Logging y Auditoría

**User Story:** Como sistema, quiero registrar todas las decisiones, errores y cambios de estado para auditoría, debugging y cumplimiento.

#### Criterios de Aceptación

1. THE sistema SHALL generar logs estructurados en formato JSON con al mínimo los campos: `timestamp`, `level` (INFO/WARNING/ERROR), `component`, `session_id`, `message`, `data` (objeto con contexto).

2. WHEN request llega a cualquier endpoint, THE sistema SHALL logear: `{level: INFO, message: "Request recibida", data: {method, path, session_id, timestamp}}`.

3. WHEN LLM_Layer genera interpretación, THE sistema SHALL logear: `{level: INFO, message: "Preferencias interpretadas", data: {session_id, interests, confidence_score, weights_generated}}`.

4. WHEN Scoring_Engine calcula ranking, THE sistema SHALL logear: `{level: INFO, message: "Ranking generado", data: {session_id, top_3_careers, scores}}`.

5. WHEN error ocurre (parsing JSON falla, LLM retorna invalido, etc.), THE sistema SHALL logear: `{level: ERROR, message: "Error en [componente]", data: {session_id, error_type, error_detail, timestamp}}` Y intentar recuperación o fallback.

6. WHEN usuario valida recomendación, THE sistema SHALL logear: `{level: INFO, message: "Feedback de usuario", data: {session_id, validation_score, career_selected, timestamp}}`.

7. WHEN sesión expira, THE sistema SHALL logear: `{level: INFO, message: "Sesión expirada", data: {session_id, inactivity_duration, timestamp}}`.

8. FOR seguridad, los logs NO deben incluir tokens, credenciales, o datos PII. IF necesario loguear contexto de usuario, usar solo `session_id` (anónimo).

9. THE logs SHALL ser enviados a stdout O archivo local (agnóstico: puede ser ELK Stack, CloudWatch, Datadog, etc., en producción).

10. THE retention de logs: para demo, 7 días mínimo; para producción, 90 días mínimo.

---

### Requerimiento 18: Manejo de Errores y Fallback

**User Story:** Como sistema, quiero manejar errores gracefully y proporcionar fallback para mantener servicio operativo.

#### Criterios de Aceptación

1. IF **LLM_Layer** falla a generar JSON válido (3 intentos fallidos consecutivos), THE sistema SHALL:
   - Logear error.
   - Retornar mensaje amigable al usuario: "Tengo dificultad para entender tu entrada. Por favor, intenta reformular."
   - Permitir usuario reintentar con otro mensaje.

2. IF **Scoring_Engine** falla a cargar `features.csv`, THE sistema SHALL:
   - Logear error crítico.
   - Retornar respuesta HTTP 503 Service Unavailable con mensaje: "Servicio temporalmente no disponible. Intenta más tarde."
   - NO intentar proceder con scoring vacío.

3. IF pesos generados por LLM no suman 1.0 (fuera de tolerancia ±0.01), THE **Orchestration_Layer** SHALL:
   - Normalizar: `w'ᵢ = wᵢ / Σwᵢ`.
   - Logear warning con pesos originales y normalizados.
   - Continuar con pesos normalizados.

4. IF **Embedding_Service** no disponible (para RAG), THE **RAG_Module** SHALL:
   - Logear error.
   - Retornar mensaje al usuario: "Detalles de carrera no disponibles en este momento."
   - NO bloquear ranking principal (RAG es opcional).

5. IF **Feedback_Storage** falla inserción, THE sistema SHALL:
   - Logear error.
   - Intentar reintento exponencial (1s, 2s, 4s) máximo 3 veces.
   - IF persiste, almacenar en cola local (en-memory o disco) para sincronización posterior.

6. IF `session_token` expira durante `/chat` request, THE sistema SHALL:
   - Retornar HTTP 401 Unauthorized.
   - Cliente SHALL redirigir a inicio de sesión.

7. IF usuario inactivo > 8 horas, THE **Auth_Service** SHALL:
   - Revocar token automáticamente.
   - Loguear cierre de sesión.
   - En siguiente request, retornar 401.

8. FOR la fórmula de scoring, IF falta una variable (ej: `affinity_score` NULL), THE **Scoring_Engine** SHALL:
   - Marcar esa combinación carrera-universidad como "no evaluable".
   - Excluirla de ranking.
   - Loguear warning.

---

### Requerimiento 19: Aislamiento de Datos entre Usuarios

**User Story:** Como sistema, quiero garantizar que datos (consultas, perfiles, rankings, feedback) de un usuario nunca sean accesibles por otro usuario.

#### Criterios de Aceptación

1. THE **Auth_Service** SHALL generar único `session_id` por usuario. Session IDs de diferentes usuarios NO comparten prefijo ni son predecibles.

2. EVERY solicitud a `/chat`, `/feedback`, `/rag`, etc. require `session_token` validado contra un único `session_id`. THE endpoint SHALL verificar que token pertenece a usuario esperado.

3. WHEN **Session_Manager** recupera contexto, SHALL usar `session_id` como clave primaria. IF `session_id` en request no coincide con `session_id` en token, THE solicitud SHALL ser rechazada HTTP 403 Forbidden.

4. WHEN **Feedback_Storage** guarda registro, SHALL incluir `session_id` como campo obligatorio. Queries posteriores a Feedback_Storage DEBEN filtrar por `session_id`.

5. IF implementación usa base de datos multi-tenant, THE schema SHALL incluir `session_id` en primary key o única clave de acceso para aislar datos por usuario.

6. THE **Web_Interface** SHALL limpiar todos los datos de sesión (contexto, rankings, feedback) de memoria y local storage WHEN usuario cierra sesión explícitamente O token expira.

7. IF auditor revisa logs para debugging, logs contendrán `session_id` (anónimo) pero NUNCA expondrán datos de otro usuario aunque tenga acceso a la misma máquina.

---

### Requerimiento 20: Estructura de Salida y Artefactos

**User Story:** Como proyecto, quiero mantener estructura clara y versionada de todos los artefactos generados para facilitar entrega, reproducibilidad y auditoría.

#### Criterios de Aceptación

1. THE repositorio raíz SHALL contener:
   ```
   careermatch-peru/
   ├── README.md                       (instrucciones generales)
   ├── requirements.txt                (dependencias Python)
   ├── Dockerfile                      (containerización)
   ├── docker-compose.yml              (orquestación local)
   ├── .gitignore                      (archivos ignorados)
   ├── LICENSE                         (licencia del proyecto)
   │
   ├── data_pipeline/
   │   ├── __init__.py
   │   ├── ingestion.py                (descarga automatizada)
   │   ├── data_clean.py               (limpieza y estandarización)
   │   ├── feature_engineering.py      (imputación + normalización)
   │   └── config.py                   (configuración compartida)
   │
   ├── data/
   │   ├── raw.xlsx                    (última descarga)
   │   ├── filtered.csv                (últimas features limpias)
   │   ├── features.csv                (últimas features normalizadas)
   │   ├── feature_config.json         (última configuración de imputación)
   │   ├── AFINIDAD_METHOD.md           (documentación)
   │   ├── SCORING_STRATEGY.md          (documentación)
   │   └── career_affinity_mapping.json (si aplica)
   │
   ├── snapshots/
   │   ├── raw/
   │   │   └── raw_YYYYMMDD_HHMMSS.xlsx (histórico)
   │   ├── features/
   │   │   └── features_YYYYMMDD_HHMMSS.csv (histórico)
   │   └── configs/
   │       └── feature_config_YYYYMMDD_HHMMSS.json (histórico)
   │
   ├── backend/
   │   ├── app.py                      (punto de entrada FastAPI/similar)
   │   ├── auth.py                     (Auth_Service)
   │   ├── session.py                  (Session_Manager)
   │   ├── llm_service.py              (LLM_Layer)
   │   ├── scoring.py                  (Scoring_Engine)
   │   ├── rag.py                      (RAG_Module)
   │   ├── feedback.py                 (Feedback_Storage)
   │   ├── orchestration.py            (Orchestration_Layer)
   │   ├── models.py                   (esquemas Pydantic/similar)
   │   └── utils.py                    (utilidades)
   │
   ├── frontend/
   │   ├── index.html
   │   ├── styles.css
   │   ├── app.js                      (lógica conversacional)
   │   ├── components/
   │   │   ├── chat.js
   │   │   ├── ranking.js
   │   │   └── feedback.js
   │   └── utils/
   │       ├── api.js
   │       └── auth.js
   │
   ├── notebooks/
   │   ├── 00_exploratory_data_analysis.ipynb
   │   ├── 01_pipeline_validation.ipynb
   │   ├── 02_affinity_calculation.ipynb
   │   └── 03_demo_integration.ipynb    (integración LLM-Scoring end-to-end)
   │
   ├── tests/
   │   ├── test_pipeline.py
   │   ├── test_scoring.py
   │   ├── test_llm.py
   │   └── test_auth.py
   │
   └── docs/
       ├── ARCHITECTURE.md              (diagrama y descripción técnica)
       ├── API.md                       (especificación de endpoints)
       ├── DEVELOPMENT.md               (guía de desarrollo)
       └── DEPLOYMENT.md                (guía de despliegue)
   ```

2. THE `README.md` SHALL incluir:
   - Descripción breve del proyecto.
   - Requisitos: Python 3.10+, Chrome instalado (para Selenium).
   - Instrucciones de instalación: `pip install -r requirements.txt`.
   - Instrucciones de ejecución: cómo correr pipeline, cómo iniciar servidor backend, cómo acceder a frontend.
   - Contacto y licencia.

3. THE `requirements.txt` SHALL listar todas las dependencias Python con versiones mínimas especificadas.

4. THE `Dockerfile` SHALL:
   - Usar imagen base `python:3.10-slim`.
   - Instalar dependencias desde `requirements.txt`.
   - Exponer puerto 8000 (backend).
   - Comando por defecto: ejecutar servidor backend.

5. THE `docker-compose.yml` SHALL orquestar:
   - Servicio backend (puerto 8000).
   - Servicio frontend (puerto 3000, si es web framework).
   - Volúmenes para persistencia de datos y snapshots.
   - Variables de entorno (MINEDU_URL, etc.).

6. THE `.gitignore` SHALL excluir como mínimo:
   - `*.pyc`, `__pycache__/`
   - `*.xlsx`, `*.csv` (datos grandes; opcional si se versionan)
   - `.env`, credenciales
   - `node_modules/`
   - `venv/`, `.venv/`

7. EVERY archivo Python SHALL incluir docstring en módulo que describe propósito y responsabilidades principales.

---

### Requerimiento 21: Configuración y Variables de Entorno

**User Story:** Como desarrollador, quiero configurar comportamiento del sistema (URLs, timeouts, modelos LLM, etc.) mediante variables de entorno sin modificar código.

#### Criterios de Aceptación

1. THE sistema SHALL soportar configuración mediante archivo `.env` O variables de entorno del sistema.

2. THE variables requeridas (mnemonicamente):
   - `MINEDU_URL`: URL de Ponte en Carrera (default: `https://ponteencarrera.minedu.gob.pe/pec-portal-web/`)
   - `LLM_PROVIDER`: `gemini` | `openai` | `local` (default: `gemini`)
   - `LLM_MODEL`: ej `gemini-1.5-flash`, `gpt-4o-mini`, etc.
   - `LLM_API_KEY`: clave de autenticación (si aplica)
   - `EMBEDDING_PROVIDER`: `openai` | `google` | `huggingface` | `local` (default: `huggingface`)
   - `EMBEDDING_MODEL`: ej `sentence-transformers/multilingual-MiniLM-L12-v2`
   - `VECTOR_DB_TYPE`: `faiss` | `pinecone` | `chroma` | `weaviate` (default: `faiss`)
   - `DATABASE_URL`: URI de base de datos SQL/NoSQL (default: `sqlite:///feedback.db`)
   - `SESSION_TIMEOUT_MINUTES`: duración sesión (default: `480`, 8 horas)
   - `LOG_LEVEL`: `DEBUG` | `INFO` | `WARNING` | `ERROR` (default: `INFO`)

3. IF variable requerida no está definida, THE sistema SHALL usar valor por defecto especificado O fallar con mensaje claro indicando variable faltante.

4. THE `.env` ejemplo SHALL ser incluido como `.env.example` en repositorio.

---

### Requerimiento 22: Tests Unitarios

**User Story:** Como desarrollador, quiero ejecutar tests para validar que componentes funcionan correctamente en aislamiento.

#### Criterios de Aceptación

1. THE proyecto SHALL incluir tests unitarios para al mínimo:
   - `test_auth.py`: validación de tokens, creación de sesiones, expiración.
   - `test_pipeline.py`: limpieza de datos, imputación, normalización.
   - `test_scoring.py`: cálculo de scores, ranking, ordenamiento.
   - `test_llm.py`: parsing de respuestas JSON, validación de pesos.

2. THE suite de tests SHALL ser ejecutable con comando: `pytest tests/`

3. THE cobertura de código SHALL ser >= 60% para componentes críticos (Auth, Scoring, Pipeline).

4. EVERY test SHALL ser independiente: no depender de orden de ejecución ni estado compartido.

5. IF test falla, el mensaje de error SHALL ser descriptivo indicando qué esperaba y qué obtuvo.

---

### Requerimiento 23: Documentación Técnica

**User Story:** Como desarrollador futuro, quiero entender la arquitectura, flujos y decisiones técnicas sin necesidad de leer código.

#### Criterios de Aceptación

1. THE `docs/ARCHITECTURE.md` SHALL incluir:
   - Diagrama conceptual de componentes (text-based o referencia a imagen).
   - Descripción de cada componente y responsabilidades.
   - Flujo de datos de entrada a salida.
   - Decisiones de arquitectura y trade-offs.

2. THE `docs/API.md` SHALL especificar:
   - Todos los endpoints: método HTTP, path, parámetros, respuesta esperada.
   - Códigos de error y mensajes.
   - Ejemplos de curl o Postman.

3. THE `docs/DEVELOPMENT.md` SHALL incluir:
   - Cómo configurar entorno de desarrollo.
   - Cómo ejecutar cada componente localmente.
   - Cómo ejecutar tests.
   - Convenciones de código (naming, imports, etc.).

4. THE `docs/DEPLOYMENT.md` SHALL incluir:
   - Cómo desplegar con Docker.
   - Cómo configurar variables de entorno para producción.
   - Checklist pre-deployment.

---

### Requerimiento 24: Reproducibilidad de Resultados (Demo)

**User Story:** Como evaluador, quiero verificar que el sistema es determinista y que puedo reproducir exactamente una recomendación anterior.

#### Criterios de Aceptación

1. GIVEN mismo usuario input, mismo `session_id`, mismo dataset features.csv, THE sistema SHALL generar exactamente mismo ranking (Top-3 en mismo orden) en 100% de las ejecuciones.

2. GIVEN mismo `timestamp` de ranking generado, evaluador SHALL poder recuperar snapshots de datos y configuración usados en ese momento y reproducir cálculo manualmente.

3. IF estrategia de scoring es suma ponderada (`w1*a + w2*b + w3*c + w4*d`), THE evaluador debe poder verificar cada término de ecuación manualmente con datos de snapshots.

4. THE sistema SHALL permitir "replay" de cualquier consulta histórica: dado `ranking_id`, recuperar session snapshot, inputs, pesos, y reproducir exactamente los scores calculados.

5. THE **Feedback_Storage** registro incluirá field `reproducibility_metadata`: {`dataset_snapshot_id`, `config_snapshot_id`, `prompt_version`, `llm_model_used`, `timestamp`} para facilitar auditoría posterior.

---

### Requerimiento 25: Métricas de Validación (Demo)

**User Story:** Como equipo de evaluación, quiero medir si el motor de scoring funciona adecuadamente con casos de prueba.

#### Criterios de Aceptación

1. THE sistema SHALL ejecutar suite de 10–15 casos de prueba predefinidos (perfil estudiante + perfiles carrera esperados en Top-3).

2. FOR cada caso, THE sistema SHALL registrar:
   - Input (consulta usuario).
   - Pesos generados por LLM.
   - Top-3 ranking generado.
   - Validación esperada vs. validación real (escala Likert promedio).

3. THE métrica primaria es `Satisfacción promedio (Likert)`: promedio de validaciones ≥ 3.5 / 5.0 para Top-3.

4. THE métrica secundaria es `Consistencia de ranking`: mismo input en diferentes runs genera mismo Top-3.

5. THE métrica terciaria es `Coherencia de explicaciones`: evaluador humano verifica que explicaciones son coherentes y basadas en datos (evaluación cualitativa).

6. WHEN demo es evaluada, THE documento de resultados SHALL incluir tabla:
   | Caso | Input | Validación Promedio | Ranking Consistente | Explicación Coherente |
   |------|-------|---------------------|---------------------|-----------------------|
   | 1    | ...   | 4.0                 | ✓                   | ✓                     |
   | ...  | ...   | ...                 | ...                 | ...                   |

7. IF validación promedio < 3.5, THE sistema debe investigar si fallo es en: (a) método de afinidad, (b) pesos dinámicos del LLM, (c) estrategia de scoring, O (d) formulación de criterios.

---

### Requerimiento 26: Seguridad Básica (Demo)

**User Story:** Como sistema, quiero implementar medidas básicas de seguridad para proteger datos de usuario y sesiones.

#### Criterios de Aceptación

1. THE `session_token` SHALL ser criptográficamente seguro (mínimo 32 bytes de entropía, generado con `secrets` módulo Python O equivalente).

2. THE `session_token` SHALL ser opaco: no contendrá información decibrable (no base64 de username, no timestamp en claro).

3. IF almacenamiento de tokens es necesario (ej: sesiones persistentes), SHALL ser hasheado con bcrypt o argon2 antes de persista.

4. THE API endpoints críticos (`/chat`, `/feedback`, `/rag`) SHALL requerir `session_token` en header `Authorization: Bearer {token}`. IF ausente, retornar HTTP 401.

5. THE **Web_Interface** SHALL almacenar `session_token` en cookie HTTP-only (no accesible por JavaScript) O local storage con flag seguro. Para demo, local storage es aceptable.

6. IF datos en tránsito son sensibles (cosa que no son en demo, pues son datos públicos de Ponte en Carrera + preferencias), conexión DEBE ser TLS 1.2+. Para demo, HTTP local es aceptable.

7. THE sistema SHALL NO loguear tokens, credenciales API, o datos PII.

8. WHEN usuario cierra sesión, THE token SHALL ser revocado inmediatamente en servidor y cliente.

---

### Requerimiento 27: Integración End-to-End (Demo)

**User Story:** Como equipo, quiero un notebook que demuestre el flujo completo: usuario input → LLM interpretación → Scoring → Ranking → Explicación → Feedback storage.

#### Criterios de Aceptación

1. THE notebook `notebooks/03_demo_integration.ipynb` SHALL ejecutarse sin errores de principio a fin.

2. THE notebook SHALL incluir:
   - Setup: cargar features.csv, inicializar LLM, configurar Scoring_Engine.
   - Caso 1: usuario perfil 1 (énfasis en salario).
   - Caso 2: usuario perfil 2 (énfasis en costo).
   - Caso 3: usuario perfil 3 (énfasis en afinidad).
   - Cada caso incluirá: input, pesos generados, ranking, explicaciones, feedback simulado.

3. THE notebook DEBE ser ejecutable con: `jupyter notebook notebooks/03_demo_integration.ipynb` O `python -m nbconvert --to notebook --execute notebooks/03_demo_integration.ipynb`.

4. THE notebook output DEBE mostrar claramente:
   - Input del usuario.
   - Perfil interpretado (interests, pesos).
   - Top-3 ranking con scores desglosados.
   - Explicaciones personalizadas.
   - Simulated feedback Likert.

5. THE notebook SHALL ejecutarse en < 30 segundos (sin llamadas extensas a APIs externas; usar mocks si es necesario).

---

### Requerimiento 28: Compatibilidad del Navegador (Demo)

**User Story:** Como usuario, quiero acceder a la plataforma desde navegadores modernos sin problemas de compatibilidad.

#### Criterios de Aceptación

1. THE **Web_Interface** SHALL ser compatible con:
   - Chrome 90+
   - Firefox 88+
   - Safari 14+
   - Edge 90+

2. THE interfaz DEBE soportar ECMAScript 2020 (ES11) mínimo.

3. WHEN usuario accede desde navegador no soportado, THE sistema SHALL mostrar mensaje amigable: "Tu navegador podría no ser completamente compatible. Te recomendamos actualizar."

4. THE interfaz DEBE ser responsivo en:
   - Desktop: 1024px+ ancho.
   - Tablet: 600px–1024px ancho.
   - Mobile: 360px–600px ancho.

5. WHEN usuario accede desde mobile, THE interfaz DEBE ser completamente funcional (chat, ranking, feedback) sin necesidad de aplicación nativa.

---

### Requerimiento 29: Métricas No Funcionales – Rendimiento

**User Story:** Como sistema, quiero operar dentro de límites de rendimiento especificados para demo y producción futura.

#### Criterios de Aceptación

1. **Para Demo (Julio 2026):**
   - Latencia `/chat` endpoint: ≤ 5 segundos (p95).
   - Latencia `/feedback` endpoint: ≤ 2 segundos.
   - Throughput: 1–2 usuarios simultáneos (no crítico).
   - Pipeline de datos: ≤ 1 hora ejecutar completo (una ejecución).

2. **Para Producción Futura (Fase 2):**
   - Latencia `/chat` endpoint: ≤ 3 segundos (p99).
   - Latencia `/feedback` endpoint: ≤ 2 segundos.
   - Throughput: ≥ 100 usuarios simultáneos.
   - Pipeline de datos: ≤ 30 minutos diarios.

3. THE scoring para 1,500 combinaciones carrera–universidad DEBE completarse en ≤ 1 segundo.

4. THE embedding + retrieval RAG DEBE completarse en ≤ 2 segundos por query.

5. THE latencias medidas son desde recepción de request hasta envío de respuesta completa (incluyendo I/O de red, LLM, scoring, etc.).

---

### Requerimiento 30: Métricas No Funcionales – Disponibilidad y Confiabilidad

**User Story:** Como usuario, quiero que la plataforma sea disponible y confiable durante mi sesión.

#### Criterios de Aceptación

1. **Para Demo:**
   - Disponibilidad: Best effort (no SLA comprometido).
   - Recuperación de errores: graceful degradation cuando componentes fallan (ej: RAG no disponible no bloquea ranking).
   - Timeout máximo para cualquier request: 10 segundos (retornar error si no completa).

2. **Para Producción Futura:**
   - Disponibilidad: 99%+ durante horarios pico (18:00–21:00 hrs).
   - RTO (Recovery Time Objective): < 5 minutos.
   - RPO (Recovery Point Objective): < 1 minuto (sin pérdida de feedback).

3. THE sistema SHALL implementar retry automático para llamadas a LLM externas y servicios de embedding (máximo 3 intentos con backoff exponencial).

4. IF servicio externo (Gemini API, embedding service) no disponible, THE sistema SHALL retornar respuesta útil basada en data local (ej: recomendación sin explicación generada por LLM).

---

### Requerimiento 31: Escalabilidad (Roadmap Futuro)

**User Story:** Como producto, quiero arquitectura preparada para escalar a 10,000+ usuarios sin rediseño fundamental.

#### Criterios de Aceptación

1. THE componentes principales (Auth, LLM, Scoring, Feedback) DEBEN estar desacoplados y ejecutables independientemente para permitir escalado horizontal.

2. THE **Feedback_Storage** debe soportar model de base de datos agnóstico: desde SQLite (demo) a PostgreSQL/MongoDB/BigQuery (producción) sin cambios en interfaz.

3. THE caché de features (features.csv) DEBE soportar recargas periódicas sin reiniciar servidor.

4. THE sesiones de usuario DEBEN ser stateless (todo estado en DB/cache, no en memoria de servidor) para permitir múltiples instancias backend.

5. THE LLM calls DEBEN tener rate limiting configurable para evitar hitting API limits de proveedores.

---

---

## Fuera de Alcance (v1)

Los siguientes elementos están explícitamente excluidos de la versión 1 (Demo, Julio 2026) y no deben ser implementados:

- Aplicación nativa móvil (iOS / Android). La plataforma es web-only, responsive a navegador móvil, pero sin app nativa.
- Ajuste fino (fine-tuning) del LLM. Se utilizan modelos generativos pre-entrenados sin modificación.
- Cobertura de economía informal. Los datos de ingreso son solo para empleo formal (limitación de Ponte en Carrera).
- Monitoreo en producción (MLflow, Prometheus, Datadog en vivo). Para demo, logging básico es suficiente.
- Integración con universidades o sistemas educativos externos. Demo es standalone.
- Portal para orientadores educativos (feature B2B). Enfocado solo en usuario estudiante/evaluador.
- Análisis de soft skills o competencias adicionales. Solo carreras + universidades + criterios existentes.
- Recomendación de especialidades dentro de carreras. Recomendación es a nivel carrera, no especialización.
- Comparativa lado a lado avanzada (más de 3 opciones). Top-3 es límite para demo.
- Sistema de chat multi-idioma. Solo español peruano para demo.
- Integración con redes sociales o login social. Solo session token básico.
- Análisis predictivo de empleabilidad a 5+ años. Solo datos históricos de Ponte en Carrera.
- Modelo propio entrenado. Se utiliza LLM generativo; modelo propio es Fase 2+ con feedback histórico suficiente.

---

## Fases de Desarrollo

| Fase | Alcance | Prioridad |
|---|---|---|
| **Fase 0: Demo (Jun 17 – Aug 9, 2026)** | Reqs 1–31: Sistema completo funcional con 3–5 usuarios, casos de evaluación, feedback capturado, RAG opcional (3–5 carreras piloto) | P0 |
| **Fase 1: Beta / Estabilización (Sep 2026 – Feb 2027)** | Aumento a 100–500 usuarios, análisis de feedback, iteración de prompts, documentación de learnings, decisión sobre evolución motor scoring | P1 |
| **Fase 2: Escala Limitada (Mar – Dic 2027)** | 10,000 usuarios, infraestructura escalable, modelo propio entrenado (si datos suficientes), integración con instituciones educativas, análisis de impacto real | P2 |
| **Fase 3: Producción Abierta (2028+)** | 50,000+ usuarios, modelo propio operativo, SLA 99%+, partnerships con universidades, portal orientadores | P3 |

---

## Dependencias Técnicas

| Componente | Versión Mínima | Propósito |
|---|---|---|
| **Python** | 3.10+ | Runtime para data pipeline, backend, scripting |
| **Selenium** | 4.15.0+ | Automatización de descarga de Ponte en Carrera (ingestion) |
| **Pandas** | 2.0.0+ | Manipulación y transformación de datos (pipeline, features) |
| **NumPy** | 1.24.0+ | Operaciones vectorizadas (normalization, scoring) |
| **OpenPyXL** | 3.10.0+ | Lectura/escritura de archivos Excel (.xlsx) |
| **FastAPI** | 0.104.0+ | Framework web para API REST (orquestación, endpoints) |
| **Uvicorn** | 0.24.0+ | ASGI server para servir FastAPI |
| **Pydantic** | 2.0.0+ | Validación de esquemas y modelos (request/response) |
| **google-generativeai** | 0.3.0+ | SDK para Gemini API (LLM principal) |
| **openai** | 1.0.0+ | SDK para OpenAI API (LLM alternativo, opcional) |
| **python-dotenv** | 1.0.0+ | Gestión de variables de entorno (.env) |
| **pytest** | 7.0.0+ | Framework de testing (unit tests) |
| **sentence-transformers** | 2.2.0+ | Modelos de embeddings (para afinidad/RAG, si local) |
| **faiss-cpu** | 1.7.0+ | Vector database local (RAG, alternativa) |
| **langchain** | 0.1.0+ | Framework que abstrae LLM + RAG (opcional, puede no usarse) |
| **Flask** o **Django** | 2.3.0+ | Alternativa a FastAPI (agnóstico) |
| **React** | 18.0.0+ | Framework frontend (agnóstico) |
| **Node.js** | 18.0.0+ | Runtime para frontend (agnóstico) |
| **Docker** | 20.10.0+ | Containerización |
| **Docker Compose** | 2.0.0+ | Orquestación local |
| **Git** | 2.30.0+ | Control de versiones |
| **Chrome** | 90.0.0+ | Para Selenium (descarga Ponte en Carrera) |
| **SQLite3** | 3.35.0+ | Base de datos local feedback (demo) |
| **PostgreSQL** | 13.0+ | Base de datos escalable (producción futura) |
| **Redis** | 7.0.0+ | Cache de sesiones / rate limiting (producción futura) |
| **Pinecone** API | — | Vector database cloud (RAG alternativa, sin versión) |
| **Chroma** | 0.4.0+ | Vector database local (RAG alternativa) |

---

**Documento de Requerimientos generado**: `requirements.md` v1.0  
**Fecha**: Junio 2026  
**Estado**: Listo para Desarrollo  
**Trazabilidad**: Generado desde PRD CareerMatch Perú v1.1
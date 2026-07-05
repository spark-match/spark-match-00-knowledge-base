# CareerMatch Perú — Documento de Requisitos de Producto (PRD)

**Versión:** 1.0  
**Fecha:** Junio 2026  
**Estado:** Documento Base  
**Audiencia:** Equipo de desarrollo, stakeholders, evaluadores

---

## Tabla de Contenidos

1. [Resumen Ejecutivo](#resumen-ejecutivo)
2. [Contexto del Negocio](#contexto-del-negocio)
3. [Definición del Problema](#definición-del-problema)
4. [Objetivos del Proyecto](#objetivos-del-proyecto)
5. [Descripción de la Solución](#descripción-de-la-solución)
6. [Requisitos Funcionales](#requisitos-funcionales)
7. [Requisitos No Funcionales](#requisitos-no-funcionales)
8. [Arquitectura Conceptual](#arquitectura-conceptual)
9. [Experiencia del Usuario](#experiencia-del-usuario)
10. [Datos y Modelos](#datos-y-modelos)
11. [Métricas de Éxito](#métricas-de-éxito)
12. [Roadmap del Producto](#roadmap-del-producto)
13. [Restricciones y Supuestos](#restricciones-y-supuestos)
14. [Referencias](#referencias)

---

## 1. Resumen Ejecutivo

**CareerMatch Perú** es una plataforma de orientación vocacional impulsada por inteligencia artificial generativa que ayuda a estudiantes de educación secundaria y sus familias en Perú a tomar decisiones informadas sobre la elección de carrera universitaria.

La plataforma utiliza un **asistente conversacional basado en LLM** que interpreta preferencias expresadas en lenguaje natural y genera recomendaciones personalizadas de combinaciones carrera–universidad, sustentadas en datos oficiales del Ministerio de Educación y un **motor de scoring transparente y auditables basado en múltiples criterios**.

**Propuesta de Valor:**
- Reduce la brecha de información en orientación vocacional en contextos de recursos limitados
- Combina datos oficiales con inteligencia conversacional para recomendaciones contextualizadas
- Mantiene transparencia y auditabilidad mediante desacoplamiento de componentes LLM y scoring
- Aplica prácticas de MLOps para reproducibilidad y rastreabilidad completa

---

## 2. Contexto del Negocio

### 2.1 Industria y Mercado

**Sector:** Tecnología educativa (EdTech) aplicada a orientación vocacional y educación superior  
**Geografía Primaria:** Perú  
**Audiencia Primaria:** Estudiantes de 4º y 5º año de secundaria (16–18 años)  
**Audiencia Secundaria:** Padres de familia, orientadores educativos, instituciones educativas

### 2.2 Situación Actual del Mercado

En el contexto peruano, la elección de carrera es **una de las decisiones más críticas para los jóvenes**, con implicaciones económicas y profesionales de largo plazo. Sin embargo, enfrenta desafíos significativos:

**a) Fragmentación de información:**
- Datos sobre carreras, universidades, costos e ingresos esperados están dispersos en múltiples fuentes
- No existe un sistema centralizado que integre información oficial verificable
- Estudiantes recurren a recomendaciones informales o percepciones sesgadas

**b) Falta de orientación especializada:**
- Muchas instituciones educativas (especialmente en zonas rurales) no cuentan con orientadores vocacionales capacitados
- La relación orientador-estudiante es limitada en la mayoría de contextos

**c) Desconocimiento de mercado laboral:**
- Los estudiantes desconocen la empleabilidad real de carreras
- Existe desinformación sobre ingresos esperados y demanda laboral
- Las decisiones frecuentemente no alinean intereses con viabilidad económica

### 2.3 Oportunidad de Negocio

**Mercado Objetivo:**
- ~200,000 estudiantes/año que egresan de secundaria en Perú
- Instituciones educativas (colegios, academias preuniversitarias)
- Plataformas digitales de educación superior

**Estimación de Tamaño:**
- Si el 10% de estudiantes accede a herramientas de orientación vocacional: 20,000 usuarios/año
- Modelo B2B2C: escuelas + plataformas de RRSS + alianzas con universidades

**Modelos de Ingresos Potenciales (fuera del alcance actual):**
- Freemium: acceso básico gratuito, reportes detallados de pago
- B2B: licencia para instituciones educativas
- Partnerships: ingresos por referrales a universidades

---

## 3. Definición del Problema

### 3.1 Problema Principal

**Estudiantes y familias en Perú carecen de un sistema accesible, integrado y personalizado para tomar decisiones informadas sobre la elección de carrera basado en datos verificables y sus preferencias individuales.**

### 3.2 Cuantificación del Problema

**Datos de respaldo:**

| Métrica | Valor | Fuente |
|---|---|---|
| Estudiantes/año que egresan de secundaria en Perú | ~200,000 | MINEDU 2024 |
| % sin acceso a orientación vocacional especializada | 60–70% | Estimación basada en cobertura de orientadores |
| % que reporta dificultad en elección de carrera | 75% | Estudio cualitativo docentes peruanos |
| % que cambia de carrera en primer año | 15–20% | Datos SUNEDU instituciones piloto |
| Costo medio de año de estudios perdido (USD) | 2,000–3,000 | Estimación costo promedio privada + costo de oportunidad |
| **Costo total anual por malas decisiones vocacionales** | **$60–120 millones** | 20,000 × 3,000 USD promedio |

### 3.3 Problema Secundario: Asimetría de Información

Aunque el portal **Ponte en Carrera (MINEDU)** publica datos oficiales sobre:
- Ingresos promedio de egresados por carrera y universidad
- Costos de matrícula y arancel
- Tasas de admisión
- Duración de programas
- Ubicación geográfica

**Estos datos NO se transforman fácilmente en recomendaciones personalizadas** porque:
1. Requieren análisis manual de múltiples variables
2. No se adaptan a preferencias y restricciones individuales
3. No existen explicaciones contextuales que justifiquen recomendaciones

### 3.4 Dolor del Usuario

| Rol | Problema | Impacto |
|---|---|---|
| **Estudiante** | No sabe qué carrera es compatible con sus intereses y capacidades | Inseguridad, decisiones basadas en influencias externas |
| **Estudiante** | Desconoce ingresos realistas y costos de diferentes opciones | Presión financiera familiar, deserción |
| **Padre/Madre** | Falta información para aconsejar o apoyar la decisión | Ansiedad, asesoramiento inadecuado |
| **Orientador educativo** | Capacidad limitada para atender a todos los estudiantes | Recomendaciones genéricas, insatisfacción |

---

## 4. Objetivos del Proyecto

### 4.1 Objetivo de Negocio

**Desarrollar una solución de orientación vocacional basada en IA que:**
- Mejore la tasa de alineación entre preferencias de estudiantes y decisiones vocacionales
- Reduzca la tasa de cambio de carrera en primer año
- Sea accesible, transparente y verificable con datos oficiales
- Escale de manera sostenible en el contexto educativo peruano

**KPIs de Negocio (largo plazo):**
- Aumento de 30% en satisfacción con la decisión de carrera (1 año post-ingreso)
- Reducción de 25% en tasa de cambio de carrera (año 1)
- 50,000 usuarios activos en 2 años
- NPS (Net Promoter Score) ≥ 50 en población objetivo

### 4.2 Objetivos Técnicos

**O1:** Implementar un pipeline de datos reproducible que integre datos oficiales de Ponte en Carrera con transformaciones normalizadas para scoring

**O2:** Desarrollar un motor de recomendación multi-criterio que genere rankings personalizados basado en:
- Afinidad con intereses del estudiante
- Ingresos esperados de egresados
- Costo de estudios
- Facilidad de ingreso (tasa de admisión)
- Factibilidad geográfica

**O3:** Implementar un asistente conversacional que interprete preferencias en lenguaje natural y traduzca dinámicamente los pesos de criterios de scoring

**O4:** Garantizar reproducibilidad y auditabilidad mediante versionado completo de datos, prompts y configuración

**O5:** Asegurar claridad en las recomendaciones mediante explicaciones fundamentadas en datos verificables

### 4.3 Diferencia entre Objetivos de Negocio y Técnicos

| Aspecto | Objetivo Negocio | Objetivo Técnico |
|---|---|---|
| **Enfoque** | Valor para el usuario y viabilidad comercial | Implementación de funcionalidades |
| **Métrica** | Satisfacción, cambio de carrera, usuarios activos | Reproducibilidad, latencia, exactitud |
| **Horizonte** | Largo plazo (1–3 años) | Corto plazo (sprint iterativo) |
| **Ejemplo** | "Reducir deserción vocacional 25%" | "Motor scoring con latencia < 2s" |

---

## 5. Descripción de la Solución

### 5.1 Propuesta de Valor Diferencial

A diferencia de soluciones existentes, CareerMatch Perú **integra tres componentes que no se encuentran juntos en el mercado local:**

| Componente | CareerMatch | Alternativas Existentes |
|---|---|---|
| **Datos Oficiales Integrados** | Ponte en Carrera (MINEDU) actualizado automáticamente | Portales estáticos o datos no verificables |
| **Conversación Personalizada** | LLM que interpreta preferencias naturales | Tests psicométricos genéricos o manuales |
| **Motor Transparente** | Scoring con pesos dinámicos basado en preferencias | Caja negra o recomendaciones sin explicación |
| **Reproducibilidad** | Versionado completo de datos/prompts/configuración | Sin trazabilidad de decisiones |

### 5.2 Flujo de Interacción Principal

```
┌────────────────────────────────────────────────────────────────┐
│                    USUARIO ESTUDIANTE                         │
└────────────────────────────────────────────────────────────────┘
                              ↓
                   Expresar preferencias en lenguaje natural:
              "Quiero carrera con buen sueldo, pero tengo 
               presupuesto limitado. Me gustan matemáticas"
                              ↓
┌────────────────────────────────────────────────────────────────┐
│          CAPA DE INTERFAZ CONVERSACIONAL (Frontend)           │
│  - Recibe entrada del usuario                                 │
│  - Aplica filtros iniciales (región, presupuesto, tipo inst.) │
│  - Mantiene contexto de sesión                                │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│    CAPA LLM: Interpretación de Preferencias (Backend)         │
│  - Recibe consulta y contexto de sesión                       │
│  - LLM interpreta: intereses, restricciones financieras       │
│  - Genera pesos dinámicos [w1, w2, w3, w4] para criterios    │
│  - Output: JSON con pesos + perfil interpretado               │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│  CAPA SCORING: Ranking de Recomendaciones (Backend)           │
│  - Carga features.csv (datos normalizados de Ponte en Carrera)│
│  - Calcula Score = w1×afinidad + w2×salario + w3×costo +    │
│                    w4×facilidad para cada carrera-universidad│
│  - Genera Top-N rankings ordenados por score descendente      │
│  - Aplica filtros geográficos si es necesario                 │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│   CAPA LLM: Generación de Explicación (Backend)              │
│  - LLM redacta explicación personalizada para cada opción     │
│  - Justifica por qué esa carrera coincide con el perfil       │
│  - Incluye datos duros verificables (salario, costo, etc.)    │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│  RESPUESTA AL USUARIO (Frontend)                              │
│  ✓ Ranking Top-3 de combinaciones carrera-universidad        │
│  ✓ Explicaciones personalizadas para cada opción              │
│  ✓ Datos verificables: ingresos, costos, tasas de admisión   │
│  ✓ Opción de explorar más opciones o refinar búsqueda        │
└────────────────────────────────────────────────────────────────┘
                              ↓
                  USUARIO TOMA DECISIÓN INFORMADA
```

### 5.3 Componentes Principales de la Solución

#### A. Pipeline de Datos

Convierte datos crudos de fuente oficial en variables listas para scoring:

```
ENTRADA: Ponte en Carrera (Excel)
    ↓
[1] INGESTION: Descarga automatizada con versionado
    ↓
[2] LIMPIEZA: Estandarización de columnas, validación de tipos
    ↓
[3] FEATURE ENGINEERING: Imputación jerárquica, normalización
    ↓
SALIDA: features.csv (variables de scoring [0,1])
```

**Garantías:**
- Reproducibilidad exacta (snapshots con timestamp)
- Trazabilidad completa (flags de imputación)
- Actualización automática cuando MINEDU publica nuevos datos

#### B. Motor de Scoring Multi-Criterio

Calcula un puntaje para cada combinación carrera–universidad:

```
Score = w₁ × Afinidad + w₂ × Ingresos + w₃ × Costo + w₄ × Facilidad

Donde:
- Afinidad [0,1]: Qué tan alineada está la carrera con los 
                   intereses interpretados del estudiante
- Ingresos [0,1]: Ingreso promedio normalizado de egresados
- Costo [0,1]:    Invertido (mayor score = menor costo real)
- Facilidad [0,1]: Tasa de admisión normalizada

w₁, w₂, w₃, w₄ ∈ [0,1], Σwᵢ = 1.0
```

**Características:**
- Pesos NO son fijos: se generan dinámicamente por el LLM según preferencias
- Transparencia: Cada score es calculable manualmente con datos públicos
- Auditable: Todas las transformaciones (log, min-max, inversión) documentadas

#### C. Asistente Conversacional (LLM)

Interpreta preferencias naturales y genera explicaciones:

**Función 1: Interpretación de Preferencias**
```
INPUT: "Me gustan matemáticas y datos. Quiero un buen sueldo 
        pero mi familia tiene presupuesto limitado"

OUTPUT (JSON):
{
  "interests": ["matemáticas", "análisis de datos", "estadística"],
  "career_families_aligned": ["Ingeniería", "Estadística", "Ciencia de Datos"],
  "salary_priority": 0.8,
  "cost_sensitivity": 0.9,
  "admission_difficulty_tolerance": 0.5,
  
  "weights_generated": {
    "affinity": 0.40,
    "salary": 0.35,
    "cost": 0.20,
    "admission": 0.05
  },
  "reasoning": "Priorizas ingresos y tienes restricción presupuestaria..."
}
```

**Función 2: Generación de Explicaciones**
```
INPUT: Ranking Top-3 + perfil del usuario + pesos utilizados

OUTPUT: 
"Te recomendamos Estadística en la UNMSM porque:
- Fuerte alineación con tus intereses en análisis y matemáticas
- Ingreso promedio de egresados: S/. 3,200/mes (percentil 80)
- Costo anual: S/. 1,500 (accesible para tu presupuesto)
- Tasa de admisión: 18% (desafiante pero posible)
Esta combinación maximiza tu objetivo de ingresos altos 
manteniendo viabilidad financiera."
```

#### D. Capa de Orquestación

Coordina flujo entre componentes:
- Gestión de sesiones de usuario
- Integración LLM → Scoring → Interfaz
- Manejo de errores y fallbacks
- Logging y auditoría de decisiones

---

## 6. Requisitos Funcionales

### 6.1 Funcionalidades del Lado del Usuario

| ID | Funcionalidad | Descripción | Prioridad |
|---|---|---|---|
| **F1.1** | Ingreso conversacional | Usuario escribe consulta en lenguaje natural sobre sus intereses y restricciones | ALTA |
| **F1.2** | Filtros contextuales | Usuario especifica región geográfica, tipo de institución (pública/privada), presupuesto máximo | ALTA |
| **F1.3** | Visualización de ranking | Sistema muestra Top-N (recomendación: 3–5) combinaciones carrera–universidad ordenadas por relevancia | ALTA |
| **F1.4** | Explicaciones personalizadas | Cada recomendación incluye explicación de por qué coincide con el perfil del usuario | ALTA |
| **F1.5** | Comparativa de opciones | Usuario puede comparar 2–3 opciones lado a lado (salario, costo, dificultad de ingreso) | MEDIA |
| **F1.6** | Refinamiento iterativo | Usuario puede hacer seguimiento: "Muéstrame carreras con menor costo" o "¿Y qué hay en Arequipa?" | MEDIA |
| **F1.7** | Descarga de reporte | Usuario puede descargar reporte PDF con recomendaciones y datos verificables | MEDIA |
| **F1.8** | Historial de sesión | Sistema mantiene contexto de consulta anterior en la misma sesión | MEDIA |

### 6.2 Funcionalidades del Lado del Sistema

| ID | Funcionalidad | Descripción | Prioridad |
|---|---|---|---|
| **F2.1** | Descarga automática de datos | Sistema descarga datos de Ponte en Carrera según cronograma (semanal/mensual) | ALTA |
| **F2.2** | Validación de integridad | Pipeline valida que datos descargados contienen campos requeridos y tipos correctos | ALTA |
| **F2.3** | Imputación de datos faltantes | Sistema completa valores faltantes usando estrategia jerárquica (mediana familia+tipo → mediana familia → fallback) | ALTA |
| **F2.4** | Normalización de variables | Sistema transforma variables brutas a scores [0,1] con transformaciones logarítmicas y min-max scaling | ALTA |
| **F2.5** | Cálculo de afinidad | Sistema calcula afinidad entre perfil de estudiante y cada carrera (mediante embeddings, mapeo semántico u otro método definido) | ALTA |
| **F2.6** | Generación de pesos dinámicos | LLM interpreta preferencias y retorna pesos [w1, w2, w3, w4] válidos (suma = 1.0) | ALTA |
| **F2.7** | Ranking con scoring | Sistema calcula Score para cada carrera–universidad usando fórmula ponderada | ALTA |
| **F2.8** | Generación de explicación | LLM redacta explicación personalizada de cada recomendación con datos verificables | ALTA |
| **F2.9** | Versionado de datos | Sistema mantiene snapshots con timestamp de datasets, configs y prompts | MEDIA |
| **F2.10** | Logging y auditoría | Sistema registra: entrada del usuario, pesos generados, scores, ranking final, timestamp | MEDIA |

### 6.3 Requisitos de Integración de Datos

| Requisito | Descripción |
|---|---|
| **Fuente Oficial** | Datos provenientes de Ponte en Carrera (MINEDU), descargables en formato Excel |
| **Frecuencia** | Actualización mínima mensual; objetivo: semanal |
| **Variables Requeridas** | Carrera, familia de carrera, institución, ubicación, tipo institución, tipo gestión, duración, costo, tasa de admisión, ingreso promedio, becas disponibles |
| **Cobertura Mínima** | Todas las universidades e institutos públicos; al menos 50% de privadas |
| **Calidad** | Tratamiento de valores faltantes, validación de rangos, documentación de imputaciones |

---

## 7. Requisitos No Funcionales

### 7.1 Rendimiento

| Requisito | Especificación |
|---|---|
| **Latencia de consulta** | ≤ 3 segundos desde que usuario envía consulta hasta recibir Top-3 recomendaciones |
| **Throughput** | Sistema debe soportar ≥ 100 usuarios concurrentes sin degradación |
| **Tiempo de procesamiento de datos** | Pipeline diario debe completarse en ≤ 30 minutos |
| **Disponibilidad** | 99.0% uptime en horarios de mayor uso (17:00–21:00 hrs weekdays) |

### 7.2 Confiabilidad y Seguridad

| Requisito | Especificación |
|---|---|
| **Persistencia de datos** | Todos los datos de usuario, scores y logs deben persistir con redundancia (backups diarios mínimo) |
| **Privacidad** | Datos de usuario (preferencias, consultas) NO se almacenan sin consentimiento explícito |
| **Encryption** | Datos en tránsito cifrados (TLS 1.2+); datos en reposo cifrados si contienen información sensible |
| **Auditoría** | Todas las decisiones del sistema (pesos generados, scores, explicaciones) son trazables y reproducibles |
| **Reproducibilidad** | Sistema puede reproducir exactamente los resultados de cualquier consulta anterior usando snapshots versionados |

### 7.3 Escalabilidad

| Requisito | Especificación |
|---|---|
| **Crecimiento de datos** | Pipeline debe soportar 3–5× aumento en número de carreras y universidades sin rediseño |
| **Usuarios concurrentes** | Arquitectura debe escalar de 100 a 10,000 usuarios activos simultáneos |
| **Almacenamiento** | Base de datos de features debe crecer a ≤ 500 MB en 5 años bajo proyecciones realistas |
| **Integración de nuevos criterios** | Motor de scoring debe incorporar nuevas variables (p.ej., reputación universitaria) con mínimo rediseño |

### 7.4 Mantenibilidad

| Requisito | Especificación |
|---|---|
| **Documentación de código** | Código debe incluir docstrings, comentarios en lógica compleja y README con instrucciones de ejecución |
| **Modularidad** | Componentes (pipeline, scoring, LLM, frontend) deben estar desacoplados y testables independientemente |
| **Versionado** | Todos los artefactos (datos, prompts, configuración) versionados con timestamps e historial accesible |
| **Logs** | Logs estructurados (JSON) con niveles INFO/WARNING/ERROR que permitan debugging sin acceso a código |
| **Testing** | Mínimo 60% cobertura de tests unitarios para pipeline y motor de scoring |

### 7.5 Usabilidad

| Requisito | Especificación |
|---|---|
| **Interfaz intuitiva** | Interfaz debe ser usable sin instrucciones previas por estudiantes de 16+ años |
| **Tiempo de onboarding** | Usuario debe completar primer ranking en ≤ 2 minutos desde inicio de sesión |
| **Accesibilidad** | Interfaz debe cumplir estándares WCAG 2.1 nivel AA (fuentes legibles, contraste, navegación por teclado) |
| **Responsividad** | Interfaz funciona en desktop, tablet y mobile (resoluciones ≥ 360px) |
| **Idioma** | Interfaz y explicaciones en español peruano; prompts optimizados para este dialecto |

---

## 8. Arquitectura Conceptual

### 8.1 Vista Lógica de Componentes

```
┌─────────────────────────────────────────────────────────────────┐
│                      USUARIO FINAL                             │
└─────────────────────────────────────────────────────────────────┘
                              ↑ ↓
┌─────────────────────────────────────────────────────────────────┐
│              CAPA DE PRESENTACIÓN (Frontend)                   │
│  - Interfaz Conversacional (Web/Mobile)                        │
│  - Input: Consulta en lenguaje natural + Filtros             │
│  - Output: Ranking visual + Explicaciones + Datos             │
│  - Tecnología agnóstica: Web framework moderno               │
└─────────────────────────────────────────────────────────────────┘
                              ↑ ↓
┌─────────────────────────────────────────────────────────────────┐
│                CAPA DE ORQUESTACIÓN (Backend)                  │
│  - API REST con endpoints: /chat, /recommend, /compare        │
│  - Manejo de sesiones y contexto conversacional              │
│  - Validación de inputs y gestión de errores                 │
│  - Logging y auditoría de todas las decisiones               │
│  - Tecnología agnóstica: Framework web escalable             │
└─────────────────────────────────────────────────────────────────┘
         ↑                    ↑                    ↑
         │                    │                    │
    ┌────┴────┐      ┌────────┴────────┐      ┌───┴────┐
    │   LLM   │      │ Motor Scoring   │      │Pipeline│
    │ Layer   │      │                 │      │ Datos  │
    └────┬────┘      └────────┬────────┘      └───┬────┘
         │                    │                    │
┌────────┴──────────┐  ┌──────┴────────────┐  ┌──┴──────────┐
│ Interpretación    │  │ Cálculo de Scores │  │ Ingestion   │
│ de preferencias   │  │ Ranking Top-N     │  │ Limpieza    │
│ Generación de     │  │ Explicaciones     │  │ Features    │
│ explicaciones     │  │                   │  │             │
└────────┬──────────┘  └──────┬────────────┘  └──┬──────────┘
         │                    │                    │
         └────────────────────┴────────────────────┘
                              ↓
                    ┌──────────────────────┐
                    │  Capa de Datos       │
                    │  - features.csv      │
                    │  - config.json       │
                    │  - Snapshots versionados
                    │  - Prompts v1/v2/v3  │
                    └──────────────────────┘
                              ↓
                    ┌──────────────────────┐
                    │  Capa de Persistencia│
                    │  - Base de datos     │
                    │  - Almacenamiento    │
                    │  - Backups           │
                    └──────────────────────┘
```

### 8.2 Flujo de Datos

```
USUARIO
  ↓
Consulta en lenguaje natural
  ↓
API (/chat endpoint)
  ↓
Conversation Manager (recuperar contexto de sesión)
  ↓
LLM: Interpretación
  ├─ Input: Consulta + contexto
  ├─ Process: few-shot prompting con ejemplos
  └─ Output: JSON {interests, weights, profile}
  ↓
Motor de Scoring
  ├─ Input: weights + features.csv + affinity_scores
  ├─ Process: Score = Σ wᵢ × scoreᵢ para cada carrera
  └─ Output: Ranking ordenado [carrera1, carrera2, carrera3]
  ↓
LLM: Explicación
  ├─ Input: Ranking + perfil usuario + pesos
  ├─ Process: Redacción de justificación personalizada
  └─ Output: Explicaciones en español [expl1, expl2, expl3]
  ↓
Logging & Auditoría
  ├─ Registrar: timestamp, entrada usuario, pesos, ranking
  └─ Almacenar: snapshot de estado del sistema
  ↓
API Response
  ├─ JSON con ranking
  ├─ Explicaciones
  └─ Datos verificables
  ↓
Frontend
  ├─ Renderizar UI
  ├─ Mostrar recomendaciones
  └─ Permitir refinamiento iterativo
  ↓
USUARIO recibe recomendación
```

### 8.3 Componentes Principales

#### A. Pipeline de Datos

**Responsabilidades:**
- Descargar datos de Ponte en Carrera
- Limpiar y estandarizar
- Generar features normalizadas
- Mantener versionado y snapshots

**Interfaz:**
- Input: Endpoint Ponte en Carrera
- Output: `features.csv`, `feature_config.json`
- Dependencias: Datos oficiales MINEDU

**Tecnología agnóstica:**
- Lenguaje: Python, Go, TypeScript
- Almacenamiento: CSV, Parquet, base de datos relacional
- Orquestación: Cron job, Airflow, Cloud Scheduler, Prefect

#### B. Motor de Scoring

**Responsabilidades:**
- Cargar features y calcular afinidad
- Generar scores usando fórmula multi-criterio
- Aplicar filtros (región, presupuesto, tipo institución)
- Ordenar y retornar Top-N

**Interfaz:**
- Input: `{weights, features_df, filters}`
- Output: `{ranked_results, scores, explanations}`

**Tecnología agnóstica:**
- Lenguaje: Python, R, SQL
- Librerías: Pandas, NumPy, PySpark para escala
- Cálculo: Vectorizado (preferible sobre loops)

#### C. LLM Layer

**Responsabilidades:**
- Interpretar preferencias del usuario
- Generar pesos dinámicos validados
- Redactar explicaciones personalizadas

**Interfaz:**
- Input: `{user_query, session_context, ranking}`
- Output: `{weights_json, explanations}`

**Tecnología agnóstica:**
- Modelo: Cualquier LLM generativo (cloud API o local)
- Prompting: Few-shot con ejemplos contextualizados
- Validación: JSON schema para outputs estructurados

#### D. Orquestación

**Responsabilidades:**
- Enrutar requests a componentes correctos
- Mantener sesiones y contexto
- Manejo de errores y fallbacks
- Logging de auditoría

**Interfaz:**
- API REST con endpoints JSON
- Session management
- Error handling con fallbacks graceful

**Tecnología agnóstica:**
- Framework: FastAPI, Express, Flask, Django
- Session store: Redis, base de datos, archivo
- Logging: ELK Stack, CloudWatch, Datadog

### 8.4 Decisiones de Arquitectura

| Decisión | Razón | Trade-off |
|---|---|---|
| **Desacoplamiento LLM-Scoring** | Transparencia, auditabilidad, facilita cambios de proveedor | Requiere dos llamadas separadas |
| **Pesos dinámicos generados por LLM** | Personalizabilidad máxima, adaptación a preferencias individuales | Menos reproducible que pesos fijos, requiere validación |
| **Datos oficiales de MINEDU** | Verificabilidad, confianza, actualización automática | Limitación geográfica a Perú, latencia en actualización |
| **Versionado completo** | Reproducibilidad y auditoría | Overhead de almacenamiento y gestión |

---

## 9. Experiencia del Usuario

### 9.1 Escenario de Uso Principal

**Persona:** Carlos, estudiante de 5º año de secundaria, 17 años, Lima  
**Contexto:** Necesita elegir carrera antes de fin de semestre

**Flujo:**

1. **Acceso:** Carlos entra a la plataforma CareerMatch desde su teléfono
2. **Bienvenida:** Interfaz muestra propuesta de valor en 2 líneas
3. **Filtros iniciales:** Se solicita:
   - "¿En qué región quieres estudiar?" (Carlos selecciona Lima)
   - "¿Universidad pública o privada?" (Carlos selecciona "cualquiera")
   - "¿Presupuesto anual máximo?" (Carlos ingresa S/. 6,000)
4. **Chat conversacional:** Carlos escribe:
   > "Me gustan las matemáticas y el análisis de datos. Quiero ganar bien después de egresar, pero mi familia tiene presupuesto limitado. No me importa si es difícil entrar."

5. **Procesamiento invisible:** Sistema:
   - LLM interpreta intereses → afinidad por Estadística, Ingeniería, Ciencia de Datos
   - LLM genera pesos: afinidad=0.40, salario=0.35, costo=0.20, admisión=0.05
   - Motor scoring calcula ranking

6. **Recomendaciones:** Sistema muestra:

   **Top 1: Estadística — UNMSM (Lima)**
   - Score: 0.847/1.0
   - Ingreso promedio: S/. 3,200/mes
   - Costo anual: S/. 1,200
   - Tasa de admisión: 18%
   - Explicación: "Te recomendamos Estadística en la UNMSM porque ofrece una fuerte alineación con tus intereses analíticos, tiene uno de los mejores ingresos promedio entre carreras técnicas (S/. 3,200/mes) y mantiene un costo anual accesible (S/. 1,200). Aunque la tasa de admisión es selectiva (18%), es una carrera que premia el esfuerzo académico en matemáticas."

   **Top 2: Ingeniería Civil — UNI (Lima)**
   - Score: 0.823/1.0
   - [Datos e explicación similar]

   **Top 3: Ingeniería Industrial — PUCP (Lima)**
   - Score: 0.801/1.0
   - [Datos e explicación similar]

7. **Refinamiento:** Carlos pregunta:
   > "¿Y qué hay con costos menores? ¿Algo que pueda estudiar sin examen de admisión?"
   
   Sistema ajusta pesos (costo=0.40, admisión=0.30, salario=0.20, afinidad=0.10) y retorna nuevo ranking.

8. **Comparativa:** Carlos compara Estadística UNMSM vs Ing. Sistemas Universidad Autónoma lado a lado
9. **Descarga:** Carlos descarga PDF con sus Top-3, datos y explicaciones
10. **Decisión:** Carlos se siente más confiado sobre sus opciones

### 9.2 Escenarios Secundarios

**Escenario 2: Padre/Madre apoyando la decisión**
- Accede plataforma, recibe resumen ejecutivo de Top-3 recomendaciones
- Puede ver datos verificables (ingresos, costos) para evaluar viabilidad financiera
- Realiza seguimiento: "¿Qué pasa si tenemos S/. 4,000/año?"

**Escenario 3: Orientador educativo**
- Accede panel de agregación (fuera de MVP)
- Ve tendencias de intereses en su institución
- Usa recomendaciones para enfocar sesiones de tutoría

### 9.3 Wireframe de Interfaz (Conceptual)

```
┌─────────────────────────────────────────────┐
│  CareerMatch Perú — Orientación Vocacional │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Filtros Iniciales                           │
│                                             │
│ Región: [Lima ▼]                          │
│ Tipo institución: [Cualquiera ▼]          │
│ Presupuesto: S/. [6000] /año               │
│                                             │
│ [Continuar →]                              │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Chat                                        │
│                                             │
│ Bot: ¿Qué te interesa estudiar?            │
│                                             │
│ Usuario: Me gustan matemáticas...          │
│                                             │
│ Bot: Entendido. Generando recomendaciones  │
│      (⏳ procesando...)                     │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Recomendaciones (Top-3)                     │
│                                             │
│ 🥇 Estadística — UNMSM                      │
│    Score: 8.47/10                          │
│    Ingresos: S/. 3,200/mes  💰             │
│    Costo: S/. 1,200/año     💵             │
│    Admisión: 18%            📈             │
│    "Te recomendamos porque..."             │
│    [Ampliar] [Comparar]                    │
│                                             │
│ 🥈 Ingeniería Civil — UNI                   │
│    ...                                      │
│                                             │
│ 🥉 Ing. Industrial — PUCP                   │
│    ...                                      │
│                                             │
│ [Refinar búsqueda] [Descargar PDF]         │
└─────────────────────────────────────────────┘
```

---

## 10. Datos y Modelos

### 10.1 Fuente de Datos

**Fuente Primaria:** Ponte en Carrera (MINEDU)  
**URL:** `https://ponteencarrera.minedu.gob.pe/pec-portal-web/`  
**Formato:** Excel descargable mensualmente  
**Cobertura:** ~150 universidades, ~50 institutos superiores, ~1,500 carreras

**Variables Capturadas:**

| Variable | Tipo | Rango | Descripción |
|---|---|---|---|
| career_family | Categórica | 10–15 valores | Familia de carrera (Ingeniería, Educación, Salud, etc.) |
| career | Texto | - | Nombre de la carrera específica |
| institution | Texto | - | Nombre de la institución educativa |
| location | Categórica | 24 regiones | Región geográfica del Perú |
| institution_type | Categórica | {Universidad, Instituto} | Tipo de institución |
| management_type | Categórica | {Público, Privado} | Tipo de gestión |
| duration_years | Numérica | 3–7 años | Duración de la carrera |
| annual_cost | Numérica | S/. 0 – 60,000 | Costo anual promedio |
| admission_rate | Numérica | 0–100% | Porcentaje ingresantes/postulantes |
| monthly_income | Numérica | S/. 800 – 10,000 | Ingreso promedio de egresados |
| scholarships_available | Booleana | {Sí, No} | Disponibilidad de becas |

### 10.2 Procesamiento de Datos

**Pipeline de transformación:**

```
Raw Data (Excel)
    ↓
[1] Limpieza
    - Estandarizar nombres
    - Remover registros incompletos
    - Convertir tipos
    ↓
[2] Imputación Jerárquica
    - Nivel 1: Mediana por (career_family, institution_type)
    - Nivel 2: Mediana por career_family
    - Nivel 3: Fallback configurado
    ↓
[3] Flags de Imputación
    - Marcar qué registros fueron imputados
    - Facilitar auditoría
    ↓
[4] Normalización
    - Log1p(income) → Min-Max [0,1]
    - Log1p(cost) → Min-Max → Invertir
    - Duration → Min-Max → Invertir
    - Admission_rate → Min-Max
    ↓
Features [0,1] listos para scoring
```

### 10.3 Modelo de Lenguaje

**Función:** Interpretación de preferencias + Generación de explicaciones

**Estrategia de Prompting:**

La estrategia exacta se definirá en documentos posteriores (PRD de Engineering de Prompts), pero incluirá:

- **Few-Shot Prompting:** Ejemplos de usuarios con diferentes perfiles
- **Structured Output:** JSON schema para validar pesos generados
- **Chain-of-Thought:** Desglose del razonamiento del LLM
- **Validation Loop:** Asegurar que Σ wᵢ = 1.0 y wᵢ ∈ [0,1]

**Ejemplo de System Prompt (ilustrativo):**

```
Eres CareerMatch, un orientador vocacional especializado en Perú.

Tu tarea: Analizar lo que dice un estudiante sobre sus intereses, 
restricciones y prioridades, para generar pesos personalizados 
que guíen la recomendación de carreras.

Retorna siempre JSON válido:
{
  "interests": ["matemáticas", "análisis"],
  "career_families": ["Estadística", "Ingeniería"],
  "weights": {
    "affinity": 0.40,
    "salary": 0.30,
    "cost": 0.20,
    "admission": 0.10
  }
}

Ejemplos:
[Few-shot examples aquí]
```

### 10.4 Modelo de Afinidad

**Definición:** Medida [0,1] de alineación entre intereses del estudiante y perfil de una carrera

**Métodos candidatos (a evaluar):**

| Método | Ventajas | Desventajas |
|---|---|---|
| **Embeddings + Similitud Coseno** | Captura semántica, flexible | Requiere modelo embedding, latencia |
| **Tabla de Mapeo Semántico** | Rápido, interpretable, auditable | Manual, menos flexible |
| **Score Directo del LLM** | Contextual | Menos reproducible, costo API |
| **Combinación (embeddings + tabla)** | Robusto, con fallback | Complejidad adicional |

**Decisión:** Se evaluarán en sprint de Data (15–28 jun) y se documentará en PRD de Feature Engineering

### 10.5 Métricas de Calidad de Datos

| Métrica | Objetivo | Acción si Falla |
|---|---|---|
| Completitud de variables obligatorias | ≥ 95% | Investigar fuente, aumentar fallback |
| Rango válido de admission_rate | [0, 90%] | Clipear o marcar como imputado |
| Rango válido de duration | [3, 7] años | Imputar por mediana |
| Actualización de fuente | Mensual mínimo | Alertar a equipo, usar última versión válida |
| Coherencia intra-registro | Ingresos > costo (aproximadamente) | Revisar manualmente |

---

## 11. Métricas de Éxito

### 11.1 Métricas de Negocio

| Métrica | Descripción | Target | Horizonte |
|---|---|---|---|
| **Adopción de usuarios** | Número de usuarios activos mensuales (MAU) | 5,000 MAU (Año 1) | 12 meses |
| **Satisfacción del usuario** | NPS (Net Promoter Score) medido post-sesión | ≥ 50 | Contínuo |
| **Tasa de aceptación de recomendación** | % usuarios que reportan "útil" la recomendación | ≥ 70% | Contínuo |
| **Permanencia en carrera** | Reducción en tasa de cambio de carrera (1 año post-ingreso) | Reducir 25% vs. baseline | 18 meses |
| **Alineación intereses-carrera** | Puntuación de alineación post-1año (auto-reportada) | ≥ 7/10 promedio | 18 meses |
| **Penetración geográfica** | % de regiones del Perú representadas en usuarios | ≥ 50% de regiones | 12 meses |
| **Ingresos (si aplica)** | Ingresos mensuales (freemium/B2B) | Define modelo primero | 24 meses |

### 11.2 Métricas Técnicas

#### A. Pipeline de Datos

| Métrica | Descripción | Target |
|---|---|---|
| **Completitud** | % de registros con todas variables obligatorias | ≥ 95% |
| **Validez de tipos** | % de conversiones numéricas exitosas | 100% |
| **Reproducibilidad** | ¿Reproduce exactamente resultados con snapshots versionados? | 100% (binario) |
| **Latencia de update** | Tiempo de descarga → features.csv | ≤ 30 min |
| **Cobertura de fuente** | % de carreras+universidades de Ponte en Carrera incluidas | ≥ 95% |

#### B. Motor de Scoring

| Métrica | Descripción | Target |
|---|---|---|
| **Latencia de scoring** | Tiempo de cálculo de Top-N | ≤ 500 ms |
| **Validez de scores** | Todos los scores ∈ [0,1] | 100% |
| **Suma de pesos** | Verificar Σ wᵢ = 1.0 para cada usuario | 100% |
| **Estabilidad de ranking** | Cambios mínimos en ranking ante pequeñas variaciones de pesos | ≥ 80% top-3 estable |

#### C. LLM Layer

| Métrica | Descripción | Target |
|---|---|---|
| **Tasa de error de parseo JSON** | % de respuestas LLM con JSON inválido | ≤ 5% |
| **Validez de pesos** | % de pesos ∈ [0,1] y Σ = 1.0 | ≥ 95% |
| **Latencia de LLM** | Tiempo respuesta API LLM | ≤ 2 segundos |
| **Coherencia de explicaciones** | Evaluación humana: ¿la explicación es coherente y basada en datos? | ≥ 85% |

#### D. Sistema General

| Métrica | Descripción | Target |
|---|---|---|
| **Disponibilidad** | % uptime durante horarios de uso | ≥ 99.0% |
| **Error rate** | % de requests que resultan en error | ≤ 1% |
| **Latencia end-to-end** | Tiempo total desde input usuario a respuesta | ≤ 3 segundos (p99) |
| **Test coverage** | % de código cubierto por tests | ≥ 60% |

### 11.3 Métricas de Evaluación

| Métrica | Descripción | Método de Medición |
|---|---|---|
| **Recall@3** | % de recomendaciones Top-3 que usuario considera "relevantes" | Survey post-uso |
| **NDCG (Normalized Discounted Cumulative Gain)** | Evalúa calidad de ranking considerando posición | Comparar vs. baseline |
| **Serendipidad** | % de usuarios que descubren carreras no consideradas previamente | Survey de sorpresa positiva |
| **Claridad de explicación** | % usuarios que reportan entender por qué se recomienda una carrera | Survey 5-punto Likert |
| **Reproducibilidad** | ¿Puede auditor reproducir exactamente la decisión del sistema? | Verificación técnica |

---

## 12. Roadmap del Producto

### 12.1 Fases de Desarrollo

#### **Fase 0: MVP (Mínimo Producto Viable) — Hoy a Julio 2026**

**Objetivo:** Sistema funcional end-to-end que demuestre viabilidad técnica y valor

**Entregables:**
- ✅ Pipeline de datos operativo
- ✅ Motor de scoring con 4 criterios
- ✅ Integración LLM para interpretación + explicación
- ✅ Interfaz web básica conversacional
- ✅ Manejo de 100+ usuarios concurrentes

**Limitaciones aceptadas:**
- No hay ajuste fino del LLM
- Afinidad calculada por método simple (se define en sprint)
- Sin monitoreo en producción
- Cobertura geográfica: Principalmente Lima y zonas urbanas

#### **Fase 1: Beta — Agosto a Diciembre 2026**

**Objetivo:** Validación con usuarios reales y retroalimentación

**Entregables:**
- Integración con 5+ instituciones educativas
- Panel de monitoreo en vivo
- Ajuste fino de prompts v2, v3 basado en feedback
- Análisis de satisfacción de usuario
- Documentación de aprendizajes

**KPIs Target:**
- 1,000 usuarios activos mensuales
- NPS ≥ 40
- Tasa de "recomendación útil" ≥ 60%

#### **Fase 2: Expansión — 2027**

**Objetivo:** Escala a nivel nacional e integración de nuevas capacidades

**Entregables:**
- Modelo de negocio implementado (freemium u otro)
- Cobertura de ≥ 50% de regiones del Perú
- Nuevos criterios: reputación universitaria, empleabilidad por sector
- Portal para orientadores educativos (B2B)
- Integración con universidades para feedback automático

**KPIs Target:**
- 50,000 usuarios activos mensuales
- NPS ≥ 50
- Evidencia de impacto en reducción de cambio de carrera

#### **Fase 3: Consolidación y Nuevas Aplicaciones — 2027+**

**Objetivo:** Solidificar posición y explorar nuevos mercados

**Opciones:**
- Expansión a otros países (LATAM)
- Integración con análisis de soft skills
- Recomendación de especialidades dentro de carreras
- Tracking de empleabilidad a 5 años
- Partnerships con empresas (matching estudiante-empleador)

### 12.2 Timeline Detallado (Próximas 16 semanas)

```
SEMANA 1-2 (Jun 17 – Jun 30)
├─ Publicar repositorio GitHub
├─ Definir método de cálculo de Afinidad
├─ Validación de datos
└─ Análisis exploratorio (EDA)

SEMANA 3-4 (Jul 1 – Jul 14)
├─ Feature engineering completo
├─ Motor de scoring v1 operativo
├─ SYSTEM_PROMPT_V1 con ejemplos
└─ Integración LLM-Scoring en notebook demo

SEMANA 5-6 (Jul 15 – Jul 28)
├─ Frontend Streamlit integrado
├─ API FastAPI funcional
├─ Docker operativo
└─ CI/CD configurado (GitHub Actions)

SEMANA 7-8 (Jul 29 – Aug 12)
├─ MLOps: versionado operativo
├─ Evaluación inicial: test cases
├─ Redacción informe paralela
├─ Grabación video demo
└─ Ajustes finales

ENTREGA (Aug 9)
├─ Informe final PDF
├─ Repositorio con código completo
├─ Video de demostración
└─ Presentación oral (todos integrantes)
```

---

## 13. Restricciones y Supuestos

### 13.1 Restricciones

| Restricción | Impacto | Mitigación |
|---|---|---|
| **Datos de Ponte en Carrera solo para Perú** | Solución geograficamente acotada inicialmente | Documentar como oportunidad de expansión futura |
| **Ingreso reportado = empleo formal únicamente** | Subestima ingresos reales en economía informal (40% del trabajo en Perú) | Documentar limitación, señalar en explicaciones |
| **Presupuesto limitado de cómputo** | No es posible usar GPUs masivas ni entrenamientos extensos | Usar LLMs cloud API y no hacer fine-tuning costoso |
| **Latencia de actualización de datos (mensual)** | Recomendaciones desactualizado si oferta cambia | Comunicar periodicidad a usuarios |
| **Falta de base de empleo de egresados actualizada** | No es posible validar con datos de empleabilidad futuros en esta fase | Planificar feedback loop con universidades |

### 13.2 Supuestos

| Supuesto | Justificación | Riesgo |
|---|---|---|
| **Estudiantes accederán a internet y smartphone** | 85%+ penetración de smartphone en Lima/zonas urbanas | Bajo en audiencia primaria |
| **MINEDU mantendrá portal actualizado** | Portal público oficial desde 2016 | Bajo (institución pública) |
| **Estudiantes pueden expresar preferencias en lenguaje natural** | Jóvenes de 16+ hablan español naturalmente | Bajo |
| **LLM interpretará preferencias con ≥85% exactitud** | LLMs modernos tienen desempeño alto en español | Medio (depende de ejemplos few-shot) |
| **Usuarios confiarán en recomendaciones basadas en datos oficiales** | Datos MINEDU tienen legitimidad | Medio (requiere comunicación clara) |
| **La decisión de carrera es principalmente racional** | Supuesto simplificador (hay factores emocionales/sociales) | Medio (documentar limitación) |

---

## 14. Referencias

Al-Dossari, H., Abu Nughaymish, F., Al-Qahtani, Z., Alkahlifah, M., & Alqahtani, A. (2020). A machine learning approach to career path choice for information technology graduates. Engineering, Technology & Applied Science Research, 10(6), 6589–6596.

Bender, E. M., Gebru, T., McMillan-Major, A., & Shmitchell, S. (2021). On the dangers of stochastic parrots: Can language models be too big? En Proceedings of the 2021 ACM Conference on Fairness, Accountability, and Transparency (pp. 610–623).

Bommasani, R., Hudson, D. A., Adeli, E., Altman, R., Arora, S., et al. (2021). On the opportunities and risks of foundation models. arXiv preprint arXiv:2108.07258.

Esteban, A., Zafra, A., & Romero, C. (2020). Helping university students to choose elective courses by using a hybrid multi-criteria recommendation system with genetic optimization. Knowledge-Based Systems, 194, 105385.

Ezz, M., & Elshenawy, A. (2020). Adaptive recommendation system using machine learning algorithms for predicting student's best academic program. Education and Information Technologies, 25(4), 2733–2746.

Gao, Y., Xiong, Y., Gao, X., Jia, K., Pan, J., Bi, Y., Dai, Y., Sun, J., Wang, M., & Wang, H. (2024). Retrieval-augmented generation for large language models: A survey. arXiv preprint arXiv:2312.10997.

Huang, C., Yu, T., Xie, K., Zhang, S., Yao, L., & McAuley, J. (2024). Foundation models for recommender systems: A survey and new perspectives. arXiv preprint arXiv:2402.11143.

---

## Apéndice A: Términos y Definiciones

| Término | Definición |
|---|---|
| **Afinidad** | Medida [0,1] de alineación entre los intereses expresados de un estudiante y el perfil académico de una carrera |
| **Motor de Scoring** | Componente que calcula un puntaje para cada combinación carrera–universidad basado en criterios ponderados |
| **Pesos Dinámicos** | Coeficientes (w₁, w₂, w₃, w₄) generados por el LLM en cada sesión basados en preferencias del usuario, a diferencia de pesos fijos predeterminados |
| **Ponte en Carrera** | Portal oficial del MINEDU que publica datos sobre carreras, universidades, costos, ingresos y empleabilidad en Perú |
| **Reproducibilidad** | Capacidad de obtener exactamente los mismos resultados utilizando los mismos datos y configuración |
| **Versionado** | Práctica de mantener snapshots históricos de datos, configuración y prompts para auditoría y reproducibilidad |

---

**Documento Preparado Por:** Equipo CareerMatch Perú  
**Fecha de Publicación:** Junio 2026  
**Estado:** Documento Base - Sujeto a Iteración  
**Versión Siguiente:** PRD v1.1 (Engineering de Prompts y Feature Engineering)
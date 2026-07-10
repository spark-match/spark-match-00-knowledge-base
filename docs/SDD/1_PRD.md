---
title: "CareerMatch Perú — PRD Actualizado v1.1"
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
  - "[[docs/SDD/2_requirements]]"
  - "[[docs/SDD/3_design]]"
  - "[[docs/SDD/ARCHITECTURE]]"
---

# CareerMatch Perú — PRD Actualizado v1.1

**Versión:** 1.1  
**Fecha:** Junio 2026  
**Estado:** Documento Base Actualizado  
**Cambios:** Aclaración scope demo vs. producción, feedback de usuario, RAG opcional, replanteamiento del motor de scoring

---

## Tabla de Contenidos

1. [Resumen Ejecutivo](#resumen-ejecutivo)
2. [Contexto del Negocio](#contexto-del-negocio)
3. [Definición del Problema](#definición-del-problema)
4. [Objetivos del Proyecto](#objetivos-del-proyecto)
5. [Descripción de la Solución](#descripción-de-la-solución)
6. [Scope: Demo vs. Producción Futura](#scope-demo-vs-producción-futura)
7. [Requisitos Funcionales](#requisitos-funcionales)
8. [Requisitos No Funcionales](#requisitos-no-funcionales)
9. [Arquitectura Conceptual](#arquitectura-conceptual)
10. [Experiencia del Usuario](#experiencia-del-usuario)
11. [Motor de Scoring: Propuesta Inicial y Evolución](#motor-de-scoring-propuesta-inicial-y-evolución)
12. [Módulo RAG Opcional](#módulo-rag-opcional)
13. [Feedback del Usuario y Base de Datos de Entrenamiento](#feedback-del-usuario-y-base-de-datos-de-entrenamiento)
14. [Datos y Modelos](#datos-y-modelos)
15. [Métricas de Éxito](#métricas-de-éxito)
16. [Roadmap del Producto](#roadmap-del-producto)
17. [Restricciones y Supuestos](#restricciones-y-supuestos)
18. [Referencias](#referencias)

---

## 1. Resumen Ejecutivo

**CareerMatch Perú** es una plataforma de orientación vocacional impulsada por inteligencia artificial generativa que ayuda a estudiantes de educación secundaria y sus familias en Perú a tomar decisiones informadas sobre la elección de carrera universitaria.

La plataforma utiliza un **asistente conversacional basado en LLM** que interpreta preferencias expresadas en lenguaje natural, mantiene un diálogo iterativo para capturar preferencias incompletas, y genera recomendaciones personalizadas de combinaciones carrera–universidad, sustentadas en datos oficiales del Ministerio de Educación y un **motor de evaluación multi-criterio transparente y auditable**.

**Propuesta de Valor:**
- Reduce la brecha de información en orientación vocacional en contextos de recursos limitados
- Combina datos oficiales con inteligencia conversacional para recomendaciones contextualizadas
- Mantiene transparencia y auditabilidad mediante evaluación estructurada y desacoplada
- Construye históricamente una base de datos de feedback de usuario para entrenar modelos propios en el futuro
- Proporciona acceso conversacional a detalles de carreras via RAG (opcional)

**Estado Actual (Junio 2026):**
- Esta versión describe una **demo funcional** para evaluación técnica y viabilidad de concepto
- Destinada a usuarios limitados (evaluadores, casos de prueba)
- Arquitectura diseñada para evolucionar hacia producción abierta sin rediseño fundamental

---

## 2. Contexto del Negocio

### 2.1 Industria y Mercado

**Sector:** Tecnología educativa (EdTech) aplicada a orientación vocacional y educación superior  
**Geografía Primaria:** Perú  
**Audiencia Primaria (Producción Futura):** Estudiantes de 4º y 5º año de secundaria (16–18 años)  
**Audiencia Secundaria (Producción Futura):** Padres de familia, orientadores educativos, instituciones educativas  
**Audiencia Actual (Demo):** Evaluadores, panel de prueba (≤100 usuarios)

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

**Mercado Objetivo (Largo Plazo):**
- ~200,000 estudiantes/año que egresan de secundaria en Perú
- Instituciones educativas (colegios, academias preuniversitarias)
- Plataformas digitales de educación superior

**Estimación de Tamaño:**
- Si el 10% de estudiantes accede a herramientas de orientación vocacional: 20,000 usuarios/año
- Modelo B2B2C: escuelas + plataformas de RRSS + alianzas con universidades

**Modelos de Ingresos Potenciales (fuera del alcance actual):**
- Freemium: acceso básico gratuito, reportes detallados de pago
- B2B: licencia para instituciones educativas
- Partnerships: ingresos por referrals a universidades

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

### 4.1 Objetivo de Negocio — Largo Plazo (Producción)

**Desarrollar una solución de orientación vocacional basada en IA que:**
- Mejore la tasa de alineación entre preferencias de estudiantes y decisiones vocacionales
- Reduzca la tasa de cambio de carrera en primer año
- Sea accesible, transparente y verificable con datos oficiales
- Escale de manera sostenible en el contexto educativo peruano
- Construya históricamente una base de datos de feedback para entrenar modelos propios

**KPIs de Negocio (largo plazo — no medibles aún en demo):**
- Aumento de 30% en satisfacción con la decisión de carrera (1 año post-ingreso)
- Reducción de 25% en tasa de cambio de carrera (año 1)
- 50,000 usuarios activos en 2 años
- NPS (Net Promoter Score) ≥ 50 en población objetivo
- Modelo propio de recomendación entrenado con datos históricos de feedback (año 2+)

### 4.2 Objetivo de Demo (Julio 2026)

**Demostrar viabilidad técnica y valor conceptual mediante:**
- Pipeline de datos reproducible y auditado
- Motor de evaluación multi-criterio que mantenga consistencia y confiabilidad
- Integración LLM que interprete preferencias naturales y genere recomendaciones fundamentadas
- Captura de feedback de usuario para construir base de datos de entrenamiento futuro
- Sistema conversacional que capture información incompleta sin ser invasivo

**Casos de evaluación:**
- 3–5 personas (evaluadores + panel de prueba)
- 10–15 consultas de prueba con diferentes perfiles
- Validación de reproducibilidad de resultados
- Evaluación cualitativa de calidad de recomendaciones

### 4.3 Objetivos Técnicos (Demo)

**O1:** Implementar un pipeline de datos reproducible que integre datos oficiales de Ponte en Carrera con transformaciones normalizadas para evaluación

**O2:** Desarrollar un motor de evaluación multi-criterio que:
- Sea transparente y auditable (cada decisión verificable manualmente)
- Genere rankings personalizados consistentes
- Sea adaptable a diferentes estrategias de ponderación sin rediseño fundamental
- Pueda escalar a métodos alternativos en el futuro (p.ej., modelos propios entrenados)

**O3:** Implementar un asistente conversacional que:
- Interprete preferencias en lenguaje natural
- Mantuviera diálogos iterativos para capturar información incompleta
- Genere recomendaciones fundamentadas en datos
- Adapte dinámicamente la evaluación sin ser directo o invasivo

**O4:** Garantizar reproducibilidad y auditabilidad mediante versionado completo de datos, prompts y configuración

**O5:** Asegurar claridad en las recomendaciones mediante explicaciones fundamentadas en datos verificables

---

## 5. Descripción de la Solución

### 5.1 Propuesta de Valor Diferencial

A diferencia de soluciones existentes, CareerMatch Perú **integra tres componentes que no se encuentran juntos en el mercado local:**

| Componente | CareerMatch | Alternativas Existentes |
|---|---|---|
| **Datos Oficiales Integrados** | Ponte en Carrera (MINEDU) actualizado automáticamente | Portales estáticos o datos no verificables |
| **Conversación Personalizada Iterativa** | LLM que interpreta preferencias naturales y mantiene diálogo para completar información | Tests psicométricos genéricos o manuales |
| **Motor Transparente y Adaptable** | Evaluación multi-criterio con pesos dinámicos, verificable y preparada para modelos propios | Caja negra o recomendaciones sin explicación |
| **Captura de Feedback Histórico** | Base de datos de decisiones usuario→feedback para entrenar modelos futuros | Sin mecanismo de aprendizaje iterativo |
| **Acceso a Detalles de Carrera (RAG)** | Consultas conversacionales a documentos oficiales de carreras seleccionadas | Información estática o no contextual |

### 5.2 Flujo de Interacción Principal

```
┌────────────────────────────────────────────────────────────────┐
│                    USUARIO ESTUDIANTE                         │
│                 (Demo: evaluador / panel)                      │
└────────────────────────────────────────────────────────────────┘
                              ↓
                   Expresar preferencias en lenguaje natural:
              "Quiero carrera con buen sueldo, pero tengo 
               presupuesto limitado. Me gustan matemáticas"
                              ↓
┌────────────────────────────────────────────────────────────────┐
│          CAPA DE INTERFAZ CONVERSACIONAL (Frontend)           │
│  - Recibe entrada del usuario                                 │
│  - Aplica filtros iniciales opcionales (región, presupuesto)  │
│  - Mantiene contexto de sesión                                │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│    CAPA LLM: Interpretación de Preferencias (Backend)         │
│  - Recibe consulta y contexto de sesión                       │
│  - LLM interpreta: intereses, restricciones financieras       │
│  - Evalúa si hay información suficiente para recomendar       │
│  - Si NO: genera pregunta conversacional (no invasiva) para   │
│    completar información                                      │
│  - Si SÍ: genera perfil estructurado + indicadores para       │
│    evaluación multi-criterio                                  │
└────────────────────────────────────────────────────────────────┘
                              ↓
                    [¿Información Suficiente?]
                    /                      \
                 NO                        SÍ
                /                            \
               ↓                              ↓
        "¿Qué región prefieres?"     Motor de Evaluación
        "¿Importa el costo?"              ↓
        (No invasivo, conversacional)  [Genera Ranking]
               ↓                            ↓
        [Vuelve a [1]]         ┌────────────────────────────┐
                               │  CAPA LLM: Explicación     │
                               │  - Redacta justificación   │
                               │    personalizada           │
                               │  - Incluye datos duros     │
                               └────────────────────────────┘
                                           ↓
┌────────────────────────────────────────────────────────────────┐
│  RESPUESTA AL USUARIO (Frontend)                              │
│  ✓ Ranking Top-3 de combinaciones carrera-universidad        │
│  ✓ Explicaciones personalizadas para cada opción              │
│  ✓ Datos verificables: ingresos, costos, tasas de admisión   │
│  ✓ [OPCIONAL] Acceso RAG a detalles de carrera elegida       │
│  ✓ Opción de: validar recomendación, explorar más, refinar   │
└────────────────────────────────────────────────────────────────┘
                              ↓
                   [Usuario valida recomendación]
                   Escala Likert 1–5: ¿Qué tan útil?
                              ↓
┌────────────────────────────────────────────────────────────────┐
│  ALMACENAMIENTO DE FEEDBACK (Backend)                         │
│  - Guardar: consulta usuario, perfil interpretado, ranking,   │
│    explicación, feedback Likert, timestamp, sesión ID         │
│  - Estos datos = base para entrenar modelo propio futuro      │
└────────────────────────────────────────────────────────────────┘
                              ↓
                  USUARIO TOMA DECISIÓN INFORMADA
```

### 5.3 Componentes Principales de la Solución

#### A. Pipeline de Datos

Convierte datos crudos de fuente oficial en variables listas para evaluación:

```
ENTRADA: Ponte en Carrera (Excel)
    ↓
[1] INGESTION: Descarga automatizada con versionado
    ↓
[2] LIMPIEZA: Estandarización de columnas, validación de tipos
    ↓
[3] FEATURE ENGINEERING: Imputación jerárquica, normalización
    ↓
SALIDA: features.csv (variables de evaluación [0,1])
```

**Garantías:**
- Reproducibilidad exacta (snapshots con timestamp)
- Trazabilidad completa (flags de imputación)
- Actualización automática cuando MINEDU publica nuevos datos

#### B. Motor de Evaluación Multi-Criterio

**IMPORTANTE:** El motor descrito a continuación es una **propuesta inicial** para evaluación de viabilidad técnica. No es definitivo ni se asume que será la solución final. La intención es demostrar que es posible construir un sistema de recomendación **consistente, confiable y auditable** que puede evolucionar hacia alternativas mejores (incluyendo modelos propios entrenados con feedback histórico).

**Propuesta Inicial:**

Calcula un indicador de concordancia para cada combinación carrera–universidad basado en múltiples criterios:

```
Indicador de Concordancia = f(Afinidad, Ingresos, Costo, Facilidad)

Donde cada criterio es normalizado a [0,1]:
- Afinidad [0,1]: Qué tan alineada está la carrera con los 
                   intereses interpretados del estudiante
- Ingresos [0,1]: Ingreso promedio normalizado de egresados
- Costo [0,1]:    Invertido (mayor valor = menor costo real)
- Facilidad [0,1]: Tasa de admisión normalizada

Posibles estrategias de combinación (a evaluar):
  a) Suma ponderada: w₁×A + w₂×I + w₃×C + w₄×F
  b) Promedio geométrico: (A^w₁ × I^w₂ × C^w₃ × F^w₄)^(1/Σw)
  c) Min de criterios (garantiza mínimo aceptable en todos)
  d) Modelo propio entrenado (futuro, con datos de feedback)
```

**Características:**
- **Transparencia:** Cada componente es calculable manualmente con datos públicos
- **Auditabilidad:** Todas las transformaciones (log, min-max, inversión) documentadas
- **Flexibilidad:** Pesos pueden ser dinámicos (generados por LLM) o explorados manualmente
- **Preparación para evolución:** La estructura permite reemplazar por métodos alternativos sin cambiar el flujo general

**¿Por qué esta propuesta es defendible incluso siendo inicial?**

1. **Basada en principios:** Multi-criterio es enfoque estándar en teoría de decisiones
2. **Verificable:** A diferencia de caja negra, cada recomendación es justificable
3. **Adaptable:** Los pesos pueden ajustarse sin rediseño si se descubre mejor estrategia
4. **Prototipo de producción:** No es un juguete; es la estructura que escala

**Evaluación y posibles mejoras durante desarrollo:**

Se explorarán durante el sprint (15–28 jun):
- ¿Qué estrategia de combinación produce rankings más coherentes?
- ¿Funcionan mejor pesos fijos o dinámicos?
- ¿Qué feedback de usuarios sugiere sobre la calidad de recomendaciones?

Si se identifica enfoque superior, se documentará y se planificará transición en Fase 1.

#### C. Asistente Conversacional (LLM)

Interpreta preferencias naturales y mantiene diálogo para información incompleta:

**Función 1: Interpretación de Preferencias y Detección de Información Incompleta**

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
  "geographic_preference": null,  ← FALTA INFORMACIÓN
  "institution_type_preference": null,  ← FALTA INFORMACIÓN
  "missing_info": [
    "geographic_preference",
    "institution_type_preference"
  ],
  "confidence_for_recommendation": 0.65,  ← No es suficiente aún
  "suggested_follow_up": "¿Hay alguna región del Perú donde prefieras estudiar, o tienes flexibilidad?"
}
```

**Función 2: Generación de Preguntas Conversacionales (No Invasivas)**

```
Estrategia:
- Si confidence < 0.70: generar pregunta seguimiento
- La pregunta debe ser abierta, contextual, no forzada
- Mencionar por qué la información ayuda

EJEMPLO:
Profesional: "¿Hay alguna región del Perú donde prefieras estudiar, 
             o tienes flexibilidad? Algunos estudiantes prefieren 
             ciudades grandes para más oportunidades."

NO hacer: "¿Qué región? (Respuesta obligatoria)"
```

**Función 3: Generación de Explicaciones Personalizadas**

```
INPUT: Ranking Top-3 + perfil del usuario + criterios evaluados

OUTPUT: 
"Te recomendamos Estadística en la UNMSM porque:
- Tu afinidad con el análisis de datos es alta (82/100) y esta 
  carrera es especializada en eso
- Ingresos promedio: S/. 3,200/mes (superior a mayoría de carreras)
- Costo anual: S/. 1,200 (accesible para tu presupuesto de S/. 6,000)
- Tasa de admisión: 18% (selectiva, pero menos que ingeniería pura)

Esta combinación maximiza tu objetivo de ingresos altos 
mientras mantiene viabilidad financiera."
```

**Función 4: Gestión de Conversación Multi-Turno**

```
- Mantener contexto entre turnos (no repetir preguntas)
- Actualizar perfil interpretado con nueva información
- Re-evaluar confidence para recomendación
- Cuando confidence ≥ 0.70: presentar Top-3

Máximo 3–4 turnos de seguimiento antes de presentar mejor 
recomendación con información disponible (graceful degradation)
```

#### D. Capa de Orquestación

Coordina flujo entre componentes:
- Gestión de sesiones de usuario
- Integración LLM → Evaluación → Interfaz
- Manejo de errores y fallbacks
- Logging y auditoría de decisiones
- Almacenamiento de feedback

---

## 6. Scope: Demo vs. Producción Futura

### 6.1 Clarificación: Demo Funcional (Junio—Agosto 2026)

**Definición:**
La versión actual es una **demo técnica** destinada a:
- Demostrar viabilidad conceptual y técnica
- Permitir evaluación de calidad de recomendaciones
- Construir base de datos inicial de feedback
- Servir como base para iteración futura

**Características Demo:**

| Aspecto | Demo | Producción Futura |
|---|---|---|
| **Número de usuarios** | 3–5 (evaluadores + panel prueba) | 10,000+ |
| **Sesiones concurrentes** | 1–2 simultáneas | 100+ simultáneas |
| **Casos de evaluación** | 10–15 consultas de prueba | Miles de consultas reales |
| **Plataforma** | Interfaz web navegador | Web responsive + API |
| **Almacenamiento** | Local o cloud simple | Base de datos escalable |
| **Uptime requerido** | Best effort (no crítico) | 99%+ durante horarios pico |
| **Privacidad** | Datos de prueba OK | GDPR/LGPD compliance |
| **Feedback de usuario** | Recolectado manualmente o simple | Capturado automáticamente a escala |

### 6.2 Evolución Planificada hacia Producción

**Fase 1 (Post-demo):** Estabilización
- Aumentar users a 100–500
- Monitoreo en vivo
- Ajuste de prompts basado en logs
- Documentación de aprendizajes

**Fase 2:** Escala limitada
- 5,000–10,000 usuarios
- Infraestructura escalable
- Modelo de feedback operativo
- Análisis de impacto real

**Fase 3:** Producción abierta
- 50,000+ usuarios
- Modelo propio entrenado
- Optimizaciones de rendimiento
- Integración con instituciones educativas

---

## 7. Requisitos Funcionales

### 7.1 Funcionalidades del Lado del Usuario

| ID | Funcionalidad | Descripción | Prioridad | Scope |
|---|---|---|---|---|
| **F1.1** | Ingreso conversacional | Usuario escribe consulta en lenguaje natural sobre sus intereses y restricciones | ALTA | Demo |
| **F1.2** | Diálogo iterativo | Sistema mantiene conversación para completar información incompleta sin ser invasivo | ALTA | Demo |
| **F1.3** | Filtros contextuales opcionales | Usuario puede especificar región geográfica, tipo institución, presupuesto | MEDIA | Demo |
| **F1.4** | Visualización de ranking | Sistema muestra Top-3 combinaciones carrera–universidad ordenadas por relevancia | ALTA | Demo |
| **F1.5** | Explicaciones personalizadas | Cada recomendación incluye explicación de por qué coincide con el perfil del usuario | ALTA | Demo |
| **F1.6** | Feedback de recomendación | Usuario valida si recomendación le parece útil (escala Likert 1–5) | ALTA | Demo |
| **F1.7** | Acceso RAG a detalles | [OPCIONAL] Usuario puede hacer preguntas sobre detalles de carrera recomendada (qué cursos, se dedica a qué, etc.) | MEDIA | Demo (carreras piloto) |
| **F1.8** | Comparativa de opciones | Usuario puede comparar 2–3 opciones lado a lado | MEDIA | Producción futura |
| **F1.9** | Refinamiento iterativo | Usuario puede hacer seguimiento: "Muéstrame carreras con menor costo" | MEDIA | Producción futura |
| **F1.10** | Descarga de reporte | Usuario puede descargar reporte PDF con recomendaciones | MEDIA | Producción futura |

### 7.2 Funcionalidades del Lado del Sistema

| ID | Funcionalidad | Descripción | Prioridad | Scope |
|---|---|---|---|---|
| **F2.1** | Descarga de datos | Sistema descarga datos de Ponte en Carrera (manual o automatizado) | ALTA | Demo |
| **F2.2** | Validación de integridad | Pipeline valida datos cargados | ALTA | Demo |
| **F2.3** | Imputación de datos faltantes | Sistema completa valores faltantes usando estrategia jerárquica | ALTA | Demo |
| **F2.4** | Normalización de variables | Sistema transforma variables brutas a scores [0,1] | ALTA | Demo |
| **F2.5** | Cálculo de afinidad | Sistema calcula afinidad entre perfil y carreras (método a definir) | ALTA | Demo |
| **F2.6** | Interpretación de preferencias | LLM interpreta consulta y retorna perfil estructurado | ALTA | Demo |
| **F2.7** | Detección de información incompleta | Sistema identifica qué información falta para buena recomendación | ALTA | Demo |
| **F2.8** | Generación de seguimiento | Sistema genera pregunta contextual para completar información | ALTA | Demo |
| **F2.9** | Evaluación multi-criterio | Sistema calcula indicador de concordancia para cada carrera | ALTA | Demo |
| **F2.10** | Ranking consistente | Sistema ordena opciones manteniendo consistencia en criterios | ALTA | Demo |
| **F2.11** | Generación de explicación | LLM redacta justificación personalizada | ALTA | Demo |
| **F2.12** | Almacenamiento de feedback | Sistema persiste: consulta, perfil, ranking, validación usuario | ALTA | Demo |
| **F2.13** | Versionado de datos | Sistema mantiene snapshots con timestamp | MEDIA | Demo |
| **F2.14** | Logging y auditoría | Sistema registra todas las decisiones | MEDIA | Demo |
| **F2.15** | RAG sobre carreras | [OPCIONAL] Sistema consulta documentos embeddeados de carreras seleccionadas | MEDIA | Demo (piloto) |

### 7.3 Requisitos de Integración de Datos

| Requisito | Descripción | Demo | Producción |
|---|---|---|---|
| **Fuente Oficial** | Datos de Ponte en Carrera (MINEDU), descargables en Excel | Descarga manual o semanal | Automática diaria |
| **Frecuencia** | Actualización de datos | Una vez para demo | Semanal mínimo |
| **Variables Requeridas** | Carrera, institución, ubicación, costo, tasa admisión, ingresos, etc. | Todas 15+ variables | Todas + nuevas |
| **Cobertura Mínima** | Número de carreras/universidades | 100+ combinaciones | 1,500+ combinaciones |
| **Calidad** | Tratamiento de valores faltantes, validación de rangos | Imputación jerárquica | Imputación + validación continua |

---

## 8. Requisitos No Funcionales

### 8.1 Rendimiento (Demo vs. Producción)

| Requisito | Demo Target | Producción Target |
|---|---|---|
| **Latencia de consulta** | ≤ 5 segundos (best effort) | ≤ 3 segundos (p99) |
| **Throughput** | 1–2 usuarios simultáneos | ≥ 100 usuarios concurrentes |
| **Tiempo pipeline de datos** | ≤ 1 hora (una sola ejecución) | ≤ 30 minutos diarios |
| **Disponibilidad** | Best effort (no crítico) | 99.0% uptime horas pico |

### 8.2 Confiabilidad y Seguridad

| Requisito | Especificación | Scope |
|---|---|---|
| **Persistencia de datos** | Datos guardados con backup básico | Demo: backup manual; Producción: redundancia |
| **Privacidad** | Datos de usuario no compartidos sin consentimiento | Demo: usuarios conscientes de prueba; Producción: full compliance |
| **Encryption** | Datos en tránsito cifrados (TLS 1.2+) | Producción; Demo: best effort |
| **Auditoría** | Todas las decisiones reproducibles y trazables | Demo y Producción |
| **Reproducibilidad** | Sistema puede reproducir exactamente resultados anteriores | Demo y Producción |

### 8.3 Escalabilidad

| Requisito | Demo | Producción |
|---|---|---|
| **Crecimiento de datos** | No crítico | Pipeline soporta 3–5× aumento |
| **Usuarios concurrentes** | 1–2 | Escalar de 100 a 10,000 sin rediseño |
| **Almacenamiento** | Archivo local OK | Base de datos escalable |

### 8.4 Mantenibilidad

| Requisito | Especificación |
|---|---|
| **Documentación** | Código con docstrings, README con instrucciones |
| **Modularidad** | Componentes desacoplados (pipeline, evaluación, LLM, frontend) |
| **Versionado** | Datos, prompts, configuración versionados |
| **Logs** | Logs estructurados (JSON) con niveles INFO/WARNING/ERROR |
| **Testing** | Mínimo 60% cobertura tests unitarios |

### 8.5 Usabilidad

| Requisito | Especificación | Scope |
|---|---|---|
| **Interfaz intuitiva** | Usable sin instrucciones previas | Demo + Producción |
| **Tiempo onboarding** | Primer ranking en ≤ 3 minutos | Demo: acceptable; Producción: ≤ 2 min |
| **Accesibilidad** | WCAG 2.1 nivel AA (futuro) | Producción |
| **Responsividad** | Funciona en navegador desktop; accesible vía mobile browser (sin app nativa) | Demo + Producción |
| **Idioma** | Español peruano | Demo + Producción |

---

## 9. Arquitectura Conceptual

### 9.1 Vista Lógica de Componentes

```
┌─────────────────────────────────────────────────────────────────┐
│                      USUARIO FINAL                             │
│              (Demo: evaluador/panel prueba)                    │
└─────────────────────────────────────────────────────────────────┘
                              ↑ ↓
┌─────────────────────────────────────────────────────────────────┐
│              CAPA DE PRESENTACIÓN (Frontend)                   │
│  - Interfaz Conversacional (Web, responsive, no app nativa)    │
│  - Input: Consulta en lenguaje natural + Filtros opcionales   │
│  - Output: Chat + Ranking visual + Explicaciones + RAG [opt]  │
│  - Tecnología agnóstica: Web framework moderno (React, etc.)  │
└─────────────────────────────────────────────────────────────────┘
                              ↑ ↓
┌─────────────────────────────────────────────────────────────────┐
│                CAPA DE ORQUESTACIÓN (Backend)                  │
│  - API REST con endpoints: /chat, /recommend, /feedback       │
│  - Manejo de sesiones y contexto conversacional               │
│  - Validación de inputs y gestión de errores                  │
│  - Logging y auditoría de todas las decisiones                │
│  - Tecnología agnóstica: Framework web escalable              │
└─────────────────────────────────────────────────────────────────┘
     ↑                  ↑                   ↑                  ↑
     │                  │                   │                  │
┌────┴─────┐    ┌───────┴──────────┐  ┌───┴────┐         ┌──┴────┐
│   LLM    │    │ Motor Evaluación │  │Pipeline│         │ RAG   │
│ Layer    │    │                  │  │ Datos  │         │Module │
└────┬─────┘    └───────┬──────────┘  └───┬────┘         └──┬────┘
     │                  │                  │                  │
┌─────┴────────────┐  ┌──┴────────────┐  ┌┴────────────┐   ┌──┴────┐
│ Interpretación   │  │ Cálculo de    │  │ Ingestion  │   │ Docum.│
│ Seguimiento      │  │ Indicadores   │  │ Limpieza   │   │ Embed.│
│ Explicaciones    │  │ Ranking       │  │ Features   │   │ Search│
└─────┬────────────┘  └──┬────────────┘  └┴────────────┘   └──┬────┘
      │                  │                  │                  │
      └──────────────────┴──────────────────┴──────────────────┘
                              ↓
                    ┌──────────────────────┐
                    │  Capa de Datos       │
                    │  - features.csv      │
                    │  - config.json       │
                    │  - feedback.db       │ ← NUEVO
                    │  - Snapshots version │
                    │  - Prompts v1/v2/v3  │
                    │  - Documentos RAG    │ ← NUEVO
                    └──────────────────────┘
                              ↓
                    ┌──────────────────────┐
                    │  Capa de Persistencia│
                    │  - Base de datos     │
                    │  - Almacenamiento    │
                    │  - Backups           │
                    └──────────────────────┘
```

### 9.2 Flujo de Datos (Detallado)

```
[1] USUARIO INICIA CONSULTA
USER INPUT: "Me gustan matemáticas y datos..."
    ↓
[2] API RECIBE Y VALIDA
/chat endpoint
Session ID creado
    ↓
[3] LLM: INTERPRETACIÓN
Input: {query, session_context}
Process: 
  - Extraer intereses, restricciones, prioridades
  - Generar perfil estructurado
  - Calcular confidence (0–1) de información
Output: {profile_json, confidence, missing_fields}
    ↓
[4] DECISIÓN: ¿INFORMACIÓN SUFICIENTE?
IF confidence >= 0.70 THEN → [6] EVALUACIÓN
ELSE → [5] SEGUIMIENTO
    ↓
[5] LLM: GENERACIÓN DE SEGUIMIENTO (NO INVASIVO)
Output: Pregunta contextual abierta
Retorna al usuario
Usuario responde (CICLO ITERATIVO)
Máximo 3–4 turnos
    ↓
[6] MOTOR DE EVALUACIÓN
Input: Perfil interpretado + features.csv + afinidad scores
Process:
  - Calcular afinidad para cada carrera
  - Evaluación multi-criterio
  - Generar indicadores de concordancia
Output: {rankings_by_carrera}
    ↓
[7] ORDENAMIENTO
Ordenar por indicador descendente
Top-3 seleccionadas
    ↓
[8] LLM: GENERACIÓN DE EXPLICACIONES
For each top_3:
  Input: {ranking_score, carrera, datos_verificables, perfil_usuario}
  Output: {explicación_personalizada}
    ↓
[9] ALMACENAMIENTO DE CONSULTA
INSERT INTO feedback_table:
  {timestamp, session_id, user_profile, ranking, explanations, 
   confidence_score}
    ↓
[10] RETORNO A USUARIO (FRONTEND)
JSON response:
  {
    "top_3": [
      {carrera, universidad, score, datos, explicación},
      ...
    ],
    "rag_available": [list of carreras con RAG habilitado]
  }
    ↓
[11] FRONTEND RENDERIZA
  - Top-3 con datos e explicaciones
  - Botón validación Likert
  - Links a RAG [si disponible]
    ↓
[12] USUARIO VALIDA
Escala Likert 1–5: "¿Qué tan útil fue la recomendación?"
    ↓
[13] ALMACENAMIENTO DE FEEDBACK
UPDATE feedback_table
SET user_validation = 4  ← escala Likert
    ↓
[14] CIERRE DE SESIÓN
Sesión almacenada completa para análisis posterior
```

---

## 10. Experiencia del Usuario

### 10.1 Escenario Principal (Demo)

**Persona:** Carlos, estudiante de 5º año, 17 años, usuario de prueba  
**Contexto:** Evaluando la plataforma CareerMatch

**Flujo:**

1. **Acceso:** Carlos entra a `careermatch-peru.demo` desde navegador (desktop o mobile)
2. **Bienvenida:** Pantalla muestra propuesta y botón "Comenzar conversación"
3. **Chat inicial:** Carlos escribe:
   > "Me gustan las matemáticas y el análisis de datos. Quiero ganar bien después de egresar, pero mi familia tiene presupuesto limitado."

4. **Procesamiento LLM:** Sistema interpreta → confidence = 0.68 (información incompleta)
5. **Seguimiento no invasivo:** Sistema responde:
   > "Entendido, te interesan análisis y matemáticas. ¿Hay alguna región del Perú donde prefieras estudiar, o tienes flexibilidad de ubicación? Algunos estudiantes lo dejan abierto para más opciones."

6. **Usuario responde:** "Flexible, preferiblemente Lima"
7. **Re-interpretación:** Confidence sube a 0.82
8. **Evaluación:** Motor genera ranking
9. **Respuesta (Top-3):**

   **🥇 Estadística — UNMSM (Lima)**
   - Indicador: 0.847/1.0
   - Ingreso: S/. 3,200/mes | Costo: S/. 1,200/año | Admisión: 18%
   - Explicación: "Te recomendamos Estadística en UNMSM porque tu afinidad con análisis es muy alta (85/100). Esta carrera ofrece uno de los mejores ingresos (S/. 3,200) y es accesible financieramente. Aunque selectiva (18%), es factible si tienes fortaleza en matemáticas."
   - [Ver detalles de carrera via RAG] (si disponible)

   **🥈 Ing. Civil — UNI**  
   **🥉 Ing. Sistemas — UPC**

10. **Validación:** Carlos hace click en escala Likert: "4 — Muy útil"
11. **Exploración RAG (opcional):** Si RAG disponible para Estadística:
    > "¿Qué cursos lleva Estadística?" 
    > RAG retorna: "En los primeros años: Cálculo I-II, Álgebra Lineal, Introducción a Programación. Luego: Probabilidad, Métodos Estadísticos, Análisis Multivariado..."

12. **Cierre:** Sistema confirma que feedback fue guardado. Carlos puede hacer otra consulta o salir.

### 10.2 Escenario Alternativo: Información Insuficiente

**Usuario:** Patricia, 16 años, consulta vaga
> "No sé qué estudiar, ayuda"

**Flujo:**
1. LLM interpreta → confidence = 0.25 (información muy insuficiente)
2. Sistema genera primer seguimiento:
   > "¡Claro, te ayudamos a explorar! Para empezar, ¿hay algo que te apasione o que disfrutes haciendo? (Por ejemplo: trabajar con números, ayudar a otros, creatividad, etc.)"

3. Patricia responde: "Me gusta dibujar y diseño"
4. Confidence sube a 0.45
5. Sistema genera segundo seguimiento:
   > "Perfecto. Entonces te interesan carreras creativas. ¿Qué importancia tiene el dinero después de egresar? ¿Es crítico que ganes mucho desde el principio, o tienes flexibilidad?"

6. Patricia: "Necesito ganar, mi familia tiene presupuesto limitado"
7. Confidence = 0.72 → GENERAR RANKING
8. Recomendaciones creativas con viabilidad económica

**Puntos clave:**
- Sistema no es invasivo (pregunta abierta, contextual)
- Aprende a cada turno (actualiza perfil)
- Máximo 4 turnos antes de recomendar con lo disponible
- Usuario nunca se siente forzado

### 10.3 Wireframe de Interfaz (Conceptual)

```
┌─────────────────────────────────────────┐
│  CareerMatch Perú  |  Demo v1.0         │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Chat Conversacional                     │
│                                         │
│ Bot: ¿Qué te interesa estudiar?        │
│                                         │
│ User: Me gustan matemáticas...         │
│                                         │
│ Bot: ¿Alguna región prefieres?         │
│ (⏳ procesando...)                      │
│                                         │
│ User: Lima                             │
│                                         │
│ Bot: Generando recomendaciones...      │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Top-3 Recomendaciones                   │
│                                         │
│ 🥇 Estadística — UNMSM                  │
│    Score: 8.47/10                      │
│    💰 Ingresos: S/. 3,200/mes           │
│    💵 Costo: S/. 1,200/año              │
│    📈 Admisión: 18%                     │
│    "Te recomendamos porque..."          │
│    [Ver detalles] [Comparar]            │
│                                         │
│ 🥈 Ing. Civil — UNI (...)              │
│ 🥉 Ing. Sistemas — UPC (...) │
│                                         │
│ ¿Qué tan útil fue? [1] [2] [3] [4] [5]│
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ [Opcional] Detalles de Carrera (RAG)    │
│                                         │
│ Pregunta: "¿Qué cursos lleva?"         │
│                                         │
│ Respuesta RAG:                          │
│ "Estadística comprende: Cálculo,       │
│  Álgebra, Probabilidad, Métodos Estad.│
│  [Fuente: Plan Curricular Oficial]"    │
└─────────────────────────────────────────┘
```

---

## 11. Motor de Scoring: Propuesta Inicial y Evolución

### 11.1 ¿Por Qué No es "Definitivo"?

El motor de scoring descrito aquí es una **propuesta técnicamente robusta pero abierta a evolución**. Las razones:

1. **Desconocimiento inicial sobre datos reales:** 
   - No sabemos hasta qué punto los 4 criterios (afinidad, ingresos, costo, facilidad) predicen satisfacción real del usuario
   - El feedback de usuarios (scores Likert) nos dirá si la propuesta funciona

2. **Comportamiento del LLM aún incierto:**
   - Los pesos generados por LLM pueden ser óptimos o no
   - Pueden ser predecibles o inconsistentes
   - El feedback histórico nos lo dirá

3. **Oportunidad de mejora obvia:**
   - Con suficientes datos de feedback (500+ casos), es posible entrenar un modelo propio
   - Ese modelo sería más específico al contexto peruano que LLM genérico
   - Pero necesitamos datos para hacerlo

### 11.2 Propuesta Inicial: Indicador Multi-Criterio Ponderado

**Estrategia base (a evaluar y posiblemente reemplazar):**

```
Indicador_de_Concordancia = f(Afinidad, Ingresos, Costo, Facilidad)

Opción A (Simple):
  Score = w₁×A + w₂×I + w₃×C + w₄×F
  donde w₁+w₂+w₃+w₄ = 1.0

Opción B (Geométrico):
  Score = (A^w₁ × I^w₂ × C^w₃ × F^w₄)
  
Opción C (Satisfacción mínima):
  Score = min(A, I, C, F)  ← garantiza que todos criterios OK

Opción D (Ponderado no lineal):
  Score = w₁×A² + w₂×I² + w₃×C² + w₄×F²  ← penaliza desbalance
```

**¿Cuál será elegida?**
Se evaluará durante desarrollo (15–28 jun) basado en:
- ¿Qué produce rankings más coherentes?
- ¿Cuál alinha mejor con feedback de usuarios de prueba?
- ¿Cuál es más interpretable y justificable?

### 11.3 Criterios Específicos

#### Afinidad [0,1]

**Definición:** Medida de alineación entre intereses del estudiante y carrera

**Métodos candidatos:**

| Método | Descripción | Ventajas | Desventajas |
|---|---|---|---|
| **Embeddings + Similitud** | Usar modelo embeddings (ej: multilingual BERT) para generar vectores de intereses y carrera, calcular similitud coseno | Flexible, semántico, robusto | Requiere modelo embedding, latencia |
| **Tabla de Mapeo** | Tabla estática: {interés} → {career_families_afines} con scores fijos | Rápido, auditable, sin ML | Manual, menos flexible |
| **LLM Score Directo** | El LLM genera directamente afinidad [0,1] para cada carrera | Contextual, simple | Menos reproducible, costo API |
| **Modelo Propio (Futuro)** | Entrenar modelo con datos feedback histórico | Optimizado para contexto | Requiere 500+ datos |

**Decisión en sprint Data:** Se evaluarán Embeddings + Tabla como fallback

**Ejemplo de aplicación:**

```
Intereses del usuario: ["matemáticas", "análisis de datos", "programación"]

Opción A (Embeddings):
  - Generar embedding de cada interés
  - Generar embedding de cada carrera
  - Calcular similitud coseno
  - Afinidad_Estadística = 0.87
  - Afinidad_Derecho = 0.12

Opción B (Tabla):
  - interés "matemáticas" → career_families con afinidad alta: 
    Ingeniería (1.0), Estadística (0.9), Física (0.85)
  - Promediar afinidades relevantes para cada carrera
  - Afinidad_Estadística = 0.88
  - Afinidad_Derecho = 0.05
```

#### Ingresos [0,1]

**Definición:** Ingreso promedio normalizado de egresados

```
income_score = MinMax(log1p(monthly_income))

donde:
- monthly_income: Ingreso promedio mensual de egresados (dato Ponte en Carrera)
- log1p: Log natural para reducir efecto de outliers
- MinMax: Normalizar a [0,1] usando min y max del dataset
```

#### Costo [0,1]

**Definición:** Costo invertido (mayor score = menor costo real)

```
cost_score = 1 - MinMax(log1p(annual_cost))

donde:
- annual_cost: Costo anual promedio (dato Ponte en Carrera)
- 1 - x: Invertir para que mayor score = mejor (menor costo)
- log1p: Reducir efecto de outliers
```

#### Facilidad [0,1]

**Definición:** Tasa de admisión normalizada

```
admission_score = MinMax(admission_rate)

donde:
- admission_rate: % de postulantes aceptados (dato Ponte en Carrera)
- MinMax: Normalizar a [0,1]
```

### 11.4 Validación y Posibles Mejoras

**Durante desarrollo (15–28 jun):**

Evaluar experimentalmente:
1. ¿Qué combinación de criterios produce rankings más coherentes?
2. ¿Funcionan mejor pesos fijos o dinámicos?
3. ¿Qué feedback recibimos en casos de prueba (escalas Likert)?

**Métricas a monitorear:**

| Métrica | Target | Acción si falla |
|---|---|---|
| Consistency | Mismo input → mismo ranking en 100% casos | Debuggear lógica |
| User validation (Likert) | ≥ 3.5/5 promedio en top-3 | Ajustar pesos o criterios |
| Explainability | Usuario entiende por qué se recomienda | Mejorar explicaciones LLM |
| Diversity | Top-3 no monopolizadas por 1–2 universidades | Revisar pesos si es deseable |

**Si durante demo se identifica enfoque superior:**
1. Documentar findings
2. Evaluar effort de transición
3. Planificar en Fase 1 (post-demo)
4. NO bloquear entrega demo actual

### 11.5 Roadmap de Evolución del Motor

```
DEMO (Junio—Agosto 2026)
└─ Propuesta inicial multi-criterio
   Pesos dinámicos por LLM
   Evaluación con 10–15 casos prueba
   
FASE 1 (Post-demo)
└─ Análisis de feedback de prueba
   Si resultados OK: mantener y refinar
   Si resultados pobres: explorar alternativas (ensembles, nuevos criterios)
   
FASE 2 (2027)
└─ Entrenar modelo propio con 500+ casos reales
   Modelo propio reemplaza o complementa LLM-scoring
   
FASE 3+ (2027+)
└─ Feedback loop cerrado
   Más datos → mejor modelo propio
   Posible A/B testing entre versiones
```

---

## 12. Módulo RAG Opcional

### 12.1 Propósito

Proporcionar acceso conversacional a detalles específicos de carreras recomendadas (qué es la carrera, qué cursos lleva, a qué se dedica, etc.), basado en documentos oficiales de las universidades.

**Ejemplo de valor:**
- Usuario recibe recomendación: "Ing. Industrial — UNI"
- Usuario pregunta: "¿Qué cursos lleva Ing. Industrial?"
- RAG retorna: "En primeros años: Matemática, Física, Dibujo Técnico. Cursos profesionales: Optimización, Gestión de Producción, Calidad..." [Fuente: Plan Curricular Oficial UNI]

### 12.2 Scope para Demo

**Limitación aceptable:** Inicialmente SOLO 3–5 carreras piloto tendrán RAG habilitado

**Por qué limitación:**
- Requiere búsqueda manual de PDFs de planes curriculares de universidades
- Requiere chunking y embedding (manual o con herramienta)
- Es trabajo manual que requiere verificación
- 3–5 es suficiente para demostrar viabilidad

**Carreras piloto sugeridas (a confirmar):**
1. Ing. Industrial — UNI
2. Estadística — UNMSM
3. Ciencias de Datos (si existe oficial)
4. Contabilidad — PUCP
5. Derecho — PUCP

### 12.3 Arquitectura de RAG

```
┌──────────────────────────────────────────┐
│ Documentos Oficiales (PDFs)              │
│ - Plan Curricular                        │
│ - Perfil Profesional                     │
│ - Descripción de Carrera                 │
└──────────────────────────────────────────┘
            ↓
┌──────────────────────────────────────────┐
│ Preprocessing                            │
│ - Extracción de texto de PDF            │
│ - Limpieza y normalización               │
│ - Chunking (fragmentos pequeños)         │
└──────────────────────────────────────────┘
            ↓
┌──────────────────────────────────────────┐
│ Embedding + Indexación                   │
│ - Generar embeddings de chunks           │
│ - Indexar en vector DB (FAISS, Pinecone,│
│   Chroma, o servicio cloud)              │
└──────────────────────────────────────────┘
            ↓
[Durante Interacción]
            ↓
┌──────────────────────────────────────────┐
│ Query del Usuario                        │
│ "¿Qué cursos lleva Ing. Industrial?"    │
└──────────────────────────────────────────┘
            ↓
┌──────────────────────────────────────────┐
│ Retrieval (RAG)                          │
│ - Embedder query                         │
│ - Search en vector DB top-k chunks       │
│ - Retorna chunks relevantes              │
└──────────────────────────────────────────┘
            ↓
┌──────────────────────────────────────────┐
│ Generación (LLM)                         │
│ - Input: {query, retrieved_chunks}      │
│ - LLM: Redacta respuesta coherente       │
│ - Output: "Ing. Industrial cursa:..."    │
└──────────────────────────────────────────┘
            ↓
┌──────────────────────────────────────────┐
│ Respuesta al Usuario                    │
│ + citación de fuente (documento oficial) │
└──────────────────────────────────────────┘
```

### 12.4 Tecnologías Agnósticas

**Embedding Model:**
- OpenAI API (cost)
- Google VertexAI Embeddings
- Hugging Face open-source (local)
- AWS SageMaker

**Vector Database:**
- FAISS (local, open-source)
- Pinecone (managed, cost)
- Weaviate (open-source)
- Chroma (simple, open-source)
- AWS OpenSearch Serverless

**LLM para Generación:**
- Mismo LLM principal (Gemini / GPT / etc.)

**Preprocessing:**
- PyPDF2 / pdfplumber (Python)
- Langchain / LlamaIndex (frameworks que abstraen detalles)

### 12.5 Implementación en Demo

**Fases:**

1. **Pre-Demo:** Obtener 3–5 PDFs de planes curriculares oficiales de universidades
2. **Durante Demo:** Generar embeddings, indexar localmente (FAISS o Chroma)
3. **Integración:** Cuando usuario selecciona carrera, mostrar botón "Detalles de carrera" si RAG disponible
4. **Fallback:** Si usuario pregunta detalle de carrera sin RAG, mostrar mensaje: "Información detallada no disponible aún para esta carrera"

### 12.6 Métricas de Éxito para RAG (Demo)

| Métrica | Target |
|---|---|
| Relevancia de chunks recuperados | ≥ 80% de chunks relevantes a query |
| Claridad de respuesta LLM | Usuario entiende respuesta (evaluación cualitativa) |
| Disponibilidad | 3–5 carreras con RAG funcional |
| Latencia | ≤ 2 segundos para respuesta RAG |

---

## 13. Feedback del Usuario y Base de Datos de Entrenamiento

### 13.1 Objetivo

Construir históricamente una **base de datos de feedback del usuario** que sirva como dataset para entrenar un modelo propio de recomendación en el futuro, reemplazando o complementando el LLM genérico.

### 13.2 Estructura de Feedback Capturado

Cuando usuario valida una recomendación, se almacena:

```json
{
  "session_id": "s_1234567890",
  "timestamp": "2026-07-15T14:32:00Z",
  
  "user_input": {
    "query": "Me gustan matemáticas y datos...",
    "region": "Lima",
    "budget_max": 6000,
    "institution_type_pref": null
  },
  
  "user_profile_interpreted": {
    "interests": ["matemáticas", "análisis de datos"],
    "salary_priority": 0.8,
    "cost_sensitivity": 0.9,
    "admission_tolerance": 0.5,
    "confidence_score": 0.82
  },
  
  "generated_weights": {
    "affinity": 0.40,
    "salary": 0.30,
    "cost": 0.20,
    "admission": 0.10
  },
  
  "ranking_generated": [
    {
      "rank": 1,
      "career": "Estadística",
      "institution": "UNMSM",
      "score": 0.847,
      "scores_by_criterion": {
        "affinity": 0.85,
        "salary": 0.82,
        "cost": 0.90,
        "admission": 0.60
      }
    },
    ...
  ],
  
  "user_validation": {
    "recommendation_liked": true,  ← Likert escala 1-5
    "validation_score": 4,
    "selected_career": "Estadística — UNMSM",
    "timestamp_validation": "2026-07-15T14:35:00Z",
    "notes": "Usuario comentó que le gustó mucho la carrera"
  }
}
```

### 13.3 Usos del Feedback Histórico

#### A. Evaluación Continua (Corto Plazo — Demo + Fase 1)

**Pregunta:** ¿Nuestro motor funciona?

```
Métrica: Score de validación promedio de Top-3
Target: ≥ 3.5 / 5.0

Si < 3.5:
  - Analizar qué criterios fallaron
  - ¿Fue afinidad? → ajustar método afinidad
  - ¿Fue costo? → revisar normalización
  - Iterar prompts o pesos base
```

#### B. Análisis de Patrones (Mediano Plazo — Fase 1)

**Preguntas:**
- ¿Qué perfiles de usuario tienden a validar más alto?
- ¿Hay carreras que consistentemente reciben validación baja?
- ¿Hay sesgos (ej: usuarios de cierta región siempre validan bajo)?

**Acciones:**
- Investigar si hay problema sistemático
- Ajustar criterios o ponderaciones por subgrupos

#### C. Entrenamiento de Modelo Propio (Largo Plazo — Fase 2, 2027+)

**Objetivo:** Reemplazar o complementar el motor genérico actual

**Con 500+ casos reales, se puede:**

1. **Regresión supervisada:**
   - Features: user_profile + weights + scores_by_criterion
   - Target: validation_score (continuo 1–5)
   - Modelo: Random Forest, XGBoost, Red Neuronal
   - Output: Predictor de qué tan satisfecho estará usuario

2. **Learning to Rank:**
   - Features: características de cada carrera
   - Target: ranking (user prefirió opción 1 sobre 2)
   - Modelo: LambdaMART, RankNet
   - Output: Ordenador de carreras optimizado para contexto peruano

3. **Clustering de usuarios:**
   - Segmentar por perfiles similares
   - Entrenar sub-modelos por cluster
   - Output: Recomendaciones personalizadas por tipo de perfil

### 13.4 Privacidad y Consentimiento

**Para Demo:**
- Usuarios saben que participan en prueba
- Datos claramente almacenados para mejora futura
- Consentimiento explícito al iniciar sesión

**Para Producción (Fase 2+):**
- GDPR/LGPD compliance
- Opción de opt-out de almacenamiento de feedback
- Anonimización después de N meses
- Política clara sobre uso de datos

### 13.5 Métricas de Feedback (Demo)

| Métrica | Target | Interpretation |
|---|---|---|
| **N de sesiones con validación** | ≥ 8/10 consultas validadas | Usuarios dan feedback |
| **Score promedio Likert (Top-3)** | ≥ 3.5/5.0 | Recomendaciones útiles |
| **Score promedio por criterio** | Variación > 0.5 | Criterios capturan variabilidad real |
| **Consistencia (mismo input → mismo output)** | 100% | Sistema determinista |
| **Tiempo de validación** | < 10 seg desde respuesta | UX fluido |

---

## 14. Datos y Modelos

### 14.1 Fuente de Datos

**Fuente Primaria:** Ponte en Carrera (MINEDU)  
**URL:** `https://ponteencarrera.minedu.gob.pe/pec-portal-web/`  
**Formato:** Excel descargable  
**Periodicidad Demo:** Una descarga inicial; actualizaciones manuales si es necesario  
**Periodicidad Producción:** Semanal mínimo

**Cobertura Demo:** ~100–150 combinaciones carrera-universidad  
**Cobertura Producción:** 1,500+ combinaciones

**Variables Capturadas:**

| Variable | Tipo | Rango | Descripción |
|---|---|---|---|
| career_family | Categórica | 10–15 valores | Familia de carrera |
| career | Texto | - | Nombre específico |
| institution | Texto | - | Universidad/Instituto |
| location | Categórica | 24 regiones | Región geográfica |
| institution_type | Categórica | {Universidad, Instituto} | Tipo |
| management_type | Categórica | {Público, Privado} | Tipo gestión |
| duration_years | Numérica | 3–7 años | Duración |
| annual_cost | Numérica | S/. 0–60,000 | Costo anual |
| admission_rate | Numérica | 0–100% | Tasa admisión |
| monthly_income | Numérica | S/. 800–10,000 | Ingreso egresados |
| scholarships_available | Booleana | {Sí, No} | Becas disponibles |

### 14.2 Procesamiento de Datos

**Pipeline:**

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
    - Facilitar auditoría y análisis de sensibilidad
    ↓
[4] Normalización
    - Log1p(income) → Min-Max [0,1]
    - Log1p(cost) → Min-Max → Invertir
    - Duration → Min-Max → Invertir
    - Admission_rate → Min-Max
    ↓
Features [0,1] listos para evaluación
```

### 14.3 Modelo de Lenguaje (LLM)

**Funciones:**

1. Interpretación de preferencias
2. Detección de información incompleta
3. Generación de preguntas seguimiento
4. Generación de explicaciones
5. (Opcional) Generación de pesos dinámicos

**Estrategia de Prompting:**

Detallada en PRD de Ingeniería de Prompts (documento futuro), pero incluye:

- Few-shot ejemplos de diferentes perfiles
- Structured output (JSON schema)
- Validación de pesos Σwᵢ = 1.0
- Chain-of-thought para explicaciones

**Ejemplo System Prompt Ilustrativo:**

```
Eres CareerMatch, asistente de orientación vocacional para estudiantes en Perú.

Tu rol es:
1. Interpretar intereses, restricciones y prioridades que expresa un estudiante
2. Detectar si hay información incompleta para buena recomendación
3. Generar preguntas contextual y conversacionales (NO invasivas) si falta info
4. Redactar explicaciones que justifiquen recomendaciones con datos

## Formato de salida (JSON):

Para interpretación:
{
  "interests": [...],
  "constraints": {...},
  "missing_info": [...],
  "confidence_score": 0.0–1.0,
  "suggested_follow_up": "Pregunta opcional"
}

Para explicación:
{
  "explanation": "Texto justificado con datos...",
  "data_cited": ["source1", "source2"]
}

## Importante:
- Si confidence < 0.70: SIEMPRE generar follow-up
- Follow-up debe ser abierto y contextual, no forzado
- NUNCA pedir información ya mencionada
- MÁXIMO 3–4 turnos antes de recomendar con lo disponible
```

### 14.4 Modelo de Afinidad (Método a Definir)

Ver sección 11.3 (Motor de Scoring). Será elegido durante sprint de Data basado en evaluación empírica.

---

## 15. Métricas de Éxito

### 15.1 Métricas Técnicas (Demo)

#### A. Pipeline de Datos

| Métrica | Target Demo |
|---|---|
| Completitud de variables | ≥ 95% |
| Validez de tipos | 100% conversiones exitosas |
| Reproducibilidad | 100% (mismo input → mismo output) |
| Latencia de carga | ≤ 5 minutos (demo scale) |

#### B. Motor de Evaluación

| Métrica | Target |
|---|---|
| Latencia de scoring | ≤ 1 segundo |
| Validez de scores | Todos ∈ [0,1] |
| Suma de pesos | Σwᵢ = 1.0 (si aplica) |
| Consistencia de ranking | 100% (determinista) |

#### C. LLM Layer

| Métrica | Target |
|---|---|
| Tasa error JSON | ≤ 5% |
| Validez de weights | ≥ 95% (sum = 1.0) |
| Latencia | ≤ 3 segundos |
| Coherencia de explicaciones | ≥ 85% (evaluación cualitativa) |

#### D. Sistema General

| Métrica | Target |
|---|---|
| Latencia end-to-end | ≤ 5 segundos |
| Disponibilidad | Best effort (demo) |
| Error rate | ≤ 2% |

### 15.2 Métricas de Usabilidad (Demo)

| Métrica | Target |
|---|---|
| Facilidad de uso (observación) | Usuario completa flujo sin problemas |
| Tiempo onboarding | ≤ 3 minutos hasta primer ranking |
| Claridad de explicaciones | Usuario entiende justificación |

### 15.3 Métricas de Evaluación (Demo)

| Métrica | Descripción | Target |
|---|---|---|
| **Satisfacción de usuario (Likert)** | % validaciones ≥ 3/5 | ≥ 70% |
| **Consistencia de recomendación** | ¿Ranking estable ante pequeñas variaciones? | ≥ 80% top-3 estable |
| **Reproducibilidad** | ¿Auditor puede reproducir decisión? | 100% |
| **Cobertura de criterios** | ¿Cada ranking justifica > 1 criterio? | 100% |

### 15.4 Métricas de Negocio (Largo Plazo — NO medibles en demo)

| Métrica | Descripción | Target Producción |
|---|---|---|
| **MAU (Monthly Active Users)** | Usuarios activos mensuales | 50,000 (año 1) |
| **NPS** | Net Promoter Score | ≥ 50 |
| **Satisfacción a 1 año post-ingreso** | % que reporta "buena decisión" | ≥ 70% |
| **Reducción en cambio de carrera** | % reducción en tasa cambio año 1 | ≥ 25% |
| **Alineación intereses-carrera** | Autorreporte post-ingreso | ≥ 7/10 |

---

## 16. Roadmap del Producto

### 16.1 Fases de Desarrollo

#### **Fase 0: Demo (Junio—Agosto 2026) ← ACTUAL**

**Duración:** 8 semanas (con soporte de herramientas de coding asistido)

**Objetivo:** Sistema funcional end-to-end que demuestre viabilidad

**Entregables:**
- ✅ Pipeline de datos operativo
- ✅ Motor de evaluación multi-criterio
- ✅ Integración LLM para interpretación + explicación
- ✅ Interfaz web conversacional (navegador)
- ✅ Almacenamiento de feedback de usuario
- ✅ RAG opcional (3–5 carreras piloto)
- ✅ 10–15 casos de evaluación completados
- ✅ Repositorio con código documentado

**Limitaciones Aceptadas:**
- 3–5 usuarios simultáneos máximo
- No hay ajuste fino del LLM
- Cobertura geográfica: Lima + zonas urbanas
- SLA no crítico

**Responsables:** Equipo 4 personas (Nikolai, Ángel, Fabiola, David)

#### **Fase 1: Beta / Estabilización (Septiembre 2026—Febrero 2027)**

**Objetivo:** Validación con usuarios reales, análisis de feedback, refinamiento

**Entregables:**
- Aumento a 100–500 usuarios
- Monitoreo en vivo
- Análisis de feedback (qué validaciones bajas, por qué)
- Iteración de prompts v2, v3
- Documentación de aprendizajes
- Decisión: ¿mantener motor actual o evolucionar?

**KPIs:**
- Likert promedio ≥ 3.5/5.0
- 500+ casos en base de datos feedback
- Identificados problemas sistemáticos (si hay)

#### **Fase 2: Escala Limitada (Marzo—Diciembre 2027)**

**Objetivo:** Validar modelo de negocio, escalar a 10,000 usuarios

**Entregables:**
- Infraestructura escalable
- Modelo de feedback operativo
- Posible modelo propio entrenado (si datos suficientes)
- Integración con 5+ instituciones educativas
- Análisis de impacto real en decisiones vocacionales

**KPIs:**
- 10,000 MAU
- NPS ≥ 40
- 2,000+ casos feedback histórico
- Evidencia de impacto (si es posible medir)

#### **Fase 3: Producción Abierta (2028+)**

**Objetivo:** Escala nacional, modelo de negocio establecido

**Entregables:**
- 50,000+ usuarios activos
- Modelo propio entrenado y operativo
- SLA 99%+
- Partnerships con universidades
- Portal para orientadores educativos

---

### 16.2 Timeline Detallado (Próximas 8 semanas)

```
SEMANA 1 (Jun 17 – Jun 23)
├─ Publicar repositorio GitHub
├─ Definir método cálculo Afinidad
├─ Diseño de DB para feedback
└─ Setup inicial de herramientas

SEMANA 2 (Jun 24 – Jun 30)
├─ Feature engineering completo
├─ Validación dataset
├─ Primeros tests del motor evaluación
└─ EDA preliminar

SEMANA 3 (Jul 1 – Jul 7)
├─ Motor evaluación v1 operativo
├─ SYSTEM_PROMPT v1 con ejemplos
├─ Primeros tests LLM integración
└─ Frontend skeleton

SEMANA 4 (Jul 8 – Jul 14)
├─ Integración LLM-Evaluación en notebook demo
├─ Almacenamiento de feedback funcional
├─ Frontend chat conversacional básico
└─ RAG preparación (obtención de PDFs piloto)

SEMANA 5 (Jul 15 – Jul 21)
├─ Frontend completo
├─ API endpoints /chat, /recommend, /feedback
├─ Diálogo iterativo (captura información incompleta)
├─ RAG integrado (3–5 carreras)
└─ Testing manual

SEMANA 6 (Jul 22 – Jul 28)
├─ Casos de evaluación (10–15 consultas prueba)
├─ Recolección de feedback Likert
├─ Análisis preliminar de validaciones
├─ Docker + deployment simple
└─ Documentación de código

SEMANA 7 (Jul 29 – Aug 4)
├─ Ajustes basados en feedback evaluación
├─ Pruebas de reproducibilidad
├─ Verificación de auditoría
├─ Video demo
└─ Informe preliminar

SEMANA 8 (Aug 5 – Aug 9)
├─ Última ronda de bugfixes
├─ Informe final
├─ Repositorio limpio y listo
├─ Presentación oral
└─ ENTREGA FINAL
```

---

## 17. Restricciones y Supuestos

### 17.1 Restricciones Técnicas

| Restricción | Impacto | Mitigación |
|---|---|---|
| **20 días de desarrollo** | Scope muy acotado, requiere priorización | Usar herramientas code-assisted (Claude, Codex, etc.) |
| **No hay app móvil nativa** | Solo navegador web (responsive pero no optimizado) | Aceptable para demo; producción posterior puede tener app |
| **Datos de Ponte en Carrera solo** | Limitado a Perú, solo empleo formal | Documentar como limitación; oportunidad de expansión |
| **Ingreso data refleja solo formalidad** | Subestima ingresos reales (40% economía informal) | Documentar, señalar en explicaciones |
| **Pesos dinámicos dependen de LLM** | Inconsistencia si LLM varia | Validación de suma pesos, fallback a pesos fijos |
| **RAG solo para 3–5 carreras** | No coverage completa | Plantea escalabilidad futura como feature |

### 17.2 Restricciones de Negocio

| Restricción | Impacto | Mitigación |
|---|---|---|
| **Demo, no producción abierta** | Usuarios limitados, sin compromiso de SLA | Comunicar claramente que es prototype |
| **Sin monetización aún** | No ingresos, proyecto investigativo | Definir modelo en Fase 2 |
| **Dependencia de datos MINEDU** | Si portal cambia, impacto desconocido | Monitorear actualizaciones, documentar cambios |

### 17.3 Supuestos Técnicos

| Supuesto | Justificación | Riesgo | Mitigación |
|---|---|---|---|
| **LLM funciona en español peruano** | LLMs modernos soportan español bien | MEDIO | Few-shot ejemplos contextualizados |
| **Interpretación preferencias ≥85% exactitud** | Tareas de clasificación de texto son estándar en LLMs | MEDIO | Validación con casos prueba |
| **Pesos generados son válidos** | LLM entiende constraint Σwᵢ=1.0 | BAJO | JSON schema validation |
| **Embedding models disponibles** | Modelos open-source o APIs existentes | BAJO | Comparar alternativas en desarrollo |
| **RAG es viable con 3–5 docs** | Chunking y embedding estándar | BAJO | POC rápido con herramientas existentes |

### 17.4 Supuestos de Negocio

| Supuesto | Justificación | Riesgo | Mitigación |
|---|---|---|---|
| **Estudiantes expresan preferencias en lenguaje natural** | Jóvenes 16+ hablan español naturalmente | BAJO | Interfaz accesible, ejemplos claros |
| **Feedback Likert es proxy de satisfacción real** | Métrica estándar en investigación UX | MEDIO | Comparar con entrevistas cualitativas |
| **Base de datos feedback → modelo futuro** | Suficientes datos historicizados → patterns | MEDIO | Recolectar mínimo 500 casos para viabilidad |
| **Datos oficiales inspirarán confianza** | Fuente MINEDU tiene legitimidad | BAJO | Pero requiere comunicación clara de limitaciones |

---

## 18. Recomendaciones para Mejora del PRD

Durante la actualización de este documento, identifiqué varias oportunidades para mejorar el PRD y fortalecer la propuesta:

### 18.1 **RECOMENDACIÓN 1: Separar explícitamente "Demo" de "Visión Futura"**

**Estado:** ✅ Ya implementado

**Descripción:** El PRD ahora tiene sección 6 (Scope: Demo vs. Producción) que clarifica qué es prototipo y qué es aspiración. Esto ayuda a evitar confusión y establece expectativas realistas.

**Beneficio:** Evaluadores entienden claramente el alcance y no penalizan por limitaciones aceptadas en una demo.

---

### 18.2 **RECOMENDACIÓN 2: Proponer estrategia de "Motor Adaptable" en lugar de "Motor Fijo"**

**Estado:** ✅ Ya implementado (sección 11)

**Descripción:** En lugar de presentar un motor de scoring como "definitivo", presentarlo como "propuesta inicial evaluable", con opciones claras para evolucionar.

**Beneficio:**
- Mayor honestidad intelectual
- Justifica mejor por qué es defensible (es transparente, auditable, preparado para cambios)
- Abre diálogo sobre alternativas sin parecer que no hay plan

---

### 18.3 **RECOMENDACIÓN 3: Incluir "Diálogo Iterativo" como Feature Diferenciador**

**Estado:** ✅ Ya implementado

**Descripción:** En lugar de asumir usuario tiene preferencias claras, permitir que el sistema pregunte iterativamente sin ser invasivo.

**Beneficio:**
- Más realista (muchos estudiantes no saben qué quieren)
- Diferencia CareerMatch de sistemas estáticos
- Demuestra sofisticación en interacción LLM

---

### 18.4 **RECOMENDACIÓN 4: Agregar Módulo RAG como "Opcional pero Viableversioned"**

**Estado:** ✅ Ya implementado (sección 12)

**Descripción:** Permitir que el usuario profundice en detalles de carrera recomendada. Inicialmente con 3–5 carreras.

**Beneficio:**
- Aumenta valor para usuario (no solo ranking sino información contextual)
- Demuestra capacidad técnica (RAG es tendencia 2024–2026)
- Camino claro para escala futura

---

### 18.5 **RECOMENDACIÓN 5: Diseñar "Feedback Loop" como Asset Estratégico**

**Estado:** ✅ Ya implementado (sección 13)

**Descripción:** No ver feedback solo como métrica, sino como **base de datos de entrenamiento futuro**. Esto abre posibilidad de entrenar modelo propio (vs. depender de LLM genérico).

**Beneficio:**
- Diferencia estratégica a largo plazo
- Explica por qué recolectar feedback es importante
- Propone roadmap creíble hacia independencia tecnológica
- Atrae a evaluadores: "esto no es un juguete, es investigación seria"

---

### 18.6 **RECOMENDACIÓN 6: Documentar Explícitamente "Graceful Degradation"**

**Descripción:** Si falta información, sistema aún recomienda con lo disponible (no fuerza al usuario a responder).

**Beneficio:**
- UX más resiliente
- Evita frustración si usuario no sabe algo
- Realista: los estudiantes real no tendrán claridad total

---

### 18.7 **RECOMENDACIÓN 7: Separar "Métricas Demo" de "Métricas Producción"**

**Estado:** ✅ Ya implementado (tablas con "Demo Target" vs "Producción Target")

**Descripción:** Diferentes targets para latencia, uptime, usuarios concurrentes, etc.

**Beneficio:**
- No penaliza demo por no tener SLA de producción
- Muestra crecimiento planificado

---

### 18.8 **RECOMENDACIÓN 8: Incluir "Tecnologías Agnósticas" Consistentemente**

**Estado:** ✅ Implementado en secciones clave

**Descripción:** Nunca comprometerse con tecnología específica (AWS vs GCP vs Azure vs local). Permitir múltiples opciones.

**Beneficio:**
- Flexibilidad de implementación
- Evita lock-in tecnológico
- Focus en concepto, no en herramientas

---

### 18.9 **RECOMENDACIÓN 9: Crear Tabla "Scope de Requisitos Funcionales"**

**Estado:** ✅ Implementada en sección 7

**Descripción:** F1.X y F2.X con columna "Scope" (Demo vs. Producción Futura)

**Beneficio:**
- Claridad de qué entra en MVP vs. roadmap
- Fácil referencia para equipo de desarrollo
- Comunica prioridades

---

### 18.10 **RECOMENDACIÓN 10: Documentar "Limitaciones Aceptadas" con Plan de Mitigación**

**Estado:** ✅ Implementado

**Descripción:** Cada restricción tiene fila de "Mitigación" explicando cómo se aborda

**Beneficio:**
- Muestra pensamiento crítico
- No aparenta que no hay problemas, sino que hay plan para resolverlos
- Genera confianza

---

### 18.11 **RECOMENDACIÓN 11: Agregar Sección de "Decisiones de Arquitectura"**

**Descripción (si no está):** Documentar por qué se eligió desacoplamiento LLM-Scoring, por qué pesos dinámicos, etc.

**Beneficio:**
- Justifica decisions técnicas
- Facilita discusión de trade-offs
- Base para futuras iteraciones

---

### 18.12 **RECOMENDACIÓN 12: Considerar Métrica de "Reproducibilidad" como Core**

**Estado:** ✅ Implementado

**Descripción:** Hacer que "auditor puede reproducir exactamente la decisión" sea métrica primaria, no secundaria.

**Beneficio:**
- Diferencia de LLM-only solutions (que son cajas negras)
- Atrae a evaluadores que valoran auditabilidad
- Necesario para contextos regulatorios (educación, finanzas)

---

## 19. Cambios Principales desde PRD v1.0 → v1.1

| Aspecto | v1.0 | v1.1 | Razón |
|---|---|---|---|
| **Scope** | Producción a escala | Demo + Roadmap a producción | Clarificar alcance real |
| **Motor Scoring** | Propuesta definitiva | Propuesta inicial + evaluable | Mayor honestidad intelectual |
| **Feedback Usuario** | Métrica de evaluación | Asset estratégico (dataset futuro) | Alinear con visión largo plazo |
| **RAG** | No mencionado | Módulo opcional funcional | Agregar valor, demostrar capability |
| **Diálogo Iterativo** | Mencionado | Feature central diferenciador | Más realista y sofisticado |
| **Métricas** | Genéricas | Específicas por fase (demo/producción) | Expectativas claras |
| **Plataforma** | Web + Mobile | Solo Web (responsive, no app nativa) | Realidad de 20 días dev |
| **Tecnología** | Específica (algunos lugares) | Agnóstica siempre | Flexibilidad implementación |

---

## 20. Referencias

Al-Dossari, H., Abu Nughaymish, F., Al-Qahtani, Z., Alkahlifah, M., & Alqahtani, A. (2020). A machine learning approach to career path choice for information technology graduates. Engineering, Technology & Applied Science Research, 10(6), 6589–6596.

Bender, E. M., Gebru, T., McMillan-Major, A., & Shmitchell, S. (2021). On the dangers of stochastic parrots: Can language models be too big? En Proceedings of the 2021 ACM Conference on Fairness, Accountability, and Transparency (pp. 610–623).

Bommasani, R., Hudson, D. A., Adeli, E., Altman, R., Arora, S., et al. (2021). On the opportunities and risks of foundation models. arXiv preprint arXiv:2108.07258.

Esteban, A., Zafra, A., & Romero, C. (2020). Helping university students to choose elective courses by using a hybrid multi-criteria recommendation system with genetic optimization. Knowledge-Based Systems, 194, 105385.

Ezz, M., & Elshenawy, A. (2020). Adaptive recommendation system using machine learning algorithms for predicting student's best academic program. Education and Information Technologies, 25(4), 2733–2746.

Gao, Y., Xiong, Y., Gao, X., Jia, K., Pan, J., Bi, Y., Dai, Y., Sun, J., Wang, M., & Wang, H. (2024). Retrieval-augmented generation for large language models: A survey. arXiv preprint arXiv:2312.10997.

Huang, C., Yu, T., Xie, K., Zhang, S., Yao, L., & McAuley, J. (2024). Foundation models for recommender systems: A survey and new perspectives. arXiv preprint arXiv:2402.11143.

Lewis, P., Perez, E., Piktus, A., Schwenk, H., Schwettmann, D., Yih, W. T., & Kiela, D. (2020). Retrieval-augmented generation for knowledge-intensive NLP tasks. Advances in Neural Information Processing Systems, 33, 9459–9474.

---

**Documento Preparado Por:** Especialista en Product Requirements, en colaboración con equipo CareerMatch Perú  
**Fecha de Publicación:** Junio 2026  
**Versión:** 1.1 (Actualizada)  
**Estado:** Listo para Desarrollo  
**Próxima Iteración:** v1.2 (Post-demo, basada en learnings)

---

## Apéndice A: Glosario de Términos

| Término | Definición |
|---|---|
| **Afinidad** | Medida [0,1] de alineación entre intereses del estudiante y perfil académico de una carrera |
| **Motor de Evaluación** | Sistema que calcula indicadores de concordancia para ranking de carreras |
| **Pesos Dinámicos** | Coeficientes generados por LLM en cada sesión basados en preferencias del usuario |
| **Reproducibilidad** | Capacidad de obtener exactamente los mismos resultados con mismos datos y configuración |
| **Versionado** | Práctica de mantener snapshots históricos de datos, prompts y configuración |
| **RAG** | Retrieval-Augmented Generation: técnica de generar respuestas consultando documentos externos |
| **Graceful Degradation** | Sistema que sigue funcionando con información parcial en lugar de fallar |
| **Feedback Loop** | Ciclo de recolectar validaciones de usuario para mejorar futuras recomendaciones |
| **Scope** | Alcance definido de funcionalidades y limitaciones |
| **KPI** | Key Performance Indicator: métrica de éxito de negocio |

---
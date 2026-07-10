---
title: "Frontmatter Schema"
author: "@spark-match/devops"
date: 2026-07-09
updated: 2026-07-09
tags:
  - admin
  - area/conventions
  - topic/schema
  - status/published
audience:
  - members
status: published
aliases:
  - "frontmatter-schema"
---

# 📋 Frontmatter Schema — Schema de Metadatos

> Schema enforced para todos los documentos del vault. **Cualquier doc nuevo debe cumplir este schema mínimo**.

---

## 🎯 Schema mínimo obligatorio

```yaml
---
title: "<título del documento>"
author: "@github-user"
date: YYYY-MM-DD
updated: YYYY-MM-DD
tags:
  - area/<X>
  - status/<draft|published|archived>
audience: []  # list de equipos destinatarios
status: "<draft|published|archived>"
---
```

> **Validación**: el CI del repositorio falla si falta alguno de estos campos.

## 📐 Schema completo (recomendado)

```yaml
---
# --- Identificación ---
title: ""
author: ""
date: YYYY-MM-DD
updated: YYYY-MM-DD

# --- Clasificación ---
tags: []              # list de strings
audience: []          # list de equipos
status: ""            # draft | published | archived

# --- Relación ---
related: []           # list de wikilinks
aliases: []           # list de alias

# --- Tipo de documento ---
template: ""          # nombre del template usado

# --- Campos específicos por tipo ---
# ADR:
adrNumber: ""         # "001", "002", etc.

# How-to:
estimatedTime: 0      # minutos
prerequisites: []     # list de strings

# Postmortem:
severity: ""          # Crítica | Alta | Media | Baja
incidentDate: ""
duration: ""
detectedBy: ""
resolvedBy: ""

# Research:
investigator: ""
startDate: ""
endDate: ""
hypothesis: ""
---
```

## 📊 Tipos de campos Dataview

| Tipo | Sintaxis | Ejemplo |
|---|---|---|
| `string` | `"texto"` | `title: "ADR-001"` |
| `number` | `42` | `estimatedTime: 60` |
| `date` | `YYYY-MM-DD` | `date: 2026-07-09` |
| `datetime` | `YYYY-MM-DD HH:MM` | `date: 2026-07-09 14:30` |
| `boolean` | `true/false` | `isPublic: true` |
| `list` | `[item1, item2]` | `tags: [area/decisions]` |
| `link` | `[[wikilink]]` | `related: ["[[ADR-001]]"]` |

## ⚠️ Validaciones

### ✅ Debe tener

- [ ] Campo `title` no vacío
- [ ] Campo `author` con formato `@usuario`
- [ ] Fecha `date` en formato ISO 8601
- [ ] Al menos 1 tag `area/<X>`
- [ ] Al menos 1 tag `status/<X>`
- [ ] Campo `status` con valores válidos (draft/published/archived)

### ❌ Errores comunes

| Error | Cómo evitarlo |
|---|---|
| Frontmatter con comillas desparejadas | Validar con `yaml-lint` en CI |
| Tag con mayúsculas | Usar siempre lowercase (`area/decisions` no `Area/Decisions`) |
| Tag sin prefijo (excepto `moc`, `adr`, etc.) | `topic/aws` no `aws` |
| Fecha mal formato | Usar `YYYY-MM-DD` estrictamente |
| Wikilinks con caracteres rotos | Usar `[[nombre-archivo]]` con kebab-case |

## 🔍 Validación en CI

El CI del repo ejecuta `scripts/validate-frontmatter.sh` que:

1. Verifica que todos los `.md` con frontmatter cumplan el schema mínimo
2. Reporta errores y warnings (no detiene la build, advertencias)
3. Genera reporte en `dist/frontmatter-validation.json`

```bash
# Ejecutar localmente
make validate-frontmatter
```

## 📚 Referencia rápida

### Por área

```yaml
# ADR (decisions/)
tags: [area/decisions, status/draft]
status: draft

# How-to (guides/)
tags: [area/guides, status/draft]
status: draft

# Postmortem (postmortems/)
tags: [area/postmortems, status/draft]
severity: Media  # o "Crítica", "Alta", "Baja"

# Research (research/)
tags: [area/research, status/in-progress]
status: in-progress  # o "completed", "abandoned"

# MOC (cualquier carpeta)
tags: [moc, area/<X>, status/published]
status: published
```

### Por equipo

```yaml
# Para backend-devs
audience: [backend-devs, tech-leads]

# Para frontend-devs
audience: [frontend-devs, tech-leads]

# Para todo el equipo
audience: [members]
```

## 🔗 Wikilinks relacionados

- [[00 - Home]]
- [[admin/obsidian-conventions]]
- [[admin/dataview-queries]]

## 📚 Referencias externas

- [Obsidian — YAML front matter](https://help.obsidian.md/Advanced+topics/YAML+front+matter)
- [JSON Schema — especificación](https://json-schema.org/)

---

> **Mantenedor**: `@spark-match/devops`
> **Status**: 🟢 published
> **Próxima revisión**: 2026-08-09 (1 mes)

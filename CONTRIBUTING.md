# 🤝 Guía de Contribución — Spark Match Knowledge Base

> Cómo añadir, editar y mantener documentos en este repositorio.

## 📌 Antes de empezar

- Lee el [`README.md`](./README.md) para entender la estructura.
- Revisa el [`INDEX.md`](./INDEX.md) para ver qué ya existe y evitar duplicados.
- Si tu doc es **cross-team** (afecta a varios equipos), considera abrir primero un **RFC** en [`decisions/`](./decisions/) antes de escribir el doc final.

## 🌿 Flujo de trabajo

### 1. Crear una rama

Usa prefijos descriptivos:

| Tipo | Prefijo | Ejemplo |
|---|---|---|
| Nuevo documento | `docs/` | `docs/how-to-deploy-lambdas` |
| Corrección/edición | `fix/` | `fix/typos-readme` |
| Refactor de docs existentes | `refactor/` | `refactor/restructure-architecture` |
| Mejora de tooling/config | `chore/` | `chore/update-codeowners` |

```bash
git checkout main
git pull
git checkout -b docs/<nombre-descriptivo-en-kebab-case>
```

### 2. Ubicar el documento

Coloca tu archivo en la carpeta correcta:

| Si tu doc es... | Carpeta |
|---|---|
| Un paso a paso / tutorial / how-to | `guides/` |
| Un diseño técnico o diagrama | `architecture/` |
| Una decisión cross-team (ADR) | `decisions/` |
| Una investigación o comparación | `research/` |
| Un análisis de incidente | `postmortems/` |
| Una plantilla reutilizable | `templates/` |
| Material de onboarding | `onboarding/` |

**¿No encaja?** Abre un issue o pregunta en `#spark-match` antes de crear una nueva carpeta.

### 3. Usar la plantilla

Usa la plantilla apropiada:

- ADR → [`templates/adr.md`](./templates/adr.md)
- Postmortem → [`templates/postmortem.md`](./templates/postmortem.md)
- Investigación → [`templates/research.md`](./templates/research.md)
- How-to → empieza directamente con `## Título` + `## Pasos`

### 4. Front-matter (opcional pero recomendado)

Añade metadatos al inicio en YAML:

```markdown
---
title: Cómo desplegar Lambdas con SAM
author: @angel-hincho
date: 2026-07-04
tags: [aws, lambda, sam, deploy, backend]
audience: [backend-devs, devops]
status: published
related:
  - architecture/backend-overview.md
---
```

Campos:

- `title`: título del documento
- `author`: tu usuario de GitHub
- `date`: fecha de creación (YYYY-MM-DD)
- `tags`: lista de tags para búsqueda
- `audience`: equipos a los que va dirigido
- `status`: `draft` | `published` | `archived`
- `related`: enlaces a docs relacionados

### 5. Actualizar el INDEX

Si es un documento nuevo:

1. Abre [`INDEX.md`](./INDEX.md)
2. Añade una entrada en la sección de la carpeta correspondiente
3. Usa este formato:

```markdown
| [Cómo desplegar Lambdas con SAM](./guides/deploy-lambdas-sam.md) | @angel-hincho | 2026-07-04 | backend-devs, devops |
```

### 6. Abrir el Pull Request

```bash
git add .
git commit -m "docs(guides): add how-to deploy Lambdas with SAM"
git push -u origin docs/deploy-lambdas-sam
gh pr create \
  --title "docs(guides): Cómo desplegar Lambdas con SAM" \
  --body "Añade how-to paso a paso para desplegar Lambdas con SAM. Incluye troubleshooting y links a docs oficiales."
```

**El PR template se autocompletará** con la información que Github te pida.

### 7. Esperar revisión

- CODEOWNERS del área te será asignado automáticamente como reviewer
- Si no tienes respuesta en 2 días, menciona al equipo en el PR
- Tras al menos 1 aprobación, podrás mergear

## ✅ Checklist antes de pedir review

- [ ] El documento está en la carpeta correcta
- [ ] Tiene front-matter completo (si aplica)
- [ ] Los links internos funcionan (probados localmente o con `markdown-link-check`)
- [ ] No hay typos evidentes (usar Grammarly o similar)
- [ ] El `INDEX.md` está actualizado
- [ ] He leído el documento desde el punto de vista de un陌生的 (extraño al tema) — ¿se entiende?

## 🚫 Qué NO hacer

- ❌ Subir documentos con **información sensible** (claves, tokens, datos personales)
- ❌ Duplicar docs que ya existen — actualízalos en su lugar
- ❌ Mezclar temas no relacionados en un solo archivo
- ❌ Hacer commits directos a `main` (las protecciones lo impedirán)
- ❌ Subir archivos binarios pesados (PDFs > 5MB, imágenes > 1MB) — usa S3 + enlace
- ❌ Plagiar contenido sin citar la fuente

## 🔒 Seguridad y secretos

Si descubres un secreto expuesto en un PR:

1. **No lo commitees visiblemente** — usa placeholders como `<SECRET_AQUI>`
2. **Notifica inmediatamente** a `@spark-match/devops`
3. El secreto será rotado y el historial limpiado con `git-filter-repo`

## 🧹 Mantenimiento

### Documentos obsoletos

Si un documento ya no es válido:

1. Cambia el `status` en front-matter a `archived`
2. Añade una nota al inicio: `> ⚠️ ARCHIVADO: <razón>. Ver <doc-nuevo> en su lugar.`
3. No borres el archivo — puede servir de referencia histórica

### Renombrar o mover

Para renombrar o mover un documento:

1. Crea un PR que haga el cambio
2. Incluye un redirect en `INDEX.md` si el documento es referenciado frecuentemente
3. Actualiza todos los enlaces internos al nuevo path

### Limpieza periódica

Cada trimestre, `@spark-match/devops` revisará:

- Documentos con `status: draft` por más de 90 días
- Documentos con links rotos (CI automático)
- Carpetas vacías o subutilizadas

## 🆘 ¿Necesitas ayuda?

- Abre un **issue** con la etiqueta `question`
- Pregunta en `#spark-match` (canal de Slack/Discord interno)
- Etiqueta a `@spark-match/devops` si es urgente

---

> **Recuerda**: si lo aprendiste, alguien más lo necesita aprender también. Documentar es regalar tiempo a tu yo futuro y a tus compañeros. 🌟
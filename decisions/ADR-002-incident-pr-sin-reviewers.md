---
title: "ADR-002: Incident — PRs mergeados sin CODE OWNER review"
author: "@ahincho"
date: 2026-07-05
tags: [decisions, governance, codeowners, incident]
audience: [product-owners, devops, members]
status: published
related:
  - REVIEW_PERMISSIONS.md
---

# ADR-002: Incident — PRs mergeados sin CODE OWNER review

## Contexto

El PR #3 ("docs: reglas de negocio del agente + ADR-001 backend híbrido") se mergeó
el 2026-07-05 a las 20:58 UTC sin reviewers automáticos, a pesar de que la branch
protection del repo requiere 1 aprobación de CODE OWNER. Este ADR documenta:

- La causa raíz del problema
- La remediación aplicada
- Las lecciones aprendidas

## Causa raíz

El archivo `.github/CODEOWNERS` referenciaba los teams `@spark-match/product-owners`
y `@spark-match/devops` como CODE OWNERS de varios paths. Sin embargo, estos teams
**NO estaban agregados como colaboradores del repo `00-knowledge-base`**.

La API de CODEOWNERS reportó 12 errores "Unknown owner" con el mensaje:
> "make sure the team @spark-match/X exists, is publicly visible, and has write access to the repository"

Como los CODE OWNERS no eran válidos, GitHub:
1. NO asignó reviewers automáticos al PR
2. NO bloqueó el merge con el CODE OWNER check (porque no había CODE OWNER que chequear)
3. Permitió que el admin (`ahincho`) mergeeara con bypass

### Por qué funcionaba en 08-deep-agent pero no aquí

El CODEOWNERS de `08-deep-agent` referencia los mismos teams (`product-owners`,
`devops`), pero ese repo SÍ los tiene agregados como colaboradores. Por eso ahí
funciona correctamente y aquí no.

### Por qué mi diagnóstico inicial fue incorrecto

Inicialmente pensé que el problema era la privacidad `closed` de los teams (la doc
de GitHub dice que los CODE OWNERS teams deben ser "publicly visible"). El usuario
intentó cambiar la privacidad a `public` desde la UI, pero el cambio no se aplicó
(varias razones posibles: la API REST no permite el cambio en GitHub Free plan, o
el cambio requiere otra configuración que no identifiqué).

El diagnóstico real se obtuvo al comparar los CODEOWNERS errors de ambos repos:
- `08-deep-agent`: 3 errores (solo blockquotes `>`)
- `00-knowledge-base`: 15 errores (3 blockquotes + 12 unknown owners)

El endpoint `orgs/{org}/teams/{team}/repos/{repo}` reveló que `product-owners` y
`devops` tienen acceso a 8 y 2 repos respectivamente, pero **NO a `00-knowledge-base`**.

## Remediación aplicada

1. **Agregados los teams faltantes como colaboradores** del repo:
   - `product-owners` con acceso `push` (ya estaba en 8 repos, faltaba este)
   - `devops` con acceso `push` (ya estaba en 2 repos, faltaba este)

2. **Resultado**: los CODEOWNERS errors bajaron de **15 a 3** (solo los blockquotes `>`)

3. **Validación**: el PR #5 abrió un PR contra `main` y GitHub **SÍ asignó el team
   `devops` como reviewer automático** (`requested_teams: [{name: "devops"}]`,
   `mergeStateStatus: BLOCKED`, `reviewDecision: REVIEW_REQUIRED`)

4. **Pendiente**: refinar el CODEOWNERS para quitar los 3 blockquotes `>` (cosmético).
   Esto se hará en un PR posterior que pueda ser aprobado por `product-owners`.

## Consecuencias

### Positivas

- CODEOWNERS ahora funciona en este repo
- PRs futuros tendrán reviewers automáticos
- Se documenta el incidente para prevenir recurrencia

### Negativas

- El PR #3 quedó mergeado sin revisión formal (irreversible sin revert)
- El CODEOWNERS de la rama `docs/SDD` (paralela) sigue con el código antiguo

### Mitigaciones

- Pull request template exige CODE OWNER review (configurado en branch protection)
- CODEOWNERS errors se pueden monitorear con: `gh api repos/.../codeowners/errors`
- Tests futuros pueden validar que los CODE OWNERS están bien configurados

## Lecciones aprendidas

1. **CODEOWNERS requiere que los teams tengan acceso al repo**, no solo que existan
2. **El error "Unknown owner" tiene 3 causas posibles**: no existe, no es público, o no tiene acceso
3. **Comparar CODEOWNERS errors entre repos similares** ayuda a identificar el problema
4. **El endpoint `orgs/{org}/teams/{team}/repos` es la mejor forma de verificar acceso**
5. **La doc de GitHub es ambigua**: "publicly visible OR has explicit access" — el "OR" no se enfatiza

## Referencias

- PR #3: https://github.com/spark-match/spark-match-00-knowledge-base/pull/3
- PR #5 (cerrado): https://github.com/spark-match/spark-match-00-knowledge-base/pull/5
- `REVIEW_PERMISSIONS.md` — auditoría de permisos
- 08-deep-agent CODEOWNERS — referencia funcional
- [GitHub Docs — About code owners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)

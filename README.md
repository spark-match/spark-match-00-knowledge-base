# 📚 Knowledge Base — Spark Match

> Repositorio compartido de conocimiento para todos los equipos de **Spark Match**.
> Guías, decisiones, investigación, postmortems, plantillas y onboarding en un solo lugar.

## 🎯 ¿Qué encontrarás aquí?

| Carpeta | Contenido | Audiencia |
|---|---|---|
| 📖 [`guides/`](./guides/) | How-tos, tutoriales paso a paso, procedimientos operativos | Todos |
| 🏗️ [`architecture/`](./architecture/) | Documentos de diseño, diagramas, decisiones técnicas profundas | Arquitectos, leads |
| ✅ [`decisions/`](./decisions/) | Decisiones cross-team (no bounded-context). ADRs organizacionales | Leads, PMs |
| 🔬 [`research/`](./research/) | Investigaciones, papers, spikes, prototipos, comparativas | Todos |
| 🚨 [`postmortems/`](./postmortems/) | Análisis de incidentes y aprendizaje organizacional | DevOps, leads |
| 📋 [`templates/`](./templates/) | Plantillas reutilizables (ADR, postmortem, RFC, RFC, diseño) | Todos |
| 🌱 [`onboarding/`](./onboarding/) | Material de bienvenida para nuevos miembros | Nuevos miembros |

📑 **Catálogo completo**: [`INDEX.md`](./INDEX.md)

## ✍️ ¿Cómo contribuir?

Este repositorio está abierto a **todos los miembros de Spark Match** para compartir conocimiento.

### Flujo rápido (TL;DR)

1. Crea una rama: `git checkout -b docs/<tu-aporte>`
2. Añade/edita el documento en la carpeta apropiada
3. Actualiza el [`INDEX.md`](./INDEX.md) si es un documento nuevo
4. Abre un **Pull Request**
5. CODEOWNERS del área te revisará (o `@spark-match/devops` si es general)
6. Tras aprobación, mergea 🚀

📘 **Guía detallada**: [`CONTRIBUTING.md`](./CONTRIBUTING.md)

## 🧭 Reglas de oro

> 1. **Documenta lo que aprendiste**, no solo lo que funcionó.
> 2. **Cita fuentes** cuando tomes prestado de internet, papers u otros repos.
> 3. **Mantén el `INDEX.md` actualizado** — es el mapa del repositorio.
> 4. **Un documento = un tema.** Si tu doc cubre tres cosas, divídelo.
> 5. **Revisa antes de pedir review.** Ortografía, links, formato.

## 🔍 Búsqueda rápida

```bash
# Buscar por palabra clave en todo el repo
grep -ri "bedrock" --include="*.md" .

# Buscar por tag en front-matter
grep -r "^tags:" --include="*.md" . | grep "aws"
```

O usa la búsqueda nativa de GitHub en [spark-match/00-knowledge-base/search](https://github.com/spark-match/spark-match-00-knowledge-base/search).

## 🏛️ Estructura del repo

```
00-knowledge-base/
├── README.md              ← este archivo
├── CONTRIBUTING.md        ← guía detallada de contribución
├── INDEX.md               ← catálogo completo
│
├── guides/                ← how-tos paso a paso
├── architecture/          ← documentos de diseño
├── decisions/             ← ADRs cross-team
├── research/              ← investigaciones y prototipos
├── postmortems/           ← análisis de incidentes
├── templates/             ← plantillas reutilizables
├── onboarding/            ← material de bienvenida
│
└── .github/
    ├── CODEOWNERS         ← ownership por carpeta
    └── pull_request_template.md
```

## 👥 ¿Quién mantiene esto?

- **CODEOWNERS por defecto**: `@spark-match/devops`
- **CODEOWNERS por carpeta**: definido en `.github/CODEOWNERS`
- **PRs abiertos** se notifican automáticamente al equipo correspondiente

## 📜 Licencia

MIT — ver [`LICENSE`](./LICENSE).

---

> _"Si no está documentado, no existe."_ — Dicho popular del equipo Spark Match 🌟
<!-- codeowners-validation-test: este comentario se revertirá después de validar -->

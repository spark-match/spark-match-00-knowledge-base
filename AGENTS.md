# AGENTS.md — Spark Match Knowledge Base (00)

> Working agreement for AI agents (and humans) contributing to this repo.
> Last updated: 2026-07-30 (PR-#80: workflow formalization)

## Repo at a glance

- **Type:** Documentation-only (Markdown). No code, no build, no tests.
- **Branching model:** `main` (production) + `dev` (integration).
  Feature PRs go to `dev`; sync PRs promote to `main`.
- **CODEOWNERS:** `/decisions/`, `/onboarding/`, `/postmortems/`,
  plus root `README.md` / `CONTRIBUTING.md` / `LICENSE` —
  `@spark-match/product-owners`. Everything else —
  `@spark-match/tech-leads`. See `.github/CODEOWNERS` for the
  full per-path list.
- **License:** MIT (see `LICENSE`).
- **Tooling:** None. No CI, no linters, no coverage gates. The
  CI gate is *human review* via CODEOWNERS.

## Hard quality gates (cannot be relaxed)

| Gate | Mechanism |
|---|---|
| At least 1 CODEOWNER approval per PR | Branch protection (dev + main) |
| `INDEX.md` updated when a doc is added | Manual review (checklist in PR template) |
| Front-matter complete (if used) | Manual review |
| `status:` field reflects reality (published / draft / archived) | Manual review |

The first row is the only **blocking** gate. The other three are
non-blocking but checked in the PR template.

## Local guardrails (run before pushing)

There are **no local tests or builds**. Pre-PR checklist:

```text
- [ ] Document is in the correct folder (see README § "Estructura")
- [ ] If new doc, INDEX.md updated
- [ ] If new doc, .gitkeep removed (or kept if folder is shared)
- [ ] Filename is kebab-case lowercase (see § "Naming convention" below)
- [ ] Front-matter present if the doc is "real" (vs. a stub)
- [ ] status: field set correctly
- [ ] Links to other docs in this repo use relative paths
- [ ] If new CODEOWNERS path, add it to .github/CODEOWNERS
```

If your doc covers three topics, **split it into three docs**. One
doc = one topic (per README § "Reglas de oro").

### Naming convention

- **Folders**: kebab-case lowercase, descriptive
  (`docs/SDD/`, `decisions/`, `templates/`, `guides/`,
  `architecture/`, `research/`, `postmortems/`, `onboarding/`).
- **Files**: kebab-case lowercase (`.md` extension). Use
  `nnn-` numeric prefix only for **ordered** sequences
  (e.g. `001-create-foo.md`); for SDD we use semantic names
  (`prd.md`, `requirements.md`, `design.md`, `architecture.md`).
- **ADRs**: `ADR-NNN-kebab-case-name.md` (3 digits, sequential).
- **PR title**: Conventional Commits — `type(scope): summary`
  (e.g. `docs(decisions): add ADR-004 pg`).

## Workflow

1. **Branch off `dev`** (not `main`): `git checkout dev && git
   checkout -b <type>/<scope>`. Allowed prefixes: `docs/`,
   `fix/`, `refactor/`, `chore/`.
2. Add or edit the doc. If a new doc, add a row to `INDEX.md`
   in the corresponding section.
3. **Run the pre-PR checklist** above.
4. **Push branch**, open a **PR to `dev`** (not `main`).
5. CODEOWNERS auto-assigned. Wait for at least 1 approval.
6. **Merge with squash** (ruleset enforces this).
7. **Dev → main sync** is a dedicated chore PR
   (`chore(sync): dev -> main (PR #NN ...)`). Each sprint ends
   with a sync PR. The sync PR **admin-bypasses** the review
   requirement because:
   - Underlying feature PRs already passed CODEOWNER review on
     `dev`.
   - The sync adds no new content, just rebases dev's HEAD onto
     main.
   - Without admin-bypass, we'd need a CODEOWNER to approve
     every sync, which is operationally noisy.

## Out of scope for agents

- Force-push to `main` or `dev`. Force-push to your own branch
  is OK to incorporate PR review feedback.
- Skipping CODEOWNER review via admin-bypass on **feature** PRs
  (sync PRs are the only exception).
- Modifying the LICENSE.
- Renaming top-level folders (this requires team discussion).

## Sprint history

- **Sprint 1** (2026-07-04, governance bootstrap): PRs #1-#13
  established the team permissions, CODEOWNERS, ADR-001, ADR-002
  and the first incident postmortem. Repo went from empty to
  8 governance/onboarding/ADR docs.
- **Sprint 2** (2026-07-29, audit cleanup): PR #17 created the
  4 missing folders with stub READMEs, archived ADR-001
  (Python/Lambda hybrid superseded by PR #62 of the backend),
  fixed INDEX.md (added ADR-002 + SDD section + stats). This
  PR (#80) adds the workflow doc you're reading now.

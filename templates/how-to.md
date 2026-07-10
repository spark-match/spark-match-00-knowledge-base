---
title: "How-to: {{title}}"
author: "{{author}}"
date: <% tp.date.now("YYYY-MM-DD") %>
updated: <% tp.date.now("YYYY-MM-DD") %>
tags:
  - area/guides
  - status/draft
audience:
  - backend-devs
  - frontend-devs
  - devops
  - ai-devs
status: draft
template: how-to
estimatedTime: 0
prerequisites: []
related: []
aliases: []
---

# 📖 How-to: {{title}}

> [!info] **Metadata**
> **Audiencia**: {{audience}}
> **Tiempo estimado**: <% tp.user.default("30 minutos") %>
> **Prerrequisitos**: {{prerequisites}}

---

## 🎯 Objetivo

<!-- 1-2 oraciones: qué lograrás al seguir esta guía. -->

## ✅ Prerrequisitos

- [ ] Prerrequisito 1
- [ ] Prerrequisito 2

## 📋 Pasos

### Paso 1: <Título>

<!-- Descripción clara y concisa. -->

```bash
# Comando exacto que el usuario debe correr
aws s3 ls
```

### Paso 2: <Título>

<!-- ... -->

### Paso 3: <Título>

<!-- ... -->

## ✅ Verificación

<!-- ¿Cómo verifica el usuario que todo funcionó? -->

```bash
# Comando de verificación
curl https://api.example.com/health
```

**Salida esperada**:

```
{"status": "ok"}
```

## 🐛 Troubleshooting

### Problema: <Síntoma>

**Causa probable**: ...

**Solución**:

```bash
# Comando para resolver
```

### Problema: <Otro síntoma>

**Causa probable**: ...

**Solución**: ...

## 🔗 Wikilinks relacionados

- [[00 - Home]]
- [[MOC-guides]]

## 📚 Referencias

- [Documentación oficial](https://...)
- [Tutorial relacionado](./ruta-relativa.md)

---
title: Setup del entorno de desarrollo
author: @spark-match/devops
date: 2026-07-04
tags: [onboarding, setup, devops, github, aws, git]
audience: [backend-devs, frontend-devs, ai-devs, devops]
status: published
related:
  - welcome.md
---

# 🛠️ Setup del Entorno de Desarrollo

> Configuración inicial para contribuir a los repos de Spark Match.

## 🎯 Objetivo

Tener tu máquina lista para clonar repos, abrir PRs y desplegar (si aplica) en menos de 30 minutos.

## ✅ Prerrequisitos

- [ ] Cuenta de GitHub con acceso a la organización `spark-match`
- [ ] Sistema operativo: Windows 10+, macOS 12+ o Ubuntu 20.04+
- [ ] Acceso a internet

## 📋 Instalación

### Paso 1: Git

```bash
# macOS
brew install git

# Ubuntu
sudo apt install git -y

# Windows
winget install Git.Git
```

Configura tu identidad:

```bash
git config --global user.name "Tu Nombre"
git config --global user.email "tu-email@ejemplo.com"
git config --global init.defaultBranch main
```

### Paso 2: GitHub CLI

```bash
# macOS
brew install gh

# Ubuntu
sudo apt install gh -y

# Windows
winget install GitHub.cli
```

Autentícate:

```bash
gh auth login
```

Sigue el wizard:

- **Where do you use GitHub?** → GitHub.com
- **What is your preferred protocol?** → HTTPS o SSH (recomendado SSH)
- **Authenticate Git with your GitHub credentials?** → Yes
- **How would you like to authenticate?** → Login with a web browser

### Paso 3: SSH Key (recomendado)

Si elegiste SSH:

```bash
ssh-keygen -t ed25519 -C "tu-email@ejemplo.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Añade la clave a tu cuenta
gh ssh-key add ~/.ssh/id_ed25519.pub --title "Tu máquina ($(hostname))"
```

Verifica:

```bash
ssh -T git@github.com
```

### Paso 4: AWS CLI (solo DevOps, Backend, AI, Data)

```bash
# macOS
brew install awscli

# Ubuntu
sudo apt install awscli -y

# Windows
winget install Amazon.AWSCLI
```

> ⚠️ **No configures access keys.** Spark Match usa **OIDC + roles IAM** para autenticación desde GitHub Actions. Solo necesitas la CLI si vas a desplegar localmente.

### Paso 5: Node.js 20 (Backend, Frontend, DevOps)

```bash
# Usa nvm (recomendado)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
nvm install 20
nvm use 20
```

### Paso 6: Python 3.12 (Backend AI/Data, AI Devs)

```bash
# macOS
brew install python@3.12

# Ubuntu
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt install python3.12 python3.12-venv

# Windows
winget install Python.Python.3.12
```

Instala `uv` (gestor de deps rápido):

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### Paso 7: Terraform (solo DevOps)

```bash
# macOS
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Ubuntu
sudo apt install terraform -y

# Windows
winget install Hashicorp.Terraform
```

### Paso 8: SAM CLI (solo Backend)

```bash
# macOS
brew install aws-sam-cli

# Ubuntu/Linux
pip install aws-sam-cli

# Windows
winget install Amazon.SAM-CLI
```

## 📂 Clonar los repos

```bash
# Crear carpeta de trabajo
mkdir -p ~/spark-match && cd ~/spark-match

# Clonar los repos principales
gh repo clone spark-match/spark-match-00-knowledge-base
gh repo clone spark-match/spark-match-01-devops
gh repo clone spark-match/spark-match-02-infrastructure
gh repo clone spark-match/spark-match-03-backend
gh repo clone spark-match/spark-match-04-frontend
gh repo clone spark-match/spark-match-05-data-pipeline
gh repo clone spark-match/spark-match-06-model-training
gh repo clone spark-match/spark-match-07-article
```

## 🔐 Configurar acceso a AWS (si aplica)

Si eres DevOps, Backend o AI Devs y necesitas acceso a AWS:

1. Pide a `@spark-match/devops` que te añada al rol IAM correspondiente
2. Configura tu perfil local:

```bash
# Edita ~/.aws/config
[profile spark-match-prod]
region = us-east-1
output = json

[profile spark-match-dev]
region = us-east-1
output = json
```

3. Configura las credenciales (si usas SSO) o deja que OIDC las gestione

## ✅ Verificación

Ejecuta este checklist:

- [ ] `git --version` muestra 2.30+
- [ ] `gh --version` muestra 2.0+
- [ ] `gh auth status` muestra tu cuenta autenticada
- [ ] `git clone git@github.com:spark-match/spark-match-00-knowledge-base.git` funciona
- [ ] `node --version` muestra 20.x (si aplica)
- [ ] `python3 --version` muestra 3.12.x (si aplica)
- [ ] `terraform --version` muestra 1.6+ (si aplica)
- [ ] `sam --version` muestra 1.100+ (si aplica)

## 🐛 Troubleshooting

### `git@github.com: Permission denied (publickey)`

Tu clave SSH no está añadida a GitHub. Ejecuta:

```bash
gh ssh-key add ~/.ssh/id_ed25519.pub --title "Mi máquina"
```

### `gh: command not found`

Reinstala GitHub CLI siguiendo el Paso 2.

### `aws: command not found`

Reinstala AWS CLI siguiendo el Paso 4.

### Python 3.12 no disponible en Ubuntu

Usa el PPA:

```bash
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.12
```

## 📚 Referencias

- [GitHub CLI docs](https://cli.github.com/manual/)
- [Git SSH setup](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)
- [AWS CLI docs](https://docs.aws.amazon.com/cli/)
- [Terraform install](https://learn.hashicorp.com/tutorials/terraform/install-cli)
- [SAM CLI install](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)
- [Bienvenida a Spark Match](./welcome.md)
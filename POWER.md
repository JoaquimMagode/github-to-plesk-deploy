---
name: "Deploy projects from GitHub to Plesk"
displayName: "Deploy to Plesk"
description: "Deploy Node.js + React (Vite) projects from GitHub to Plesk shared hosting using GitHub Actions, SSH, and rsync."
keywords: ["Deploy","Node.js","React (Vite)","GitHub","Plesk","CI/CD","SSH","Workflows","PM2"]
author: "Joaquim Magode"
---


# github-to-plesk-deploy

Deploy Node.js + React (Vite) projects from GitHub to Plesk shared hosting using GitHub Actions, SSH, and rsync.

---

## What this power does

This power turns Kiro into a CI/CD specialist for Plesk deployments. It knows how to:

- Generate a complete, ready-to-use `.github/workflows/deploy.yml`
- Set up SSH keys correctly for automated pipelines
- Configure PM2 on a remote Plesk server
- Deploy a React + Vite frontend and Node.js + Express backend via rsync
- Diagnose and fix the most common deployment failures

---

## Included

| Type | Name | Purpose |
|---|---|---|
| Agent | `github-actions-plesk-deploy` | CI/CD specialist — fixes workflows, diagnoses errors, generates pipelines |
| Skill | `github-actions-plesk-deploy` | Interactive wizard that generates a complete `deploy.yml` from scratch |
| Steering | `github-actions-plesk-deploy` | Reference guide with canonical SSH, rsync, PM2, and .env patterns |

---

## How to use

### Generate a new deploy.yml

Just ask:

```
generate a deploy.yml for my Plesk server
```

Kiro will ask for your server path, PM2 app name, Node.js version, and which secrets you need, then write the complete workflow file.

### Fix a broken workflow

Paste your error log or broken `deploy.yml` and ask:

```
fix my GitHub Actions deployment - getting "Load key: invalid format"
```

### Add the steering guide to context

Reference it manually when you want the full rules loaded:

```
#github-actions-plesk-deploy
```

---

## Keywords that activate this power

`deploy` `plesk` `github-actions` `ci-cd` `ssh` `rsync` `pm2` `nodejs` `react`

---

## Required GitHub Secrets

| Secret | Description |
|---|---|
| `SSH_HOST` | Plesk server IP or hostname |
| `SSH_PORT` | SSH port (usually 22) |
| `SSH_USERNAME` | SSH user (e.g. root) |
| `SSH_PRIVATE_KEY` | Full PEM content of ed25519 private key |
| `VITE_API_URL` | Frontend API base URL |
| `DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` | Database credentials |
| `JWT_SECRET` | JWT signing secret |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | OAuth credentials |
| `SESSION_SECRET` | Express session secret |
| `EMAIL_USER`, `EMAIL_PASSWORD` | SMTP credentials |

---

## Common errors this power can fix

- `Permission denied (publickey)` - wrong or missing SSH key
- `Load key "...": invalid format` - key written incorrectly in workflow
- `sharp` module error on linux-x64 - native binary platform mismatch
- `MODULE_NOT_FOUND` after deploy - node_modules not reinstalled on server
- Frontend API calls failing - VITE_API_URL not injected at build time

---

## Author

Joaquim Magode
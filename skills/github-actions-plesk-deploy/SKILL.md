---
name: github-actions-plesk-deploy
description: >
  Generates a complete, ready-to-use GitHub Actions deploy.yml for deploying a
  Node.js + React (Vite) project to a Plesk server via SSH/rsync. Asks the user
  for their specific values, then produces the full workflow file. Use this skill
  when the user wants to create or regenerate their deploy.yml from scratch.
tools: ["read", "write"]
---

# Skill: Generate deploy.yml

Follow this exact five-step process whenever this skill is invoked.

---

## Step 1 - Gather Information

Ask the user for the values below. If any value is already visible in context (e.g. an existing `deploy.yml` is open), pre-fill it and confirm rather than asking again.

| # | What to ask | Example |
|---|---|---|
| 1 | Server base path (Plesk vhost root) | `/var/www/vhosts/example.com/subdomains/myapp` |
| 2 | Frontend target directory (relative to base path) | `.httpdocs` |
| 3 | Backend target directory (relative to base path) | `backend` |
| 4 | PM2 app name | `myapp-backend` |
| 5 | Backend entry file | `server.js` |
| 6 | Backend health check URL (or `none` to skip) | `http://localhost:3000/api/health` |
| 7 | Node.js version | `20` |
| 8 | Local frontend directory | `frontend` |
| 9 | Local backend directory | `Backend` |
| 10 | Does the project use OAuth (Google)? | yes / no |
| 11 | Does the project use email (Nodemailer)? | yes / no |
| 12 | Does the project use a database? | yes / no |

---

## Step 2 - Generate the Workflow

Using the answers above, produce a complete `.github/workflows/deploy.yml` with this structure:

```
name: CI/CD - Build & Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  NODE_VERSION: "<NODE_VERSION>"

jobs:

  build:
    name: Build
    runs-on: ubuntu-latest
    steps:
      - Checkout
      - Backend: setup node, install, test (continue-on-error: true)
      - Frontend: setup node, install, build (with VITE_API_URL env)
      - Upload frontend/dist as artifact (retention-days: 1)
      - Upload Backend/ as artifact (retention-days: 1)

  deploy:
    name: Deploy to Plesk
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    steps:
      - Checkout
      - Download frontend artifact to frontend/dist
      - Download backend artifact to Backend/
      - Generate backend .env (heredoc, no single-quote delimiter)
      - Setup SSH (canonical pattern - verbatim)
      - Prepare server directories (mkdir -p both paths)
      - Deploy frontend (rsync -az --delete, no exclusions needed for dist)
      - Deploy backend (rsync -az --delete --exclude .env --exclude node_modules --exclude uploads)
      - Upload backend .env via scp
      - Install deps + PM2 restart (remote SSH heredoc with set -e)
      - Health check (curl -sf, exit 1 on failure with pm2 logs output)
```

---

## Step 3 - Rules to Apply While Generating

Apply every rule from the steering guide:

- SSH setup step must be the canonical verbatim pattern - never simplify it
- All secrets referenced in shell must be declared in `env:` on the step, not inline
- `.env` generated with an unquoted `<<EOF` heredoc
- rsync to backend always has the three `--exclude` flags
- PM2 step uses the `if pm2 describe ... ; then restart; else start; fi` pattern with `pm2 save`
- Health check uses `curl -sf` and exits 1 on failure, printing `pm2 logs`
- `chmod 600` applied to `.env` after upload
- Node version references the top-level `env.NODE_VERSION` variable
- Both jobs use: `actions/checkout@v4`, `actions/setup-node@v4`, `actions/upload-artifact@v4`, `actions/download-artifact@v4`

---

## Step 4 - Write the File

Write the generated workflow to `.github/workflows/deploy.yml`. If a file already exists there, read it first and ask the user whether to overwrite or merge.

---

## Step 5 - Post-Generation Checklist

After writing the file, output this checklist:

```
deploy.yml created at .github/workflows/deploy.yml

Next steps:

1. Generate SSH key pair (if not done):
   ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions_deploy -N ""

2. Add public key to server:
   ssh-copy-id -i ~/.ssh/github_actions_deploy.pub -p 22 USER@HOST

3. Add secrets to GitHub (Settings > Secrets and variables > Actions):
   - SSH_HOST, SSH_PORT, SSH_USERNAME, SSH_PRIVATE_KEY
   - VITE_API_URL
   - DB_HOST, DB_USER, DB_PASSWORD, DB_NAME   (if using database)
   - JWT_SECRET
   - GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET   (if using OAuth)
   - SESSION_SECRET
   - EMAIL_USER, EMAIL_PASSWORD               (if using email)

4. Test SSH connection locally:
   ssh -i ~/.ssh/github_actions_deploy -p PORT USER@HOST "echo 'SSH OK'"

5. Push to main to trigger the pipeline.

Troubleshooting:
- "Load key: invalid format"       -> re-paste private key in GitHub Secrets (full PEM with headers)
- "Permission denied (publickey)"  -> re-add .pub to /root/.ssh/authorized_keys on server
- "MODULE_NOT_FOUND"               -> check npm ci ran successfully on server
```
# github-to-plesk-deploy

A [Kiro Power](https://kiro.dev/docs/powers/) that turns Kiro into a CI/CD specialist for deploying Node.js + React (Vite) projects from GitHub to Plesk shared hosting via GitHub Actions, SSH, and rsync.

---

## What it does

- Generates a complete, ready-to-use `.github/workflows/deploy.yml`
- Sets up SSH keys correctly for automated pipelines
- Configures PM2 on a remote Plesk server
- Deploys a React + Vite frontend and Node.js + Express backend via rsync
- Diagnoses and fixes the most common deployment failures

---

## Installation

### From GitHub (recommended for sharing)

1. Clone or download this repository:
   ```bash
   git clone https://github.com/Joaquim-Magode/github-to-plesk-deploy.git
   ```
2. Open Kiro → Powers panel
3. Click **Add Custom Power** → **Import power from a folder**
4. Select the cloned folder
5. The power appears as **Deploy to Plesk** by Joaquim Magode

### From a local folder

1. Open Kiro → Powers panel
2. Click **Add Custom Power** → **Import power from a folder**
3. Select this folder

---

## Included

| Type | Name | Purpose |
|---|---|---|
| Agent | `github-actions-plesk-deploy` | CI/CD specialist — fixes workflows, diagnoses errors, generates pipelines |
| Skill | `github-actions-plesk-deploy` | Interactive wizard that generates a complete `deploy.yml` from scratch |
| Steering | `github-actions-plesk-deploy` | Reference guide with canonical SSH, rsync, PM2, and .env patterns |

---

## Usage

### Generate a new deploy.yml

```
generate a deploy.yml for my Plesk server
```

Kiro will ask for your server path, PM2 app name, Node.js version, and which secrets you need, then write the complete workflow file.

### Fix a broken workflow

Paste your error log or broken `deploy.yml` and ask:

```
fix my GitHub Actions deployment - getting "Load key: invalid format"
```

### Load the steering guide manually

```
#github-actions-plesk-deploy
```

---

## Project structure

```
github-to-plesk-deploy/
├── plugin.json                                      # Power manifest
├── POWER.md                                         # Power documentation
├── README.md                                        # This file
├── skills/
│   └── github-actions-plesk-deploy/
│       └── SKILL.md                                 # deploy.yml generator skill
└── dev.kiro/
    ├── agents/
    │   └── github-actions-plesk-deploy.md           # CI/CD specialist agent
    └── steering/
        └── github-actions-plesk-deploy.md           # Deployment rules reference
```

---

## MCP Servers

This power includes an `mcp.json` with two optional servers. Edit the placeholders with your own values after installing.

### github (`@modelcontextprotocol/server-github`)

Gives Kiro live access to your GitHub repo — check Actions run status, view workflow files, manage secrets.

**Setup:** Create a [GitHub Personal Access Token](https://github.com/settings/tokens) with `repo` and `workflow` scopes, then replace `<YOUR_GITHUB_PAT>` in `mcp.json`.

**Enabled by default.** Requires `npx` (comes with Node.js).

### ssh-mcp (`ssh-mcp`)

Lets Kiro run commands directly on your Plesk server — check PM2 status, tail logs, verify deployments, test SSH connectivity.

**Setup:** Replace the four placeholders in `mcp.json`:
- `<YOUR_PLESK_SERVER_IP>` — server IP or hostname
- `<YOUR_SSH_PORT>` — SSH port (usually `22`)
- `<YOUR_SSH_USERNAME>` — SSH user (e.g. `root`)
- `<PATH_TO_YOUR_PRIVATE_KEY>` — local path to your ed25519 private key (e.g. `~/.ssh/github_actions_deploy`)

**Disabled by default** (`"disabled": true`) — enable it after filling in your values.

> Neither server transmits your credentials anywhere other than the target (GitHub API / your own server). Review `mcp.json` before enabling.

---

## Keywords that activate this power

`deploy` · `plesk` · `github-actions` · `ci-cd` · `ssh` · `rsync` · `pm2` · `nodejs` · `react`

---

## Required GitHub Secrets

| Secret | Description |
|---|---|
| `SSH_HOST` | Plesk server IP or hostname |
| `SSH_PORT` | SSH port (usually `22`) |
| `SSH_USERNAME` | SSH user (e.g. `root`) |
| `SSH_PRIVATE_KEY` | Full PEM content of ed25519 private key including `-----BEGIN/END-----` headers |
| `VITE_API_URL` | Frontend API base URL (injected at Vite build time) |
| `DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` | Database credentials |
| `JWT_SECRET` | JWT signing secret |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | Google OAuth credentials |
| `SESSION_SECRET` | Express session signing secret |
| `EMAIL_USER`, `EMAIL_PASSWORD` | Nodemailer SMTP credentials |

---

## Common errors this power can fix

| Error | Cause |
|---|---|
| `Permission denied (publickey)` | Wrong or missing SSH key in `authorized_keys` |
| `Load key "...": invalid format` | Key written with `echo` or inline secret in workflow |
| `sharp` module error on linux-x64 | Native binary platform mismatch |
| `MODULE_NOT_FOUND` after deploy | `node_modules` not reinstalled on server after rsync |
| Frontend API calls failing in production | `VITE_API_URL` not injected at build time |

---

## SSH key setup (one-time)

```bash
# Generate a dedicated key pair
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions_deploy -N ""

# Add public key to server
ssh-copy-id -i ~/.ssh/github_actions_deploy.pub -p 22 root@YOUR_SERVER_IP

# Copy private key into GitHub Secrets as SSH_PRIVATE_KEY
cat ~/.ssh/github_actions_deploy

# Test the connection
ssh -i ~/.ssh/github_actions_deploy -p 22 root@YOUR_SERVER_IP "echo 'SSH OK'"
```

---

## License

MIT — Joaquim Magode

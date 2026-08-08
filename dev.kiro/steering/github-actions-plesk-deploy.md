---
inclusion: manual
---

# GitHub Actions → Plesk Deployment Guide

Reference guide for CI/CD pipelines that build and deploy this project (React + Vite frontend, Node.js + Express backend) to a Plesk Linux VPS via SSH and rsync.

---

## Project Stack

| Layer | Technology | Local path | Server path |
|---|---|---|---|
| Frontend | React + TypeScript + Vite | `frontend/` | `.httpdocs/` (rsync from `frontend/dist/`) |
| Backend | Node.js + Express | `Backend/` | `backend/` (rsync from `Backend/`) |
| Process manager | PM2 | — | runs on server |
| Hosting | Plesk on Linux VPS | — | `/var/www/vhosts/<domain>/subdomains/<sub>/` |
| Deployment | SSH + rsync | — | — |

---

## Required GitHub Secrets

| Secret | Description |
|---|---|
| `SSH_HOST` | Plesk server IP or hostname |
| `SSH_PORT` | SSH port (usually `22`) |
| `SSH_USERNAME` | SSH user (e.g. `root`) |
| `SSH_PRIVATE_KEY` | Full PEM content of ed25519 private key including `-----BEGIN/END-----` headers |
| `VITE_API_URL` | Frontend API base URL (injected at build time by Vite) |
| `DB_HOST` | Database host |
| `DB_USER` | Database username |
| `DB_PASSWORD` | Database password |
| `DB_NAME` | Database name |
| `JWT_SECRET` | JWT signing secret |
| `GOOGLE_CLIENT_ID` | Google OAuth client ID |
| `GOOGLE_CLIENT_SECRET` | Google OAuth client secret |
| `SESSION_SECRET` | Express session signing secret |
| `EMAIL_USER` | Nodemailer SMTP username |
| `EMAIL_PASSWORD` | Nodemailer SMTP password |

---

## SSH Key Setup (one-time)

Generate a dedicated key pair — never reuse existing keys:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions_deploy -N ""
```

Add the **public** key to the server:
```bash
ssh-copy-id -i ~/.ssh/github_actions_deploy.pub -p 22 root@YOUR_SERVER_IP
# or manually:
cat ~/.ssh/github_actions_deploy.pub | ssh -p 22 root@YOUR_SERVER_IP "cat >> /root/.ssh/authorized_keys"
```

Copy the **private** key content into GitHub:
```bash
cat ~/.ssh/github_actions_deploy
```
Paste the full output (including `-----BEGIN OPENSSH PRIVATE KEY-----` / `-----END OPENSSH PRIVATE KEY-----`) into GitHub → Settings → Secrets → `SSH_PRIVATE_KEY`.

Test before running the pipeline:
```bash
ssh -i ~/.ssh/github_actions_deploy -p 22 root@YOUR_SERVER_IP "echo 'SSH OK'"
```

---

## Canonical SSH Setup Step (copy verbatim)

```yaml
- name: Setup SSH
  env:
    SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
    SSH_HOST: ${{ secrets.SSH_HOST }}
    SSH_PORT: ${{ secrets.SSH_PORT }}
    SSH_USERNAME: ${{ secrets.SSH_USERNAME }}
  run: |
    mkdir -p "$HOME/.ssh"
    printf '%s\n' "$SSH_PRIVATE_KEY" > "$HOME/.ssh/id_ed25519"
    chmod 600 "$HOME/.ssh/id_ed25519"
    ssh-keyscan -p "$SSH_PORT" -H "$SSH_HOST" >> "$HOME/.ssh/known_hosts"
    ssh \
      -o BatchMode=yes \
      -o StrictHostKeyChecking=yes \
      -o IdentitiesOnly=yes \
      -i "$HOME/.ssh/id_ed25519" \
      -p "$SSH_PORT" \
      "$SSH_USERNAME@$SSH_HOST" \
      "echo 'SSH OK'"
```

**Why these rules matter:**
- `env:` block — secrets are injected as environment variables, never interpolated inside shell strings (prevents format corruption)
- `printf '%s\n'` — preserves exact key bytes including trailing newline; `echo` adds an extra newline that breaks PEM format
- `-o IdentitiesOnly=yes` — prevents SSH agent from offering other keys and causing auth confusion
- `ssh-keyscan -p "$SSH_PORT"` — must use the correct port or the host key won't match

---

## .env Generation Pattern

Use an unquoted heredoc so secrets expand:

```yaml
- name: Generate backend .env
  run: |
    cat > Backend/.env <<EOF
    DB_HOST=${{ secrets.DB_HOST }}
    DB_USER=${{ secrets.DB_USER }}
    DB_PASSWORD=${{ secrets.DB_PASSWORD }}
    DB_NAME=${{ secrets.DB_NAME }}
    JWT_SECRET=${{ secrets.JWT_SECRET }}
    PORT=3000
    NODE_ENV=production
    GOOGLE_CLIENT_ID=${{ secrets.GOOGLE_CLIENT_ID }}
    GOOGLE_CLIENT_SECRET=${{ secrets.GOOGLE_CLIENT_SECRET }}
    SESSION_SECRET=${{ secrets.SESSION_SECRET }}
    EMAIL_USER=${{ secrets.EMAIL_USER }}
    EMAIL_PASSWORD=${{ secrets.EMAIL_PASSWORD }}
    EOF
```

**Never** use `<<'EOF'` (single-quoted) — it prevents secret expansion.

---

## rsync Rules

All rsync calls to the backend **must** exclude:

```
--exclude ".env"        # managed separately via scp
--exclude "node_modules" # reinstalled on server
--exclude "uploads"     # server-side user files, never overwrite
```

Always pass `-o IdentitiesOnly=yes` inside the `-e` string:

```yaml
rsync -az --delete \
  --exclude ".env" \
  --exclude "node_modules" \
  --exclude "uploads" \
  -e "ssh -o IdentitiesOnly=yes -i $HOME/.ssh/id_ed25519 -p $SSH_PORT" \
  Backend/ \
  "$SSH_USERNAME@$SSH_HOST:/path/to/backend/"
```

Frontend rsync doesn't need exclusions (dist/ is a clean build output):

```yaml
rsync -az --delete \
  -e "ssh -o IdentitiesOnly=yes -i $HOME/.ssh/id_ed25519 -p $SSH_PORT" \
  frontend/dist/ \
  "$SSH_USERNAME@$SSH_HOST:/path/to/.httpdocs/"
```

---

## PM2 Restart Pattern

```bash
if pm2 describe APP_NAME >/dev/null 2>&1; then
  pm2 restart APP_NAME
else
  pm2 start server.js --name APP_NAME
fi
pm2 save
```

Always run `pm2 save` after so PM2 resurrects the process after a server reboot.

---

## Native Module: sharp Replacement

`sharp` uses native binaries compiled per platform. When it fails on GitHub Actions runners or Plesk servers, replace it with `@resvg/resvg-js` (Rust-based prebuilt — no compile step):

```bash
npm uninstall sharp
npm install @resvg/resvg-js
```

```js
const { Resvg } = require('@resvg/resvg-js');

async function svgToPng(svgPath, size) {
  const svg = fs.readFileSync(svgPath, 'utf8');
  const resvg = new Resvg(svg, { fitTo: { mode: 'width', value: size } });
  return resvg.render().asPng();
}
```

---

## Git Hygiene

Files that must never be committed:

```bash
# Untrack if already committed
git rm --cached Backend/.env
git rm --cached -r Backend/uploads/
git commit -m "chore: untrack .env and uploads"
```

`.gitignore` entries required:

```
.env
uploads/
!uploads/.gitkeep
```

Create the gitkeep so the `uploads/` directory exists after a fresh clone:

```bash
touch Backend/uploads/.gitkeep
git add Backend/uploads/.gitkeep
```

---

## Common Errors and Fixes

| Error | Root Cause | Fix |
|---|---|---|
| `Permission denied (publickey)` | Public key not in `authorized_keys` or wrong key pasted in secret | Regenerate key pair, re-add `.pub` to server, re-paste private key in GitHub Secrets |
| `Load key "...": invalid format` | Key written with `echo` or `${{ secrets.X }}` inline | Rewrite Setup SSH step using `printf '%s\n'` + `env:` block |
| `ssh-keyscan` fails / empty host | `SSH_HOST` secret not set or typo | Add/fix the secret in GitHub repo settings |
| `sharp` module error on linux-x64 | Native binary platform mismatch | Replace with `@resvg/resvg-js` |
| `MODULE_NOT_FOUND` after deploy | `node_modules` not reinstalled on server | Run `npm ci --omit=dev` on server after rsync |
| Frontend API calls fail in production | `VITE_API_URL` not injected at build time | Add `env: VITE_API_URL: ${{ secrets.VITE_API_URL }}` to the frontend build step |
| `.env` file has `undefined` values | Secrets not set in GitHub or wrong secret names | Check Settings → Secrets for exact names matching the workflow |
| PM2 not found on server | PM2 not globally installed | Add `npm install -g pm2` guard in the remote SSH step |

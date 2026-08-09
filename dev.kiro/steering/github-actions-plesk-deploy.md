---
inclusion: auto
---

# GitHub Actions to Plesk — Deployment Standards

You are assisting with a CI/CD pipeline that deploys a **React + Vite frontend** and **Node.js + Express backend** to a **Plesk Linux VPS** using GitHub Actions, SSH, and rsync.

Apply every rule in this guide whenever you write, edit, or review any GitHub Actions workflow for this project. These rules are non-negotiable — never simplify, skip, or reword the canonical patterns.

---

## Project Layout

| Layer | Local path | Server path |
|---|---|---|
| Frontend | `frontend/` | `.httpdocs/` (deploy from `frontend/dist/`) |
| Backend | `Backend/` | `backend/` (deploy from `Backend/`) |
| Process manager | — | PM2 on the remote server |
| Hosting | — | `/var/www/vhosts/<domain>/subdomains/<sub>/` |

---

## Rule 1 — SSH Setup Step (use verbatim, every time)

Always write the SSH setup step exactly like this. Never use `echo`, never inline `${{ secrets.X }}` in shell, never simplify.

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

**Why:** `printf '%s\n'` preserves exact PEM bytes. `echo` appends an extra newline that corrupts the key format. The `env:` block prevents GitHub from masking the key mid-write.

---

## Rule 2 — Secrets in Shell Steps

All secrets used in `run:` steps must be declared in an `env:` block on that step and referenced as `$VARIABLE_NAME`. Never write `${{ secrets.X }}` inside a shell command.

```yaml
# CORRECT
- name: Deploy backend
  env:
    SSH_PORT: ${{ secrets.SSH_PORT }}
    SSH_USERNAME: ${{ secrets.SSH_USERNAME }}
    SSH_HOST: ${{ secrets.SSH_HOST }}
  run: |
    rsync ... -p "$SSH_PORT" "$SSH_USERNAME@$SSH_HOST:/path/"

# WRONG - causes "Load key: invalid format"
- name: Deploy backend
  run: |
    rsync ... -p "${{ secrets.SSH_PORT }}" "${{ secrets.SSH_USERNAME }}@${{ secrets.SSH_HOST }}:/path/"
```

---

## Rule 3 — rsync to Backend (three excludes, always)

Every rsync call targeting the backend directory must include all three exclusions and `-o IdentitiesOnly=yes`:

```yaml
- name: Deploy backend
  env:
    SSH_PORT: ${{ secrets.SSH_PORT }}
    SSH_USERNAME: ${{ secrets.SSH_USERNAME }}
    SSH_HOST: ${{ secrets.SSH_HOST }}
    SERVER_PATH: /var/www/vhosts/example.com/subdomains/myapp/backend
  run: |
    rsync -az --delete \
      --exclude ".env" \
      --exclude "node_modules" \
      --exclude "uploads" \
      -e "ssh -o IdentitiesOnly=yes -i $HOME/.ssh/id_ed25519 -p $SSH_PORT" \
      Backend/ \
      "$SSH_USERNAME@$SSH_HOST:$SERVER_PATH/"
```

Frontend rsync deploys `frontend/dist/` only — no exclusions needed since dist is a clean build output.

---

## Rule 4 — Backend .env Generation (unquoted heredoc)

Generate the backend `.env` file using an unquoted `<<EOF` heredoc so `${{ secrets.X }}` values expand:

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

Never use `<<'EOF'` — the single-quoted delimiter prevents secret expansion, leaving literal `${{ secrets.X }}` strings in the file.

After uploading to the server via `scp`, always set permissions:

```bash
chmod 600 /path/to/backend/.env
```

---

## Rule 5 — PM2 Restart Pattern

Always use the idempotent pattern with `pm2 save`:

```bash
if pm2 describe APP_NAME >/dev/null 2>&1; then
  pm2 restart APP_NAME
else
  pm2 start server.js --name APP_NAME
fi
pm2 save
```

`pm2 save` is mandatory — it persists the process list so PM2 restores it after a server reboot.

---

## Rule 6 — Workflow Structure

Every workflow must follow this two-job structure:

```yaml
jobs:
  build:
    name: Build
    runs-on: ubuntu-latest
    steps:
      # 1. Checkout
      # 2. Backend: setup-node, npm ci, npm test (continue-on-error: true)
      # 3. Frontend: setup-node, npm ci, npm run build (env: VITE_API_URL)
      # 4. Upload frontend/dist artifact (retention-days: 1)
      # 5. Upload Backend/ artifact (retention-days: 1)

  deploy:
    name: Deploy to Plesk
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    steps:
      # 1. Checkout
      # 2. Download frontend artifact
      # 3. Download backend artifact
      # 4. Generate backend .env (unquoted heredoc)
      # 5. Setup SSH (verbatim canonical step)
      # 6. mkdir -p both server directories
      # 7. rsync frontend/dist/ to .httpdocs/
      # 8. rsync Backend/ to backend/ (with three excludes)
      # 9. scp .env to server + chmod 600
      # 10. Remote SSH: npm ci --omit=dev + PM2 restart + pm2 save
      # 11. Health check: curl -sf, exit 1 on failure + pm2 logs
```

Always use pinned action versions: `actions/checkout@v4`, `actions/setup-node@v4`, `actions/upload-artifact@v4`, `actions/download-artifact@v4`.

Define `NODE_VERSION` as a top-level `env:` variable and reference it in all `setup-node` steps.

---

## Rule 7 — Git Hygiene

These files must never be committed. Check `.gitignore` whenever `.env` or `uploads/` are mentioned:

```gitignore
.env
uploads/
!uploads/.gitkeep
```

If already tracked:

```bash
git rm --cached Backend/.env
git rm --cached -r Backend/uploads/
git commit -m "chore: untrack .env and uploads"
```

---

## Rule 8 — Native Module: sharp

`sharp` fails on Plesk/GitHub Actions because its native binary is compiled per platform. Replace with `@resvg/resvg-js` (Rust prebuilt, no compile step):

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

## GitHub Secrets Reference

| Secret | Description |
|---|---|
| `SSH_HOST` | Plesk server IP or hostname |
| `SSH_PORT` | SSH port (usually `22`) |
| `SSH_USERNAME` | SSH user (e.g. `root`) |
| `SSH_PRIVATE_KEY` | Full PEM content of ed25519 private key including `-----BEGIN/END-----` headers |
| `VITE_API_URL` | Frontend API base URL — injected at Vite build time |
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

## Error Diagnosis Reference

| Error | Root Cause | Fix |
|---|---|---|
| `Permission denied (publickey)` | Public key missing from `authorized_keys` or wrong key pasted | Regenerate key pair, re-add `.pub` to server, re-paste private key in Secrets |
| `Load key "...": invalid format` | Key written with `echo` or `${{ secrets.X }}` inline in shell | Rewrite SSH step using Rule 1 verbatim |
| `ssh-keyscan` fails or empty host | `SSH_HOST` secret missing or typo | Add/fix secret in GitHub repo settings |
| `sharp` module error on linux-x64 | Native binary platform mismatch | Replace with `@resvg/resvg-js` (Rule 8) |
| `MODULE_NOT_FOUND` after deploy | `node_modules` not reinstalled on server | Run `npm ci --omit=dev` on server after rsync |
| Frontend API calls fail in production | `VITE_API_URL` not passed to build step | Add `env: VITE_API_URL: ${{ secrets.VITE_API_URL }}` to frontend build step |
| `.env` values are `undefined` on server | Secrets not set or wrong names in GitHub | Check Settings > Secrets, names must match exactly |
| PM2 not found on server | PM2 not globally installed | Add `npm install -g pm2` guard before PM2 commands |

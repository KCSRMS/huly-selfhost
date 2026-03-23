# GitHub integration — next steps & operations guide

This guide captures the follow-up after a self-hosted Huly instance (`huly.proxy.kcsgis.com`) is wired for GitHub Apps and the **Tracker ↔ GitHub** flow is working.

## What should already be in place

| Area | Check |
|------|--------|
| **GitHub App** | Callback URL, **Setup URL** → `https://<HOST>/github`; Webhook → `https://<HOST>/_github/api/webhook`; webhook secret matches `GITHUB_WEBHOOK_SECRET`. |
| **Post-install** | **Redirect on update** enabled on the GitHub App so installs/updates send users back to Huly to finish workspace linking. |
| **Traefik** | Hostname routes to **nginx:80** only; **nginx** has `web` and **`websecure`** Traefik labels so HTTPS hits nginx (not `front` directly). |
| **Nginx** | `location /_github/` strips prefix and proxies to `github:3500` (see `.huly.nginx`). |
| **`github` service** | `compose.yml` with `GITHUB_*` env; `GITHUB_PRIVATE_KEY` is a **full PEM** (often one line with `\n` for Dokploy). |
| **Dokploy** | Environment variables for the stack; **Domains** points at **nginx** only. |

Reference files in this repo: `compose.yml`, `.huly.nginx`, `kcsrms.env.example`, README **GitHub Service** section.

## Day-to-day next steps

1. **Map repositories to Tracker**  
   Huly: **Integrations → GitHub → Github Repositories**. Use **Connect to Huly** (or equivalent) to attach repos to the right Tracker projects. See [Huly docs — GitHub](https://docs.huly.io/integrations/github/).

2. **Org vs personal installs**  
   If repos live under a **GitHub organization**, install/configure the app for that **org**; “Installed GitHub Apps” links in UIs often open **user** settings first — use **Org → Settings → GitHub Apps** when needed.

3. **Verify webhooks after changes**  
   GitHub App → **Advanced** / webhook deliveries: expect **200** for events. Failures → Traefik → nginx → `github` service (`GITHUB_WEBHOOK_SECRET`, routing).

4. **Smoke test (optional)**  
   ```bash
   curl -sS -X POST "https://<HOST>/_github/api/webhook" -H "Content-Type: application/json" -d "{}"
   ```  
   A JSON error about **missing GitHub headers** (not HTML `Cannot POST /_github/...`, not **502**) means routing reaches the **github** service.

## If something breaks

| Symptom | Likely cause |
|--------|----------------|
| `Cannot POST /_github/...` (Express HTML) | HTTPS → `front` instead of nginx; fix **websecure** router to nginx, or duplicate Traefik route. |
| **502** from nginx | `github` container down, wrong network, or bad env so process exits. |
| `secretOrPrivateKey must be an asymmetric key` in **github** logs | `GITHUB_PRIVATE_KEY` truncated or malformed; use full PEM or `\n`-escaped single line. |
| “No Platform GitHub App” for workspace | Setup redirect not completed; enable **Redirect on update**, repeat **Install** from Huly, let browser land on `https://<HOST>/github`. |
| **URL typos** | GitHub App URLs must match **`HOST`** exactly (e.g. `kcsgis` vs `krcgis`). |

## Security

- Never commit real keys; keep `kcsrms.env` gitignored (see `.gitignore`).
- Rotate **GitHub App private keys** if they were ever exposed; update `GITHUB_PRIVATE_KEY` in Dokploy and redeploy.

## Optional hardening

- Monitor **github** service logs and GitHub webhook delivery failures.
- Document **Dokploy** env copies for disaster recovery (without pasting secrets into git).

---

*Last aligned with repo layout: `compose.yml`, `.huly.nginx`, `kcsrms.env.example`.*

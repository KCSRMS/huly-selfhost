# Optional services implementation plan

This document is an **artifact** describing how to enable optional Huly services for this self-host stack: **GitHub integration**, **web push notifications**, **mail**, **Love (LiveKit)**, **HulyPulse**, **print**, and **export**. It aligns with `compose.yml`, `README.md`, and `.huly.nginx` in this repository.

**Security:** Never commit real credentials. Use a secrets manager for production. Rotate any values that were ever exposed in logs, chats, or tickets.

**Environment file for this repo:** Add or change compose-related variables in **`kcsrms.env`** at the repo root (tracked in git). Copy from there into **Dokploy** or pass `--env-file kcsrms.env` when running Compose locally. The repo root `.env` is not used for this stack unless you mirror values intentionally.

---

## Dokploy (Traefik) — do we need to change the plan?

**Mostly no.** This stack uses **one public entry**: Traefik routes by hostname to **`nginx` on port 80** (see `traefik.*` labels on `nginx` in `compose.yml` and attachment to `dokploy-network`). Nginx then routes paths (`/`, `/_accounts`, `/_transactor`, future `/_github`, `/_pulse`, …) to internal containers.

| Topic | With Dokploy |
|-------|----------------|
| **New optional services** | Still add containers to `compose.yml` and wire **only** `.huly.nginx` for new `location` prefixes. You do **not** need extra Traefik router labels per service unless you stop using nginx as the single front door. |
| **Environment variables** | Maintain keys in **`kcsrms.env`**, then mirror them in **Dokploy** (same names as `compose.yml`). Prefer Dokploy secrets for passwords and keys. |
| **Deploy / reload** | Redeploy or **restart** the stack from Dokploy after changing `compose.yml` or `.huly.nginx`. No separate `nginx -s reload` on the host if Dokploy recreates the `nginx` container with the updated mounted config. |
| **TLS** | If Traefik terminates HTTPS, the browser sees `https://`. Keep `HOST_ADDRESS` and `SECURE` (and `front` URLs) consistent with that public URL. GitHub/LiveKit callback URLs must use the **same** scheme and host users hit in the browser. |
| **`dokploy-network`** | Keep `nginx` attached to **`dokploy-network` (external)** so Traefik can reach it; backend services stay on `huly_net` behind nginx. |

**Summary:** The phased plan (services + nginx paths + env) is unchanged. Treat Dokploy as the **orchestrator and env store**; Traefik stays a **single hop to nginx** unless you intentionally expose a service directly (not recommended for this layout).

---

## Current baseline

- Core services are defined in `compose.yml` (nginx, cockroach, redpanda, minio, elastic, rekoni, transactor, collaborator, account, workspace, front, fulltext, stats, kvs).
- Optional services still **not** in `compose.yml` (unless you add them): `mail`, `github`, `love`, `hulypulse`, `print`, `exports`. **`notification`** + `WEB_PUSH_URL` on `transactor` are wired when enabled in your tracked `compose.yml`.
- `.huly.nginx` contains **commented** `location` blocks for `/_print`, `/_love`, `/_pulse`, `/_github`, and `/_export` that must be enabled when those backends exist.

---

## Implementation order

| Phase | Services | Rationale |
|-------|----------|-----------|
| 1 | VAPID keys + `notification` + `transactor` `WEB_PUSH_URL` | Web push (0.7.x uses `notification`, not legacy `ses`). |
| 2 | `mail` + `MAIL_URL` on `account` and `transactor` | Transactional email; required for many flows and user email prefs. |
| 3 | `hulypulse` + `PULSE_URL` + nginx `/_pulse` | Presence, typing, call knock/invite (in-memory backend OK on v0.7.375+). |
| 4 | `love` + nginx `/_love` + LiveKit env on `front` | Audio/video via LiveKit. |
| 5 | `print` + `PRINT_URL` + nginx `/_print` | Server-side print/PDF paths. |
| 6 | `exports` + `EXPORT_URL` + nginx `/_export` | Export workflows. |
| 7 | GitHub App + `github` + nginx `/_github` + `front` GitHub vars | Repo sync (issues/PRs); separate from GitHub OAuth sign-in. |

---

## 1. Notifications (web push)

**References:** `README.md` — *Web Push Notifications*, *notification service*.

1. Generate VAPID keys (e.g. `web-push generate-vapid-keys` or equivalent).
2. Add `notification` service:
   - Image: `hardcoreeng/notification:${HULY_VERSION}`
   - `PORT=8091`, `SOURCE` (sender identity string, often same as outbound mail from address), `STATS_URL=http://stats:4900`
   - `PUSH_PUBLIC_KEY` / `PUSH_PRIVATE_KEY` (or align naming with existing `WEBPUSH_VAPID_*` if the images expect those—match `README` and image docs).
3. On `transactor`, set: `WEB_PUSH_URL=http://notification:8091`
4. On `front`, ensure the **public** VAPID key and subject match what the notification service uses (see existing `WEBPUSH_VAPID_*` pattern in `compose.yml`).

---

## 2. Mail service

**References:** `README.md` — *Mail Service*, *SMTP Configuration*.

1. Add `mail` service:
   - Image: `hardcoreeng/mail:${HULY_VERSION}`
   - `PORT=8097`, `SOURCE=${EMAIL_FROM}`
   - For SMTP: `SMTP_HOST`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD` (use app passwords for Gmail, not account passwords).
2. On **both** `account` and `transactor`, set: `MAIL_URL=http://mail:8097`
3. Users configure event email under **Settings → Notifications** (per-user).

**Note:** SMTP and SES are mutually exclusive; this stack typically uses SMTP.

---

## 3. HulyPulse

**References:** `README.md` — *HulyPulse (Push / real-time updates)*.

1. Uncomment `hulypulse` in `compose.yml` (default `HULY_BACKEND=memory` is fine for single-node).
2. `transactor`: `PULSE_URL=http://hulypulse:8099`
3. `front`: `PULSE_URL=http${SECURE:+s}://${HOST_ADDRESS}/_pulse` (match your URL scheme and proxy path).
4. Uncomment `location /_pulse` in `.huly.nginx`, reload nginx, recreate stack.

**Optional:** Uncomment `redis` and switch HulyPulse to Redis for multi-node/HA (see README).

---

## 4. Love service (LiveKit)

**References:** `README.md` — *Love Service (Audio & Video calls)*.

1. Provision LiveKit (e.g. LiveKit Cloud): note `LIVEKIT_HOST` (e.g. `wss://…`), `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`.
2. Add `love` service (port `8096`) with `SECRET`, `ACCOUNTS_URL`, `DB_URL`, `STORAGE_*`, and LiveKit variables.
3. `front`: set `LIVEKIT_WS` to the same WebSocket base as `LIVEKIT_HOST` (see README).
4. Uncomment `location /_love` in `.huly.nginx`.

`front` already references `LOVE_ENDPOINT` in `compose.yml`; verify it matches the public URL pattern after nginx is enabled.

---

## 5. Print service

**References:** `README.md` — *Print Service*.

1. Add `print` container (`hardcoreeng/print:${HULY_VERSION}`, port `4005`) with `SECRET`, `ACCOUNTS_URL`, `STORAGE_CONFIG`, `STATS_URL`.
2. `front`: `PRINT_URL=http${SECURE:+s}://${HOST_ADDRESS}/_print`
3. Uncomment `/_print` in `.huly.nginx`.

---

## 6. Export service

**References:** `README.md` — *Export Service*.

1. Add `exports` container (`hardcoreeng/export:${HULY_VERSION}`, port `4009`) with `SERVICE_ID=export-service`, `DB_URL`, `SECRET`, `ACCOUNTS_URL`, `STORAGE_CONFIG`, `PORT=4009`.
2. `front`: `EXPORT_URL` → public URL for `/_export` (use `https://` or the same `SECURE` pattern as other services—be consistent with Traefik/TLS).
3. Uncomment `/_export` in `.huly.nginx`.

---

## 7. GitHub integration (GitHub App)

**References:** `README.md` — *GitHub Service* (not the same as *Configure GitHub OAuth*).

### Concepts

- **GitHub OAuth** (optional sign-in): OAuth App, `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` on `account`, callback `…/_account/auth/github/callback`.
- **GitHub Service** (issues/PR sync): **GitHub App** with webhook, PEM key, and the `github` container; public paths `/_github` and app URLs documented in README.

Environment variables typically include:

| Variable | Purpose |
|----------|---------|
| `GITHUB_APPID` | Numeric App ID (General → About). |
| `GITHUB_APPNAME` | App **slug** (URL-safe name). |
| `GITHUB_CLIENTID` / `GITHUB_CLIENT_ID` | Client ID (standardize naming in compose). |
| `GITHUB_CLIENT_SECRET` | Client secret from the app. |
| `GITHUB_PRIVATE_KEY` | PEM contents (multiline; often file-mounted). |
| `GITHUB_WEBHOOK_SECRET` | Random shared secret for webhooks (not a PAT). |

### Replacing an old app name (e.g. legacy `sonarqube-*` slug)

1. In GitHub: **Settings → Developer settings → GitHub Apps → New GitHub App** (or org settings).
2. Choose a **new unique GitHub App name** → this becomes the new **`GITHUB_APPNAME` (slug)**.
3. Set **Homepage URL** to your Huly public origin (e.g. `https://<HOST_ADDRESS>`).
4. **Callback URL** and **Setup URL** (with redirect): `https://<HOST_ADDRESS>/github` (per README).
5. **Webhook URL**: `https://<HOST_ADDRESS>/_github/api/webhook`
6. **Webhook secret**: generate a strong random string; set the same value in GitHub and in the `github` container (`WEBHOOK_SECRET`).
7. Apply **permissions** and **events** exactly as listed in README (*Configure Permissions*, *Subscribe to Events*).
8. Create a **client secret**; generate a **private key** (PEM) and store securely → `GITHUB_PRIVATE_KEY`.
9. **Install** the app on the org/account/repos that should sync.
10. Add `github` service to `compose.yml` per README (`APP_ID`, `CLIENT_ID`, `CLIENT_SECRET`, `PRIVATE_KEY`, `WEBHOOK_SECRET`, `BOT_NAME`, etc.).
11. `front`: `GITHUB_URL`, `GITHUB_APP`, `GITHUB_CLIENTID` (match variable names in **`kcsrms.env`** / Dokploy).
12. Uncomment `/_github` in `.huly.nginx`; recreate services.

Deprecate or delete the old GitHub App after traffic uses the new endpoints.

---

## Post-change verification

1. Redeploy or `docker compose up -d --force-recreate` (with env from Dokploy or `--env-file kcsrms.env`).
2. `docker ps` (or Dokploy container view) — all new containers healthy.
3. Logs: `docker logs <service>` for `mail`, `notification`, `github`, `love`, `hulypulse`, `print`, `exports`.
4. Browser: exercise sign-in, notifications, mail, calls, print/export, and GitHub connect flows through the **public hostname** Traefik serves (same URL users use day to day).

---

## Related files

| File | Role |
|------|------|
| `compose.yml` | Service definitions and env wiring |
| `kcsrms.env` | Canonical compose env for this repo (tracked); copy into Dokploy |
| `.huly.nginx` | Path routing to internal services |
| `README.md` | Authoritative snippets for each optional service |
| `guides/gmail-configuration.md` | Gmail OAuth for in-app mail (separate from SMTP mail service) |

---

*Generated as a planning artifact for the huly-selfhost deployment. Update this file when the stack or README changes.*

# Obtain a Huly account JWT (login token)

Use this when you need a **bearer token** for scripts, API calls, or tools (for example an MCP server) that accept a Huly JWT. The token is issued by the **account** service after a successful **`login`** request.

This deployment exposes the account API at **`/_accounts`** on the same host as the web app (see `.huly.nginx`).

## 1. Create a JSON request file

Create a file (for example `.tmp-signin.json` in the repo root) with this shape:

```json
{
  "method": "login",
  "params": {
    "email": "you@example.com",
    "password": "your-password"
  }
}
```

### Important

- The method must be **`login`**. A method named `signIn` is **not** supported by the account service and returns `UnknownMethod`.
- **Do not commit** this file. This repository’s `.gitignore` includes `.tmp-signin.json`; use another ignored path or a secrets manager if you prefer a different filename.

## 2. Call the account endpoint with curl

Replace `YOUR_HULY_HOST` with your real origin (no trailing path except as shown).

**Windows (use `curl.exe` so PowerShell does not alias `curl` to `Invoke-WebRequest`):**

```powershell
curl.exe -sS -X POST "https://YOUR_HULY_HOST/_accounts/" `
  -H "Content-Type: application/json" `
  --data-binary "@C:\path\to\.tmp-signin.json"
```

**Linux or macOS:**

```bash
curl -sS -X POST "https://YOUR_HULY_HOST/_accounts/" \
  -H "Content-Type: application/json" \
  --data-binary "@/path/to/.tmp-signin.json"
```

**Optional — write the response to a file instead of the terminal:**

```powershell
curl.exe -sS -X POST "https://YOUR_HULY_HOST/_accounts/" `
  -H "Content-Type: application/json" `
  --data-binary "@.tmp-signin.json" `
  -o "$env:TEMP\huly-login-response.json"
```

## 3. Read the token from the response

On success the body is JSON with a **`result`** object. The JWT is in **`result.token`**.

Example (fields may vary by version):

```json
{
  "result": {
    "account": "<uuid>",
    "token": "<jwt>",
    "name": "Your Name",
    "socialId": "<id>"
  }
}
```

Use the token in the `Authorization` header:

```http
Authorization: Bearer <paste result.token here>
```

**Extract `token` with jq (if installed):**

```bash
jq -r '.result.token' huly-login-response.json
```

## 4. PowerShell alternative (no JSON file)

```powershell
$body = '{"method":"login","params":{"email":"you@example.com","password":"your-password"}}'
Invoke-RestMethod -Uri "https://YOUR_HULY_HOST/_accounts/" -Method Post -ContentType "application/json" -Body $body
```

## Troubleshooting

| Symptom | Likely cause |
| ------- | ------------ |
| `UnknownMethod` / `signIn` | Use **`login`**, not `signIn`. |
| `AccountNotFound` or invalid credentials | Wrong email or password. |
| 404 on `/_accounts/` | Reverse proxy path differs; check your nginx config for the account `location`. |
| curl errors on Windows with inline `-d` JSON | Prefer **`--data-binary @file`** or `Invoke-RestMethod` to avoid quoting issues. |

## Security

- The JWT grants access to your account; treat it like a password.
- Remove or clear the JSON file after use; prefer environment variables or a secret store for automation.
- If the password or token was ever committed or shared, **change the password** and obtain a new token.

## Related

- CI example (signup + bearer call): `.github/workflows/main.yaml`
- Account route: `.huly.nginx` (`location /_accounts`)

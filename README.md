# ChatGPT_team.py

Standalone single-file script for:

1. **Registering/logging into ChatGPT Web** through the current enterprise SSO flow
2. **Completing Codex OAuth**
3. **Saving the first usable Codex refresh token**

This file is intentionally self-contained:

- No dependency on `chatgpt_register.py`
- No dependency on `codex_login_extract.py`
- No dependency on `hero_sms.py`
- No stock-upload / merchant-sync logic

---

## Install

```bash
python -m pip install --upgrade pip
python -m pip install curl_cffi
```

---

## Minimal Config

Create `ChatGPT_team.config.local.json` next to this file:

```json
{
  "proxy": "http://127.0.0.1:7890"
}
```

If you do not need a proxy, you can omit the file and run directly.

---

## Usage

### Register and save refresh token

```bash
python ChatGPT_team.py --total 1 --workers 1
```

### Batch registration (10 accounts, 5 concurrent)

```bash
python ChatGPT_team.py --total 10 --workers 5
```

### Check existing `codex_tokens/*.json` plan type

```bash
python ChatGPT_team.py --check-tokens
```

When `--check-tokens` sees a `401`, it will:

1. Load the matching local session from `chatgpt_sessions/`
2. Run Codex OAuth again
3. Overwrite the original token file with the fresh RT/AT/ID token

---

## Command Line Options

| Flag | Default | Description |
|------|---------|-------------|
| `-p`, `--proxy` | from config | Proxy URL |
| `-n`, `--total` | `1` | Number of accounts to register |
| `-w`, `--workers` | `1` | Number of concurrent threads |
| `-o`, `--output` | `registered_only.txt` | Output file for successful accounts |
| `--check-tokens` | `false` | Check existing token files and auto-refresh on 401 |

---

## Outputs

| File/Dir | Description |
|----------|-------------|
| `registered_only.txt` | Format: `email----password----rt----refresh_token` |
| `register_only_failed.txt` | Failed registration logs |
| `chatgpt_sessions/` | Session cache per email (JSON) |
| `codex_tokens/` | Codex refresh/access/id token per email (JSON) |

---

## Config Options

| Key | Type | Description |
|-----|------|-------------|
| `proxy` | `string` | Main proxy URL |
| `proxy_chain_enabled` | `bool` | Enable proxy chain |
| `proxy_chain_upstream_proxy` | `string` | Upstream proxy template (`{session}` will be replaced) |
| `proxy_chain_dynamic_per_account` | `bool` | Create sticky proxy per account |

Also available via environment variables:

```bash
export PROXY="http://127.0.0.1:7890"
export PROXY_CHAIN_ENABLED=true
export PROXY_CHAIN_UPSTREAM_PROXY="http://user-region-{session}-t-5:pass@host:port"
export PROXY_CHAIN_DYNAMIC_PER_ACCOUNT=true
```

---

## Registration Flow

```
1. Visit homepage → get cookies
2. GET CSRF token
3. POST signin → get auth URL
4. GET auth URL → handle SSO redirect
5. (if SSO) → complete_sixoner_external_flow → authorize_continue → consent
6. (if about-you) → create_account
7. GET /api/auth/session → get access_token
8. Save session cache
9. PATCH onboarding (role=engineering)
10. Codex OAuth: GET /oauth/authorize → follow redirects → extract code
11. POST /oauth/token → get refresh_token
12. Save token to file
```

## Token Check Flow

```
1. Read all codex_tokens/*.json
2. GET WHAM usage endpoint with access_token
3. If 200 → display plan
4. If 401 → auto refresh:
   a. Load session cache
   b. Run Codex OAuth again
   c. Overwrite token file
   d. Re-verify
```

---

## Directory Structure

```
.
├── ChatGPT_team.py
├── ChatGPT_team.config.local.json   # optional
├── chatgpt_sessions/                 # auto-created
│   └── user@example.com.json
├── codex_tokens/                     # auto-created
│   └── user@example.com.json
├── registered_only.txt               # auto-created
└── register_only_failed.txt          # auto-created
```

---

## Environment Variables

| Variable | Description |
|----------|-------------|
| `PROXY` | Default proxy URL |
| `PROXY_CHAIN_ENABLED` | `true/false` — enable chain proxy |
| `PROXY_CHAIN_UPSTREAM_PROXY` | Upstream proxy template |
| `PROXY_CHAIN_DYNAMIC_PER_ACCOUNT` | `true/false` — sticky session per account |

---

## Tech Stack

- **curl_cffi** — TLS fingerprint impersonation (anti-bot bypass)
- **Sentinel token generator** — OpenAI proof-of-work
- **PKCE** — OAuth code challenge (S256)
- **Proxy chain** — LOCAL → MAIN PROXY → UPSTREAM PROXY

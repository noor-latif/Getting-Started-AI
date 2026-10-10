# RESEARCH DIGEST — verified 2026-10-09 (source of truth for README/ROADMAP/skills writers)

Every claim below was verified against live docs this session. `[UNVERIFIED]` marks the exceptions.

## 1. Accounts, keys, costs (the complete list)

| # | Account | Cost | What to create | Env var |
|---|---------|------|----------------|---------|
| 1 | Surplus Intelligence — https://www.surplusintelligence.ai | $0 signup; prepaid usage (USDC on Base or card credit). **The USER provides the working key — friend creates NO account.** | Nothing. Key `inf_`-prefixed | `SURPLUS_API_KEY` |
| 2 | Groq — https://console.groq.com (key at console.groq.com/keys) | Free tier, no card | Account + 1 API key | `GROQ_API_KEY` |
| 3 | Telegram @BotFather — https://t.me/BotFather | Free | 1 bot via `/newbot` (username must end `bot`); token format `123456789:ABCdef...`, shown once, `/revoke` if leaked | `TELEGRAM_BOT_TOKEN` |
| 4 | Tailscale — https://login.tailscale.com | Personal plan free: up to 6 users, unlimited user devices, no card | Free tailnet; Windows client (https://tailscale.com/docs/install/windows) + Android app (com.tailscale.ipn); same login on both | none (GUI login) |
| 5 | HostUp (OPTIONAL, only for custom-domain routing) — https://hostup.se | Domain at-cost (**.se sold natively here** — Cloudflare Registrar does NOT sell .se); DNS management included; same account can spin up cheap VPSes later | Account; API key with scopes `read:dns`, `write:dns`, `read:domains` (what the lego hostup DNS-01 provider requires) | `HOSTUP_API_KEY` |
| 6 | Edge TTS | $0, no key, no account | Nothing — Hermes default TTS | none |
| 7 | Hermes Agent itself | $0 open source; Nous Portal is OPTIONAL paid — NOT used here | No account | — |

- Groq free STT limits (whisper-large-v3-turbo): 20 RPM, 2,000 req/day, 28,800 audio-sec/day (8h), 25 MB/file — vastly more than one user's voice memos need.
- Per-tool keys (FIRECRAWL_API_KEY, BROWSER_USE_API_KEY, FAL_KEY, ELEVENLABS_API_KEY) are all OPTIONAL — leave unset.
- Telegram `TELEGRAM_ALLOWED_USERS` = friend's numeric user id (get from @userinfobot). NEVER `TELEGRAM_ALLOW_ALL_USERS`.
- Secrets live ONLY in `%LOCALAPPDATA%\hermes\.env` — not Windows user env vars.

## 2. Hermes Agent on native Windows (Tier 1, x64)

- Install (PowerShell, NO admin): `iex (irm https://hermes-agent.nousresearch.com/install.ps1)`
- Installs to `%LOCALAPPDATA%\hermes\` (code `hermes-agent\`, launchers `bin\`, data/HERMES_HOME = same root: config.yaml, .env, sessions, logs).
- Bundles everything: uv, Python 3.14, Node.js, npm, ripgrep, **FFmpeg**, Git for Windows pin. No prerequisites.
- Post-install: open a NEW PowerShell window (PATH refresh); verify `Get-Command hermes`, `hermes --version`.
- `'hermes' is not recognized` → new window, or `& "$env:LOCALAPPDATA\hermes\bin\hermes.exe"`.
- Never delete `%LOCALAPPDATA%\hermes` to repair — it holds user data; use `hermes uninstall` (check `--dry-run` first).
- MSIX desktop app exists but needs Win11 22H2+; PowerShell path covers Win10+11 x64.
- Runs shell commands through Git Bash; UTF-8 handled automatically (use Windows Terminal if output garbles).

### LLM = Surplus Intelligence (cloud only)
Independent config, verified pattern (providers docs, Named Custom Providers):
```yaml
# %LOCALAPPDATA%\hermes\config.yaml
providers:
  surplus:
    api: https://api.surplusintelligence.ai/v1
    key_env: SURPLUS_API_KEY
model:
  default: deepseek-v4.1-flash   # exact string, dot before 1
  provider: custom:surplus
```
- `/v1/models` verified live (430 models incl. deepseek-v4.1-flash).
- Alternative: `hermes model` → "Custom endpoint" → base URL + key + model (persists to config.yaml).
- Mid-session switch: `/model custom:surplus:deepseek-v4.1-flash`.

### STT = Groq Whisper (independent of LLM)
```yaml
stt:
  enabled: true
  provider: groq        # MUST pin — auto-fallback order is local > groq > openai
  groq:
    language: ''        # blank = auto-detect
```
Default model whisper-large-v3-turbo. `GROQ_API_KEY=gsk_...` in .env.

### TTS = Edge TTS (free default)
```yaml
tts:
  provider: edge
  edge:
    voice: en-US-AriaNeural   # 322 voices, 74 languages
```
No key. ffmpeg needed for MP3→Opus (Telegram voice bubble) — installer provisions it. Without ffmpeg the reply arrives as a rectangular audio file, still works.

### Telegram gateway
- `hermes gateway setup` → select Telegram → paste token + user id (writes .env).
- Easiest: dashboard Messaging → Telegram "Create with QR" (creates bot, detects user id, writes config).
- Start: `hermes gateway` (outbound long-polling — no ports, no webhook, works behind home NAT).
- **24/7 autostart: `hermes gateway install`** → schtasks ONLOGON task (Hermes_Gateway, /RL LIMITED, no admin/UAC), Startup-folder fallback. Manage: `hermes gateway status|start|stop|restart|uninstall`. Installed ONLY on explicit action. Verify with `hermes gateway status` after first reboot (older-install task bugs fixed in v2026.7.1; fresh installs get hardened XML registration).
- Voice commands in bot chat: `/voice on` (spoken reply when you send voice), `/voice tts` (spoken reply to everything), `/voice off`, `/voice status`. Persists across restarts.
- Windows caveats: Telegram photo-send "invalid characters" bug = raw backslash paths in custom plugins (Hermes normalizes its own); Bot API 20 MB getFile cap (voice memos are tiny — non-issue); Groq 25 MB STT cap with chunking.

## 3. Pocket ID v2.18.0 (2026-10-04) — Windows single binary (PREFERRED over Docker)

- Asset: https://github.com/pocket-id/pocket-id/releases/latest/download/pocket-id_windows_amd64.exe (checksums.txt alongside; arm64 exe exists — NOT for x64).
- Single Go binary, embedded frontend, built-in SQLite (data/ dir created in working dir: pocket-id.db + uploads/).
- Required env (`.env` next to exe, godotenv autoload): `APP_URL` (exact final URL, no path — passkeys bind to it; changing later invalidates passkeys) + `ENCRYPTION_KEY` (≥16 bytes, e.g. `openssl rand -base64 32`).
- `PORT` default 1411; `DB_CONNECTION_STRING` default SQLite; `PUBLIC_URL` DOES NOT EXIST (renamed APP_URL in v1); `APP_ENV` exists but unnecessary — omit. `TRUST_PROXY=127.0.0.1` when behind local Caddy. `STATIC_API_KEY` (≥16 chars) enables headless admin API for OIDC client creation.
- Run: `.\pocket-id.exe`. Windows Firewall will prompt on first run — pre-authorize inbound 1411 (`netsh advfirewall`) to avoid interactive popup `[INFERENCE: standard Windows behavior]`.
- First-run: open `APP_URL/setup` — create account + enroll passkey. **This step is inherently interactive (one human/browser step). NO default admin credentials exist.**
- Recovery (passkey lost): `.\pocket-id.exe one-time-access-token <username>` → prints /lc/<token> URL valid 1h.
- HTTPS MANDATORY for passkeys (WebAuthn secure context) except localhost. Options: Tailscale serve (`tailscale serve --bg 1411` → https://machine.tailnet.ts.net, auto LE cert, set APP_URL to it) — simplest; or Caddy `id.example.com { reverse_proxy localhost:1411 }`; or native TLS via TLS_CERT_FILE/TLS_KEY_FILE (redundant with tailscale serve).
- OIDC client: Administration → OIDC Clients → Add (name, callback URL from app's docs) → get client ID/secret/discovery URL `https://id.example.com/.well-known/openid-configuration`. **New clients allow NO ONE by default** — Access tab → All Users or group. Scopes: openid profile email.
- Upgrade: replace exe, restart; migrations automatic; downgrades blocked without ALLOW_DOWNGRADE=true. Docker alternative (only if Docker Desktop already present): official compose `pocketid/pocket-id:v2`, port 1411, ./data volume — but binary wins for agent automation (no WSL2 dependency).

## 4. Private web routing — two options

### Option A: HostUp DNS + lego DNS-01 wildcard + Caddy (custom domain)
- Domain: buy/register/transfer at HostUp (hostup.se) — `.se` sold natively (Cloudflare Registrar does NOT sell .se); DNS zone auto-created.
- HostUp API key scopes: `read:dns`, `write:dns`, `read:domains` (exactly what lego's hostup provider requires). DNS records API: `GET/POST /api/v2/dns-zones/{id}/records` (https://developer.hostup.se).
- Cert path: **lego** (go-acme) has a native HostUp DNS-01 provider since v5.0 (https://go-acme.github.io/lego/dns/hostup/). lego is NOT in the winget catalog — install from GitHub releases zip (`lego_v<ver>_windows_amd64.zip` → `C:\caddy\lego\lego.exe`). Key via file: `HOSTUP_API_KEY_FILE=C:\caddy\secrets\hostup_api_key` (hostup provider supports `_FILE` env suffixes). ALWAYS pin `--path C:\caddy\lego` on run AND renew so Caddy serves the renewed files: `lego --path C:\caddy\lego --dns hostup -d "*.example.com" -d example.com run` / `... renew --days 30` (daily Task Scheduler + `caddy reload`).
- **Because lego fetches the cert, a STANDARD Caddy build works** (`winget install CaddyServer.Caddy`) — no download-builder module, no xcaddy. Caddy serves the lego cert files (`auto_https off`, `tls <crt> <key>`).
- HostUp DNS: wildcard A record `*` → PC's Tailscale IP (`tailscale ip -4`). Optional AAAA with `tailscale ip -6`. Specific records override the wildcard.
- Caddyfile (adjust domain/ports):
```
{
    auto_https off
}
*.example.com:443 {
    tls C:\caddy\lego\certificates\_.example.com.crt C:\caddy\lego\certificates\_.example.com.key
    @agent host agent.example.com
    handle @agent { reverse_proxy 127.0.0.1:PORT }
}
```
- Always-on: run as Windows service (NSSM) or Task Scheduler at startup.
- Direct route: Caddy `reverse_proxy` accepts full host:port for cross-machine targets.

### Option B (LAZY PATH — no domain, no Caddy, no HostUp): Tailscale serve
- Prereqs (one-time, Tailscale admin console → DNS): MagicDNS enabled + HTTPS Certificates enabled.
- `tailscale serve --bg 3000` (= `--bg --https=443 localhost:3000`) → https://machine.tailnet-name.ts.net — persists across reboots, auto Let's Encrypt cert for the ts.net name, public CA-trusted.
- Reset: `tailscale serve reset`; status: `tailscale serve status`. Public exposure variant: `tailscale funnel 3000`.
- Limits: fixed *.ts.net hostname, no arbitrary-hostname wildcard routing, tailnet name leaks in URL, only localhost backends in reverse-proxy mode.
- **Recommendation: Option B first (zero cost, 2 commands), Option A only if friend buys a domain.**

## 5. Full source list
- https://hermes-agent.nousresearch.com/docs/ (+ /getting-started/installation, /user-guide/windows-native, /integrations/providers, /user-guide/messaging/telegram, /user-guide/features/voice-mode)
- https://github.com/NousResearch/hermes-agent (issues #35692, PR #45610)
- https://pocket-id.org/docs/ (+ setup/installation, setup/reverse-proxy, setup/connect-an-app, configuration/environment-variables, troubleshooting/account-recovery)
- https://github.com/pocket-id/pocket-id/releases (v2.18.0)
- https://developer.hostup.se (API v2: dns-zones/records, scopes), https://go-acme.github.io/lego/dns/hostup/ (lego hostup DNS-01 provider, since v5.0), https://hostup.se/en/support/guide-for-hantering-av-dns/
- https://caddyserver.com/download (standard build sufficient in the lego path)
- https://tailscale.com/docs/reference/tailscale-cli/serve, https://tailscale.com/kb/1242/tailscale-serve
- https://console.groq.com/docs/rate-limits, /docs/deprecations
- https://api.surplusintelligence.ai/v1/models (live 200, deepseek-v4.1-flash present)
- HostUp domain pricing: client area (at-cost; .se natively supported)

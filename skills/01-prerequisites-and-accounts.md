# 01 — Prerequisites and Accounts

## Purpose
Create every account and key needed for the whole setup, before installing anything. Total cost: $0 (LLM usage is prepaid by the key already in hand).

## Prerequisites
- Windows 10/11 x64 PC.
- A web browser and the Telegram app (phone or desktop).
- The friend's `SURPLUS_API_KEY` (starts with `inf_`). **The friend already has this — no Surplus account is created by anyone in this step.**

## Accounts to create (complete list)

| # | Account | Cost | Signup URL | Env var | Who does it |
|---|---------|------|------------|---------|-------------|
| 1 | Surplus Intelligence | $0 signup, prepaid usage | https://www.surplusintelligence.ai | `SURPLUS_API_KEY` | Nobody — key handed over by friend |
| 2 | Groq | Free tier, no card | https://console.groq.com | `GROQ_API_KEY` | HUMAN STEP |
| 3 | Telegram bot | Free | https://t.me/BotFather | `TELEGRAM_BOT_TOKEN` | HUMAN STEP |
| 4 | Tailscale | Free (≤6 users, no card) | https://login.tailscale.com | none (GUI login) | HUMAN STEP |
| 5 | HostUp | Domain at-cost (.se natively supported); DNS included | https://hostup.se | `HOSTUP_API_KEY` | OPTIONAL — skip now, needed only for custom-domain routing (skills 05–06). Key scopes: `read:dns`, `write:dns`, `read:domains` |
| 6 | Edge TTS | $0, no key | none | none | Nothing to do — Hermes default TTS |

## Steps

### 1. Receive the Surplus key (no signup)
- **HUMAN STEP (friend):** hand over the `inf_`-prefixed key. Store it now in a private note; it goes into `.env` in skill 02.
- Do NOT put it in Windows user environment variables. Secrets live ONLY in `%LOCALAPPDATA%\hermes\.env`.

### 2. Groq (STT — voice transcription)
1. **HUMAN STEP:** open https://console.groq.com, sign up (free, no card).
2. **HUMAN STEP:** go to https://console.groq.com/keys → create one API key → copy it (starts `gsk_`).
3. Free-tier limits (whisper-large-v3-turbo): 20 requests/min, 2,000/day, 28,800 audio-sec/day (8 h), 25 MB/file — far more than one person's voice memos need.

### 3. Telegram bot
1. **HUMAN STEP:** open Telegram, chat with https://t.me/BotFather.
2. **HUMAN STEP:** send `/newbot`, follow prompts. Bot username must end in `bot`.
3. BotFather replies with the token, format `123456789:ABCdef...`. It is shown ONCE — copy it immediately.
4. **HUMAN STEP:** get the friend's numeric Telegram user id: message @userinfobot — it replies with the id. That number is the value of `TELEGRAM_ALLOWED_USERS`.
5. If the token ever leaks: send `/revoke` to BotFather, use the new token.

### 4. Tailscale (private access, used in skills 04–06)
1. **HUMAN STEP:** create a free account at https://login.tailscale.com.
2. **HUMAN STEP (PC):** install the Windows client — https://tailscale.com/docs/install/windows — and log in with the same account.
3. **HUMAN STEP (phone):** install the Android app (`com.tailscale.ipn`) and log in with the SAME account.
4. One tailnet, both devices. No env var — it is a GUI login.

### 5. Where every value lands
Final home for secrets: `%LOCALAPPDATA%\hermes\.env` (created in skill 02):

```
SURPLUS_API_KEY=inf_...
GROQ_API_KEY=gsk_...
TELEGRAM_BOT_TOKEN=123456789:ABCdef...
TELEGRAM_ALLOWED_USERS=123456789
```

Optional, only for skills 05–06: the HostUp API key does NOT go in this .env and NOT in system env vars — skill 05 stores it in `C:\caddy\secrets\hostup_api_key` and points lego at it via `HOSTUP_API_KEY_FILE`. Get the key (scopes `read:dns`, `write:dns`, `read:domains`) from the HostUp dashboard when Phase 4 runs.

## Security rules (non-negotiable)
- Hermes secrets ONLY in `%LOCALAPPDATA%\hermes\.env`. Never in Windows user/system env vars, never in chat logs, never in screenshots.
- NEVER set `TELEGRAM_ALLOW_ALL_USERS`. Only `TELEGRAM_ALLOWED_USERS` with the friend's numeric id.
- ONE exception, introduced only if Phase 4 runs: the HostUp key lives in `C:\caddy\secrets\hostup_api_key` (a plain file), because lego is not a Hermes process and scheduled tasks must read it headlessly. It never goes into Hermes' .env, system env vars, or anywhere else. Set up in skill 05, not here.
- Telegram token leaked → `/revoke` in BotFather immediately, replace in `.env`, restart gateway.
- Per-tool keys (`FIRECRAWL_API_KEY`, `BROWSER_USE_API_KEY`, `FAL_KEY`, `ELEVENLABS_API_KEY`) are optional — leave unset.

## Verification
Run in PowerShell, confirm all four lines print a non-empty value (substitute your real path if `.env` doesn't exist yet — then this check runs after skill 02):

```powershell
Get-Content "$env:LOCALAPPDATA\hermes\.env" | Select-String '^(SURPLUS_API_KEY|GROQ_API_KEY|TELEGRAM_BOT_TOKEN|TELEGRAM_ALLOWED_USERS)='
```

- Exactly 4 matches, none with an empty value after `=`.
- `SURPLUS_API_KEY` starts `inf_`, `GROQ_API_KEY` starts `gsk_`, `TELEGRAM_BOT_TOKEN` matches `^\d+:[A-Za-z0-9_-]+$`.
- `TELEGRAM_ALLOWED_USERS` is the number @userinfobot reported.

If `.env` doesn't exist yet, park the four values securely and verify after skill 02 creates it.

## Rollback / Troubleshooting
- **BotFather token lost before saving:** send `/revoke` to BotFather → new token → use it. (Do this anyway if it was pasted anywhere public.)
- **Wrong user id in `TELEGRAM_ALLOWED_USERS`:** bot ignores messages. Re-check with @userinfobot — must be the numeric id, not the @username.
- **Groq key not working later:** rate limits (20 RPM / 2,000/day) — see skill 03 troubleshooting.
- **Created a Surplus account by accident:** harmless; the handed-over key is still what gets used.

---
Source: RESEARCH.md §1 (Accounts, keys, costs).

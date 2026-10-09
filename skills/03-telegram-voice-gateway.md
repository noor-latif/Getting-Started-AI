# 03 — Telegram Voice Gateway (24/7)

## Purpose
Connect the Hermes bot to Telegram so the father can text or send voice memos from his phone and get replies (text or spoken) — running 24/7 on the PC with automatic start at logon.

## Prerequisites
- Skill 02 done: `hermes --version` works, chat round-trip OK.
- From skill 01: `TELEGRAM_BOT_TOKEN` (BotFather), `TELEGRAM_ALLOWED_USERS` (friend's numeric id from @userinfobot).
- No ports, no admin, no port-forwarding needed: the gateway uses outbound long-polling and works behind home NAT.

## Steps

### 1. Link the bot
Either path works:

- **CLI:** `hermes gateway setup` → select Telegram → paste the bot token and the numeric user id. It writes both into `%LOCALAPPDATA%\hermes\.env`.
- **Easiest — dashboard QR:** open the Hermes dashboard → Messaging → Telegram → "Create with QR". It creates the bot, detects the user id, and writes the config for you.

Result in `.env` (check either way):
```
TELEGRAM_BOT_TOKEN=123456789:ABCdef...
TELEGRAM_ALLOWED_USERS=123456789
```

### 2. Start the gateway (manual first run)
```powershell
hermes gateway
```
Leave it running.

### 3. Test, then enable 24/7 autostart
1. From the friend's Telegram account, DM the bot a text message → bot replies.
2. Once confirmed, install autostart:
```powershell
hermes gateway install
```
- Creates a Windows scheduled task `Hermes_Gateway` (`schtasks`, ONLOGON trigger, `/RL LIMITED`) — no admin prompt, no UAC. (Startup-folder fallback exists if task creation fails.)
- Autostart is installed ONLY by this explicit command.

### 4. Manage the gateway
```powershell
hermes gateway status     # installed? running?
hermes gateway start      # start the scheduled task
hermes gateway stop       # stop it
hermes gateway restart    # restart
hermes gateway uninstall  # remove the task
```

### 5. Voice mode
In the bot chat, the father uses:
- `/voice on` — send a voice memo, get a spoken reply.
- `/voice tts` — every reply is spoken.
- `/voice off` — text only.
- `/voice status` — current mode.

Voice settings persist across gateway restarts.

### 6. How a voice memo flows (all free, all cloud)
Groq Whisper STT (whisper-large-v3-turbo, `GROQ_API_KEY`) → LLM (`deepseek-v4.1-flash` via Surplus) → Edge TTS (`en-US-AriaNeural`, no key) → MP3→Opus via FFmpeg → arrives as a Telegram voice bubble.
FFmpeg is provisioned by the Hermes installer (skill 02) — nothing to do.

### 7. Power settings — keep the PC awake (24/7 requirement)
- **HUMAN STEP (or agent via settings):** Settings → System → Power → set "When plugged in, put my device to sleep" to **Never** (desktop: Screen and sleep → Never). Disable hibernate/Modern Standby sleep for unattended hours.
- If the PC sleeps, the gateway stops until wake — voice messages pile up undelivered.

## Verification checklist
- [ ] Bot answers a plain text DM from the friend's account.
- [ ] Voice memo in → voice reply out (voice bubble, not a text dump).
- [ ] `/voice status` reports the expected mode; switching `/voice on|tts|off` works.
- [ ] `hermes gateway status` shows the scheduled task installed (after `hermes gateway install`).
- [ ] Reboot the PC, log in, wait ~1 minute: `hermes gateway status` shows running — gateway auto-started, no manual action.

## Rollback / Troubleshooting
- **Bot silent:** `TELEGRAM_ALLOWED_USERS` must be the numeric id (not @username) of the account actually messaging the bot. NEVER set `TELEGRAM_ALLOW_ALL_USERS`.
- **Token rejected:** leaked/revoked token → BotFather `/revoke`, paste new token into `.env`, `hermes gateway restart`.
- **Photos fail with "invalid characters":** known Telegram photo-send bug = raw backslash paths in custom plugins. Hermes normalizes its own paths — the bug only hits self-written third-party plugins. Avoid raw backslash paths there.
- **"File is too big":** Bot API getFile cap is 20 MB. Voice memos are tiny — non-issue for this use case. (Groq STT cap is 25 MB with chunking.)
- **Task didn't fire after reboot:** older installs had task bugs — fixed in v2026.7.1 (fresh installs get hardened XML registration). Update Hermes, then `hermes gateway uninstall` and `hermes gateway install` again; verify with `hermes gateway status` after a reboot.
- **Rollback:** `hermes gateway uninstall` removes the scheduled task; stop gateway with `hermes gateway stop`. Bot keeps existing, just offline.

---
Source: RESEARCH.md §2 (Telegram gateway, voice mode, Windows caveats).

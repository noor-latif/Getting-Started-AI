# 02 — Install Hermes Agent and Configure Cloud Providers

## Purpose
Install Hermes Agent natively on Windows (no WSL, no Docker), point it at Surplus Intelligence for the LLM, Groq Whisper for STT, and Edge TTS for speech.

## Prerequisites
- Windows 10/11 x64. PowerShell. NO admin rights needed.
- Values from skill 01: `SURPLUS_API_KEY` (`inf_...`), `GROQ_API_KEY` (`gsk_...`).

## Steps

### If Hermes is already installed (outdated version)
1. Check version: `hermes --version` (new window if `'hermes' is not recognized`).
2. Back up FIRST — copy, don't move: `Copy-Item "$env:LOCALAPPDATA\hermes\.env","$env:LOCALAPPDATA\hermes\config.yaml" "$env:USERPROFILE\hermes-backup\"` (create the folder first). NEVER delete `%LOCALAPPDATA%\hermes` to repair — it holds user data.
3. Update (verified: official docs, https://hermes-agent.nousresearch.com/docs/getting-started/updating — `hermes update` is the update path for a source/Windows-native install, which is what install.ps1 creates):
   ```powershell
   hermes update
   ```
   If `hermes update` cannot run on a very old install, re-running `iex (irm https://hermes-agent.nousresearch.com/install.ps1)` over an existing checkout is the documented rerun path ([agent must verify: installer preserves config.yaml/.env/sessions — docs say user data stays outside the package and "do not remove the data root to repair", but no explicit statement covers install.ps1 rerun]). The update itself snapshots config.yaml/.env before pulling (quick pre-update snapshot).
4. On a v2026.7.1+ build, refresh the autostart task: `hermes gateway uninstall`, then `hermes gateway install`; verify `hermes gateway status` (old task bugs fixed in v2026.7.1).
5. Reconcile the old `config.yaml` with options added since the installed version: `hermes config check`, then `hermes config migrate` (interactive; verified: https://hermes-agent.nousresearch.com/docs/getting-started/updating). The manual re-pin later in this skill only fixes the three values it pins — this picks up the rest.
6. The rest of skill 02 continues unchanged — steps 3–4 are idempotent: keep existing values, fix only what this skill pins (`model.default`, `stt.provider: groq`, `tts.provider: edge`).

### 1. Install
PowerShell (normal window, NO admin):

```powershell
iex (irm https://hermes-agent.nousresearch.com/install.ps1)
```

- Installs to `%LOCALAPPDATA%\hermes\` — code in `hermes-agent\`, launchers in `bin\`, data (HERMES_HOME: config.yaml, .env, sessions, logs) in the same root.
- Bundles everything: uv, Python 3.14, Node.js, npm, ripgrep, **FFmpeg**, Git for Windows pin. Nothing else to install.

**PATH gotcha:** the current window does not see `hermes` yet. Open a NEW PowerShell window after install (PATH refresh).

### 2. Verify install
```powershell
Get-Command hermes
hermes --version
```

### 3. Configure providers — `%LOCALAPPDATA%\hermes\config.yaml`
Create/edit the file with exactly this:

```yaml
providers:
  surplus:
    api: https://api.surplusintelligence.ai/v1
    key_env: SURPLUS_API_KEY

model:
  default: deepseek-v4.1-flash   # exact string — dot before the 1
  provider: custom:surplus

stt:
  enabled: true
  provider: groq                 # MUST pin — see below
  groq:
    language: ''                 # blank = auto-detect

tts:
  provider: edge
  edge:
    voice: en-US-AriaNeural
```

Why pin `stt.provider: groq`: Hermes auto-fallback order is local > groq > openai. Unpinned, it tries local whisper first (slow/heavy on a desktop PC) before reaching Groq. Pinning forces the free cloud STT every time.

### 4. Secrets — `%LOCALAPPDATA%\hermes\.env`
```
SURPLUS_API_KEY=inf_...
GROQ_API_KEY=gsk_...
```
(`TELEGRAM_BOT_TOKEN` and `TELEGRAM_ALLOWED_USERS` are added in skill 03 — either by `hermes gateway setup` or by hand.)

### 5. Verify model round-trip
1. In PowerShell run `hermes` to start a chat session.
2. Send any message ("say hi"). A reply must come back — the model in use is `deepseek-v4.1-flash`.
3. Type `/model` — it must list the active model as `custom:surplus` / `deepseek-v4.1-flash`.
4. Mid-session switch (if ever needed): `/model custom:surplus:deepseek-v4.1-flash`.

Alternative to editing config.yaml by hand: `hermes model` → "Custom endpoint" → paste base URL `https://api.surplusintelligence.ai/v1`, key, model `deepseek-v4.1-flash` (persists to config.yaml).

## Verification
- `hermes --version` prints a version (no `'hermes' is not recognized`).
- Chat round-trip returns a reply from `deepseek-v4.1-flash`.
- `/model` shows `custom:surplus`.
- `%LOCALAPPDATA%\hermes\config.yaml` contains the `providers.surplus`, `stt.provider: groq`, and `tts.provider: edge` blocks above.
- `.env` has non-empty `SURPLUS_API_KEY` and `GROQ_API_KEY`.

## Rollback / Troubleshooting
- **`'hermes' is not recognized`:** open a NEW PowerShell window (PATH refresh). Still broken → run the launcher directly: `& "$env:LOCALAPPDATA\hermes\bin\hermes.exe" --version`.
- **Garbled/UTF-8 broken output in shell commands:** use Windows Terminal — Hermes handles UTF-8 automatically through Git Bash, but the terminal must cooperate.
- **NEVER delete `%LOCALAPPDATA%\hermes` to "repair"** — it holds user data (sessions, logs, config). Repair = `hermes uninstall` (run with `--dry-run` first), then reinstall.
- **Model name typo** (`deepseek-v4.1` vs `deepseek-v4.1-flash`, missing dot): `/v1/models` lists the exact string; the default above is verified.
- **MSIX desktop app exists** but needs Win11 22H2+ — ignored here; the PowerShell path covers Win10 + Win11 x64.

---
Source: RESEARCH.md §2 (Hermes Agent on native Windows; LLM/STT/TTS blocks).

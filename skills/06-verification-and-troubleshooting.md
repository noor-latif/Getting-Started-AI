# Skill 06 — End-to-end verification & troubleshooting

## Purpose
Prove the whole setup works, in order, and fix the known failure modes. Run this after skills 01–05, after any reboot, and after updates.

## End-to-end acceptance checklist (run in order; stop at first failure, fix, re-run from that step)

1. **Hermes installed & on PATH** — new PowerShell window:
   ```powershell
   hermes --version
   ```
   Prints a version. If `hermes` is not recognized → Troubleshooting row 1.
2. **LLM round-trip (Surplus)** — start `hermes`, then:
   ```
   /model custom:surplus:deepseek-v4.1-flash
   ```
   Send any prompt ("say ok"). A coherent reply comes back from `deepseek-v4.1-flash`.
3. **Telegram text** — from the allowed user's Telegram account, DM the bot any text. The bot replies.
4. **Telegram voice** — hold the mic, record a voice memo, send. With `/voice on` (set in bot chat once), the bot replies as a voice bubble (Edge TTS, MP3→Opus via bundled ffmpeg).
5. **Gateway survives reboot** — after `hermes gateway install` was run and the PC is **rebooted**:
   ```powershell
   hermes gateway status
   ```
   Shows the gateway running (task Hermes_Gateway, ONLOGON). If not → Troubleshooting row 6.
6. **Private HTTPS from phone** — phone on the tailnet, open `https://<machine>.<tailnet-name>.ts.net` (Pocket ID, port 1411 mapping). Valid padlock, page loads.
7. **Pocket ID passkey login** — from that page, sign in with the enrolled passkey (Windows Hello or phone biometrics). Login succeeds.
8. **OPTIONAL Caddy wildcard** (only if Option A was built): `curl -I https://agent.example.com` from any device → valid wildcard cert + expected status.

## Troubleshooting table

| # | Symptom | Cause | Fix |
|---|---------|-------|-----|
| 1 | `'hermes' is not recognized` | PATH not refreshed after install, or install really missing | Open a NEW PowerShell window; else run `& "$env:LOCALAPPDATA\hermes\bin\hermes.exe"`. Never delete `%LOCALAPPDATA%\hermes` to "repair" — it holds user data; use `hermes uninstall` (check `--dry-run` first) |
| 2 | Garbled / mojibake output from Hermes shell commands | Terminal encoding | Use Windows Terminal; Hermes handles UTF-8 automatically otherwise |
| 3 | Custom plugin sending Telegram photos fails: "invalid characters" in path | Raw Windows backslash paths in custom plugins | Use forward slashes / normalize the path in the plugin. Hermes normalizes its own paths — bug only bites custom plugins |
| 4 | Bot can't fetch a >20 MB Telegram file | Bot API `getFile` cap = 20 MB | Keep files under 20 MB (voice memos are tiny — non-issue in normal use) |
| 5 | STT rejects a large audio file | Groq free-tier 25 MB per-file cap | Chunk the audio before sending to Whisper; under 25 MB nothing to do |
| 6 | `hermes gateway status` shows nothing after reboot | ONLOGON task not installed, older-install task bug, or the PC rebooted (e.g. Windows Update) and nobody signed in — ONLOGON needs a logon | Install it: `hermes gateway install`, then reboot AND sign in, re-check. v2026.7.1+ has hardened XML registration. For unattended survival after update-reboots see skill 03 step 3 (auto-logon caveat) |
| 7 | All Pocket ID passkeys suddenly rejected | `APP_URL` changed (passkeys are bound to it) | Restore the original URL in `%LOCALAPPDATA%\pocket-id\.env`, restart exe. Old passkeys stay invalid → log in with recovery: `.\pocket-id.exe one-time-access-token <username>` (prints `/lc/<token>` URL, valid 1h), re-enroll passkey |
| 8 | Locked out of Pocket ID, passkey lost/unusable | No second factor | Recovery token (row 7's command). If the admin account name is forgotten: [agent must verify: username recovery path — RESEARCH.md only documents the token command] |
| 9 | Pocket ID refuses to start after an older version was restored | Downgrade across a DB migration | migrations are forward-only; set `ALLOW_DOWNGRADE=true` only if you truly must roll back (accept the risk), or restore the matching newer exe |
| 10 | Wildcard cert issuance fails (lego) | Key file missing/unreadable, or key lacks scopes | Confirm `%LOCALAPPDATA%\caddy\secrets\hostup_api_key` exists and contains only the raw key; re-create the key with scopes `read:dns`, `write:dns`, `read:domains` (skill 05 step 2). Task must set `HOSTUP_API_KEY_FILE` |
| 11 | Custom-domain site unreachable; works via ts.net | Wildcard A record missing or points at the wrong IP | HostUp DNS: A record `*` → PC's Tailscale IP (`tailscale ip -4`). Verify with `Resolve-DnsName agent.example.com` |
| 12 | Pocket ID not reachable after reboot (browser/`Invoke-WebRequest` on the ts.net URL fails) until started manually | PocketID ONLOGON task not created (skill 04 step 9), or the PC rebooted (e.g. Windows Update) and nobody signed in — ONLOGON needs a logon (skill 03 caveat) | Re-run skill 04 step 9 (`Register-ScheduledTask -TaskName PocketID ... -WorkingDirectory "$env:LOCALAPPDATA%\pocket-id"`), then reboot AND sign in, re-check. For unattended survival after update-reboots see skill 03 step 3 |

## Maintenance

- **Hermes updates:** `hermes update` (verified: https://hermes-agent.nousresearch.com/docs/getting-started/updating — it snapshots config.yaml/.env, pulls code, and restarts gateways; the docs list it as the update path for source installs like the Windows-native one). Check first with `hermes uninstall --dry-run` only if you intend to remove the install. Do NOT delete `%LOCALAPPDATA%\hermes` — `.env`, config, sessions, logs live there. See skill 02's "If Hermes is already installed" branch.
- **Pocket ID upgrade:** stop the exe, replace `pocket-id.exe` with the new amd64 build (verify checksum), start it. Migrations run automatically. Never copy an older exe back without re-reading row 9.
- **Surplus key rotation:** if `SURPLUS_API_KEY` is ever leaked, revoke it in the provider dashboard and paste the new `inf_...` key into `%LOCALAPPDATA%\hermes\.env`, then restart Hermes. Same pattern for `GROQ_API_KEY` / `TELEGRAM_BOT_TOKEN` (`/revoke` in @BotFather).
- **BACKUP (critical):** `%LOCALAPPDATA%\hermes\.env` holds ALL Hermes secrets (Surplus, Groq, Telegram). `%LOCALAPPDATA%\hermes\data\` holds the SQLite DB. Also back up `%LOCALAPPDATA%\pocket-id\.env` (ENCRYPTION_KEY + APP_URL) and `%LOCALAPPDATA%\pocket-id\data\` (pocket-id.db + uploads), and Phase 4's `%LOCALAPPDATA%\caddy\secrets\hostup_api_key` if it exists. Lose these = re-enroll every passkey and re-create every key.

Source: RESEARCH.md §1–§4 caveats + maintenance notes.

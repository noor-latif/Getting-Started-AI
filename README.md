# Hermes Voice Assistant for Windows

An agent-executable roadmap for standing up a private AI assistant on a native Windows 10/11 x64 PC: **Hermes Agent** talking Telegram voice notes in and spoken replies out, auto-starting at sign-in, with Pocket ID single sign-on guarding private web services, all reachable from the phone through Tailscale. (True unattended 24/7 across Windows-Update reboots needs the skill 03 sign-in caveat — Telegram queues messages meanwhile.) Every cloud dependency is free-tier or prepaid by the owner; the roadmap is written so an AI agent can execute it top-to-bottom, with the few unavoidable human steps (bot token, signups, passkey enrollment) explicitly marked **HUMAN STEP**.

## Accounts & keys

Source: RESEARCH.md §1. Hermes secrets live ONLY in `%LOCALAPPDATA%\hermes\.env` — never Windows env vars. (Phase 4 exception: the HostUp key lives in a lego-owned file, skills 01/05.)

| # | Account | Who creates it | Cost | What to create | Env var |
|---|---------|----------------|------|----------------|---------|
| 1 | [Surplus Intelligence](https://www.surplusintelligence.ai) | **USER** hands over the working key (`inf_`-prefixed); the friend creates NO account | $0 signup; prepaid usage (USDC on Base or card credit) | Nothing | `SURPLUS_API_KEY` |
| 2 | [Groq](https://console.groq.com) (key at console.groq.com/keys) | Friend — signup is a **HUMAN STEP** | Free tier, no card | Account + 1 API key | `GROQ_API_KEY` |
| 3 | [Telegram @BotFather](https://t.me/BotFather) | Friend — **HUMAN STEP** (BotFather chat, `/newbot`, username must end `bot`; token shown once, `/revoke` if leaked) | Free | 1 bot + token `123456789:ABCdef...` | `TELEGRAM_BOT_TOKEN` |
| 4 | [Tailscale](https://login.tailscale.com) | Friend — **HUMAN STEP** (same login on PC and Android) | Personal plan free: up to 6 users, unlimited user devices, no card | Free tailnet; Windows client + Android app (com.tailscale.ipn) | none (GUI login) |
| 5 | [HostUp](https://hostup.se) (OPTIONAL, custom-domain routing only) | Friend — **HUMAN STEP** (optional; domain buy too) | Domain at-cost (.se natively supported — Cloudflare Registrar does NOT sell .se); DNS included | Account; API key with scopes `read:dns`, `write:dns`, `read:domains` (lego DNS-01 + dashboard DNS) | `HOSTUP_API_KEY_FILE` → `C:\caddy\secrets\hostup_api_key` (NOT the Hermes .env — skill 05) |
| 6 | Edge TTS | Nobody — no account, no key | $0 | Nothing — Hermes default TTS | none |
| 7 | Hermes Agent | Nobody | $0 open source; Nous Portal is OPTIONAL paid — NOT used here | No account | — |

Optional tool keys (`FIRECRAWL_API_KEY`, `BROWSER_USE_API_KEY`, `FAL_KEY`, `ELEVENLABS_API_KEY`) stay unset. Telegram `TELEGRAM_ALLOWED_USERS` = the friend's numeric user id (get it from @userinfobot) — NEVER `TELEGRAM_ALLOW_ALL_USERS`.

## Architecture

![Diagram, top to bottom: your phone (Telegram) sends a voice note down to Telegram servers, which relay it to Hermes Agent on your always-on Windows PC. Hermes calls three cloud voice services — 1 the ears (STT, Groq, free), 2 the brain (LLM, Surplus deepseek-v4.1-flash, prepaid), 3 the voice (TTS, Edge TTS, free) — and replies up the same chain. Below: your private apps (Caddy optional, Pocket ID passkey login, Tailscale private tunnel) reached from your phone.](docs/architecture.svg)

The solid blue arrows are **your voice going down**: you hold the mic in Telegram and talk → Telegram servers hold it (no open ports on your PC — Hermes polls outbound) → Hermes on the PC pulls it in. Hermes then calls the cloud helpers, numbered as in the diagram: **1 the ears** (STT — Groq, free) turn speech into text, **2 the brain** (LLM — Surplus's `deepseek-v4.1-flash`, prepaid by you) thinks, **3 the voice** (TTS — Edge, free) turns the answer into speech. The dotted arrows are **the reply going back up** the same chain, landing as a voice bubble in Telegram. Everything else on the PC — Pocket ID (one passkey for private logins), Tailscale (encrypted tunnel), optional Caddy (custom-domain proxy) — exists so your phone can reach your private apps from anywhere without exposing them to the internet.

## Why native Windows — no WSL2, no Linux emulation

WSL2 is built by Microsoft for inner-loop development, not always-on servers: its VM shuts down ~60 s after the last terminal closes (`vmIdleTimeout`), and it cannot be started headless at boot — Task Scheduler "run whether logged on or not" lands in Session 0 and gets "Access is denied" (microsoft/WSL #9231, acknowledged by-design). Windows sleep also breaks WSL networking (#12747) and skews its clock (official megathread #10006) — fatal for TLS and scheduled jobs.

This setup needs one boring, reliable thing: a process that survives reboots and stays online. Native Windows + Hermes' own installer + a logon scheduled task does exactly that with zero emulation layers.

Linux friends don't need this repo at all: Hermes is cross-platform — they run the Linux installer directly on their own machine.

## Repo map

| File | Covers |
|---|---|
| `CONTEXT.md` | Read FIRST if you are an AI agent executing this: mental model, non-negotiables, decided answers, failure etiquette |
| `RESEARCH.md` | Single source of truth — verified facts, URLs, costs, commands. |
| `skills/01-prerequisites-and-accounts.md` | Windows 10/11 x64 checks, PowerShell (no admin), accounts/keys from §1, secret handling, .env rules |
| `skills/02-install-hermes.md` | Hermes PowerShell installer, PATH verification, Surplus LLM config (config.yaml), Groq STT, Edge TTS |
| `skills/03-telegram-voice-gateway.md` | `hermes gateway setup` (Telegram), voice mode `/voice on|tts|off`, `hermes gateway install` schtasks autostart, status/restart |
| `skills/04-tailscale-and-pocket-id.md` | Tailscale PC+phone install, MagicDNS/HTTPS certs, `tailscale serve --bg 1411`, Pocket ID Windows binary (checksum, `.env` APP_URL/ENCRYPTION_KEY, firewall), first-run passkey, recovery, OIDC client |
| `skills/05-private-web-routing.md` | Option B `tailscale serve` default (mappings are additive); optional Option A HostUp DNS + lego wildcard cert + Caddy |
| `skills/06-verification-and-troubleshooting.md` | End-to-end acceptance checklist, troubleshooting table, maintenance (updates, key rotation, backup) |

## Cost summary

Everything is $0: Hermes open source, Edge TTS keyless, Groq free tier, Tailscale personal plan. The only money ever spent: Surplus usage, prepaid by the user; optional domain at HostUp if Phase 4 is run (`.se` natively supported).

Source note: RESEARCH.md §1, §2, §3, §4.

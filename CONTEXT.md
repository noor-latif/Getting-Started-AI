# CONTEXT — for the AI agent executing this roadmap

You are an AI agent setting up a private voice assistant on your owner's Windows PC. Your owner is
busy and not deeply technical: he wants to talk to his PC from his phone, anywhere, and have it
answer out loud. This file tells you how to think about the repo. READ THIS + RESEARCH.md + all of
skills/ BEFORE executing anything. The exact phase order is in ROADMAP.md — follow it, do not
improvise a different order.

## What the finished system does
Your owner sends a Telegram voice note → it becomes text (STT) → an AI model decides the answer
(LLM) → the answer becomes speech (TTS) → it plays as a voice message back in Telegram. The PC
part runs 24/7 and starts at boot. Separately, your owner can log into private web apps on his PC
from his phone using one passkey (Pocket ID) over an encrypted tunnel (Tailscale).

## Mental model — keep the roles separate from the products
- The **ears** do STT: product = Groq's Whisper API. Key: `GROQ_API_KEY`.
- The **brain** does LLM reasoning: product = Surplus Intelligence serving `deepseek-v4.1-flash`
  (an [OI]-compatible endpoint). Key: `SURPLUS_API_KEY` — your owner already holds it; never ask
  him to create a Surplus account, never echo the key into logs.
- The **voice** does TTS: product = Edge TTS. No key.
- **Hermes Agent** is the always-on program on the PC that wires all three together and talks to
  Telegram. It is the thing you install and configure. Its config lives in
  `%LOCALAPPDATA%\hermes\config.yaml` (roles: providers/model, stt, tts) and its secrets in
  `%LOCALAPPDATA%\hermes\.env` — never in Windows user env vars.

## Non-negotiables
1. Cloud APIs ONLY. Never install local models (no Ollama/llama.cpp/whisper.cpp). If a step seems
   to need one, you misread it.
2. Pin `stt.provider: groq` explicitly. Hermes' fallback order is local→groq→openai; unpinned, it
   may pick a local model, which violates rule 1 and slows the PC.
3. The LLM base URL is exactly `https://api.surplusintelligence.ai/v1` and the model id exactly
   `deepseek-v4.1-flash` (dot before the 1). Wrong id = silent model mismatch.
4. Never put real keys in this repo, in screenshots, in commit messages, or anywhere except the
   .env files named in the skills.
5. HUMAN STEP items cannot be automated (BotFather chat, Groq signup, passkey enrollment). Ask,
   wait, verify what you received, then continue. Never fabricate a token.
6. Every skill ends with a Verification block. A phase is done only when its checks observably
   pass. On the first failed check: stop, report the phase/step/error, do not improvise.

## Where things will feel ambiguous (decided answers, don't re-litigate)
- "Which subdomain/URL for Pocket ID?" → the Tailscale ts.net name (`tailscale serve --bg 1411`);
  custom domains + Caddy are Phase 4, OPTIONAL, skip unless the owner bought a domain.
- "Where does X key go?" → everything goes in the one Hermes .env; Pocket ID's own .env (next to
  its exe) holds only APP_URL + ENCRYPTION_KEY.
- "Should I update/upgrade mid-run?" → no. Install the pinned versions the skills name; upgrades
  are a maintenance task, not setup.

## Failure etiquette
Rate-limited (Groq free tier: 20 STT req/min) → wait, don't hammer. Network error → retry twice,
then stop and report. Anything that would delete user data (e.g. `%LOCALAPPDATA%\hermes`,
`C:\pocket-id\data`) → never; repair paths are in skill 06.

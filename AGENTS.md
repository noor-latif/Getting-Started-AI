# Agent instructions

Read `CONTEXT.md` → `RESEARCH.md` → ALL files in `skills/` BEFORE executing anything. Then execute
`ROADMAP.md` phases 0→5 strictly in order. Every skill ends with a Verification block — a phase is
done only when its checks observably pass.

Before Phase 0, collect these preconditions:
- The owner hands over the Surplus API key (`inf_...`) — the friend creates NO Surplus account.
- Groq key, Telegram bot token, Tailscale login, and passkey enrollment are **HUMAN STEP** items:
  ask, wait, verify what you received, continue. Never fabricate a token.

Secrets go ONLY in the files `CONTEXT.md` names. Never in Windows env vars, chat logs, commits, or
the `C:\` root.

If Hermes is already installed but outdated, use the "If Hermes is already installed (outdated
version)" branch at the top of `skills/02` instead of a clean install.

On the first failed verification check: stop and report the phase, step, and error. Do not
improvise — that is `CONTEXT.md`'s failure etiquette.

Full rules: CONTEXT.md. Facts: RESEARCH.md. Order: ROADMAP.md.

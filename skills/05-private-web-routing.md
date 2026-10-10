# Skill 05 — Private web routing (expose more than Pocket ID)

## Purpose
Route `https://<name>.ts.net` (or, optionally, your own domain's subdomains) to services on the PC. Option B (Tailscale serve) is the default — zero cost, no domain, 2 commands. Option A (HostUp DNS + lego wildcard cert + Caddy) is an OPTIONAL upgrade, only with a purchased domain.

## Prerequisites
- Skill 04 done: Tailscale running, MagicDNS + HTTPS Certificates enabled in the admin console.

---

## Option B — Tailscale serve (DEFAULT)

### Steps
1. Pick the local port of the service (e.g. Hermes or any app listening on 3000).
2. Serve it:
   ```powershell
   tailscale serve --bg 3000
   ```
   Equivalent to `--bg --https=443 localhost:3000`. Yields `https://<machine>.<tailnet-name>.ts.net` with an auto Let's Encrypt cert (public CA-trusted). **`--bg` makes it persist across reboots** — no service wrapper needed.
3. Manage:
   ```powershell
   tailscale serve status    # what is mapped
   tailscale serve reset     # remove ALL serve mappings (nuclear option — see step 4)
   ```
4. **Mappings are additive — adding a second service does NOT remove the first.** Pocket ID on 1411 (skill 04) keeps working when you add another service on a different port:
   ```powershell
   tailscale serve --bg --https=8443 localhost:3000   # listen port is a flag, target is the one positional
   ```
   Access form becomes `https://<machine>.<tailnet>.ts.net:8443`. For path-based URLs on the SAME port use `--set-path` (`tailscale serve --bg --set-path=/app localhost:3000`). `serve reset` wipes every mapping — including Pocket ID's — so only use it when you truly want a clean slate; check with `serve status` first.
5. Public (non-tailnet) exposure variant, if ever needed: `tailscale funnel 3000` — NOT used in this setup.

### Limits (know before relying on it)
- Fixed `*.ts.net` hostname — no arbitrary custom hostnames, no wildcard routing of many subdomains.
- Tailnet name is visible in the URL.
- Reverse-proxy backends must be localhost.

### Verification
From another tailnet device (phone), or from the PC itself:
```powershell
curl -I https://<machine>.<tailnet-name>.ts.net
```
Returns `HTTP/2 200` (or the service's expected status) over valid TLS. If Pocket ID (skill 04) is served, port 1411 is the mapping to test.

---

## Option A — HostUp DNS + lego wildcard cert + Caddy (OPTIONAL upgrade)

**DECISION BOX — stop here unless:** the friend has purchased a domain and actually needs custom hostnames. Without a domain, Option B already covers everything. Do NOT start Option A "to try it".

**Why HostUp (not Cloudflare):** HostUp (hostup.se) sells and manages `.se` domains natively (Cloudflare Registrar does NOT sell .se), DNS is included, the same account can spin up cheap VPSes later, and — decisive — the lego ACME client has a native HostUp DNS-01 provider (`--dns hostup`, since lego v5.0). That means a **standard Caddy build works**: no special download-builder module, no xcaddy. lego fetches the wildcard cert via DNS-01; Caddy just serves the files.

### Steps
1. **Domain at HostUp.** Buy/register (or transfer) the domain in the HostUp client area — DNS zone is created automatically. `.se` works natively here.
2. **HostUp API key.** HostUp dashboard → API keys → create a key with scopes `read:dns`, `write:dns`, `read:domains` (exactly what the lego hostup provider requires).
3. **Give the key to lego.** HostUp has no key prefix (any "hu_"-style pattern you see elsewhere is invented) — use the key verbatim. lego runs as a scheduled task later, so it must see the key without a logged-in session. Simplest reliable way on Windows: a dedicated secrets file only the lego task reads:
   ```powershell
   mkdir C:\caddy\secrets -Force
   # Save the key INTO the file (no name=, just the raw key):
   Set-Content -Path C:\caddy\secrets\hostup_api_key -Value "<paste-your-hostup-api-key>" -NoNewline
   ```
   lego reads it via `HOSTUP_API_KEY_FILE=C:\caddy\secrets\hostup_api_key` (the hostup provider supports `_FILE` suffixes natively). The scheduled task (step 5) sets that variable itself, so nothing machine-global leaks. Hermes secrets stay in `%LOCALAPPDATA%\hermes\.env` per skill 01 — this file is a separate, lego-only exception.
4. **Install lego** (the ACME client that does DNS-01). lego is NOT in the winget catalog — download the release zip:
   ```powershell
   mkdir C:\caddy\lego -Force
   # Get the current version from https://github.com/go-acme/lego/releases/latest
   # then (adjust version number to what the page shows):
   curl.exe -L -o C:\caddy\lego\lego.zip https://github.com/go-acme/lego/releases/latest/download/lego_v4.25.1_windows_amd64.zip
   Expand-Archive C:\caddy\lego\lego.zip -DestinationPath C:\caddy\lego -Force
   C:\caddy\lego\lego.exe --version
   ```
   ([agent must verify: the exact `lego_vX.Y.Z_windows_amd64.zip` filename on the releases page — the version above is an example, the URL pattern `releases/latest/download/lego_v<ver>_windows_amd64.zip` needs the concrete current version.])
5. **Issue the wildcard cert** (DNS-01 — no open ports needed). Use ONE fixed lego home so issuance and renewal touch the same files:
   ```powershell
   $env:HOSTUP_API_KEY_FILE = "C:\caddy\secrets\hostup_api_key"
   C:\caddy\lego\lego.exe --path C:\caddy\lego --dns hostup -d "*.example.com" -d example.com run
   ```
   Certs land in `C:\caddy\lego\certificates\` (`_.example.com.crt` + `.key`). The Caddyfile (step 8) reads from there directly. Renewal runs the SAME `--path` so it updates the files Caddy serves:
   ```
   lego --path C:\caddy\lego --dns hostup -d "*.example.com" -d example.com renew --days 30
   ```
   Schedule that daily via Task Scheduler (task also sets `HOSTUP_API_KEY_FILE`), and reload Caddy afterwards: `C:\caddy\caddy.exe reload --config C:\caddy\Caddyfile` (append it to the same task).
6. **Wildcard DNS record** in HostUp DNS management:
   - A record: name `*` → value = PC's Tailscale IP (`tailscale ip -4`).
   - Optional AAAA with `tailscale ip -6`.
   - Specific records override the wildcard — remove stale ones.
   (Headless alternative, no dashboard: `POST /api/v2/dns-zones/{id}/records` with the API key.)
7. **Install Caddy — STANDARD build is fine now:**
   ```powershell
   winget install CaddyServer.Caddy
   ```
   No custom module needed — TLS termination uses the lego cert files.
8. **Caddyfile.** Create `C:\caddy\Caddyfile` (replace domain and ports) — pointing at lego's actual output:
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
   Add one `@name host ...` + `handle` pair per subdomain. The proxy target may be any `host:port`, including other machines on the tailnet.
9. **Test-run:**
   ```powershell
   cd C:\caddy
   .\caddy.exe run --config C:\caddy\Caddyfile
   ```
10. **Run always-on as a service.** Either:
    - NSSM: `nssm install caddy C:\caddy\caddy.exe` (args: `run --config C:\caddy\Caddyfile`), or
    - Task Scheduler: task at startup, run `C:\caddy\caddy.exe run --config C:\caddy\Caddyfile`.
    ([agent must verify: exact NSSM arg syntax and Task Scheduler action fields.])

### Verification (Option A)
From any device (works outside the tailnet too, since DNS resolves publicly):
```powershell
curl -I https://agent.example.com
```
Valid wildcard cert (`*.example.com`), expected HTTP status. Check the certificate issuer/subject covers the wildcard. After 30+ days confirm the lego renew task kept it fresh (`lego ... renew` is idempotent).

### Rollback
- Stop/disable the Caddy service; delete the wildcard DNS record; revoke the HostUp API key.
- `tailscale serve` mappings are untouched by Option A — Option B keeps working.

Source: RESEARCH.md §4 (private routing, Options A and B), §1 (HostUp API key, domain cost); HostUp API: https://developer.hostup.se (dns-zones/records, scopes read:dns write:dns read:domains); lego hostup provider: https://go-acme.github.io/lego/dns/hostup/ (since lego v5.0).

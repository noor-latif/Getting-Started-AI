# Skill 05 — Private web routing (expose more than Pocket ID)

## Purpose
Route `https://<name>.ts.net` (or, optionally, your own domain's subdomains) to services on the PC. Option B (Tailscale serve) is the default — zero cost, no domain, 2 commands. Option A (Caddy + Cloudflare DNS-01) is an OPTIONAL upgrade, only with a purchased domain.

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
   tailscale serve reset     # remove ALL serve mappings
   ```
4. Different service? `tailscale serve reset` first, then serve the new port. (One HTTPS mapping per tailnet node; check `serve status` for the current one.)
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

## Option A — Caddy + Cloudflare DNS-01 wildcard (OPTIONAL upgrade)

**DECISION BOX — stop here unless:** the friend has purchased a domain (e.g. .com, $10.46/yr at registrar.cloudflare.com; .se is NOT sold by Cloudflare — buy elsewhere and point NS at Cloudflare) and actually needs custom hostnames. Without a domain, Option B already covers everything. Do NOT start Option A "to try it".

### Steps
1. **Build Caddy with the Cloudflare DNS module.** Use the official download builder https://caddyserver.com/download — Platform "Windows amd64", add module `github.com/caddy-dns/cloudflare` → download → unzip `caddy_windows_amd64.exe` into `C:\caddy\`.
   **WARNING: winget / choco / scoop ship the standard build WITHOUT the cloudflare module. DNS-01 will fail with those. Use the builder.**
2. **Cloudflare API token.** Create at dash.cloudflare.com with permissions `Zone → DNS → Edit` + `Zone → Zone → Read`, scoped to the zone (token is `cfut_`-prefixed).
3. **Set the token as a SYSTEM environment variable** (so the service sees it, not just your session):
   ```powershell
   setx /M CLOUDFLARE_DNS_API_TOKEN "cfut_..."
   ```
   Admin PowerShell required. Open a NEW shell afterwards so it's loaded.
4. **Cloudflare DNS records.** In the Cloudflare dashboard:
   - Wildcard A record: name `*` → value = PC's Tailscale IP (`tailscale ip -4`).
   - **DNS-ONLY (grey cloud) — turn the orange proxy OFF.** Cloudflare's proxy cannot reach CGNAT/Tailscale addresses (100.64/10).
   - Optional: AAAA record with `tailscale ip -6`, also DNS-only.
   - Specific records override the wildcard — remove stale ones.
5. **Caddyfile.** Create `C:\caddy\Caddyfile` (replace `example.com` and ports):
   ```
   *.example.com {
       tls {
           dns cloudflare {env.CLOUDFLARE_DNS_API_TOKEN}
           resolvers 1.1.1.1
       }
       @agent host agent.example.com
       handle @agent { reverse_proxy 127.0.0.1:PORT }
   }
   ```
   Add one `@name host ...` + `handle` pair per subdomain. The proxy target may be any `host:port`, including other machines on the tailnet.
6. **Test-run:**
   ```powershell
   cd C:\caddy
   .\caddy.exe run
   ```
   Certificate is obtained via DNS-01 (no open ports needed).
7. **Run always-on as a service.** Either:
   - NSSM: `nssm install caddy C:\caddy\caddy.exe` (args: `run --config C:\caddy\Caddyfile`), or
   - Task Scheduler: task at startup, run `C:\caddy\caddy.exe run`, highest privileges not required but SYSTEM/`/RL HIGHEST` avoids token env issues.
   ([agent must verify: exact NSSM arg syntax and Task Scheduler action fields — RESEARCH.md names both mechanisms but not exact flags.])

### Verification (Option A)
From any device (works outside the tailnet too, since DNS resolves publicly):
```powershell
curl -I https://agent.example.com
```
Valid wildcard cert (`*.example.com`), expected HTTP status. Check the certificate issuer/subject covers the wildcard.

### Rollback
- Stop/disable the Caddy service; delete the wildcard DNS record; revoke the Cloudflare token.
- `tailscale serve` mappings are untouched by Option A — Option B keeps working.

Source: RESEARCH.md §4 (private routing, Options A and B), §1 (Cloudflare token, domain cost).

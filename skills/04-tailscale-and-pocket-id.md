# Skill 04 — Tailscale + Pocket ID (SSO on your PC)

## Purpose
Expose Pocket ID (running on the Windows PC) to the tailnet over a valid public HTTPS URL (`https://<machine>.tailnet.ts.net`), create the Pocket ID admin account, and register OIDC clients. WebAuthn passkeys require HTTPS — Tailscale serve provides it free, with zero domain purchase.

## Prerequisites
- Windows 10/11 x64 PC on, logged in.
- Tailscale account (free personal plan). Same login on PC and phone.
- Files 01–03 done: Hermes installed, gateway running is NOT required for this skill, but Tailscale install is.

## Steps

### 1. Install Tailscale on the Windows PC
```powershell
winget install Tailscale.Tailscale
```
HUMAN STEP: The Tailscale GUI opens in the tray — click "Log in" and complete browser login (the same account will be used on the phone).
Verify:
```powershell
tailscale ip -4
```
Returns an IP like `100.x.y.z`.

### 2. Install Tailscale on the Android phone
HUMAN STEP: Install "Tailscale" (com.tailscale.ipn) from Play Store, log in with the SAME account, toggle Connected.

### 3. Enable MagicDNS + HTTPS Certificates (one-time, admin console)
HUMAN STEP: In a browser, go to https://login.tailscale.com/admin/dns and:
- Enable **MagicDNS**.
- Enable **HTTPS Certificates**.

### 4. Serve Pocket ID over HTTPS
```powershell
tailscale serve --bg 1411
```
This maps `https://<machine>.<tailnet-name>.ts.net` → `localhost:1411`, with an auto Let's Encrypt certificate trusted by public CAs. Persists across reboots.
Check the exact URL:
```powershell
tailscale serve status
```
Undo if ever needed: `tailscale serve reset` — but note it wipes ALL serve mappings on this PC (including any added later in skill 05). Prefer `tailscale serve --https=443 off` to remove just this mapping.

### 5. Install Pocket ID binary
```powershell
mkdir "$env:LOCALAPPDATA\pocket-id" -Force
cd "$env:LOCALAPPDATA\pocket-id"
curl.exe -L -o pocket-id.exe https://github.com/pocket-id/pocket-id/releases/latest/download/pocket-id_windows_amd64.exe
curl.exe -L -o checksums.txt https://github.com/pocket-id/pocket-id/releases/latest/download/checksums.txt
```
VERIFY the checksum before running (do NOT skip):
```powershell
(Get-FileHash .\pocket-id.exe -Algorithm SHA256).Hash
Get-Content .\checksums.txt | Select-String "windows_amd64"
```
The two hashes must match (case-insensitive). Do NOT use the arm64 exe on an x64 PC.

### 6. Create `.env` next to the exe
Generate the encryption key:
```powershell
$b = [byte[]]::new(32); [Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b); [Convert]::ToBase64String($b)
```
Create `%LOCALAPPDATA%\pocket-id\.env` with (replace placeholders):
```ini
APP_URL=https://<machine>.<tailnet-name>.ts.net
ENCRYPTION_KEY=<paste-the-generated-base64>
```
**CRITICAL: `APP_URL` must be the FINAL ts.net URL from step 4, exactly, no trailing slash, no path.** Passkeys bind to this URL; changing it later invalidates every enrolled passkey. Decide it now, never change it.
- `PORT` defaults to 1411 — omit unless changed.
- Do NOT set `PUBLIC_URL` (it does not exist in v2+); do NOT set `APP_ENV`.
- Only if later adding a local Caddy reverse proxy in front: add `TRUST_PROXY=127.0.0.1`.

### 7. Firewall pre-authorization (avoid interactive popup)
```powershell
netsh advfirewall firewall add rule name="Pocket ID 1411" dir=in action=allow protocol=TCP localport=1411
```
Run in an **admin** PowerShell. Otherwise Windows Firewall prompts interactively on first run.

### 8. Run Pocket ID
```powershell
cd "$env:LOCALAPPDATA\pocket-id"
.\pocket-id.exe
```
Leave this window open. It creates `data\` (SQLite DB + uploads) in the working dir.

### 9. Run Pocket ID at sign-in (24/7)
Once step 10 (first-run setup) works, install autostart so Pocket ID comes back after a reboot without anyone re-running step 8:
```powershell
$act = New-ScheduledTaskAction -Execute "$env:LOCALAPPDATA\pocket-id\pocket-id.exe" -WorkingDirectory "$env:LOCALAPPDATA\pocket-id"
$type = New-ScheduledTaskTrigger -AtLogOn
$set  = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries
Register-ScheduledTask -TaskName PocketID -Action $act -Trigger $type -Settings $set
```
- No admin needed (same-user AtLogOn task, native PowerShell cmdlets — same pattern as skill 03's `Hermes_Gateway` task). `WorkingDirectory` is REQUIRED: the exe writes its `data\` folder relative to cwd. If `Register-ScheduledTask` is refused, use the Startup-folder fallback from skill 03.
- Same honest caveat as skill 03 step 3: AtLogOn starts at SIGN-IN, not at power-on — after a Windows-Update reboot Pocket ID stays offline until someone logs in (skill 03 step 3 handles this).

### 10. HUMAN STEP — first-run account setup
In a browser (on the PC or the phone, both on the tailnet), open:
```
https://<machine>.<tailnet-name>.ts.net/setup
```
- Create the admin account.
- Enroll a passkey (Windows Hello / phone biometrics).
No default admin credentials exist; this is the one unavoidable human step. HTTPS is mandatory for passkeys — do NOT try to do this over plain http://.

### 11. Recovery token (if passkey is ever lost)
```powershell
.\pocket-id.exe one-time-access-token <username>
```
Prints a `/lc/<token>` login URL, valid 1 hour.

### 12. Create an OIDC client (when the first app needs SSO)
UI path: log in as admin → Administration → OIDC Clients → Add.
- Name: the app's name.
- Callback URL: from the app's SSO docs.
- Save → note client ID, client secret, and discovery URL `https://<machine>.<tailnet-name>.ts.net/.well-known/openid-configuration`.
- Scopes: `openid profile email`.
- **New clients allow NO ONE by default** — open the client's **Access** tab and grant All Users (or a group). Forgetting this = login always fails.
Headless alternative (no browser): set `STATIC_API_KEY` (≥16 chars) in `.env`, restart, use the admin API to create clients.

## Verification
1. From the **phone** (on the tailnet, Wi-Fi or mobile data): open `https://<machine>.<tailnet-name>.ts.net` — page loads with a valid padlock (public CA-trusted TLS, no warning).
2. Passkey login works from the browser.
3. Register a test OIDC client per step 12 and complete a full login round-trip into a sample app ([agent must verify: pick a trivial OIDC test app — e.g. any app with SSO support the owner actually plans to use; no generic test app is specified in RESEARCH.md]).
4. `tailscale serve status` shows the 1411 mapping; `hermes`-independent (Pocket ID keeps running).
5. Reboot the PC, sign in, wait ~1 minute: Pocket ID is reachable again at the ts.net URL with NO manual start — autostart task fired (`Start-ScheduledTask -TaskName PocketID` (or `schtasks /Run /TN PocketID`) is only needed if it didn't; check with `schtasks /Query /TN PocketID`).

## Rollback / Troubleshooting
- `tailscale serve reset` — removes ALL serve mappings (nuclear). To drop only Pocket ID: `tailscale serve --https=443 off`.
- Passkeys suddenly all invalid → `APP_URL` changed; fix `.env` back to the original URL. Passkeys bound to the old URL cannot be re-pointed; use the one-time-access-token recovery (step 11) to log back in.
- Firewall popup on exe start → run the netsh rule (step 7) as admin.
- Phone can't load the URL → phone not on the same tailnet; check the Tailscale app is Connected.
- To upgrade Pocket ID later: stop the exe, replace it, restart — migrations run automatically; downgrades are blocked without `ALLOW_DOWNGRADE=true`.

Source: RESEARCH.md §3 (Pocket ID), §4 Option B (Tailscale serve).

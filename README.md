# SafeLine WAF — Web Application Firewall Lab

**Home Lab — Project Walkthrough & Notes**

| Field | Detail |
|---|---|
| **Reporter** | Yash Paghada |
| **Stack** | DVWA (vulnerable app) + SafeLine WAF v9.3.9 |
| **Server / DVWA Host** | 10.60.245.167 (hostname: Wazuh) |
| **Attacker Host** | 10.60.245.192 (used for the SQLi test) |
| **Date** | June 17–18, 2026 |
| **Features Covered** | DVWA setup, SSL cert, WAF install, app onboarding, SQLi test, rate-limit review |

**Web Application Firewall (WAF) — Reverse Proxy Protection**

## Phase 1: Setting Up DVWA (the Vulnerable Web App)

DVWA (Damn Vulnerable Web App) is deliberately insecure, used here as the target application that SafeLine WAF will sit in front of and protect. It's installed on the same machine that's also acting as the attacker box (hostname "Wazuh").

- **1.1 — Install the LAMP stack:** `sudo apt install apache2 php php-mysql mysql-server -y`. Apache serves the web pages, PHP runs DVWA's application code, and MySQL stores its data.
- **1.2 — Clone DVWA** into `/var/www/html` — Apache's default web root, so anything placed here becomes reachable over HTTP.
- **1.3 — Set file permissions**, handing ownership to the `www-data` user (the user Apache runs as) so it can actually serve the files without permission errors.
- **1.4 — Create the database & DB user.** DVWA needs its own database and a dedicated MySQL user (rather than root); `GRANT ALL` gives that user full rights over the `dvwa` database only, and `FLUSH PRIVILEGES` reloads the permission tables immediately.
- **1.5 — Configure `config.inc.php`** to point DVWA at the database. A plain `cp` without `sudo` failed with "Permission denied" since the config folder is owned by `www-data`; `sudo cp` and `sudo nano` were needed to actually persist the edits. The `db_server` value was changed from the default `localhost` to `127.0.0.1` — a known fix noted in DVWA's own config comments, for cases where MySQL's local socket connection doesn't resolve cleanly via `localhost`.
- **1.6 — Restart Apache and verify:** `sudo systemctl restart apache2`. After restarting, DVWA's login page loaded correctly over plain HTTP — confirming the LAMP stack, database connection, and file permissions were all working before adding the WAF layer.

## Phase 2: Preparing DVWA to Sit Behind the WAF

SafeLine WAF works as a reverse proxy — it needs to occupy the public-facing port itself, with the real application listening on a different, internal port behind it. DVWA's Apache instance was moved off port 80 onto 8080 for this reason.

- **2.1** Edited `ports.conf`, changing `Listen 80` to `Listen 8080` to free up port 80/443 for SafeLine WAF to take over as the front door, while Apache (and DVWA) keep running quietly behind it on 8080.
- **2.2** Updated the default VirtualHost block to match the new port — `ports.conf` alone tells Apache which port to open a socket on; the VirtualHost block tells it which port that specific site configuration should respond on. Both need to agree, or the site won't bind correctly to 8080.

## Phase 3: Creating a Self-Signed SSL Certificate

SafeLine WAF terminates HTTPS at the proxy layer, so it needs its own certificate and private key to present to browsers. A self-signed cert is sufficient for a home lab (browsers will just show an untrusted-certificate warning, which is expected).

- **3.1** Generated the certificate & key with `openssl req` — a 2048-bit RSA key pair and a self-signed (`-x509`) certificate valid for 365 days, with no passphrase on the key (`-nodes`) so SafeLine can read it without manual intervention. Certificate subject details: Country = IN, State = Gujarat, Organization = dvwa.local.
- **3.2** Verified both cert and key files existed and were readable — the private key file is correctly locked down to root, which is why a plain `cat` failed and `sudo cat` was needed to view it, confirming the key was generated with safe permissions.

## Phase 4: Installing SafeLine WAF

SafeLine provides an official one-line install script. It runs as a set of Docker containers, so Docker itself is installed automatically as part of the process if it isn't already present.

```
sudo bash -c "$(curl -fsSLk https://waf.chaitin.com/release/latest/manager.sh)" -- --en
```

This downloads and runs Chaitin's official SafeLine manager script as root. From the menu, option 1 (INSTALL) was selected. The script detected Docker wasn't installed and offered to install it automatically — accepted, since SafeLine's containers depend on it. Once installation finished, the script printed an admin username, a generated password, and the dashboard URL (port 9443). Logging in confirmed the WAF engine and its management UI were both running correctly.

## Phase 5: Onboarding DVWA as a Protected Application

With the WAF running, the next step was telling it about DVWA: which domain to listen for, and which internal address to forward clean traffic to.

| Setting | Value |
|---|---|
| **Domain** | dvwa.local |
| **Listening Port** | 443 (HTTPS) |
| **Reverse Proxy Upstream** | http://10.60.245.167:8080 |
| **Application Name** | DVWA |

> At the point this was submitted, the SSL Cert field (marked required) was still empty — no certificate had been attached to the application yet, picked up in the next phase.

## Phase 6: Adding the SSL Certificate to SafeLine — In Progress

This phase is not fully complete yet.

- **6.1** Opened Settings → SSL Cert to check the certificate store, which showed "Total certs 0" — confirming no cert had been successfully added yet. SafeLine keeps uploaded certificates in one place under Settings, so they can be reused across applications.
- **6.2** Attempted to upload the `dvwa.crt` / `dvwa.key` files.

> **⚠ Issue found:** The SSL Cert and Private Key text boxes ended up with the literal placeholder words typed in as plain text, rather than the actual certificate/key content (which should start with `-----BEGIN CERTIFICATE-----` and `-----BEGIN PRIVATE KEY-----` respectively) or a file uploaded via the Upload Cert File / Upload Key File buttons underneath each box. As it stands, this submission would not produce a working certificate — the real `dvwa.crt` and `dvwa.key` content from `/etc/ssl/dvwa/` needs to be pasted in, or uploaded directly, before HTTPS on the WAF side will work.

## Phase 7: Testing SQL Injection Protection

With DVWA onboarded, the next test was to confirm SafeLine actually intercepts malicious traffic before it reaches the application.

Submitted a classic SQLi payload into the User ID field:
```
'OR'1'='1
```
This is one of the most well-known SQL injection payloads — it manipulates a vulnerable query's WHERE clause so the condition always evaluates true, typically returning all rows instead of just the intended one. Submitting it against DVWA (now reverse-proxied through SafeLine) tests whether the WAF flags and blocks the request before it reaches the database.

> **Status:** This capture only shows the payload being entered, before Submit was pressed — there's no follow-up confirmation yet of whether SafeLine blocked it (e.g. an "Access Forbidden" page) or whether it reached DVWA's normal SQL injection results page. That confirmation step still needs to be completed.

## Phase 8: HTTP Flood / Rate-Limiting Defense — Reviewed, Not Yet Enabled

SafeLine ships with built-in rate-limiting rules for exactly this kind of abuse. Under HTTP Flood → Rate Limiting, SafeLine ships three default rules — Access Limiting, Attack Limiting, and Error Limiting — all showing as off (gray), meaning none were currently enforcing anything.

| Rule | Default Threshold |
|---|---|
| **Basic Access Limit** | 3 requests within 10 seconds → block for 1 minute |
| **Basic Attack Limit** | 10 attack triggers within 60 seconds → block for 30 minutes |
| **Basic Error Limit** | 10 occurrences of 403/404 within 10 seconds → block for 30 minutes |

None of these rules had been switched on yet, and no flood test (e.g. with `ab` or `siege` from the attacker machine) had been run against them. Enabling Basic Access Limit and then firing rapid repeated requests at `dvwa.local` is the next thing to test.

## Project Summary

Progress so far: DVWA is fully working and reachable, SafeLine WAF is installed, and the DVWA application has been onboarded into it as a reverse-proxied site. Two pieces are still incomplete — the SSL certificate upload and confirmation that attacks are actually being blocked.

| # | Phase | Status |
|---|---|---|
| 1 | DVWA Setup | ✅ Complete — login page verified working over HTTP |
| 2 | Port 8080 Prep | ✅ Complete — Apache moved off port 80/443 |
| 3 | SSL Certificate | ✅ Generated successfully on the server |
| 4 | SafeLine Install | ✅ Complete — dashboard accessible on :9443 |
| 5 | App Onboarding | ✅ Complete — dvwa.local added as reverse proxy to :8080 |
| 6 | Cert Upload to WAF | ⚠️ Not complete — placeholder text entered instead of real cert/key |
| 7 | SQL Injection Test | ⚠️ Payload entered — result not yet confirmed |
| 8 | Rate Limiting | ⚠️ Reviewed only — rules still disabled, untested |

**Open items to finish next:**

- Re-upload the SSL certificate in Settings → SSL Cert, pasting the actual contents of `dvwa.crt` and `dvwa.key` (or using the Upload Cert File / Upload Key File buttons), then re-attach it to the DVWA application.
- Submit the SQL injection payload and capture whether SafeLine blocks it or whether it reaches DVWA — check Attacks → Logs either way.
- Try a reflected XSS payload (e.g. `<script>alert(1)</script>`) as a second attack type to confirm broader coverage.
- Enable the Basic Access Limit rule under HTTP Flood, then test it with repeated rapid requests from the attacker machine.
- Set up the Authentication Sign-In gateway and a custom Deny Rule blocking the attacker IP outright — not yet started.

**Status:** DVWA: `10.60.245.167:8080` | WAF Dashboard: `10.60.245.167:9443` | App onboarded, hardening still in progress.

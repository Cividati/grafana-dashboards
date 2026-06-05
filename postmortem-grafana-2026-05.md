# Post-Mortem: Grafana Dashboard Outage
**Date:** May 2026  
**Duration:** ~Several hours across multiple sessions  
**Severity:** Medium — internal monitoring dashboard unavailable  
**Status:** Resolved ✅

---

## Summary

Grafana became completely inaccessible during a planned security hardening of the home server stack. What started as a simple port-exposure fix cascaded into three independent but compounding issues: a password policy breaking login, incorrect manual hash patching, and a Docker networking failure. All three had to be resolved in sequence before Grafana was usable again.

---

## What Broke

- `https://grafana.pandoraserver.duckdns.org` returned **502 Bad Gateway**
- Login with `admin` / `admin` returned **"Invalid username or password"**
- Grafana was effectively fully inaccessible

---

## Timeline

### Phase 1 — Security Hardening (Root Cause Introduced)

**Action taken:** Removed direct port bindings from `compose.yaml` to prevent services from being reachable by bypassing Caddy. As part of this, `GF_SECURITY_ADMIN_PASSWORD` was removed from the environment to avoid storing plaintext credentials in the config file.

**What went wrong:** Without `GF_SECURITY_ADMIN_PASSWORD` set, Grafana v13 auto-generated a random password on first init instead of using the default `admin`. The admin user now had an unknown password.

---

### Phase 2 — Password Reset Attempts (Compounding Failures)

Multiple attempts were made to reset the Grafana admin password, each failing for a different reason:

| Attempt | Method | Why It Failed |
|---|---|---|
| 1 | Login with `admin` / `admin` | Grafana v13 enforces minimum 12-character passwords; `admin` was rejected |
| 2 | `grafana cli admin reset-admin-password` (while server was running) | CLI reported success but the live Grafana process had the old hash cached in memory |
| 3 | Direct bcrypt hash patch in SQLite DB | Grafana does **not** use bcrypt — it uses PBKDF2-SHA256 |
| 4 | PBKDF2-SHA256 hash patch with formula `pbkdf2(password+salt, password+salt)` | Wrong formula — Grafana uses `pbkdf2(password, salt)`, not the concatenated form |
| 5 | Too many failed login attempts | Triggered Grafana's login lockout; had to clear the `login_attempt` table |
| 6 | Container recreation | Changing `GF_SECURITY_ADMIN_PASSWORD: admin` in compose.yaml was silently rejected (too short), so Grafana stored a new random password again |

---

### Phase 3 — Docker DNS Failure (502 Error)

Even after the password issues were partially addressed, Grafana was still returning **502 Bad Gateway**. Investigation of Caddy logs revealed:

```
"dial tcp: lookup grafana on 127.0.0.11:53: no such host"
```

The Docker internal DNS (`127.0.0.11`) could no longer resolve the `grafana` container by name from within the Caddy container. The `monitoring` bridge network had gotten into a bad state, likely due to repeated partial restarts during the troubleshooting process.

**This was the reason the password fix couldn't even be verified** — all requests were failing at the network level before reaching Grafana.

---

### Phase 4 — Resolution

Three steps were executed in order:

**Step 1 — Fix Docker networking**
```bash
docker compose down && docker compose up -d
```
This destroyed and recreated the `monitoring` bridge network, forcing all containers to re-register with Docker DNS. The 502 error disappeared immediately after.

**Step 2 — Discover the correct PBKDF2 formula**

By extracting the live DB and testing all formula variants against the stored hash, the correct Grafana algorithm was identified:
```python
# Correct
pbkdf2_hmac('sha256', password.encode(), salt.encode(), 10000, 50)

# What was being used (wrong)
pbkdf2_hmac('sha256', (password + salt).encode(), (password + salt).encode(), 10000, 50)
```

**Step 3 — Clean initialization via env var**

Instead of patching the database manually again, the clean approach was used:
1. Added `GF_SECURITY_ADMIN_PASSWORD: AdminAdmin12` to `compose.yaml` (12+ chars to meet v13 minimum)
2. Deleted `grafana.db` from the Docker volume so Grafana would initialize from scratch
3. Restarted Grafana — it created a fresh DB using the env var password with its own correct hashing

Login confirmed working:
```
POST /login → {"message":"Logged in","redirectUrl":"/"} HTTP 200
```

---

## Root Causes

| # | Root Cause |
|---|---|
| 1 | Removing `GF_SECURITY_ADMIN_PASSWORD` without first setting a known alternative left the admin account with an unknown auto-generated password |
| 2 | Grafana v13 silently rejects passwords shorter than 12 characters at initialization (instead of erroring), making the failure non-obvious |
| 3 | Repeated partial container restarts corrupted the Docker monitoring network's DNS state |
| 4 | Documentation and assumptions about Grafana's PBKDF2 formula were incorrect (`password+salt` vs `password` as the key) |

---

## What Was Not the Problem

- ❌ SSL/TLS certificates — the Let's Encrypt wildcard cert for `*.pandoraserver.duckdns.org` was valid throughout
- ❌ Caddy configuration — Caddyfile was correct; Caddy was healthy
- ❌ Grafana settings — no grafana.ini changes were needed
- ❌ Pi-hole DNS — local DNS resolution was working correctly

---

## Lessons Learned

1. **Never remove credentials without setting replacements first.** When removing `GF_SECURITY_ADMIN_PASSWORD`, the correct sequence is: set a new known password → verify login → then decide whether to keep it in the config.

2. **Verify the password policy before changing credentials.** Grafana v13 enforces a 12-character minimum. Using `admin` as a password silently triggers a random fallback.

3. **Full stack restarts heal Docker DNS.** When services on a shared bridge network lose DNS resolution for each other, `docker compose down && docker compose up -d` is the fix — not restarting individual containers.

4. **Let the application hash its own passwords.** Replicating application-internal hashing logic externally is fragile. The correct approach is to pass the password via env var and let the application initialize with it.

5. **502 at the proxy layer masks application errors.** The password issue couldn't be verified until the Docker DNS/networking issue was fixed first. Always confirm the full path (DNS → network → application) before debugging application-level issues.

---

## Final State

| Component | Status |
|---|---|
| Grafana HTTPS | ✅ Working — `https://grafana.pandoraserver.duckdns.org` |
| Grafana login | ✅ `admin` / `AdminAdmin12` |
| SSL certificate | ✅ Let's Encrypt wildcard, valid until Aug 2026 |
| Docker networking | ✅ All containers resolving correctly on `monitoring` network |
| Direct port exposure | ✅ Removed — all traffic routed through Caddy |

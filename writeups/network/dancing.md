# Dancing — HackTheBox — Very Easy — Network

**Date Completed:** 27 Sep 2026
**Time to Root:** ~20 minutes
**User Flag:** N/A (single flag, no separate privilege escalation phase)
**Root Flag:** `5f61c10dffbc77a704d76016a22f1664`

---

## Executive Summary

Dancing is an introductory HackTheBox Starting Point machine demonstrating an SMB null session vulnerability. A non-default file share (`WorkShares`) was accessible with no authentication at all, exposing user directories and a flag file with no exploitation beyond straightforward enumeration.

---

## Scope and Methodology

- **Target IP:** 10.129.1.230
- **Platform:** HackTheBox (Starting Point)
- **Category:** Network
- **Methodology:** Reconnaissance — Enumeration — Exploitation — Privilege Escalation

---

## Reconnaissance

### Port Scan

```bash
sudo nmap -sV 10.129.1.230
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 135 | MSRPC | Microsoft Windows RPC | |
| 139 | NetBIOS-ssn | Microsoft Windows netbios-ssn | |
| 445 | SMB | microsoft-ds | Entry point |
| 5985 | WinRM | Microsoft HTTPAPI httpd 2.0 | Not used |

---

## Enumeration

Listed available SMB shares using a null (unauthenticated) session:
```bash
smbclient -L //10.129.1.230 -N
```

Alongside the standard administrative shares (`ADMIN$`, `C$`, `IPC$`), a non-default share named **WorkShares** was present and accessible — a strong signal of a deliberately exposed or misconfigured share.

---

## Exploitation

**Vulnerability Class:** SMB Null Session / Unauthenticated Share Access (CWE-284 — Improper Access Control)

Connected to the `WorkShares` share anonymously:
```bash
smbclient //10.129.1.230/WorkShares -N
```

Listing the share's contents revealed two user directories:

Amy.J
James.P


Navigating into `James.P` revealed a flag file, retrievable with no further authentication:
```bash
cd James.P
ls
get flag.txt
```

No credentials, brute forcing, or exploitation techniques were required beyond simply connecting to the share anonymously and browsing its contents.

---

## Privilege Escalation

Not applicable — the null SMB session itself provided direct, unauthenticated read access to the target file with no additional access level required.

---

## Proof of Completion

5f61c10dffbc77a704d76016a22f1664


---

## Vulnerability Analysis

| Finding | Severity | CVSS | MITRE ATT&CK | CWE |
|---------|----------|------|--------------|-----|
| SMB null session allowing unauthenticated share enumeration and access | High | 7.5 | T1135 (Network Share Discovery) | CWE-284 |

---

## Remediation

1. **Disable SMB null sessions** (`RestrictAnonymous` / `RestrictNullSessAccess` on Windows) so that share listings and access require valid authentication.
2. **Apply the principle of least privilege to file shares.** User-specific directories such as `James.P` and `Amy.J` should require that specific user's authentication to access, not be reachable by any anonymous connection to the parent share.
3. **Regularly audit SMB share permissions**, particularly on shares with generic or organizational names (like "WorkShares"), which are common targets for this class of misconfiguration.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service detection |
| smbclient | Anonymous SMB share enumeration and file retrieval |

---

## Lessons Learned

- SMB null sessions remain a surprisingly common finding even on modern Windows systems if administrators haven't explicitly hardened share permissions — always worth testing `smbclient -L //target -N` as an early enumeration step whenever SMB is open.
- User-named directories inside a shared folder (like `James.P`, `Amy.J`) are often a sign that per-user access controls were intended but never actually enforced at the share level — worth flagging specifically in a real assessment, since it suggests a broader access control design flaw, not just a single missing permission.


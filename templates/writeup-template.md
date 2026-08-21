# [Machine Name] — [Platform] — [Difficulty] — [Category]

**Date Completed:** [DD Mon YYYY]
**Time to Root:** [X hours Y minutes]
**User Flag:** `[flag value]`
**Root Flag:** `[flag value]`

---

## Executive Summary

[2–3 sentences. What was the machine? What was the core vulnerability? What was the impact?]

---

## Scope and Methodology

- **Target IP:** [IP from platform]
- **Platform:** [TryHackMe / HackTheBox]
- **Category:** [Web / Network / Windows / Steganography / Crypto]
- **Methodology:** Reconnaissance — Enumeration — Exploitation — Privilege Escalation

---

## Reconnaissance

### Port Scan

```bash
sudo nmap -sV -sC -p- -T4 -oN nmap-initial.txt TARGET_IP
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| [port] | [service] | [version] | [observation] |

---

## Enumeration

[Commands run, what you were looking for, what you found.]

---

## Exploitation

**Vulnerability Class:** [e.g. Unrestricted File Upload / Remote Code Execution]

[Step-by-step. Explain WHY each step works, not just WHAT you typed.]

```bash
[commands]
```

---

## Privilege Escalation

[How you moved from initial access to root. Explain the technique.]

---

## Proof of Completion

```
[root flag contents]
```

---

## Vulnerability Analysis

| Finding | Severity | CVSS | MITRE ATT&CK | CWE |
|---------|----------|------|--------------|-----|
| [name] | High | 9.0 | T1190 | CWE-434 |

---

## Remediation

1. **[Vulnerability]:** [Specific, actionable fix]
2. **[Vulnerability]:** [Specific, actionable fix]

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service detection |
| [Tool] | [Purpose] |

---

## Lessons Learned

- [Key technical takeaway]
- [Technique or tool you will remember]
- [Anything that surprised you]

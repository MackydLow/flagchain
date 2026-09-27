k# Meow — HackTheBox — Very Easy — Network

**Date Completed:** 27 Sep 2026
**Time to Root:** ~20 minutes
**User Flag:** N/A (direct root access, no separate user flag)
**Root Flag:** `b40abdfe23665f766f9c61ecba8a4c19`

---

## Executive Summary

Meow is an introductory HackTheBox Starting Point machine demonstrating the risks of running legacy, unencrypted remote access services with default or blank credentials. The target exposed a Telnet service that accepted a `root` login with no password whatsoever, granting immediate unauthenticated root-level access to the system.

---

## Scope and Methodology

- **Target IP:** 10.129.1.193
- **Platform:** HackTheBox (Starting Point)
- **Category:** Network
- **Methodology:** Reconnaissance — Enumeration — Exploitation — Privilege Escalation

---

## Reconnaissance

### Port Scan

```bash
sudo nmap -sV 10.129.1.193
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 23 | Telnet | Linux telnetd | Only open port; unencrypted remote login |

---

## Enumeration

With only Telnet exposed, no further web or SMB-style enumeration was applicable. The single open port was itself the entire attack surface — the objective was simply to test the service directly.

---

## Exploitation

**Vulnerability Class:** Use of Hard-coded/Default Credentials combined with an Unencrypted Remote Access Protocol (CWE-798 / CWE-319)

Connected directly to the Telnet service:
```bash
telnet 10.129.1.193
```

An initial login attempt with a mistyped username failed, but logging in as `root` with a completely blank password succeeded immediately:

Meow login: root
Password: [blank — Enter pressed]
Welcome to Ubuntu 20.04.2 LTS ...


This granted a fully interactive root shell with zero credential guessing or brute forcing required — the account simply had no password set at all.

---

## Privilege Escalation

Not required — the Telnet login itself grants root access directly, since the exposed service authenticates straight into the root account.

---

## Proof of Completion

b40abdfe23665f766f9c61ecba8a4c19


---

## Vulnerability Analysis

| Finding | Severity | CVSS | MITRE ATT&CK | CWE |
|---------|----------|------|--------------|-----|
| Telnet root login with no password | Critical | 9.8 | T1078 (Valid Accounts) / T1021.004 (Remote Services: Telnet) | CWE-798 / CWE-319 |

---

## Remediation

1. **Disable Telnet entirely.** Telnet transmits all data — including credentials — in plaintext, and has been considered obsolete since the introduction of SSH in the late 1990s. Replace it with SSH for any remote administration need.
2. **Never allow blank or default passwords**, especially on privileged accounts such as root. Enforce a strong password policy and disable direct root login over any remote protocol.
3. **Restrict exposure of administrative services.** Even with authentication properly configured, remote root login services should not be exposed to untrusted networks without additional controls (VPN, bastion host, IP allow-listing).

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service detection |
| Telnet client | Direct interactive connection to the exposed service |

---

## Lessons Learned

- Telnet's complete lack of encryption means that even if credentials weren't blank here, anyone capturing traffic with a tool like Wireshark during a Telnet session could read every keystroke, including the password, in plaintext.
- This machine is a strong illustration of why legacy protocols persist as security risks: Telnet is functionally obsolete but still turns up on legacy industrial equipment, older network hardware, and unpatched embedded/IoT devices.
- The simplicity of this box (no privilege escalation phase at all) highlights that the most severe vulnerabilities are often the most basic — a completely unauthenticated root login requires no advanced exploitation technique whatsoever, just checking whether the obvious thing works.

# Meow — HackTheBox — Very Easy — Network

**Date Completed:** 27 Sep 2026
**Time to Root:** ~20 minutes
**User Flag:** N/A (direct root access, no separate user flag)
**Root Flag:** `b40abdfe23665f766f9c61ecba8a4c19`

---

## Executive Summary

Meow is an introductory HackTheBox Starting Point machine demonstrating the risks of running legacy, unencrypted remote access services with default or blank credentials. The target exposed a Telnet service that accepted a `root` login with no password whatsoever, granting immediate unauthenticated root-level access to the system.

---

## Scope and Methodology

- **Target IP:** 10.129.1.193
- **Platform:** HackTheBox (Starting Point)
- **Category:** Network
- **Methodology:** Reconnaissance — Enumeration — Exploitation — Privilege Escalation

---

## Reconnaissance

### Port Scan

```bash
sudo nmap -sV 10.129.1.193
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 23 | Telnet | Linux telnetd | Only open port; unencrypted remote login |

---

## Enumeration

With only Telnet exposed, no further web or SMB-style enumeration was applicable. The single open port was itself the entire attack surface — the objective was simply to test the service directly.

---

## Exploitation

**Vulnerability Class:** Use of Hard-coded/Default Credentials combined with an Unencrypted Remote Access Protocol (CWE-798 / CWE-319)

Connected directly to the Telnet service:
```bash
telnet 10.129.1.193
```

An initial login attempt with a mistyped username failed, but logging in as `root` with a completely blank password succeeded immediately:

Meow login: root
Password: [blank — Enter pressed]
Welcome to Ubuntu 20.04.2 LTS ...


This granted a fully interactive root shell with zero credential guessing or brute forcing required — the account simply had no password set at all.

---

## Privilege Escalation

Not required — the Telnet login itself grants root access directly, since the exposed service authenticates straight into the root account.

---

## Proof of Completion

b40abdfe23665f766f9c61ecba8a4c19


---

## Vulnerability Analysis

| Finding | Severity | CVSS | MITRE ATT&CK | CWE |
|---------|----------|------|--------------|-----|
| Telnet root login with no password | Critical | 9.8 | T1078 (Valid Accounts) / T1021.004 (Remote Services: Telnet) | CWE-798 / CWE-319 |

---

## Remediation

1. **Disable Telnet entirely.** Telnet transmits all data — including credentials — in plaintext, and has been considered obsolete since the introduction of SSH in the late 1990s. Replace it with SSH for any remote administration need.
2. **Never allow blank or default passwords**, especially on privileged accounts such as root. Enforce a strong password policy and disable direct root login over any remote protocol.
3. **Restrict exposure of administrative services.** Even with authentication properly configured, remote root login services should not be exposed to untrusted networks without additional controls (VPN, bastion host, IP allow-listing).

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service detection |
| Telnet client | Direct interactive connection to the exposed service |

---

## Lessons Learned

- Telnet's complete lack of encryption means that even if credentials weren't blank here, anyone capturing traffic with a tool like Wireshark during a Telnet session could read every keystroke, including the password, in plaintext.
- This machine is a strong illustration of why legacy protocols persist as security risks: Telnet is functionally obsolete but still turns up on legacy industrial equipment, older network hardware, and unpatched embedded/IoT devices.
- The simplicity of this box (no privilege escalation phase at all) highlights that the most severe vulnerabilities are often the most basic — a completely unauthenticated root login requires no advanced exploitation technique whatsoever, just checking whether the obvious thing works.

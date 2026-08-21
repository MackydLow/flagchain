# FlagChain — CTF Writeup Portfolio

Professional CTF writeups across 12+ machines on HackTheBox and TryHackMe, spanning web exploitation, network services, Windows security, steganography, and cryptography.

Each writeup follows a penetration testing report structure with methodology, MITRE ATT&CK mapping, CVSS scoring, and remediation.

## Writeups

### Web Exploitation
| Machine | Platform | Difficulty | Key Techniques |
|---------|----------|-----------|----------------|
| [RootMe](writeups/web/rootme.md) | TryHackMe | Easy | File upload bypass, SUID abuse |
| [Pickle Rick](writeups/web/pickle-rick.md) | TryHackMe | Easy | OS command injection |
| [Vulnversity](writeups/web/vulnversity.md) | TryHackMe | Easy | Upload filter bypass, PHP reverse shell |
| [Simple CTF](writeups/web/simple-ctf.md) | TryHackMe | Easy | CMS exploitation, sudo misconfiguration |

### Network Services
| Machine | Platform | Difficulty | Key Techniques |
|---------|----------|-----------|----------------|
| [Meow](writeups/network/meow.md) | HackTheBox | Very Easy | Telnet default credentials |
| [Fawn](writeups/network/fawn.md) | HackTheBox | Very Easy | Anonymous FTP |
| [Dancing](writeups/network/dancing.md) | HackTheBox | Very Easy | SMB null session |

### Windows Security
| Machine | Platform | Difficulty | Key Techniques |
|---------|----------|-----------|----------------|
| [Blue](writeups/windows/blue.md) | TryHackMe | Easy | MS17-010 (EternalBlue), hash dumping |

### Linux Privilege Escalation
| Machine | Platform | Difficulty | Key Techniques |
|---------|----------|-----------|----------------|
| [Kenobi](writeups/network/kenobi.md) | TryHackMe | Easy | SMB, NFS mount, PATH manipulation |
| [Preignition](writeups/web/preignition.md) | HackTheBox | Very Easy | Web enumeration, default credentials |

### Steganography
| Machine | Platform | Difficulty | Key Techniques |
|---------|----------|-----------|----------------|
| [Agent Sudo](writeups/steganography/agent-sudo.md) | TryHackMe | Easy | FTP brute force, steghide, CVE-2019-14287 |

### Cryptography / Misc
| Machine | Platform | Difficulty | Key Techniques |
|---------|----------|-----------|----------------|
| [Overpass](writeups/crypto/overpass.md) | TryHackMe | Easy | Weak auth, JWT bypass, cron exploitation |

## Environment

- **Attacker OS:** Kali Linux (ARM64) running in UTM on Apple Silicon
- **Network:** OpenVPN to platform — all traffic isolated within platform private network
- **All machines are authorised practice targets on legitimate CTF platforms**


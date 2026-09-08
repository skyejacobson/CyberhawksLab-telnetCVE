# CyberHawks lab demo finding report

## Credit to: [@SafeBreach-Labs](https://github.com/SafeBreach-Labs)

**Lab:** Cyberhawks Lab  
**Target:** `10.1.1.1` (`demo.cyberhawks.lab`)  
**Finding ID:** WU-08  
**Date:** 2026-09-08  
**Severity:** Critical  
**Status:** Confirmed / Exploited  

---

## Executive Summary

During an assessment of the Cyberhawks lab environment, an unauthenticated Remote Code Execution (RCE) vulnerability was identified and successfully exploited on the target host (`10.1.1.1`). Exploitation of CVE-2026-24061 via an exposed Telnet service (TCP/23) resulted in a root-level interactive shell, granting full administrative control over the system.

---

## Scope & Environment

| Field | Value |
|---|---|
| Target IP | `10.1.1.1` |
| Target Hostname | `demo.cyberhawks.lab` |
| Operating System | Linux 6.12.41+deb13-amd64 (Debian 13) |
| Attacker OS | Parrot OS |
| Assessment Type | Penetration Test (Lab Environment) |

---

## Reconnaissance

### Port Scan

An Nmap service version scan was performed against the target to enumerate open ports and services.

**Command:**
```
nmap -sV 10.1.1.1
```

**Results Summary:**

| Port | State | Service |
|---|---|---|
| 21/tcp | Open | FTP (tcpwrapped) |
| 22/tcp | Open | SSH (tcpwrapped) |
| **23/tcp** | **Open** | **Telnet** |
| 25/tcp | Open | SMTP (tcpwrapped) |
| 53/tcp | Open | DNS (tcpwrapped) |
| 80/tcp | Open | HTTP (tcpwrapped) |
| 110/tcp | Open | POP3 (tcpwrapped) |
| 111/tcp | Open | RPC (tcpwrapped) |
| 139/tcp | Open | NetBIOS (tcpwrapped) |
| 143/tcp | Open | IMAP (tcpwrapped) |
| 445/tcp | Open | SMB (tcpwrapped) |
| 2049/tcp | Open | NFS (tcpwrapped) |
| 3306/tcp | Open | MySQL/MariaDB (tcpwrapped) |
| 3389/tcp | Open | RDP (tcpwrapped) |
| 5901/tcp | Open | VNC (tcpwrapped) |

**Notable Observations:**
- The host exposes a broad and concerning attack surface across 15 ports.
- Port **23 (Telnet)** was accessible, making it an immediate priority target.
- The presence of Telnet — an inherently insecure, cleartext protocol — on any modern host represents a critical misconfiguration. It's likely that the service version is exploitable.

---

## Vulnerability Details

| Field | Detail |
|---|---|
| **CVE** | CVE-2026-24061 |
| **Affected Service** | Telnet (GNU inetutils 2.6, TCP/23) |
| **Vulnerability Type** | Unauthenticated Remote Code Execution (RCE) |
| **CVSS Score** | Critical |
| **Authentication Required** | None |
| **User Interaction Required** | None |

### Description

CVE-2026-24061 is an unauthenticated Remote Code Execution vulnerability affecting the Telnet daemon component of GNU inetutils 2.6. Exploitation allows a remote, unauthenticated attacker to execute arbitrary commands on the target system with the privileges of the running service — in this case, `root`.

---

## Exploitation

### Tool / Exploit Used

A purpose-built Python exploit script (`telnet_rce.py`) targeting CVE-2026-24061 was executed against the vulnerable Telnet service.

**Command:**
```
python3 telnet_rce.py 10.1.1.1
```

**Exploit Output:**
```
[*] Connected to 10.1.1.1:23
[*] Interactive session started. Use Ctrl+C to quit.

Linux 6.12.41+deb13-amd64 (demo.cyberhawks.lab) (pts/0)

root@demo:~#
```

### Post-Exploitation Verification

Upon obtaining a shell, the following commands confirmed the level of access achieved:

**Identity Verification:**
```
root@demo:~# id
uid=0(root) gid=0(root) groups=0(root)
```

**Filesystem Enumeration:**
```
root@demo:~# ls -la
total 36
drwx------  6 root root 4096 Aug 28 14:47 .
drwxr-xr-x 19 root root 4096 Aug 28 15:37 ..
-rw-r--r--  1 root root  607 May 12  2025 .bashrc
drwx------  4 root root 4096 Sep  6  2025 .cache
drwxr-xr-x  3 root root 4096 Sep  6  2025 .config
drwxrwxr-x  3 root root 4096 Sep  6  2025 .local
-rw-------  1 root root  261 Sep  6  2025 .mariadb_history
-rw-r--r--  1 root root  132 May 12  2025 .profile
drwx------  2 root root 4096 Sep  6  2025 .ssh
```

**Key findings from post-exploitation:**
- `.mariadb_history` — indicates active MariaDB database usage; potential for further credential harvesting or data exfiltration.
- `.ssh/` directory — may contain private keys usable for lateral movement to other hosts.
- Full `root` (UID 0) access was confirmed with no privilege escalation step required.

---

## Impact

| Category | Assessment |
|---|---|
| **Confidentiality** | Critical — all data on the system is accessible |
| **Integrity** | Critical — attacker can modify or destroy any data |
| **Availability** | Critical — attacker can terminate services or crash the system |

Full system compromise was achieved in a single, unauthenticated step. An attacker with access to this network segment could:

- Read, modify, or exfiltrate all data stored on the host.
- Harvest credentials from shell history, config files, and the MariaDB database.
- Use SSH keys to pivot laterally to other hosts on the network.
- Establish persistence (e.g., backdoor accounts, cron jobs, rootkits).
- Disrupt availability of all services running on the host.

---

## Recommendations

### Immediate (Critical Priority)

1. **Disable and remove the Telnet service** — Telnet transmits all data in cleartext and has no place in any modern environment. Disable the `telnetd` service immediately:
   ```
   systemctl disable --now telnetd
   ```

2. **Patch or mitigate CVE-2026-24061** — Apply the vendor-supplied patch for GNU inetutils if Telnet must remain in use. Check for an updated package:
   ```
   apt update && apt install inetutils-telnetd
   ```

3. **Audit SSH keys** — Review the contents of `/root/.ssh/` for any unauthorized or unexpected keys that may have been added during the window of exposure.

4. **Rotate all credentials** — Assume all credentials stored on this host (database passwords, SSH keys, application secrets) are compromised.

### Short-Term

5. **Replace Telnet with SSH** — Use SSH (TCP/22) for all remote administration. SSH provides encrypted, authenticated sessions.

6. **Implement network segmentation and firewall rules** — Restrict access to management ports (22, 23, 3306, 3389, 5901, etc.) to trusted administrative IP ranges only. No management service should be internet-accessible.

7. **Audit all other exposed services** — The host exposes 15 ports, many of which (RDP, VNC, SMB, NFS, MySQL) carry significant risk if misconfigured. Each should be reviewed.

### Long-Term

8. **Adopt a vulnerability management program** — Regularly scan lab and production hosts for known CVEs and patch on a defined schedule.

9. **Enforce principle of least privilege** — Services should not run as `root`. The Telnet daemon running as root directly resulted in immediate full system compromise.

---

## Artifacts & Evidence

| Artifact | Description |
|---|---|
| Nmap scan output | Full port/service enumeration of `10.1.1.1` |
| `telnet_rce.py` | Exploit script used to trigger CVE-2026-24061 |
| Shell session log | Interactive root shell transcript |

---

*Report prepared for Cyberhawks Lab — internal educational use only.*

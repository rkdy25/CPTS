# CTF Lessons Learned

---

## General Methodology

- always list internal services when doing privilege escalation
  `sudo ss -tulpn` and `systemctl list-units --type=service --state=running`
- all passwords of a service may be found in .conf files
- hashcat for password cracking
- sqlmap for SQL injection
- ssh -L <localport>:127.0.0.1:<remoteport> user@target
  allows you to access an internal service running on a specific port through your browser

---

## Web / Code Injection

- if a web app uses eval() on user input it is vulnerable to SSTI — any {python_expression} inside the input will execute as the server process owner
- always check what user the web service runs as — if it runs as root, code injection gives you root immediately with no further escalation needed
- when a regex blocks your payload, map the exact allowed characters first before trying anything — saves a lot of time
- commas and spaces are the two most important characters to check — most injection payloads need one or both
- if commas are blocked, look for OOP method chaining — split a two-argument function into two chained single-argument calls
  `os.chmod(path, mode)` needs a comma but `Path(path).chmod(mode)` does not
- if spaces are blocked, stay in Python entirely and never drop to shell — shell commands need spaces, Python method calls do not
- if backslash is blocked, use real newlines inside multiline strings instead of \n
- if special characters are blocked entirely, encode your payload as hex and use bytes.fromhex() to decode it — hex only uses 0-9 and a-f which are always allowed
- __import__('module') is the function form of import — use it inside expressions and f-strings where import statements are not allowed

---

## Privilege Escalation

- if you can execute code as root, setting SUID on /bin/bash is the fastest way to get an interactive root shell
- SUID (chmod 4755) tells the kernel to run a file as its owner regardless of who launches it — the 4 at the front is what activates it
- after setting SUID on /bin/bash you must use /bin/bash -p — without -p bash drops the elevated privilege on purpose as a self-protection measure
- the kernel enforces SUID, not bash — by the time bash starts it is already running as root
- always check for SUID binaries already present on the system before trying to set your own
  `find / -perm -4000 -type f 2>/dev/null`
- always check sudo permissions for your current user
  `sudo -l`

---

## Regex Bypass Mindset

- a regex allowlist is only as strong as its most permissive allowed character — look for characters that enable Python expressions like ( ) { } ' "
- fullmatch() checks the entire string including content inside quotes — you cannot hide blocked characters inside a string literal
- the goal when bypassing a regex filter is to find a valid Python expression that achieves your goal using only the allowed characters
- pathlib is your best friend when os is too restrictive — most os operations have a pathlib equivalent that requires fewer arguments

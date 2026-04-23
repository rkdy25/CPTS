# CTF Cryptography — Complete Field Guide

**Audience:** Intermediate | **Track:** HTB / THM / CPTS / OSCP  
**Focus:** Practical methodology, zero fluff

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Core Mindset](#2-core-mindset)
3. [Hash vs Encryption](#3-hash-vs-encryption)
4. [Identification Phase](#4-identification-phase)
5. [Attack Methodology](#5-attack-methodology)
6. [Tools](#6-tools)
7. [Case Study — BCrypt](#7-case-study--bcrypt)
8. [Common Pitfalls](#8-common-pitfalls)
9. [CTF Workflow Checklist](#9-ctf-workflow-checklist)
10. [Pro Tips](#10-pro-tips)

---

## 1. Introduction

### What Cryptography Challenges Are in CTFs

Cryptography challenges in CTF competitions require you to either:

- **Recover plaintext** from encrypted or encoded data
- **Crack a hash** to retrieve the original value
- **Exploit a flawed implementation** of a known algorithm
- **Analyze protocol behavior** to extract a secret

The flag is almost always hidden behind a cryptographic operation. Your job is to reverse, bypass, or break it.

### Real-World Crypto vs CTF Crypto

| Dimension | Real-World Crypto | CTF Crypto |
|---|---|---|
| Algorithm | Correctly implemented standards (AES-GCM, RSA-OAEP) | Standard algorithms with intentional flaws |
| Keys | Properly generated, stored, rotated | Hardcoded, reused, weak, or predictable |
| Randomness | Cryptographically secure (CSPRNG) | Seeded with time, reused nonces, static IVs |
| Goal | Confidentiality, integrity, authenticity | Hide a flag; your goal is to recover it |
| Attack surface | Side channels, protocol flaws | Logic flaws, bad parameters, weak passwords |
| Scope | Nation-state / enterprise threat model | Deliberate vulnerability for learning |

> In CTFs, the algorithm is rarely broken mathematically. The flaw is almost always in the **implementation**, **parameters**, or **password choice**.

---

## 2. Core Mindset

### How to Think When Approaching Crypto Challenges

**Step 1 — Do not assume anything.**  
Read the challenge carefully. Every piece of data given is intentional. File names, variable names, comments in code — all are clues.

**Step 2 — Identify before you attack.**  
Never run a tool before you know what you are dealing with. Identification is the most important phase.

**Step 3 — Work from the outside in.**  
If data is layered (e.g., base64 of hex of rot13), you must unwrap the outer layer first. Always.

**Step 4 — Consider the simplest explanation first.**  
CTF passwords are almost always in `rockyou.txt`. The encoding is almost always base64. The key is almost always reused. Start simple.

**Step 5 — Read the source code if available.**  
If the challenge gives you Python or PHP source, read it entirely before touching any tool. The flaw is in the code.

### Common Beginner Mistakes

- Trying to decrypt a hash (hashes are one-way by design)
- Running hashcat in brute-force mode on bcrypt for hours
- Ignoring the format prefix (`$2y$`, `$6$`, etc.)
- Not checking if something is just base64-encoded, not encrypted
- Using the wrong hashcat mode and getting no results
- Assuming a complex attack when the password is `admin`
- Not verifying the cracked result before submitting

---

## 3. Hash vs Encryption

### Core Distinction

| Property | Hash | Encryption |
|---|---|---|
| Reversible? | No (one-way function) | Yes (with the correct key) |
| Key required? | No | Yes |
| Purpose | Integrity verification, password storage | Confidentiality |
| Output size | Fixed (regardless of input size) | Variable (depends on input + padding) |
| Attack method | Cracking (dictionary / brute-force) | Decryption (key recovery / exploit flaw) |
| Examples | MD5, SHA-1, SHA-256, bcrypt | AES, RSA, ChaCha20 |

### When Something Is Reversible

```
ENCODING   →  Always reversible, no key (Base64, Hex, ROT13)
ENCRYPTION →  Reversible only with the correct key (AES, RSA)
HASHING    →  Never reversible — only crackable via preimage attack
```

### Real Examples

```
Encoded (Base64):    aGVsbG8=              →  decode → hello
Hashed (MD5):        5d41402abc4b2a76b9...  →  crack  → hello
Encrypted (AES-CBC): a3f1d8...              →  decrypt with key → hello
```

> If someone says "decrypt this hash" — that is incorrect terminology and an incorrect approach. Hashes are **cracked**, not decrypted.

---

## 4. Identification Phase

### How to Recognize Algorithms from Patterns

Before running any tool, examine:

1. **Prefix / structure** — does it start with `$2y$`, `$6$`, `-----BEGIN`?
2. **Length** — is it 32, 40, 64, or 128 hex characters?
3. **Character set** — alphanumeric? Base64 alphabet? Pure hex?
4. **Context** — web app, Linux system, source code, binary?

### Quick Format Reference Table

| Format | Example / Pattern | Length | Notes |
|---|---|---|---|
| MD5 | `5f4dcc3b5aa765d61d8327deb882cf99` | 32 hex chars | Legacy, fast to crack |
| SHA-1 | `5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8` | 40 hex chars | Deprecated, fast to crack |
| SHA-256 | `5e884898da28...` | 64 hex chars | Common in modern apps |
| SHA-512 | `b109f3bbbc24...` | 128 hex chars | Very long hex string |
| BCrypt | `$2y$10$...` or `$2b$10$...` | 60 chars total | Cost factor visible |
| MD5crypt | `$1$salt$hash` | Variable | Linux legacy |
| SHA-256crypt | `$5$salt$hash` | Variable | Linux /etc/shadow |
| SHA-512crypt | `$6$salt$hash` | Variable | Most common in /etc/shadow |
| Base64 | `aGVsbG8gd29ybGQ=` | Multiple of 4, ends `=` | Not encryption |
| Hex | `68656c6c6f` | Even number of chars | 0-9, a-f only |
| ROT13 | `uryyb` | Same as input | Shifts letters by 13 |
| RSA Private Key | `-----BEGIN RSA PRIVATE KEY-----` | Block format | PEM encoded |
| RSA Public Key | `-----BEGIN PUBLIC KEY-----` | Block format | PEM encoded |
| OpenSSL encrypted | `U2FsdGVkX1...` | Starts with `Salted__` in base64 | `openssl enc` output |
| JWT | `eyJ...eyJ...signature` | Three base64 parts, dot-separated | Header.Payload.Signature |

### Quick Recognition Tips

```bash
# Check length of a hash
echo -n "5f4dcc3b5aa765d61d8327deb882cf99" | wc -c
# 32 = MD5, 40 = SHA1, 64 = SHA256, 128 = SHA512

# Detect format automatically
hashid '5f4dcc3b5aa765d61d8327deb882cf99'
hash-identifier           # interactive
nth --text '5f4dcc3b...'  # name-that-hash, most accurate

# Check if it is base64
echo "aGVsbG8=" | base64 -d

# Check if it is hex
echo "68656c6c6f" | xxd -r -p
```

### Identifying from Context

| Context | Likely Algorithm |
|---|---|
| `/etc/shadow` entry | SHA-512crypt (`$6$`) |
| PHP web app login | BCrypt (`$2y$`) |
| Old forum / CMS | MD5 (plain or salted) |
| JWT token | HMAC-SHA256 or RSA |
| CTF source code with `hashlib` | Python-standard hash (MD5/SHA family) |
| File with `.pem` extension | RSA / OpenSSL |
| String ending in `==` | Base64 |

---

## 5. Attack Methodology

### a. Hash Cracking

#### When It Works

- Password is a common word, name, or pattern
- Hash is unsalted MD5 or SHA-1
- BCrypt with a very weak password (`admin`, `password`, `1234`)

#### When It Does Not Work

- BCrypt / Argon2 with a strong or random password
- Hash is salted with an unknown salt in a non-standard way
- The "hash" is actually HMAC (requires a key, not just a password)

#### Wordlist Attack (Primary Method)

```bash
# Always try wordlist first
hashcat -m <mode> hash.txt /usr/share/wordlists/rockyou.txt

# With rules (expands wordlist coverage significantly)
hashcat -m <mode> hash.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

# Multiple rules stacked
hashcat -m <mode> hash.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule \
  -r /usr/share/hashcat/rules/toggles1.rule
```

#### Brute Force (Last Resort — Use Selectively)

```bash
# Brute force up to 6 characters, all printable ASCII
hashcat -m <mode> hash.txt -a 3 ?a?a?a?a?a?a

# Digit-only passwords up to 8 chars
hashcat -m <mode> hash.txt -a 3 ?d?d?d?d?d?d?d?d
```

> Do NOT brute-force BCrypt. At cost factor 10 on a CPU, cracking a 6-character random password
> takes decades. Only wordlist attacks are realistic.

#### Common Hashcat Modes

| Mode | Algorithm |
|---|---|
| 0 | MD5 |
| 100 | SHA-1 |
| 1400 | SHA-256 |
| 1700 | SHA-512 |
| 500 | MD5crypt (`$1$`) |
| 7400 | SHA-256crypt (`$5$`) |
| 1800 | SHA-512crypt (`$6$`) |
| 3200 | BCrypt (`$2y$`, `$2b$`, `$2a$`) |
| 13100 | Kerberos 5 TGS (etype 23) |
| 22000 | WPA-PBKDF2-PMKID+EAPOL |

```bash
# Look up any mode
hashcat --example-hashes | grep -A2 -i bcrypt
```

---

### b. Encoding Challenges

Encoding is **not** encryption. There is no key. It is always reversible.

#### Base64

```bash
# Decode
echo "aGVsbG8gd29ybGQ=" | base64 -d

# Encode
echo "hello world" | base64
```

#### Hex

```bash
# Decode hex to ASCII
echo "68656c6c6f" | xxd -r -p

# Encode ASCII to hex
echo -n "hello" | xxd -p
```

#### ROT13

```bash
echo "uryyb" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

#### Layered Encodings

Many CTF challenges stack encodings. Approach:

1. Identify the outermost layer
2. Decode it
3. Examine the result — does it look like another encoding?
4. Repeat until you reach plaintext

```python
import base64

data = "dXJsbG8="
step1 = base64.b64decode(data).decode()
# check if step1 is hex, rot13, or base64 again
print(step1)
```

> Use CyberChef's **Magic** operation to auto-detect layered encodings. It is the fastest tool for this.

---

### c. Encryption Challenges

#### AES — What to Look For

AES itself is not broken. Look for implementation flaws:

| Flaw | Description | Attack |
|---|---|---|
| Static IV | Same IV reused for every message | XOR two ciphertexts to cancel key |
| ECB mode | Each 16-byte block encrypted independently | Block rearrangement / pattern leakage |
| Key = IV | Key and IV are the same value | Recover key from known plaintext |
| Padding oracle | Server reveals padding validity | Byte-by-byte decryption (POODLE-style) |
| Hardcoded key | Key visible in source code | Direct decryption |

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = bytes.fromhex("0011223344556677...")  # extracted from source
iv  = bytes.fromhex("aabbccdd...")
ct  = bytes.fromhex("ciphertext_here...")

cipher = AES.new(key, AES.MODE_CBC, iv)
plaintext = unpad(cipher.decrypt(ct), AES.block_size)
print(plaintext)
```

#### RSA — Practical Exploitation (No Heavy Math)

RSA is broken in CTFs not mathematically, but through:

| Weakness | Condition | Attack |
|---|---|---|
| Small public exponent `e=3` | Small plaintext, no padding | Cube root attack |
| Common factor | Two moduli share a prime | GCD attack |
| Small `n` | Tiny key (512-bit or less) | Factor via factordb.com |
| `p` close to `q` | Primes are close in value | Fermat factorization |
| Wiener's attack | Very small private exponent `d` | Continued fraction analysis |
| Known plaintext | You know part of the message | Coppersmith's attack |

```python
# Check if n can be factored
from sympy import factorint
n = 123456789012345678901234567890123
print(factorint(n))

# Online: https://factordb.com
# Paste n — if pre-factored, you get p and q immediately

# Once you have p and q, recover d
from Crypto.PublicKey import RSA
from Crypto.Util.number import inverse

e = 65537
p = ...
q = ...
n = p * q
phi = (p - 1) * (q - 1)
d = inverse(e, phi)

# Decrypt
ct = ...
pt = pow(ct, d, n)
print(pt.to_bytes((pt.bit_length() + 7) // 8, 'big'))
```

---

### d. Logic / Implementation Flaws

These are the most common and most valuable to understand.

#### Weak Randomness

```python
import random
random.seed(42)          # predictable — not cryptographically secure
key = random.randbytes(16)

# Fix: always use secrets or os.urandom in production
import os
key = os.urandom(16)     # CSPRNG
```

If a CTF challenge seeds `random` with a timestamp or a known value, you can reproduce the key.

#### Reused Keys / Nonces

In stream ciphers or AES-CTR, reusing a nonce with the same key leaks the keystream:

```
C1 = P1 XOR K
C2 = P2 XOR K
C1 XOR C2 = P1 XOR P2   (key cancels out)
```

If you know `P1`, you can recover `P2`.

#### Hardcoded Secrets

Always check source code for:

```bash
grep -rEi "key|secret|password|token|iv" . --include="*.py" --include="*.php" --include="*.js"
```

---

## 6. Tools

### hashcat

```bash
# Identify mode first
hashcat --example-hashes | grep -i bcrypt

# Dictionary attack
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt

# With rules
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Show cracked results
hashcat -m 3200 hash.txt --show

# Resume a session
hashcat --session mysession --restore
```

### John the Ripper

```bash
# Auto-detect format and crack
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Force format
john --format=bcrypt hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Show cracked
john --show hash.txt

# Convert shadow file for john
unshadow /etc/passwd /etc/shadow > combined.txt
john combined.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

### CyberChef

URL: https://gchq.github.io/CyberChef/

| Use Case | Operation |
|---|---|
| Detect and decode unknown encoding | Magic |
| Decode base64 | From Base64 |
| Decode hex | From Hex |
| ROT13 | ROT13 |
| AES decrypt | AES Decrypt |
| Multiple layers | Chain operations in sequence |
| Analyze JWT | JWT Decode |

> Use the **Magic** operation first on any unknown blob. It tries dozens of encodings automatically.

### Python — hashlib and bcrypt

```python
import hashlib

# Generate common hashes
data = b"admin"
print("MD5   :", hashlib.md5(data).hexdigest())
print("SHA1  :", hashlib.sha1(data).hexdigest())
print("SHA256:", hashlib.sha256(data).hexdigest())
print("SHA512:", hashlib.sha512(data).hexdigest())

# Verify a hash
target = "21232f297a57a5a743894a0e4a801fc3"
print(hashlib.md5(b"admin").hexdigest() == target)  # True
```

```python
import bcrypt

# Verify a bcrypt hash
hashed = b"$2y$10$cmytVWFRnt1XfqsItsJRVe/ApxWxcIFQcURnm5N.rhlULwM0jrtbm"
candidate = b"admin"

# $2y$ (PHP) must be replaced with $2b$ for Python's bcrypt library
hashed_py = hashed.replace(b"$2y$", b"$2b$")

if bcrypt.checkpw(candidate, hashed_py):
    print("Match: password is", candidate.decode())
else:
    print("No match")
```

### OpenSSL

```bash
# Decrypt AES-256-CBC encrypted file (openssl enc format)
openssl enc -d -aes-256-cbc -in encrypted.bin -out decrypted.txt -k "password"

# With explicit key and IV (hex)
openssl enc -d -aes-256-cbc \
  -K 0011223344556677889900112233445566778899001122334455667788990011 \
  -iv 00112233445566778899001122334455 \
  -in encrypted.bin -out decrypted.txt

# Extract public key from certificate
openssl x509 -in cert.pem -pubkey -noout

# Read RSA key details
openssl rsa -in private.pem -text -noout

# Decrypt RSA-encrypted data
openssl rsautl -decrypt -inkey private.pem -in encrypted.bin -out decrypted.txt
```

### hash-identifier / hashid / nth

```bash
# Install name-that-hash (most accurate)
pip install name-that-hash
nth --text "5f4dcc3b5aa765d61d8327deb882cf99"

# hashid
hashid '5f4dcc3b5aa765d61d8327deb882cf99'

# hash-identifier (interactive)
hash-identifier
```

---

## 7. Case Study — BCrypt

### The Hash

```
$2y$10$cmytVWFRnt1XfqsItsJRVe/ApxWxcIFQcURnm5N.rhlULwM0jrtbm
```

### Anatomy

```
$2y$   10   $   cmytVWFRnt1XfqsItsJRVe/   ApxWxcIFQcURnm5N.rhlULwM0jrtbm
 |      |           |                              |
 |      |           Salt (22 chars)                Hash (31 chars)
 |      Cost factor (2^10 = 1024 rounds)
 Algorithm: BCrypt (PHP variant)
```

| Part | Value | Meaning |
|---|---|---|
| Algorithm | `$2y$` | BCrypt, PHP flavor |
| Cost | `10` | 1024 iterations — intentionally slow |
| Salt | `cmytVWFRnt1XfqsItsJRVe/` | Embedded — no need to extract separately |
| Hash | `ApxWxcIFQcURnm5N.rhlULwM0jrtbm` | Output of bcrypt function |

### Why It Cannot Be Decrypted

BCrypt is a **one-way hashing function**. There is no key. There is no reverse operation. The algorithm is designed so that:

1. You cannot recover the input from the output mathematically
2. Even with the full hash and salt, you must try candidate passwords one by one
3. The cost factor makes each attempt computationally expensive

There is no "bcrypt decryptor." Anyone claiming otherwise is running a lookup table (rainbow table) of pre-computed common passwords — which is exactly what you should do.

### How It Is Solved in CTFs

The password was already known (`admin`). In practice:

**Step 1 — Save the hash**

```bash
echo '$2y$10$cmytVWFRnt1XfqsItsJRVe/ApxWxcIFQcURnm5N.rhlULwM0jrtbm' > hash.txt
```

**Step 2 — Run wordlist attack**

```bash
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

**Step 3 — Check result**

```bash
hashcat -m 3200 hash.txt --show
# Output: $2y$10$cmyt...:admin
```

### Verification Process

```python
import bcrypt

hashed = b"$2y$10$cmytVWFRnt1XfqsItsJRVe/ApxWxcIFQcURnm5N.rhlULwM0jrtbm"
candidate = b"admin"

# Python's bcrypt library uses $2b$ — replace $2y$ before verifying
hashed_compat = hashed.replace(b"$2y$", b"$2b$")

result = bcrypt.checkpw(candidate, hashed_compat)
print("Verified:", result)
# Verified: True
```

> `$2y$` and `$2b$` are functionally identical. PHP introduced `$2y$` to fix an edge case
> in character handling. Python's `bcrypt` library uses `$2b$`. Always replace before verifying.

---

## 8. Common Pitfalls

| Pitfall | Why It Fails | Correct Approach |
|---|---|---|
| Trying to "decrypt" a hash | Hashes are one-way — decryption is not possible | Crack via wordlist or brute-force |
| Treating Base64 as encryption | Base64 is encoding, not encryption — there is no key | Decode directly with `base64 -d` |
| Ignoring the format prefix | Wrong hashcat mode = zero results | Always identify prefix first |
| Brute-forcing BCrypt | Too slow to be practical beyond 4-character passwords | Use wordlist only; if it fails, look for other vectors |
| Assuming a complex attack | Most CTF crypto passwords are in rockyou.txt | Try `admin`, `password`, `1234` manually before running hashcat |
| Overcomplicating the challenge | Spending hours on math when the answer is simple encoding | Always test the simplest hypothesis first |
| Not verifying the result | Submitting the wrong flag | Always verify with Python or the tool's `--show` flag |
| Forgetting about the source code | Missing a hardcoded key or flaw | Read all provided files before using tools |

---

## 9. CTF Workflow Checklist

```
[ ] 1. COLLECT
        - What data has been given? (hash, ciphertext, encoded string, source code, keys)
        - Is there source code to read? Read it entirely first.

[ ] 2. IDENTIFY FORMAT
        - Check prefix ($2y$, $6$, $1$, etc.)
        - Check length (32=MD5, 40=SHA1, 64=SHA256, 128=SHA512)
        - Check character set (hex, base64, printable only)
        - Run: hashid / nth / hash-identifier

[ ] 3. CLASSIFY TYPE
        - Encoding (Base64, Hex, ROT13) → always reversible, no key needed
        - Hash (MD5, SHA, BCrypt) → crack via wordlist
        - Encryption (AES, RSA) → need key or exploit flaw

[ ] 4. CHOOSE ATTACK VECTOR
        - Encoding     → decode directly (CyberChef, bash, Python)
        - Fast hash    → hashcat wordlist → hashcat rules → brute-force
        - Slow hash    → hashcat wordlist only (BCrypt/Argon2)
        - AES          → look for hardcoded key, static IV, ECB mode
        - RSA          → check n size, check e value, try factordb.com

[ ] 5. EXECUTE
        - Use correct tool and mode
        - Save all output

[ ] 6. VERIFY
        - Confirm result with Python before submitting
        - For hashes: bcrypt.checkpw() or hashlib comparison
        - For decryption: readable plaintext = success

[ ] 7. IF STUCK — PIVOT
        - Is there another file or parameter in the challenge?
        - Is the crypto protecting a web endpoint? Try SQLi / IDOR instead.
        - Is the algorithm correct but the key derivation weak?
        - Ask: am I overcomplicating this?
```

---

## 10. Pro Tips

### When to Stop Brute-Forcing

Stop brute-forcing when:

- BCrypt / Argon2 cost factor is high and wordlist is exhausted — the password is not in rockyou.txt
- You have been running for more than 30 minutes on a CTF challenge — re-read the challenge
- The tool reports `Exhausted` on rockyou.txt with standard rules

Pivot to:

- Custom wordlist generated from page content: `cewl http://target.com -w custom.txt`
- Challenge-specific words (usernames, app name, challenge title)
- Other attack surfaces (the crypto may not be the intended entry point)

### When to Pivot to Other Vulnerabilities

Crypto challenges in CTFs sometimes exist alongside web or binary challenges. Consider pivoting if:

- The hash is protecting a login endpoint — check for SQLi, IDOR, or broken auth instead
- The encrypted data is transmitted in a cookie — check for insecure deserialization
- An RSA key is exposed but n is large — check if the private key is accessible via a misconfigured endpoint
- The challenge gives you encrypted data and a decryption oracle — this is a padding oracle attack

### Combining Crypto with Web Exploitation

```
Common scenario:
  1. Web app uses JWT for session management
  2. JWT signed with weak secret or algorithm set to "none"
  3. Forge an admin JWT without knowing the original secret

Attack:
  - Decode JWT: base64 -d on header and payload
  - Check algorithm: if HS256, crack with hashcat -m 16500
  - If algorithm is changeable: set to "none", remove signature, submit

hashcat -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt
```

```
Common scenario:
  1. AES-CBC encrypted cookie
  2. Server returns different error for padding vs content errors
  3. Padding oracle attack → byte-by-byte plaintext recovery

Tool: padbuster
padbuster http://target.com/page COOKIE_VALUE 8 -encoding 0
```

### Building a Custom Wordlist from Target Context

```bash
# Scrape words from the target web application
cewl http://target.com -d 3 -m 4 -w custom_wordlist.txt

# Add common mutations (capitalize, append numbers)
hashcat custom_wordlist.txt --stdout -r /usr/share/hashcat/rules/best64.rule > mutated.txt

# Use it
hashcat -m 3200 hash.txt mutated.txt
```

### Quick Reference — Decision Tree

```
Unknown blob received
        |
        +-- Has prefix ($2y$, $6$, $1$)?  →  It is a hash → crack it
        |
        +-- Ends with = or ==?             →  Try base64 decode first
        |
        +-- All hex characters?            →  Try hex decode first
        |
        +-- Shifts letters?                →  Try ROT13 / Caesar
        |
        +-- Starts with eyJ?               →  JWT → decode header+payload
        |
        +-- Starts with U2FsdGVkX1?        →  openssl enc output → decrypt with openssl
        |
        +-- PEM block?                     →  RSA key or certificate → extract n, e, d
        |
        +-- Random binary data?            →  AES ciphertext → look for key in source
        |
        +-- None of the above?             →  Run: nth, hashid, CyberChef Magic
```

---

*End of document — review, study, and apply systematically.*

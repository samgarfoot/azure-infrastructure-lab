# KQL Queries — Microsoft Sentinel

A reference library of KQL (Kusto Query Language) queries written for the Azure Infrastructure Lab Sentinel workspace. Each query includes an explanation of what it detects, why it matters, and the MITRE ATT&CK technique it maps to.

All queries run against the `security-lab-workspace` Log Analytics workspace connected to `soc-target-vm` via Azure Monitor Agent.

---

## Query Index

| Query | Purpose | MITRE ATT&CK |
|---|---|---|
| Failed Login Threshold | Detect brute force login attempts | T1110 — Brute Force |
| Successful Login After Failures | Detect successful brute force | T1110.001 — Password Guessing |
| New Local Account Created | Detect persistence via new accounts | T1136.001 — Local Account |
| Privilege Escalation Attempts | Detect privilege abuse | T1078 — Valid Accounts |
| Security Log Cleared | Detect log tampering | T1070.001 — Clear Windows Event Logs |
| Process Creation with Suspicious Commands | Detect execution techniques | T1059 — Command and Scripting Interpreter |
| Logon Outside Business Hours | Detect anomalous access times | T1078 — Valid Accounts |
| Multiple Failed Logins Across Accounts | Detect password spraying | T1110.003 — Password Spraying |
| Admin Account Logon | Detect privileged account use | T1078.002 — Domain Accounts |
| Service Installation | Detect persistence via services | T1543.003 — Windows Service |
| User Added to Privileged Group | Detect privilege escalation | T1098 — Account Manipulation |
| RDP Lateral Movement | Detect lateral movement via RDP | T1021.001 — Remote Desktop Protocol |
| Scheduled Task Creation | Detect persistence via tasks | T1053.005 — Scheduled Task |
| PowerShell Execution | Detect PowerShell abuse | T1059.001 — PowerShell |
| Network Reconnaissance | Detect internal scanning | T1046 — Network Service Discovery |

---

## Authentication & Account Activity

### 1. Failed Login Threshold
Detects multiple failed login attempts against a single account — indicator of brute force activity.

**Why it matters:** Brute force against Windows accounts is one of the most common initial access techniques. Five failures in five minutes is a reasonable threshold for a lab — lower this to three in production.

**MITRE ATT&CK:** T1110 — Brute Force

```kql
SecurityEvent
| where EventID == 4625
| where TimeGenerated > ago(5m)
| summarize FailedLogins = count() by Account, Computer, IpAddress
| where FailedLogins >= 5
| project TimeGenerated = now(), Account, Computer, IpAddress, FailedLogins
| order by FailedLogins desc
```

---

### 2. Successful Login After Multiple Failures
Detects a successful login that follows multiple failed attempts — strong indicator of successful brute force.

**Why it matters:** A failed login is suspicious. A successful login immediately after multiple failures is a high-confidence indicator of compromise. This query correlates two event types to surface that pattern.

**MITRE ATT&CK:** T1110.001 — Brute Force: Password Guessing

```kql
let FailedLogins = SecurityEvent
| where EventID == 4625
| where TimeGenerated > ago(1h)
| summarize FailedCount = count() by Account, Computer
| where FailedCount >= 3;
SecurityEvent
| where EventID == 4624
| where TimeGenerated > ago(1h)
| join kind=inner FailedLogins on Account, Computer
| project
    TimeGenerated,
    Account,
    Computer,
    FailedCount,
    LogonType,
    IpAddress
| order by TimeGenerated desc
```

---

### 3. Logon Outside Business Hours
Detects successful logins outside of 08:00-18:00 Monday to Friday — anomalous access time indicator.

**Why it matters:** Legitimate users log in during business hours. Attackers operating in different time zones, or automated tools, often trigger logins at unusual times. This query baselines expected behaviour and surfaces deviations.

**MITRE ATT&CK:** T1078 — Valid Accounts

```kql
SecurityEvent
| where EventID == 4624
| where TimeGenerated > ago(24h)
| extend Hour = datetime_part("hour", TimeGenerated)
| extend DayOfWeek = dayofweek(TimeGenerated)
| where Hour < 8 or Hour > 18
| where DayOfWeek != 0 and DayOfWeek != 6
| project
    TimeGenerated,
    Account,
    Computer,
    LogonType,
    IpAddress,
    Hour,
    DayOfWeek
| order by TimeGenerated desc
```

---

### 4. Multiple Failed Logins Across Different Accounts
Detects failed login attempts targeting multiple accounts from the same source — indicator of password spraying.

**Why it matters:** Password spraying differs from brute force — instead of many attempts against one account (which triggers lockout), attackers try one password against many accounts. This query detects the lateral spread pattern rather than the vertical depth pattern.

**MITRE ATT&CK:** T1110.003 — Brute Force: Password Spraying

```kql
SecurityEvent
| where EventID == 4625
| where TimeGenerated > ago(10m)
| summarize
    AttemptedAccounts = dcount(Account),
    AccountList = make_set(Account),
    FailedCount = count()
    by IpAddress, Computer
| where AttemptedAccounts >= 3
| project
    TimeGenerated = now(),
    IpAddress,
    Computer,
    AttemptedAccounts,
    FailedCount,
    AccountList
| order by AttemptedAccounts desc
```

---

### 5. Admin Account Logon
Detects any logon by accounts with Administrator in the name — monitors privileged account usage.

**Why it matters:** Privileged accounts should only be used when necessary. Routine use of admin accounts increases the risk of credential theft and privilege abuse. This query provides visibility into when and where admin accounts are being used.

**MITRE ATT&CK:** T1078.002 — Valid Accounts: Domain Accounts

```kql
SecurityEvent
| where EventID == 4624
| where TimeGenerated > ago(24h)
| where Account contains "admin" or Account contains "Administrator"
| project
    TimeGenerated,
    Account,
    Computer,
    LogonType,
    IpAddress
| order by TimeGenerated desc
```

---

## Persistence & Privilege Escalation

### 6. New Local Account Created
Detects creation of a new local user account — common persistence technique.

**Why it matters:** Attackers create new accounts to maintain persistent access even if their initial credentials are changed or removed. Any new account creation on a server should be investigated unless it is part of a known change management process.

**MITRE ATT&CK:** T1136.001 — Create Account: Local Account

```kql
SecurityEvent
| where EventID == 4720
| where TimeGenerated > ago(24h)
| project
    TimeGenerated,
    Account,
    Computer,
    SubjectUserName,
    TargetUserName,
    TargetDomainName
| order by TimeGenerated desc
```

---

### 7. User Added to Privileged Group
Detects when a user is added to a privileged security group such as Administrators or Remote Desktop Users.

**Why it matters:** Adding a user to a privileged group is a common privilege escalation technique. Attackers with standard user access will attempt to elevate to administrator by modifying group membership.

**MITRE ATT&CK:** T1098 — Account Manipulation

```kql
SecurityEvent
| where EventID == 4728 or EventID == 4732 or EventID == 4756
| where TimeGenerated > ago(24h)
| where TargetUserName contains "Admin"
    or TargetUserName contains "Remote Desktop"
    or TargetUserName contains "Domain Admin"
| project
    TimeGenerated,
    Account,
    Computer,
    SubjectUserName,
    TargetUserName,
    MemberName,
    EventID
| order by TimeGenerated desc
```

---

### 8. Service Installation
Detects installation of a new Windows service — used by attackers for persistence and privilege escalation.

**Why it matters:** Malware commonly installs itself as a Windows service to survive reboots and run with SYSTEM privileges. Any unexpected service installation on a server warrants investigation.

**MITRE ATT&CK:** T1543.003 — Create or Modify System Process: Windows Service

```kql
SecurityEvent
| where EventID == 7045
| where TimeGenerated > ago(24h)
| project
    TimeGenerated,
    Computer,
    ServiceName,
    ServiceType,
    StartType,
    ServiceAccount,
    ImagePath
| order by TimeGenerated desc
```

---

### 9. Scheduled Task Creation
Detects creation of a new scheduled task — used by attackers for persistence and execution.

**Why it matters:** Scheduled tasks are a common persistence mechanism. Attackers create tasks that execute malicious payloads on a schedule, surviving reboots and running without user interaction.

**MITRE ATT&CK:** T1053.005 — Scheduled Task/Job: Scheduled Task

```kql
SecurityEvent
| where EventID == 4698
| where TimeGenerated > ago(24h)
| project
    TimeGenerated,
    Account,
    Computer,
    TaskName,
    TaskContent
| order by TimeGenerated desc
```

---

## Defence Evasion

### 10. Security Log Cleared
Detects clearing of the Windows Security event log — strong indicator of active attack and evidence destruction.

**Why it matters:** Clearing the security log is almost never legitimate. It is a strong indicator that an attacker is attempting to destroy evidence of their activity. This should be treated as a critical alert requiring immediate investigation.

**MITRE ATT&CK:** T1070.001 — Indicator Removal: Clear Windows Event Logs

```kql
SecurityEvent
| where EventID == 1102 or EventID == 104
| where TimeGenerated > ago(24h)
| project
    TimeGenerated,
    Account,
    Computer,
    Activity,
    EventID
| order by TimeGenerated desc
```

---

### 11. Privilege Escalation Attempts
Detects sensitive privilege use — indicates an account is attempting to use elevated permissions.

**Why it matters:** EventID 4673 fires when a privileged service is called. Repeated privilege use by unexpected accounts can indicate an attacker attempting to escalate from standard to administrator.

**MITRE ATT&CK:** T1078 — Valid Accounts

```kql
SecurityEvent
| where EventID == 4673
| where TimeGenerated > ago(1h)
| summarize PrivilegeUseCount = count() by Account, Computer, PrivilegeList
| where PrivilegeUseCount >= 10
| project
    TimeGenerated = now(),
    Account,
    Computer,
    PrivilegeList,
    PrivilegeUseCount
| order by PrivilegeUseCount desc
```

---

## Execution

### 12. PowerShell Execution
Detects PowerShell process creation — monitors for scripting interpreter abuse.

**Why it matters:** PowerShell is the most commonly abused tool in Windows environments. It is legitimate and necessary but also used extensively by attackers for execution, persistence, and lateral movement. This query surfaces all PowerShell execution for review.

**MITRE ATT&CK:** T1059.001 — Command and Scripting Interpreter: PowerShell

```kql
SecurityEvent
| where EventID == 4688
| where TimeGenerated > ago(24h)
| where NewProcessName contains "powershell"
    or NewProcessName contains "pwsh"
| project
    TimeGenerated,
    Account,
    Computer,
    NewProcessName,
    CommandLine,
    ParentProcessName
| order by TimeGenerated desc
```

---

### 13. Suspicious Command Line Execution
Detects process creation events containing commonly abused commands and flags.

**Why it matters:** Attackers use built-in Windows tools (living off the land) to avoid detection. Commands like `net user`, `whoami`, `ipconfig`, and `mimikatz` are commonly used for discovery, credential access, and lateral movement.

**MITRE ATT&CK:** T1059 — Command and Scripting Interpreter

```kql
SecurityEvent
| where EventID == 4688
| where TimeGenerated > ago(24h)
| where CommandLine contains "net user"
    or CommandLine contains "whoami"
    or CommandLine contains "mimikatz"
    or CommandLine contains "procdump"
    or CommandLine contains "wce.exe"
    or CommandLine contains "gsecdump"
    or CommandLine contains "sekurlsa"
    or CommandLine contains "lsadump"
| project
    TimeGenerated,
    Account,
    Computer,
    NewProcessName,
    CommandLine,
    ParentProcessName
| order by TimeGenerated desc
```

---

## Lateral Movement & Discovery

### 14. RDP Lateral Movement
Detects RDP logons originating from within the internal network — indicator of lateral movement.

**Why it matters:** RDP logons from the internet are blocked by NSG rules. RDP logons from inside the network — one internal machine connecting to another — can indicate an attacker moving laterally after initial compromise.

**MITRE ATT&CK:** T1021.001 — Remote Services: Remote Desktop Protocol

```kql
SecurityEvent
| where EventID == 4624
| where TimeGenerated > ago(24h)
| where LogonType == 10
| where IpAddress startswith "10."
| project
    TimeGenerated,
    Account,
    Computer,
    IpAddress,
    LogonType
| order by TimeGenerated desc
```

---

### 15. Network Reconnaissance
Detects high volumes of network connection attempts from a single host — indicator of internal scanning.

**Why it matters:** After gaining initial access, attackers typically scan the internal network to discover other hosts and services. A single host making connections to many different destinations in a short window is a strong indicator of reconnaissance activity.

**MITRE ATT&CK:** T1046 — Network Service Discovery

```kql
SecurityEvent
| where EventID == 5156
| where TimeGenerated > ago(10m)
| summarize
    ConnectionCount = count(),
    UniqueDestinations = dcount(DestAddress)
    by SourceAddress, Computer
| where UniqueDestinations >= 20
| project
    TimeGenerated = now(),
    SourceAddress,
    Computer,
    ConnectionCount,
    UniqueDestinations
| order by UniqueDestinations desc
```

---

## Composite — Multi-Stage Attack Detection

### 16. Full Attack Chain — Initial Access to Privilege Escalation
Detects the complete kill chain from brute force through to privilege escalation in a single query.

**Why it matters:** Individual alerts fire on single events. This query correlates multiple stages of an attack in sequence — a brute force followed by successful login followed by privilege escalation is a high-confidence indicator of active compromise requiring immediate response.

**MITRE ATT&CK:** T1110 → T1078 → T1098

```kql
let TimeWindow = ago(1h);
let BruteForce = SecurityEvent
| where EventID == 4625
| where TimeGenerated > TimeWindow
| summarize FailedCount = count() by Account, Computer
| where FailedCount >= 5;
let SuccessfulLogin = SecurityEvent
| where EventID == 4624
| where TimeGenerated > TimeWindow
| project Account, Computer, SuccessTime = TimeGenerated;
let PrivEsc = SecurityEvent
| where EventID == 4728 or EventID == 4732
| where TimeGenerated > TimeWindow
| project Account, Computer, PrivEscTime = TimeGenerated;
BruteForce
| join kind=inner SuccessfulLogin on Account, Computer
| join kind=inner PrivEsc on Account, Computer
| project
    Account,
    Computer,
    FailedCount,
    SuccessTime,
    PrivEscTime
| order by SuccessTime desc
```

---

## Sentinel Alert Rules

These queries are configured as scheduled analytics rules in Microsoft Sentinel — running on a defined frequency and generating incidents when conditions are met:

| Rule Name | Query | Frequency | Lookup | Threshold |
|---|---|---|---|---|
| Failed Login Threshold | Query 1 | Every 5 minutes | Last 5 minutes | >= 5 failures |
| Successful Login After Failures | Query 2 | Every 15 minutes | Last 1 hour | Any match |
| Security Log Cleared | Query 10 | Every 5 minutes | Last 24 hours | Any match |
| New Local Account Created | Query 6 | Every 15 minutes | Last 24 hours | Any match |
| Suspicious Command Line | Query 13 | Every 15 minutes | Last 24 hours | Any match |

---

## KQL Reference — Useful Operators

A quick reference for the KQL operators used in this query library:

| Operator | Purpose | Example |
|---|---|---|
| `where` | Filter rows | `where EventID == 4625` |
| `summarize` | Aggregate data | `summarize count() by Account` |
| `project` | Select columns | `project TimeGenerated, Account` |
| `extend` | Add calculated column | `extend Hour = datetime_part("hour", TimeGenerated)` |
| `join` | Combine tables | `join kind=inner Table2 on Account` |
| `dcount` | Count distinct values | `dcount(Account)` |
| `make_set` | Create array of values | `make_set(Account)` |
| `ago` | Relative time | `where TimeGenerated > ago(1h)` |
| `order by` | Sort results | `order by TimeGenerated desc` |
| `let` | Define variable | `let TimeWindow = ago(1h)` |
| `contains` | String search | `where Account contains "admin"` |
| `startswith` | String prefix match | `where IpAddress startswith "10."` |
| `datetime_part` | Extract date component | `datetime_part("hour", TimeGenerated)` |
| `dayofweek` | Get day of week | `dayofweek(TimeGenerated)` |

---

## Related Resources

- [Microsoft Sentinel KQL Reference](https://learn.microsoft.com/en-us/azure/sentinel/kusto-overview)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [Windows Security Event IDs Reference](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/security-auditing-overview)
- [Azure SOC Lab](https://github.com/samgarfoot/azure-soc-lab) — extended KQL detection engineering and SOAR automation

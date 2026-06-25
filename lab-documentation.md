# Purple Team Enterprise Lab - Lab Documentation

## Lab Purpose

This lab was built to simulate a practical Purple Team workflow in a segmented enterprise-style environment.

The purpose was to test how a Windows Server placed in a DMZ could be:

```text
Discovered -> Scanned -> Accessed -> Monitored -> Hardened -> Validated
```

The lab included both offensive and defensive activities.

Kali Linux was used as the attack machine. Windows Server Target DMZ was used as the target. Wazuh and Sysmon were used to monitor endpoint and authentication activity. OPNsense was used as the firewall and router between the ATTACK zone, DMZ zone and the existing management LAN.

The goal was not only to perform attack simulation, but also to verify if the activity was visible in Wazuh and then apply hardening based on the findings.

All testing was performed only inside an isolated local Hyper-V lab environment.

---

## Lab Scope

Only systems inside the local Hyper-V lab were included in scope.

Primary target:

```text
Hostname: WIN-TARGET-DMZ
IP address: 10.70.70.40
Network: DMZ
Operating system: Windows Server 2025 Standard Evaluation
```

Attack machine:

```text
System: Kali Linux
IP address: 10.60.60.50
Network: ATTACK
```

Monitoring system:

```text
Wazuh Server: 192.168.10.30
Wazuh Agent: WIN-TARGET-DMZ
```

The following activities were in scope:

* Network reachability testing
* Port scanning
* Vulnerability scanning
* WinRM remote access testing
* SMB enumeration
* Failed authentication testing
* Brute-force style detection review
* Wazuh alert review
* Sysmon verification
* Hardening and validation

The following activities were not performed:

* Malware execution
* Persistence
* Backdoors
* Data exfiltration
* Destructive actions
* Testing against external or unauthorized systems

---

## Lab Environment

| System / Tool             | Purpose                            |
| ------------------------- | ---------------------------------- |
| Hyper-V                   | Virtual lab platform               |
| OPNsense                  | Firewall, routing and segmentation |
| Kali Linux                | Attack machine                     |
| Windows Server Target DMZ | Target server in the DMZ           |
| Wazuh Server              | SIEM and security monitoring       |
| Wazuh Agent               | Endpoint log forwarding            |
| Sysmon                    | Windows endpoint telemetry         |
| Nessus Essentials         | Vulnerability scanning             |
| Nmap                      | Port and service discovery         |
| Evil-WinRM                | WinRM remote access testing        |
| smbclient                 | SMB enumeration                    |

---

## Network Design

The lab used OPNsense as the central router and firewall between three main areas:

| Segment        | Network           | Gateway        | Purpose                        |
| -------------- | ----------------- | -------------- | ------------------------------ |
| Management LAN | `192.168.10.0/24` | `192.168.10.1` | Wazuh, Nessus and admin access |
| ATTACK Zone    | `10.60.60.0/24`   | `10.60.60.1`   | Kali Linux                     |
| DMZ Zone       | `10.70.70.0/24`   | `10.70.70.1`   | Windows Server target          |

Key IP addresses:

| System                    | IP Address      |
| ------------------------- | --------------- |
| Kali Linux                | `10.60.60.50`   |
| OPNsense ATTACK gateway   | `10.60.60.1`    |
| Windows Server Target DMZ | `10.70.70.40`   |
| OPNsense DMZ gateway      | `10.70.70.1`    |
| Wazuh Server              | `192.168.10.30` |

Final OPNsense interface mapping:

| OPNsense Interface | Device | IP Address                    |
| ------------------ | ------ | ----------------------------- |
| WAN                | `hn0`  | Existing WAN / Default Switch |
| LAN                | `hn1`  | `192.168.10.1/24`             |
| DMZ                | `hn2`  | `10.70.70.1/24`               |
| ATTACK             | `hn3`  | `10.60.60.1/24`               |

The existing WAN and LAN were left unchanged. Only the ATTACK and DMZ networks were added for this lab.

---

## Design Decision

The existing working lab environment was not rebuilt or replaced.

The existing LAN continued to be used for management and monitoring. This included Wazuh, Nessus and administrative access. The new ATTACK and DMZ zones were added separately.

This was done to reduce the risk of breaking the existing OPNsense GUI access, internet access or Wazuh access.

The final design allowed the following flow:

```text
Kali Linux
10.60.60.50
ATTACK Zone
        |
        v
OPNsense
Firewall / Router
        |
        v
DMZ Zone
Windows Server Target DMZ
10.70.70.40
```

---

## Network Troubleshooting

Several important network issues appeared during the build. These issues were part of the learning process and were documented because they affected routing, scanning and later attack validation.

### DMZ Target Could Not Reach Gateway

The Windows Server target was configured with:

```text
IP address: 10.70.70.40
Subnet mask: 255.255.255.0
Default gateway: 10.70.70.1
DNS: 10.70.70.1
```

Initial test from Windows Server:

```powershell
ping 10.70.70.1
```

Observed result:

```text
Reply from 10.70.70.40: Destination host unreachable
Request timed out
```

The ARP table was checked:

```powershell
arp -a
```

The gateway MAC address did not appear. This showed that the Windows Server could not reach the OPNsense DMZ interface at Layer 2.

The network adapter was also checked:

```powershell
Get-NetAdapter
```

The adapter was up, which meant the Windows side was not the main issue.

The issue was traced to interface mapping between Hyper-V network adapters and OPNsense interfaces.

The working final mapping became:

```text
DMZ    = hn2
ATTACK = hn3
```

After correcting the interface assignment and confirming that the DMZ interface had `10.70.70.1/24`, the Windows Server target could reach its gateway.

Validation:

```powershell
ping 10.70.70.1
ping 192.168.10.30
```

Observation:

The Windows IP configuration was correct. The problem was caused by OPNsense interface mapping and missing or incorrect interface assignment.

---

### Kali Could Not Reach ATTACK Gateway

Kali was configured with:

```text
IP address: 10.60.60.50/24
Gateway: 10.60.60.1
Interface: eth0
```

The Kali IP configuration and route were checked:

```bash
ip a
ip route
```

Expected values:

```text
inet 10.60.60.50/24
default via 10.60.60.1 dev eth0
```

However, Kali could not ping the ATTACK gateway:

```bash
ping 10.60.60.1
```

Observed result:

```text
From 10.60.60.50 icmp_seq=1 Destination Host Unreachable
```

The ARP table was checked:

```bash
arp -a
```

Observed result:

```text
? (10.60.60.1) at <incomplete> on eth0
```

This meant Kali was asking for the MAC address of `10.60.60.1`, but no device answered.

The issue was found in OPNsense. The ATTACK interface was up, but its IPv4 address was missing. After setting the ATTACK interface to:

```text
10.60.60.1/24
```

Kali could reach the gateway and the DMZ target.

Validation:

```bash
ping 10.60.60.1
ping 10.70.70.40
```

Observation:

The Kali IP configuration was correct. The actual problem was that the OPNsense ATTACK interface did not have its IPv4 address configured.

---

## Kali Network Configuration

Kali used NetworkManager instead of `/etc/network/interfaces`.

Editing `/etc/network/interfaces` was not used because this Kali installation handled networking through NetworkManager.

The interface was identified as:

```text
eth0
```

The active connection name was checked with:

```bash
sudo nmcli con show
```

The connection name was:

```text
Wired connection 1
```

The static IP address was configured with:

```bash
sudo nmcli con mod "Wired connection 1" ipv4.addresses 10.60.60.50/24
sudo nmcli con mod "Wired connection 1" ipv4.gateway 10.60.60.1
sudo nmcli con mod "Wired connection 1" ipv4.dns 10.60.60.1
sudo nmcli con mod "Wired connection 1" ipv4.method manual
```

The connection was restarted:

```bash
sudo nmcli con down "Wired connection 1"
sudo nmcli con up "Wired connection 1"
```

Validation:

```bash
ip a
ip route
```

Confirmed result:

```text
Kali IP: 10.60.60.50/24
Default gateway: 10.60.60.1
```

Observation:

Using `nmcli` was the correct method for this Kali VM.

---

## Windows Server Target Configuration

The Windows Server target was a cloned Windows Server VM placed in the DMZ.

Final target details:

```text
Hostname: WIN-TARGET-DMZ
IP address: 10.70.70.40
Gateway: 10.70.70.1
DNS: 10.70.70.1
Network: DMZ
```

Validation commands:

```powershell
hostname
ipconfig
```

Expected values:

```text
Hostname: WIN-TARGET-DMZ
IPv4 Address: 10.70.70.40
Default Gateway: 10.70.70.1
```

Observation:

The server became the main target for scanning, attack simulation, detection and hardening.

---

## Wazuh Agent Troubleshooting and Registration

The cloned Windows Server inherited the old Wazuh agent identity from the source VM.

The Wazuh dashboard initially showed the old agent name:

```text
Dev_Anstalld
```

This happened because the cloned VM inherited the existing Wazuh agent ID and registration key.

### Wazuh Service Was Disabled

On the Windows Server target, the Wazuh service was checked:

```powershell
Get-Service | findstr Wazuh
```

The service was stopped.

Attempting to start it failed:

```powershell
Start-Service WazuhSvc
```

The service configuration was checked:

```powershell
sc.exe qc WazuhSvc
```

Observed result:

```text
SERVICE_NAME: WazuhSvc
START_TYPE: 4 DISABLED
BINARY_PATH_NAME: "C:\Program Files (x86)\ossec-agent\wazuh-agent.exe"
```

The service was set to automatic and started:

```powershell
Set-Service WazuhSvc -StartupType Automatic
Start-Service WazuhSvc
Get-Service WazuhSvc
```

Confirmed result:

```text
Status   Name       DisplayName
Running  WazuhSvc   Wazuh
```

Observation:

The Wazuh agent service existed, but it was disabled. After changing the startup type to Automatic, the service started correctly.

---

### Wazuh Server Connectivity

The Windows Server target needed to reach the Wazuh Server.

The Wazuh Server IP was checked from Ubuntu:

```bash
ip a
hostname -I
```

Confirmed Wazuh Server IP:

```text
192.168.10.30
```

Connectivity from Windows Server Target DMZ was tested:

```powershell
ping 192.168.10.30
```

Confirmed result:

```text
Reply from 192.168.10.30
```

Observation:

The Windows Server Target DMZ could reach the Wazuh Server through OPNsense routing.

---

### Removing the Old Agent Identity

On the Wazuh Server, the agent manager was opened:

```bash
sudo /var/ossec/bin/manage_agents
```

The existing agents were listed:

```text
L
```

Observed result:

```text
ID: 001, Name: Dev_Anstalld, IP: any
```

The old cloned agent was removed:

```text
R
001
```

A new agent was added:

```text
A
Name: WIN-TARGET-DMZ
IP: any
```

The new agent received:

```text
ID: 002
Name: WIN-TARGET-DMZ
IP: any
```

The agent key was extracted:

```text
E
002
```

Observation:

Removing the old cloned identity and registering a new agent fixed the incorrect Wazuh agent name.

---

### Importing the New Agent Key on Windows

On the Windows Server target, the Wazuh service was stopped:

```powershell
Stop-Service WazuhSvc
```

The Wazuh agent manager was opened:

```powershell
cd "C:\Program Files (x86)\ossec-agent"
.\manage_agents.exe
```

The new key was imported:

```text
I
```

After importing the key, the service was started again:

```powershell
Start-Service WazuhSvc
Get-Service WazuhSvc
```

Confirmed result:

```text
Running  WazuhSvc  Wazuh
```

The Wazuh dashboard then showed:

```text
Agent name: WIN-TARGET-DMZ
IP address: 10.70.70.40
Status: Active
```

Observation:

The target server was correctly registered and actively sending events to Wazuh.

---

## Manual Wazuh Event Test

A manual Windows event was created to verify that events were being generated on the target.

On Windows Server Target DMZ:

```powershell
whoami
eventcreate /T INFORMATION /ID 1000 /L APPLICATION /SO PurpleTeamLab /D "Wazuh test event"
```

Confirmed result:

```text
SUCCESS: An event of type 'INFORMATION' was created in the 'APPLICATION' log with 'PurpleTeamLab' as the source.
```

Observation:

This confirmed that Windows events could be generated on the target before the attack phase.

---

## Sysmon Installation and Verification

Sysmon was installed before baseline scanning and before the main attack phase.

Sysmon was downloaded from Microsoft Sysinternals and extracted on the target server.

The Sysmon executable was verified:

```powershell
cd "$env:USERPROFILE\Downloads\SysinternalsSuite"
.\Sysmon64.exe -?
```

Sysmon was installed:

```powershell
.\Sysmon64.exe -accepteula -i
```

The service was checked:

```powershell
Get-Service Sysmon64
```

Confirmed result:

```text
Status   Name       DisplayName
Running  Sysmon64   Sysmon64
```

Sysmon events were checked locally:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5 | Select-Object TimeCreated, Id, ProviderName, Message
```

Observed event IDs included:

```text
Event ID 1  - Process Create
Event ID 5  - Process Terminated
Event ID 16 - Sysmon config state changed
```

Observation:

Sysmon was installed and generating endpoint telemetry before attack simulation.

---

## Windows Server Attack Surface Preparation

The target server was prepared with exposed services for controlled lab testing.

### IIS Installation

IIS was installed:

```powershell
Install-WindowsFeature Web-Server -IncludeManagementTools
```

The feature was verified:

```powershell
Get-WindowsFeature Web-Server
```

Confirmed result:

```text
[X] Web Server (IIS) Installed
```

Port `80/tcp` was checked:

```powershell
netstat -ano | findstr :80
```

Confirmed result:

```text
0.0.0.0:80 LISTENING
[::]:80 LISTENING
```

Browser validation was performed from another machine:

```text
http://10.70.70.40
```

Observation:

IIS was reachable and became part of the DMZ attack surface.

---

### WinRM Configuration

WinRM was enabled:

```powershell
Enable-PSRemoting -Force
```

The listener was checked:

```powershell
winrm enumerate winrm/config/listener
```

Confirmed result:

```text
Transport = HTTP
Port = 5985
ListeningOn = 10.70.70.40
```

Port `5985/tcp` was checked locally:

```powershell
netstat -ano | findstr :5985
```

Confirmed result:

```text
0.0.0.0:5985 LISTENING
[::]:5985 LISTENING
```

Observation:

WinRM was running locally and listening on port `5985/tcp`.

---

### WinRM Firewall and Network Profile Troubleshooting

From Kali, WinRM initially showed as filtered:

```bash
nmap -Pn -p 5985 10.70.70.40
```

Observed result:

```text
5985/tcp filtered wsman
```

On the Windows Server target, WinRM was running and firewall rules were enabled:

```powershell
Get-NetFirewallRule -DisplayGroup "Windows Remote Management" | Select DisplayName, Enabled
```

Confirmed result:

```text
Windows Remote Management (HTTP-In) True
```

The Windows network profile was checked:

```powershell
Get-NetConnectionProfile
```

Observed result:

```text
NetworkCategory: Public
```

The profile was changed to Private:

```powershell
Set-NetConnectionProfile -InterfaceAlias Ethernet -NetworkCategory Private
```

Validation:

```powershell
Get-NetConnectionProfile
```

Confirmed result:

```text
NetworkCategory: Private
```

WinRM was scanned again from Kali:

```bash
nmap -Pn -p 5985 10.70.70.40
```

Confirmed result:

```text
5985/tcp open wsman
```

Observation:

The WinRM service was running, but the Windows network profile caused filtering. Changing the network category to Private allowed WinRM to become reachable from Kali.

---

### RDP Enablement

RDP was checked:

```powershell
Get-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server" -Name fDenyTSConnections
```

Observed result:

```text
fDenyTSConnections: 1
```

RDP was enabled:

```powershell
Set-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server" -Name fDenyTSConnections -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
```

Port `3389/tcp` was checked:

```powershell
netstat -ano | findstr :3389
```

Observation:

RDP was enabled as part of the controlled attack surface.

---

### Weak Local Administrator Account

A weak local administrator account was created for controlled attack simulation:

```powershell
net user purpleadmin Password123! /add
net localgroup Administrators purpleadmin /add
```

The account was verified:

```powershell
net user purpleadmin
```

Confirmed result:

```text
Account active: Yes
Local Group Memberships: Administrators, Users
Last logon: Never
```

Observation:

The account was intentionally weak and used only in this isolated lab to simulate poor credential management.

---

## Baseline Assessment

### Nmap Service Scan

From Kali, the baseline service scan was performed:

```bash
nmap -sV 10.70.70.40
```

Observed open ports:

|       Port | Service      | Observation    |
| ---------: | ------------ | -------------- |
|   `80/tcp` | HTTP         | Microsoft IIS  |
|  `135/tcp` | MSRPC        | Windows RPC    |
|  `445/tcp` | SMB          | Microsoft SMB  |
| `3389/tcp` | RDP          | Remote Desktop |
| `5985/tcp` | HTTP / WinRM | WinRM service  |

Observation:

The Nmap scan confirmed the expected attack surface from the ATTACK zone.

---

### Nessus Baseline Scan

A Nessus Basic Network Scan was created.

Scan settings:

```text
Scan name: Purple Team Baseline
Target: 10.70.70.40
Policy: Basic Network Scan
Credentials: None
```

This was a non-credentialed scan.

Important baseline result:

```text
Auth: Fail
Critical: 0
High: 0
Vulnerabilities: 25
```

Observed finding categories included:

* SSL certificate issues
* TLS protocol detection
* HTTP service information
* SMB service information
* ICMP timestamp request remote date disclosure

Observation:

The baseline scan showed exposed services and configuration findings, but no Critical or High findings from a non-credentialed perspective.

---

### Credentialed Scan Note

A later credentialed Nessus scan was tested with Windows administrator credentials.

That scan showed:

```text
Auth: Pass
More findings
Windows bulletin-related findings
```

Observation:

The credentialed scan discovered more details because Nessus could log into the target. It was not used as the main after-hardening comparison because the original baseline was non-credentialed. A fair comparison requires the same scan type before and after hardening.

---

## Attack Simulation

### Evil-WinRM Initial Failure

Evil-WinRM was first tested from Kali:

```bash
evil-winrm -i 10.70.70.40 -u purpleadmin -p 'Password123!'
```

Initial error:

```text
WinRM::WinRMAuthorizationError
```

WinRM was reachable, but Windows rejected the remote administrative session.

On the Windows Server target, WinRM configuration was checked:

```powershell
winrm get winrm/config/service
winrm quickconfig
```

Observed issue:

```text
WinRM is not set up to allow remote access to this machine for management.
Configure LocalAccountTokenFilterPolicy to grant administrative rights remotely to local users.
```

The required registry value was added:

```powershell
reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 1 /f
```

WinRM was checked again:

```powershell
winrm quickconfig
```

Confirmed result:

```text
WinRM is already set up for remote management on this computer.
```

Observation:

The issue was not the password or the network path. The problem was Windows remote administrative token filtering for local accounts.

---

### Evil-WinRM Successful Access

Evil-WinRM was tested again:

```bash
evil-winrm -i 10.70.70.40 -u purpleadmin -p 'Password123!'
```

Inside the WinRM session:

```powershell
whoami
hostname
```

Confirmed result:

```text
win-target-dmz\purpleadmin
WIN-TARGET-DMZ
```

Observation:

Remote access from Kali in the ATTACK zone to Windows Server in the DMZ was confirmed.

---

### Basic Post-Access Enumeration

Inside the Evil-WinRM session, basic enumeration commands were used:

```powershell
hostname
whoami /groups
ipconfig /all
net user
net localgroup administrators
systeminfo
```

Observation:

These commands represented typical early post-access discovery steps. They were non-destructive and used to validate access and generate endpoint activity.

---

## SMB Enumeration

### Anonymous SMB Enumeration

Anonymous SMB enumeration was tested:

```bash
smbclient -L //10.70.70.40 -N
```

Observed result:

```text
session setup failed: NT_STATUS_ACCESS_DENIED
```

Observation:

Anonymous SMB access was blocked, which is expected on modern Windows Server systems.

---

### Authenticated SMB Share Enumeration

SMB enumeration was repeated with valid credentials:

```bash
smbclient -L //10.70.70.40 -U purpleadmin
```

Password:

```text
Password123!
```

Observed shares:

```text
ADMIN$
C$
IPC$
Users
```

Additional message:

```text
Unable to connect with SMB1 -- no workgroup available
```

Observation:

The SMB enumeration was successful with valid credentials. The SMB1 workgroup listing error was not a problem because modern Windows systems commonly disable or avoid SMB1-based browsing.

---

### SMB User Profile Enumeration

The `Users` share was accessed:

```bash
smbclient //10.70.70.40/Users -U purpleadmin
```

Password:

```text
Password123!
```

Inside the SMB session:

```text
cd purpleadmin
ls
```

Observed profile folders included:

```text
Desktop
Documents
Downloads
AppData
NTUSER.DAT
Pictures
Videos
```

Observation:

The attacker could enumerate the user profile after authenticating with valid credentials. No files were modified or exfiltrated.

---

## Failed Logons and Brute Force Detection

Several failed authentication attempts were generated to validate Wazuh detection.

A simple failed WinRM attempt was performed:

```bash
evil-winrm -i 10.70.70.40 -u purpleadmin -p 'WrongPassword'
```

Wazuh detected:

```text
Logon Failure - Unknown user or bad password
Rule ID: 60122
Level: 5
```

The Wazuh event included useful fields:

```text
agent.name: WIN-TARGET-DMZ
source IP: 10.60.60.50
authentication package: NTLM
logon type: 3
```

Multiple failures later produced:

```text
Multiple Windows Logon Failures
User account locked out
```

Observed Wazuh rules:

| Wazuh Rule | Description                                  | Level |
| ---------: | -------------------------------------------- | ----: |
|    `60122` | Logon Failure - Unknown user or bad password |     5 |
|    `60204` | Multiple Windows Logon Failures              |    10 |
|    `60115` | User account locked out                      |     9 |

Observation:

This confirmed that Wazuh could detect repeated authentication failures and account lockout activity.

---

## Hydra Test Note

Hydra was tested during the lab, but the first attempt was not kept as final evidence.

Command used:

```bash
hydra -l purpleadmin -P passwords.txt 10.70.70.40 http-get /wsman
```

Observed issue:

```text
5 valid passwords found
```

This result was not valid because the module tested an HTTP GET path and interpreted the web response incorrectly. Wazuh did not generate useful authentication alerts from this specific Hydra attempt.

Observation:

The Hydra screenshot was not used in the final documentation because it did not provide reliable evidence. The Wazuh brute-force and account-lockout screenshot was kept instead because it showed real Windows authentication events.

---

## Wazuh Detection Review

Wazuh was reviewed throughout the attack phase.

Important detections included:

| Detection                                | Meaning                                        |
| ---------------------------------------- | ---------------------------------------------- |
| Successful Remote Logon Detected         | WinRM access using valid credentials           |
| Special privileges assigned to new logon | Administrator-level session                    |
| Logon Failure                            | Failed authentication attempt                  |
| Multiple Windows Logon Failures          | Repeated authentication failures               |
| User account locked out                  | Account lockout after multiple failed attempts |
| Windows User Logoff                      | Session ended                                  |

Example successful remote logon rule:

```text
Successful Remote Logon Detected
User: purpleadmin
Rule ID: 92652
Level: 6
```

Example failed logon rule:

```text
Logon Failure - Unknown user or bad password
Rule ID: 60122
Level: 5
```

Observation:

Wazuh provided useful visibility into the attack path and authentication activity from the DMZ server.

---

## Sysmon Review

Sysmon was verified locally on the Windows Server target.

Command:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 20 | Select TimeCreated,Id,Message
```

Observed event IDs:

```text
Event ID 1 - Process Create
Event ID 5 - Process Terminated
Event ID 16 - Sysmon config state changed
```

Wazuh also showed some Sysmon-related registry activity, including registry key and value changes.

Observation:

Sysmon was active and generating endpoint telemetry. However, not every process event was clearly visible in the main Wazuh Threat Hunting view during the lab. This was documented as an observation rather than overstating the result.

---

## Hardening Phase

The hardening phase focused on removing the demonstrated attack path.

### Remove Weak Local Administrator Account

Before removal, the account was checked:

```powershell
net user purpleadmin
```

The account showed:

```text
Account active: Locked
Local Group Memberships: Administrators, Users
```

The account was removed:

```powershell
net user purpleadmin /delete
```

Verification:

```powershell
net user purpleadmin
```

Expected result:

```text
The user name could not be found.
```

Observation:

The weak local administrator account used during the attack phase was removed.

---

### Evil-WinRM Validation After Account Removal

The previous Evil-WinRM command was tested again from Kali:

```bash
evil-winrm -i 10.70.70.40 -u purpleadmin -p 'Password123!'
```

Observed result:

```text
WinRM::WinRMAuthorizationError
```

Observation:

The previous WinRM attack path no longer worked with the removed account.

---

### WinRM Access Restriction

WinRM was also restricted so that it was no longer openly reachable from the ATTACK zone.

Validation from Kali:

```bash
nmap -Pn -p 5985 10.70.70.40
```

Observed result:

```text
5985/tcp filtered wsman
```

Observation:

Port `5985/tcp` was no longer openly reachable from Kali. This reduced the attack surface and validated the hardening action.

---

## After-Hardening Nessus Validation

A new non-credentialed Nessus scan was performed after hardening.

Scan settings:

```text
Scan name: Purple Team After Hardening (Non-Credentialed)
Target: 10.70.70.40
Policy: Basic Network Scan
Credentials: None
```

Result:

```text
Vulnerabilities: 25
Critical: 0
High: 0
```

Observation:

The non-credentialed Nessus result stayed mostly the same because removing `purpleadmin` does not significantly change what Nessus can see externally. The stronger validation came from Evil-WinRM failing and Nmap showing WinRM as filtered.

---

## Key Findings

| Finding                           | Result                                                           | Significance                              |
| --------------------------------- | ---------------------------------------------------------------- | ----------------------------------------- |
| Segmented network path validated  | Kali reached DMZ target through OPNsense routing                 | Positive architecture result              |
| ATTACK interface issue identified | Missing IPv4 on OPNsense ATTACK interface caused gateway failure | Important troubleshooting result          |
| DMZ mapping issue identified      | Interface mapping had to be corrected                            | Important troubleshooting result          |
| Wazuh cloned-agent issue resolved | Old agent identity removed and new one registered                | Correct monitoring identity               |
| WinRM initially filtered          | Windows network profile was Public                               | Firewall/profile troubleshooting          |
| WinRM authorization failed        | LocalAccountTokenFilterPolicy was missing                        | Windows remote admin troubleshooting      |
| Evil-WinRM access achieved        | Access confirmed as `purpleadmin`                                | Confirmed attack path                     |
| SMB enumeration successful        | Shares and user profile folders listed                           | Confirmed post-authentication enumeration |
| Wazuh detection successful        | Successful logons, failed logons and lockout detected            | Confirmed SIEM visibility                 |
| Sysmon active                     | Process and service events generated locally                     | Confirmed endpoint telemetry              |
| Weak account removed              | `purpleadmin` removed during hardening                           | Reduced credential risk                   |
| WinRM filtered after hardening    | Port `5985/tcp` no longer openly reachable                       | Attack path reduced                       |

---

## Security Impact

The lab showed that a weak local administrator account combined with exposed WinRM created a realistic remote access risk.

Before hardening:

```text
Evil-WinRM access worked with purpleadmin
WinRM port 5985 was reachable
SMB enumeration worked with valid credentials
Wazuh detected successful and failed authentication activity
```

After hardening:

```text
purpleadmin was removed
Evil-WinRM access failed
WinRM port 5985 was filtered from Kali
The previous attack path was no longer available
```

This demonstrated how detection and hardening can be connected in a practical Purple Team workflow.

---

## What Worked Well

The following parts worked successfully:

* OPNsense routed traffic between ATTACK and DMZ after interface fixes
* Kali was successfully placed in the ATTACK zone
* Windows Server Target DMZ was reachable after DMZ troubleshooting
* Wazuh agent was correctly registered as `WIN-TARGET-DMZ`
* Sysmon was installed and generated events
* IIS, SMB, RDP and WinRM were exposed for controlled testing
* Nmap identified the exposed services
* Nessus completed a non-credentialed baseline scan
* Evil-WinRM successfully connected before hardening
* SMB enumeration worked with valid credentials
* Wazuh detected successful remote logons
* Wazuh detected failed logons
* Wazuh detected multiple logon failures
* Wazuh detected account lockout
* The weak account was removed
* Evil-WinRM failed after hardening
* Nmap showed WinRM as filtered after hardening

---

## What Did Not Work

| Issue                                                  | Cause                                     | Resolution                               |
| ------------------------------------------------------ | ----------------------------------------- | ---------------------------------------- |
| DMZ target could not reach gateway                     | Interface mapping or IP issue in OPNsense | Corrected DMZ interface and IP           |
| Kali could not reach ATTACK gateway                    | ATTACK interface had no IPv4 address      | Configured `10.60.60.1/24`               |
| Wazuh showed old agent name                            | Cloned VM inherited old Wazuh identity    | Removed old agent and registered new one |
| Wazuh service would not start                          | Service startup type was Disabled         | Changed startup type to Automatic        |
| WinRM port was filtered                                | Windows network profile was Public        | Changed profile to Private               |
| Evil-WinRM failed with valid password                  | LocalAccountTokenFilterPolicy missing     | Added required registry value            |
| Hydra result was misleading                            | Wrong module/path tested HTTP response    | Not used as final evidence               |
| Sysmon process events were not always obvious in Wazuh | Dashboard/query visibility limitation     | Verified Sysmon locally                  |

Observation:

These issues were useful because they made the lab more realistic. Troubleshooting became an important part of the final project instead of being separate from the security workflow.

---

## Lessons Learned

This lab reinforced several important cybersecurity lessons:

* Network segmentation must be validated at both IP and Layer 2 levels.
* A correct IP configuration on a VM does not help if the firewall interface is missing its address.
* Interface mapping in virtual firewalls must be documented carefully.
* Cloned systems can inherit old agent identities and cause misleading monitoring data.
* Monitoring agents should be installed before attack simulation to avoid missing important logs.
* A port can be listening locally but still filtered from the attacker network.
* Windows network profiles can affect firewall exposure.
* WinRM access with local administrator accounts can require additional configuration.
* Weak credentials can quickly lead to remote administrative access.
* SMB enumeration after authentication can expose useful information.
* Wazuh can provide strong visibility into authentication activity.
* Brute-force style behaviour can trigger multiple failed logon and account lockout alerts.
* Not every tool output is useful evidence; misleading results should be excluded.
* Hardening should be validated with the same attack path that worked before.
* A strong Purple Team project should connect attack, detection and improvement.

---

## Skills Demonstrated

* Hyper-V virtual lab management
* OPNsense firewall and routing configuration
* Network segmentation troubleshooting
* Kali Linux network configuration with `nmcli`
* Windows Server static IP configuration
* IIS installation and validation
* WinRM configuration and troubleshooting
* RDP enablement
* SMB enumeration
* Wazuh agent registration and troubleshooting
* Sysmon installation and verification
* Nmap service discovery
* Nessus vulnerability scanning
* Evil-WinRM remote access validation
* Authentication failure testing
* Wazuh alert review
* Account lockout detection
* Security hardening
* Post-hardening validation
* Purple Team documentation

---

## Final Outcome

The final result was a complete Purple Team lab showing the full cycle:

```text
Design -> Build -> Troubleshoot -> Baseline -> Attack -> Detect -> Harden -> Validate
```

The most important result was that the same attack path that worked before hardening no longer worked afterwards.

Before hardening:

```text
Evil-WinRM access worked with purpleadmin
WinRM port 5985 was reachable
SMB enumeration worked with valid credentials
Wazuh detected successful and failed authentication activity
```

After hardening:

```text
purpleadmin was removed
Evil-WinRM access failed
WinRM port 5985 was filtered from Kali
The previous attack path was no longer available
```

This completed the lab as a practical Purple Team workflow combining offensive testing, defensive detection and security improvement.

---

## Author

Muhammad Mehdi
IT Security Developer Student

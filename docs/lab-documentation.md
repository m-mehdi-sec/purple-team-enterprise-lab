# Lab Documentation – Purple Team Enterprise Lab

## Technical Objective

The objective of this lab was to build and validate a controlled Purple Team workflow in a segmented Hyper-V environment.

The focus was to test a complete chain from network setup to attack simulation, detection review, hardening and validation.

The main technical question was:

```text
Can an exposed Windows Server in a DMZ be accessed from an ATTACK zone, detected by Wazuh, hardened, and then validated again?
```

The lab focused on:

* Building separate ATTACK and DMZ zones without changing the existing management LAN
* Routing traffic between lab zones through OPNsense
* Preparing a Windows Server target with exposed services
* Installing and validating Wazuh Agent and Sysmon before attack simulation
* Performing baseline scanning with Nmap and Nessus
* Testing WinRM access with Evil-WinRM
* Performing SMB enumeration with valid credentials
* Generating failed authentication events
* Reviewing detections in Wazuh
* Removing the weak account and restricting the attack path
* Validating that the previous access path no longer worked

All testing was performed only inside my own isolated local Hyper-V lab.

---

## Lab Scenario

The lab simulated a small enterprise-style environment with a monitored DMZ target and a separate attacker network.

The Windows Server target was placed in the DMZ to represent an exposed internal server. Kali Linux was placed in a dedicated ATTACK zone to represent an internal testing or adversary simulation system.

The test path was:

```text
Kali Linux
10.60.60.50
ATTACK Zone
        |
        | routed through OPNsense
        |
Windows Server Target DMZ
10.70.70.40
DMZ Zone
```

The target was monitored by Wazuh Agent and Sysmon before the attack phase started. This was important because the goal was not only to perform attack simulation, but also to verify that security events were visible afterwards.

---

## Approved Systems

Only the following lab systems were used.

| System                    | Role                                        | IP Address                                 |
| ------------------------- | ------------------------------------------- | ------------------------------------------ |
| OPNsense                  | Firewall and router                         | `192.168.10.1`, `10.60.60.1`, `10.70.70.1` |
| Kali Linux                | Attack machine                              | `10.60.60.50`                              |
| Windows Server Target DMZ | Target server                               | `10.70.70.40`                              |
| Wazuh Server              | SIEM and monitoring server                  | `192.168.10.30`                            |
| Windows 11 / Nessus       | Vulnerability scanning and dashboard access | Management LAN                             |

The existing management LAN was left unchanged. Only two new internal Hyper-V switches were added for this lab:

```text
ATTACK_Switch
DMZ_Switch
```

The existing WAN and LAN setup was not rebuilt, because it was already working and used for management, Wazuh and Nessus access.

---

## OPNsense Interface Configuration

The OPNsense VM was extended with two additional network adapters.

Final interface mapping:

| Interface | Device | Network                       | IP Address        |
| --------- | ------ | ----------------------------- | ----------------- |
| WAN       | `hn0`  | Existing WAN / Default Switch | DHCP              |
| LAN       | `hn1`  | Management LAN                | `192.168.10.1/24` |
| DMZ       | `hn2`  | DMZ Zone                      | `10.70.70.1/24`   |
| ATTACK    | `hn3`  | ATTACK Zone                   | `10.60.60.1/24`   |

The final goal was:

```text
Kali Linux -> ATTACK gateway -> OPNsense -> DMZ gateway -> Windows Server Target
```

The DMZ gateway was:

```text
10.70.70.1
```

The ATTACK gateway was:

```text
10.60.60.1
```

---

## OPNsense Interface Troubleshooting

During the build, the first major issue was incorrect interface mapping.

The Windows Server target was configured with:

```text
IP address: 10.70.70.40
Subnet mask: 255.255.255.0
Default gateway: 10.70.70.1
DNS: 10.70.70.1
```

The first gateway test from Windows Server failed.

Command used on `WIN-TARGET-DMZ`:

```powershell
ping 10.70.70.1
```

Observed result:

```text
Reply from 10.70.70.40: Destination host unreachable
```

The ARP table was checked:

```powershell
arp -a
```

The gateway MAC address was not resolved correctly. This indicated that the issue was not normal IP routing, but that the target could not reach the OPNsense DMZ interface at Layer 2.

The Windows network adapter was also checked:

```powershell
Get-NetAdapter
```

The adapter was up, so the issue was not that the Windows NIC was disabled.

The problem was traced to OPNsense interface mapping. The new Hyper-V adapters had to be matched correctly to the OPNsense interfaces.

Final corrected mapping:

```text
DMZ    = hn2
ATTACK = hn3
```

After correcting the mapping and assigning `10.70.70.1/24` to the DMZ interface, the gateway became reachable.

Validation from Windows Server:

```powershell
ping 10.70.70.1
ping 192.168.10.30
```

Observation:

The Windows Server IP configuration was correct. The real issue was incorrect OPNsense interface assignment.

---

## ATTACK Gateway Troubleshooting

Kali Linux was configured with:

```text
IP address: 10.60.60.50/24
Gateway: 10.60.60.1
Interface: eth0
```

The Kali IP configuration and routing table were checked:

```bash
ip a
ip route
```

Expected values:

```text
inet 10.60.60.50/24
default via 10.60.60.1 dev eth0
```

However, Kali could not reach the ATTACK gateway.

Command used on Kali:

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

This showed that Kali was sending ARP requests for `10.60.60.1`, but no device was answering.

The problem was found in OPNsense. The ATTACK interface was up, but the IPv4 address was missing.

The ATTACK interface was corrected to:

```text
10.60.60.1/24
```

Validation from Kali:

```bash
ping 10.60.60.1
ping 10.70.70.40
```

Observation:

The Kali IP configuration was already correct. The real issue was that the OPNsense ATTACK interface did not have its IPv4 address configured.

---

## Kali Network Configuration

Kali used NetworkManager, so the configuration was done with `nmcli`.

The active connection was checked:

```bash
sudo nmcli con show
```

The connection name was:

```text
Wired connection 1
```

Static IP configuration:

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

## Windows Server Target Preparation

The Windows Server target was created by cloning an existing Windows Server VM and placing the clone in the DMZ network.

Final target configuration:

```text
Hostname: WIN-TARGET-DMZ
IP address: 10.70.70.40
Gateway: 10.70.70.1
DNS: 10.70.70.1
Network: DMZ
```

Validation commands on Windows Server:

```powershell
hostname
ipconfig
```

Confirmed values:

```text
Hostname: WIN-TARGET-DMZ
IPv4 Address: 10.70.70.40
Default Gateway: 10.70.70.1
```

The target server became the main system for:

* Baseline scanning
* WinRM testing
* SMB enumeration
* Wazuh detection review
* Sysmon verification
* Hardening validation

---

## Wazuh Agent Issue After Cloning

The cloned Windows Server inherited the old Wazuh agent identity from the source VM.

The Wazuh dashboard initially showed the old agent name:

```text
Dev_Anstalld
```

This happened because the cloned VM kept the previous Wazuh agent registration and identity.

The Wazuh service was checked on `WIN-TARGET-DMZ`:

```powershell
Get-Service | findstr Wazuh
```

The service existed but was not running correctly.

The service configuration was checked:

```powershell
sc.exe qc WazuhSvc
```

Observed result:

```text
START_TYPE: 4 DISABLED
```

The service startup type was changed and the service was started:

```powershell
Set-Service WazuhSvc -StartupType Automatic
Start-Service WazuhSvc
Get-Service WazuhSvc
```

Confirmed result:

```text
Running  WazuhSvc  Wazuh
```

Observation:

The Wazuh agent service was installed but disabled. It had to be enabled before registration could be completed properly.

---

## Wazuh Manager IP Correction

The Wazuh Server IP was:

```text
192.168.10.30
```

The cloned agent still pointed to the old manager IP, so the agent configuration was corrected on the Windows Server target.

File checked on `WIN-TARGET-DMZ`:

```powershell
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
```

The manager address was corrected to:

```xml
<server>
  <address>192.168.10.30</address>
</server>
```

The Wazuh service was restarted:

```powershell
Restart-Service WazuhSvc
Get-Service WazuhSvc
```

Observation:

Correcting the Wazuh manager IP was required so the target could communicate with the current Wazuh Server.

---

## Wazuh Agent Re-registration

The old cloned agent identity was removed from the Wazuh Server.

Command used on the Wazuh Server:

```bash
sudo /var/ossec/bin/manage_agents
```

Existing agents were listed:

```text
L
```

Observed old agent:

```text
ID: 001
Name: Dev_Anstalld
IP: any
```

The old agent was removed:

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
```

The key was extracted:

```text
E
002
```

The key was then imported on the Windows Server target.

Commands on `WIN-TARGET-DMZ`:

```powershell
Stop-Service WazuhSvc
cd "C:\Program Files (x86)\ossec-agent"
.\manage_agents.exe
```

Inside the agent manager:

```text
I
```

After importing the key:

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

Removing the old cloned agent and registering a new one fixed the identity problem. This was important because all later detection evidence had to be linked to the correct target name.

---

## Wazuh Event Test

A manual event was created on the Windows Server target to confirm that Windows events could be generated.

Command used on `WIN-TARGET-DMZ`:

```powershell
eventcreate /T INFORMATION /ID 1000 /L APPLICATION /SO PurpleTeamLab /D "Wazuh test event"
```

Observed result:

```text
SUCCESS: An event of type 'INFORMATION' was created in the 'APPLICATION' log with 'PurpleTeamLab' as the source.
```

Observation:

This confirmed that the target could generate Windows events before the attack phase started.

---

## Sysmon Installation

Sysmon was installed before the attack simulation so endpoint telemetry would already be available.

Sysmon was downloaded from Microsoft Sysinternals and extracted on the target server.

Commands used on `WIN-TARGET-DMZ`:

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
Running  Sysmon64  Sysmon64
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

Sysmon was installed and generating endpoint telemetry before the attack simulation.

---

## IIS Preparation

IIS was installed on the Windows Server target to expose a basic web service.

Command used on `WIN-TARGET-DMZ`:

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

Browser validation:

```text
http://10.70.70.40
```

Observation:

IIS was reachable and became part of the DMZ attack surface.

---

## WinRM Preparation

WinRM was enabled on the Windows Server target.

Command used on `WIN-TARGET-DMZ`:

```powershell
Enable-PSRemoting -Force
```

The listener was checked:

```powershell
winrm enumerate winrm/config/listener
```

Confirmed listener information:

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

## WinRM Filtering Troubleshooting

From Kali, WinRM was initially shown as filtered.

Command used on Kali:

```bash
nmap -Pn -p 5985 10.70.70.40
```

Observed result:

```text
5985/tcp filtered wsman
```

On the Windows Server target, WinRM firewall rules were checked:

```powershell
Get-NetFirewallRule -DisplayGroup "Windows Remote Management" | Select DisplayName, Enabled
```

The WinRM firewall rule was enabled.

The network profile was then checked:

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

WinRM was listening locally, but the Windows network profile affected remote reachability. Changing the profile from Public to Private allowed WinRM to become reachable from Kali.

---

## RDP Preparation

RDP was enabled as part of the controlled exposed services on the target.

The current RDP setting was checked:

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

RDP was enabled as part of the controlled DMZ attack surface.

---

## Weak Local Administrator Account

A weak local administrator account was created for controlled attack simulation.

Commands used on `WIN-TARGET-DMZ`:

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
```

Observation:

The account was intentionally weak and used only inside this isolated lab to simulate poor credential management.

---

## Baseline Nmap Scan

A service and version scan was performed from Kali against the DMZ target.

Command used on Kali:

```bash
nmap -sV 10.70.70.40
```

Observed open ports:

```text
80/tcp    open  http
135/tcp   open  msrpc
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
```

Result summary:

|       Port | Service    | Meaning                     |
| ---------: | ---------- | --------------------------- |
|   `80/tcp` | HTTP / IIS | Web service exposed         |
|  `135/tcp` | MSRPC      | Windows RPC exposed         |
|  `445/tcp` | SMB        | File sharing reachable      |
| `3389/tcp` | RDP        | Remote desktop reachable    |
| `5985/tcp` | WinRM      | Remote PowerShell reachable |

Observation:

The scan confirmed that the target exposed the expected services from the ATTACK zone.

---

## Nessus Baseline Scan

A Nessus Basic Network Scan was performed against the DMZ target.

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

The baseline scan showed externally visible services and configuration findings, but no Critical or High findings from a non-credentialed perspective.

---

## Nessus Credentialed Scan Note

A credentialed Nessus scan was also tested later with Windows administrator credentials.

That scan showed:

```text
Auth: Pass
More findings
Additional Windows-related details
```

Observation:

The credentialed scan discovered more information because Nessus could authenticate to the target. It was not used as the main before-and-after comparison because the original baseline scan was non-credentialed. A fair comparison requires using the same scan type before and after hardening.

---

## Evil-WinRM Initial Failure

Evil-WinRM was tested from Kali.

Command used:

```bash
evil-winrm -i 10.70.70.40 -u purpleadmin -p 'Password123!'
```

Observed error:

```text
WinRM::WinRMAuthorizationError
```

WinRM was reachable, and the password was correct, but Windows rejected the remote administrative session.

WinRM configuration was checked on the Windows Server target:

```powershell
winrm get winrm/config/service
winrm quickconfig
```

Observed issue:

```text
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

## Evil-WinRM Successful Access

Evil-WinRM was tested again from Kali.

Command used:

```bash
evil-winrm -i 10.70.70.40 -u purpleadmin -p 'Password123!'
```

Commands inside the Evil-WinRM session:

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

Remote access from Kali in the ATTACK zone to the Windows Server target in the DMZ was confirmed.

---

## Basic Post-Access Validation

Inside the Evil-WinRM session, basic non-destructive validation commands were used.

Commands used:

```powershell
hostname
whoami
whoami /groups
ipconfig /all
net user
net localgroup administrators
systeminfo
```

Observation:

These commands were used only to validate access and generate normal endpoint activity. No persistence, malware, destructive changes or data exfiltration were performed.

---

## Anonymous SMB Enumeration

Anonymous SMB enumeration was tested first.

Command used on Kali:

```bash
smbclient -L //10.70.70.40 -N
```

Observed result:

```text
session setup failed: NT_STATUS_ACCESS_DENIED
```

Observation:

Anonymous SMB access was denied, which is expected on a modern Windows Server configuration.

---

## Authenticated SMB Share Enumeration

SMB enumeration was repeated with valid credentials.

Command used on Kali:

```bash
smbclient -L //10.70.70.40 -U purpleadmin
```

Password used:

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

The authenticated SMB enumeration worked. The SMB1 workgroup message was not treated as a failure because the share listing still succeeded and modern Windows environments commonly do not rely on SMB1 browsing.

---

## SMB User Profile Enumeration

The `Users` share was accessed with valid credentials.

Command used on Kali:

```bash
smbclient //10.70.70.40/Users -U purpleadmin
```

Password used:

```text
Password123!
```

Commands inside the SMB session:

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

Authenticated SMB access allowed the attacker to enumerate the local user profile structure. No files were modified or exfiltrated.

---

## Failed Authentication Testing

Failed authentication attempts were generated to validate Wazuh detection.

Example failed Evil-WinRM command:

```bash
evil-winrm -i 10.70.70.40 -u purpleadmin -p 'WrongPassword'
```

Observed Wazuh detection:

```text
Logon Failure - Unknown user or bad password
Rule ID: 60122
Level: 5
```

Important event fields included:

```text
agent.name: WIN-TARGET-DMZ
source IP: 10.60.60.50
authentication package: NTLM
logon type: 3
```

Observation:

Failed remote authentication attempts from Kali were visible in Wazuh.

---

## Brute-Force Style Detection

Repeated authentication failures were generated to test whether Wazuh could identify stronger authentication patterns.

Observed Wazuh detections included:

| Rule ID | Detection                                    | Level |
| ------: | -------------------------------------------- | ----: |
| `60122` | Logon Failure - Unknown user or bad password |     5 |
| `60204` | Multiple Windows Logon Failures              |    10 |
| `60115` | User account locked out                      |     9 |

Observation:

Wazuh escalated repeated failed authentication activity and detected account lockout behaviour.

---

## Hydra Test Note

Hydra was tested during the lab, but the first command was not used as final evidence.

Command used:

```bash
hydra -l purpleadmin -P passwords.txt 10.70.70.40 http-get /wsman
```

Observed issue:

```text
5 valid passwords found
```

This result was not reliable because the command tested an HTTP GET path and interpreted the web response incorrectly. It did not provide useful Windows authentication evidence in Wazuh.

Observation:

The Hydra result was excluded from the final evidence. The Wazuh brute-force and account-lockout detection was used instead because it showed real Windows authentication events.

---

## Wazuh Detection Review

Wazuh was reviewed throughout the attack phase.

Important detections included:

| Detection                                | Meaning                                        |
| ---------------------------------------- | ---------------------------------------------- |
| Successful Remote Logon Detected         | WinRM access using valid credentials           |
| Special privileges assigned to new logon | Administrator-level logon context              |
| Logon Failure                            | Failed authentication attempt                  |
| Multiple Windows Logon Failures          | Repeated authentication failures               |
| User account locked out                  | Account lockout after multiple failed attempts |
| Windows User Logoff                      | End of user session                            |

Example successful remote logon:

```text
Successful Remote Logon Detected
User: purpleadmin
Agent: WIN-TARGET-DMZ
```

Example failed logon:

```text
Logon Failure - Unknown user or bad password
User: purpleadmin
Source IP: 10.60.60.50
```

Observation:

Wazuh provided useful visibility into authentication activity on the DMZ target.

---

## Sysmon Review

Sysmon was verified locally on the Windows Server target.

Command used:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 20 | Select TimeCreated,Id,Message
```

Observed event IDs:

```text
Event ID 1  - Process Create
Event ID 5  - Process Terminated
Event ID 16 - Sysmon config state changed
```

Wazuh also showed some Sysmon-related registry activity.

Observation:

Sysmon was active and generating endpoint telemetry. However, not every process creation event was clearly visible in the main Wazuh Threat Hunting view during the lab. This was documented as an observation instead of overstating the result.

---

## Hardening Action: Remove Weak Account

The hardening phase started by removing the weak local administrator account.

Before removal, the account was checked:

```powershell
net user purpleadmin
```

Observed state:

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

The weak local administrator account used during the attack simulation was removed.

---

## Validation: Evil-WinRM After Hardening

The same Evil-WinRM command was tested again from Kali.

Command used:

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

## Hardening Action: Restrict WinRM Exposure

WinRM access from the ATTACK zone was restricted so that port `5985/tcp` was no longer openly reachable from Kali.

Validation from Kali:

```bash
nmap -Pn -p 5985 10.70.70.40
```

Observed result:

```text
5985/tcp filtered wsman
```

Observation:

Port `5985/tcp` was no longer openly reachable from the ATTACK zone. This reduced the remote administration attack surface.

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

Observed result:

```text
Vulnerabilities: 25
Critical: 0
High: 0
```

Observation:

The non-credentialed Nessus result stayed mostly the same because removing `purpleadmin` does not significantly change what Nessus can see externally. The stronger validation came from Evil-WinRM failing and Nmap showing WinRM as filtered.

---

## Important Troubleshooting Results

| Issue                                     | Cause                                   | Resolution                                        |
| ----------------------------------------- | --------------------------------------- | ------------------------------------------------- |
| DMZ target could not reach gateway        | Incorrect OPNsense interface mapping    | Corrected DMZ interface assignment                |
| Kali could not reach ATTACK gateway       | ATTACK interface had no IPv4 address    | Configured `10.60.60.1/24`                        |
| Wazuh showed old agent name               | Cloned VM inherited old Wazuh identity  | Removed old agent and registered `WIN-TARGET-DMZ` |
| Wazuh service would not start             | Service startup type was Disabled       | Changed startup type to Automatic                 |
| Wazuh agent pointed to old manager        | Old manager IP remained in `ossec.conf` | Corrected manager IP to `192.168.10.30`           |
| WinRM port was filtered                   | Windows network profile was Public      | Changed profile to Private                        |
| Evil-WinRM failed with valid password     | LocalAccountTokenFilterPolicy missing   | Added required registry value                     |
| Hydra result was misleading               | Wrong module/path tested HTTP response  | Excluded from final evidence                      |
| Sysmon events were not all clear in Wazuh | Dashboard/query limitation              | Verified Sysmon locally                           |

Observation:

These troubleshooting steps became an important part of the lab because they showed realistic problems that can appear when building, attacking and monitoring a segmented environment.

---

## Technical Findings

| Finding                        | Result                                                | Significance                           |
| ------------------------------ | ----------------------------------------------------- | -------------------------------------- |
| Segmented ATTACK-to-DMZ path   | Kali reached the target through OPNsense              | Confirmed routed lab design            |
| Target monitoring active       | Wazuh Agent active before attack                      | Logs were available during testing     |
| Sysmon installed               | Sysmon generated endpoint telemetry                   | Improved endpoint visibility           |
| Exposed services identified    | Nmap found IIS, RPC, SMB, RDP and WinRM               | Defined the attack surface             |
| Nessus baseline completed      | No Critical or High findings in non-credentialed scan | Established baseline                   |
| WinRM access achieved          | Evil-WinRM worked with weak credentials               | Confirmed remote access risk           |
| SMB enumeration worked         | Shares and user profile folders were visible          | Confirmed post-authentication exposure |
| Failed logons detected         | Wazuh detected wrong password attempts                | Confirmed authentication monitoring    |
| Account lockout detected       | Wazuh showed multiple failures and lockout            | Confirmed stronger detection signal    |
| Weak account removed           | `purpleadmin` deleted                                 | Reduced credential risk                |
| WinRM filtered after hardening | Port `5985/tcp` no longer open from Kali              | Reduced attack path                    |

---

## Security Impact

The lab showed that exposed WinRM combined with weak local administrator credentials created a realistic remote access risk.

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

The most important result was that the same attack path that worked before hardening no longer worked afterwards.

---

## Defensive Improvements

The following defensive improvements were applied or validated:

| Area               | Improvement                                                         |
| ------------------ | ------------------------------------------------------------------- |
| Account security   | Removed weak local administrator account                            |
| Remote access      | Restricted WinRM exposure from the ATTACK zone                      |
| Monitoring         | Confirmed Wazuh visibility for successful and failed authentication |
| Endpoint telemetry | Verified Sysmon was active on the target                            |
| Validation         | Re-tested the same access path after hardening                      |

Observation:

The lab connected attack simulation with detection and improvement, which is the main purpose of a Purple Team workflow.

---

## Detection Opportunities

If similar activity occurred in a real enterprise environment, the following data sources would be useful for investigation:

| Data Source            | Possible Evidence                                 |
| ---------------------- | ------------------------------------------------- |
| Wazuh alerts           | Successful logons, failed logons, account lockout |
| Windows Security logs  | Logon type, source IP, NTLM authentication        |
| Sysmon logs            | Process creation and endpoint telemetry           |
| OPNsense firewall logs | Cross-zone traffic and blocked connections        |
| Nessus scan results    | Baseline vulnerability and exposure information   |
| Nmap results           | Service exposure from the attacker network        |

Observation:

Wazuh provided useful endpoint visibility during the lab. Stronger correlation between exact timestamps, firewall logs and endpoint events would improve analysis in a larger environment.

---

## Lessons Learned

This lab reinforced several important lessons:

* Network segmentation must be validated before scanning or attack simulation.
* OPNsense interface mapping must be checked carefully when adding new Hyper-V adapters.
* A VM can have the correct IP address but still fail if the firewall interface is missing its gateway IP.
* Cloned systems can inherit old monitoring identities.
* Wazuh Agent and Sysmon should be installed before attack simulation begins.
* A port can be listening locally but still appear filtered from another zone.
* Windows network profiles can affect firewall exposure.
* WinRM with local administrator accounts may require LocalAccountTokenFilterPolicy.
* Weak credentials can quickly become a remote access path.
* SMB enumeration with valid credentials can reveal useful information.
* Not every tool output is valid evidence.
* Hardening should be validated using the same method that worked before.
* The strongest result is not only detecting the attack, but proving that the attack path was reduced afterwards.

---

## Final Technical Outcome

The completed lab followed this technical chain:

```text
Build -> Troubleshoot -> Baseline -> Attack -> Detect -> Harden -> Validate
```

Final result:

```text
The target was reachable and monitored.
The exposed services were identified.
WinRM access worked with weak credentials.
SMB enumeration worked with valid credentials.
Wazuh detected successful and failed authentication activity.
The weak account was removed.
WinRM was no longer openly reachable from the ATTACK zone.
The previous attack path no longer worked.
```

This completed the Purple Team workflow by connecting offensive testing, defensive detection and post-hardening validation.


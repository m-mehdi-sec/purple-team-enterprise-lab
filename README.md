# Purple Team Enterprise Lab

## Overview

This project documents a controlled Purple Team lab built in Hyper-V with OPNsense, Kali Linux, Windows Server, Wazuh, Sysmon, Nmap and Nessus.

The goal was to build an enterprise-style segmented lab where an attack machine in a dedicated ATTACK zone targets a Windows Server placed in a DMZ. The lab demonstrates the full workflow from network design and baseline scanning to attack simulation, detection, analysis, hardening and validation.

The lab followed this workflow:

```text
Network Design -> Baseline Scanning -> Attack Simulation -> Detection Review -> Hardening -> Validation
```

All testing was performed inside an isolated local lab environment against my own virtual machines.

---

## Lab Objective

The objective of this project was to understand how offensive activity can be detected and validated in a monitored environment.

The lab focused on:

```text
Segmentation -> Exposed Services -> Attack Simulation -> SIEM Detection -> Hardening -> Validation
```

The project was designed as a practical Purple Team exercise, combining both attack and defense perspectives.

---

## Lab Environment

| Component                 | Purpose                            |
| ------------------------- | ---------------------------------- |
| Hyper-V                   | Virtualization platform            |
| OPNsense                  | Firewall, routing and segmentation |
| Kali Linux                | Attack machine                     |
| Windows Server Target DMZ | Target server in DMZ               |
| Wazuh Server              | SIEM and log analysis              |
| Wazuh Agent               | Endpoint log collection            |
| Sysmon                    | Windows endpoint telemetry         |
| Nessus Essentials         | Vulnerability scanning             |
| Nmap                      | Port and service discovery         |

---

## Network Design

| Zone           | Network         | Purpose                             |
| -------------- | --------------- | ----------------------------------- |
| Management LAN | 192.168.10.0/24 | Management, Wazuh and Nessus access |
| ATTACK Zone    | 10.60.60.0/24   | Kali Linux attack machine           |
| DMZ Zone       | 10.70.70.0/24   | Windows Server target               |

Key systems:

| System                    | IP Address    |
| ------------------------- | ------------- |
| Kali Linux                | 10.60.60.50   |
| OPNsense ATTACK gateway   | 10.60.60.1    |
| Windows Server Target DMZ | 10.70.70.40   |
| OPNsense DMZ gateway      | 10.70.70.1    |
| Wazuh Server              | 192.168.10.30 |

---

## Attack Surface

The Windows Server target was intentionally configured with several exposed services for lab purposes.

| Port | Service | Purpose                      |
| ---: | ------- | ---------------------------- |
|   80 | IIS     | Web service exposure         |
|  135 | RPC     | Windows service discovery    |
|  445 | SMB     | File sharing and enumeration |
| 3389 | RDP     | Remote desktop exposure      |
| 5985 | WinRM   | Remote PowerShell access     |

A weak local administrator account was used during the attack simulation phase to demonstrate the risk of poor credential management.

---

## Summary of Results

| Phase             | Result                                                              |
| ----------------- | ------------------------------------------------------------------- |
| Network Design    | ATTACK and DMZ zones were created and routed through OPNsense       |
| Baseline          | Nmap and Nessus identified the exposed attack surface               |
| Attack Simulation | WinRM access and SMB enumeration were performed from Kali           |
| Detection         | Wazuh detected successful logons, failed logons and account lockout |
| Hardening         | The weak account was removed and WinRM access was restricted        |
| Validation        | Evil-WinRM failed and port 5985 was filtered after hardening        |

---

## Screenshots

### 1. OPNsense Interface Overview

![OPNsense interface overview](screenshots/01-opnsense-interface-overview.png)

OPNsense interface overview showing the segmented lab network with LAN, DMZ and ATTACK interfaces. The DMZ network uses `10.70.70.1/24` and the ATTACK network uses `10.60.60.1/24`.

---

### 2. Kali Attack IP Configuration

![Kali attack IP configuration](screenshots/02-kali-attack-ip-config.png)

Kali Linux was placed in the ATTACK zone with IP address `10.60.60.50/24` and default gateway `10.60.60.1`.

---

### 3. Windows Server Target DMZ IP Configuration

![Windows Server Target DMZ IP configuration](screenshots/03-win-target-dmz-ip-config.png)

Windows Server Target DMZ was configured with IP address `10.70.70.40/24`, gateway `10.70.70.1` and hostname `WIN-TARGET-DMZ`.

---

### 4. Wazuh Agent Active

![Wazuh agent active](screenshots/04-wazuh-agent-active.png)

Wazuh dashboard showing the `WIN-TARGET-DMZ` agent as active. This confirms that the target server was sending logs to Wazuh before the attack simulation started.

---

### 5. Nmap Baseline Open Ports

![Nmap baseline open ports](screenshots/05-nmap-baseline-open-ports.png)

Nmap service scan from Kali against `10.70.70.40`, showing exposed services including IIS, RPC, SMB, RDP and WinRM.

---

### 6. Nessus Baseline Summary

![Nessus baseline summary](screenshots/06-nessus-baseline-summary.png)

Nessus baseline scan against `10.70.70.40`. The scan was non-credentialed and showed no Critical or High findings, but identified Medium, Low and informational findings related to exposed services and TLS/SSL configuration.

---

### 7. Evil-WinRM Successful Login

![Evil-WinRM successful login](screenshots/07-evil-winrm-successful-login.png)

Evil-WinRM access from Kali to the DMZ target using the intentionally weak local administrator account. The commands `whoami` and `hostname` confirmed remote access as `win-target-dmz\purpleadmin`.

---

### 8. Wazuh Successful Remote Logon

![Wazuh successful remote logon](screenshots/08-wazuh-successful-remote-logon.png)

Wazuh detected the successful remote logon from the attack machine. The event was associated with the `WIN-TARGET-DMZ` agent and the `purpleadmin` account.

---

### 9. Wazuh Failed Logon

![Wazuh failed logon](screenshots/09-wazuh-failed-logon.png)

Wazuh event details showing a failed logon attempt for `purpleadmin`. The event includes authentication information and confirms that failed authentication attempts were logged by the target server.

---

### 10. SMB Authenticated Share Enumeration

![SMB authenticated share enumeration](screenshots/10-smb-authenticated-share-enumeration.png)

SMB enumeration from Kali using valid credentials. The target exposed administrative and user shares including `ADMIN$`, `C$`, `IPC$` and `Users`.

---

### 11. SMB User Profile Enumeration

![SMB user profile enumeration](screenshots/11-smb-user-profile-enumeration.png)

Authenticated SMB access to the `Users` share. The attacker was able to enumerate the `purpleadmin` user profile folders such as Desktop, Documents, Downloads and AppData.

---

### 12. Wazuh Brute Force Detection

![Wazuh brute force detection](screenshots/12-wazuh-bruteforce-detection.png)

Wazuh detected multiple failed logon attempts and account lockout activity. This shows how repeated authentication failures can be detected and escalated in the SIEM.

---

### 13. Purpleadmin Account Before Removal

![Purpleadmin account before removal](screenshots/13-purpleadmin-account-before-removal.png)

PowerShell verification showing the `purpleadmin` account before it was removed during the hardening phase. The account was a local administrator and was used during the attack simulation.

---

### 14. WinRM Access Denied After Hardening

![WinRM access denied](screenshots/14-winrm-access-denied.png)

After hardening, Evil-WinRM access using the previous `purpleadmin` credentials failed with an authorization error. This confirmed that the previous attack path was no longer valid.

---

### 15. WinRM Port Filtered After Hardening

![WinRM port filtered](screenshots/15-winrm-port-filtered.png)

Nmap validation from Kali showing port `5985/tcp` as filtered after hardening. This confirms that WinRM was no longer openly reachable from the ATTACK zone.

---

## Detection Summary

Wazuh successfully detected several important security events during the lab.

| Detection                       | Description                                      |
| ------------------------------- | ------------------------------------------------ |
| Successful Remote Logon         | Valid WinRM authentication                       |
| Failed Logon                    | Wrong password or invalid authentication attempt |
| Special Privileges Assigned     | Administrator-level logon activity               |
| Multiple Windows Logon Failures | Repeated failed authentication                   |
| User Account Locked Out         | Account lockout after multiple failed attempts   |

These detections confirmed that authentication activity from the DMZ target was visible in Wazuh.

---

## Hardening Summary

After the attack and detection phase, the lab was hardened to reduce the attack surface.

Hardening actions included:

* Removing the intentionally weak local administrator account
* Verifying that Evil-WinRM access no longer worked
* Restricting WinRM access from the ATTACK zone
* Validating the change with Nmap

The final validation showed that the previous WinRM-based attack path was no longer available.

---

## Key Takeaways

This lab demonstrated how offensive testing and defensive monitoring can be connected in a practical Purple Team workflow.

Key lessons from the project:

* Network segmentation helps separate attack, management and DMZ traffic.
* Baseline scanning is important before attack simulation.
* WinRM can create a serious remote access risk when weak credentials are used.
* SMB enumeration can expose useful information after authentication.
* Wazuh can detect successful logons, failed logons and account lockouts.
* Hardening should always be validated after changes.
* A complete Purple Team workflow should include attack, detection, analysis and improvement.

---

## Status

Project completed.

This lab demonstrates practical skills in:

* Network segmentation
* Firewall-based lab design
* Vulnerability scanning
* Windows attack surface analysis
* SIEM monitoring
* Endpoint logging
* Authentication detection
* SMB enumeration
* WinRM hardening
* Purple Team methodology

---

# Key Skills Demonstrated

* Enterprise network segmentation
* Firewall configuration with OPNsense
* DMZ network design
* Network reconnaissance
* Port scanning
* Service and version detection
* Vulnerability scanning
* Vulnerability assessment
* WinRM attack simulation
* SMB enumeration
* Remote authentication testing
* Wazuh SIEM monitoring
* Sysmon endpoint visibility
* Authentication monitoring
* Brute-force detection
* Account lockout detection
* Security event analysis
* Security hardening
* Hardening validation
* Purple Team methodology
* SOC and Blue Team workflow

---

# Documentation

Full lab documentation is available here:

**[Lab Documentation](LAB-DOCUMENTATION.md)**

---

# Author

Muhammad Mehdi

IT Security Developer Student


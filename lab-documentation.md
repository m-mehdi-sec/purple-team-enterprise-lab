# Lab Documentation

## 1. Lab Planning

- Why this lab was built
- Design decisions
- Purple Team workflow

---

## 2. Environment Preparation

- Hyper-V
- OPNsense
- ATTACK network
- DMZ network
- Windows Server
- Kali
- Wazuh
- Sysmon

---

## 3. OPNsense Configuration

- Interfaces
- Firewall Rules
- Routing
- Network Segmentation

---

## 4. Windows Server Configuration

- Static IP
- IIS
- SMB
- WinRM
- Local administrator account
- Wazuh Agent
- Sysmon

---

## 5. Baseline Assessment

### Nmap

- command
- findings

### Nessus

- scan policy
- non-credentialed scan
- findings

---

## 6. Attack Simulation

### Evil-WinRM

- command
- successful login
- validation

### SMB Enumeration

- smbclient commands
- share enumeration
- user profile enumeration

### Authentication Failures

- failed logons
- account lockout

---

## 7. Detection

### Wazuh

- Successful logon
- Failed logon
- Privilege assignment
- Account lockout

### Sysmon

- Process creation
- Event collection

---

## 8. Hardening

- Remove purpleadmin
- Disable WinRM attack path
- Verify firewall
- Validation

---

## 9. Validation

### Evil-WinRM

before

after

### Nmap

before

after

### Nessus comparison

baseline

after hardening

---

## 10. Lessons Learned

What worked well

What could be improved

Future improvements

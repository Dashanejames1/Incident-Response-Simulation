# Incident-Response-Simulation


**Author:** Dashane James  
**Lab Environment:** VMware Workstation | Kali Linux | Metasploitable 2  
**Purpose:** The goal of this repository is to walk through a full IR simulation using the NIST framework phases.
**Status:** 🔵 Completed

---

## 📋 Overview

For this Repository, I utilized a Kali Linux attacker VM, then used Medusa to Brute Force SSH on a Metasploitable 2 target, compromising the default msfadmin account in under a second. A Splunk Enterprise alert detected the activity, and the response is documented across seven steps: preparation, detection, analysis, containment, eradication, recovery, and lessons learned. Finding are mapped to MITRE ATT&CK.

---

## 🧪 Lab Environment

| Component | Details |
|---|---|
| Hypervisor | VMware Workstation (Host-Only Network) |
| Attacker Machine | Kali Linux 2026.1 — `192.168.79.129` |
| Target Machine | [Metasploitable 2] — `192.168.79.130` |
| Network Type | Host-Only (isolated, no internet exposure) |
| Host OS | Windows 11 — ASUS Vivobook 14 |

> ⚠️ **Note:** All activity was performed in a controlled, isolated lab environment against deliberately vulnerable machines. No unauthorized access to live networks was performed.

---

## 🛠️ Tools Used

- **SPLUNK Enterprise** — Log aggregation and alerting 
- **TCPdump** — Packet capture (on standby)  
- **iptables** — Host firewall for containment
- **Nmap** - Post-incident baseline 
- **Medusa**  - 
---

---

## 🔬 Tasks / Assessments Performed

### 1. Preparation: Document the tools, policies, and contact list

Before the incident, Splunk was configured with an SSH Brute Force Detection alert, tcpDump was available for packet capture, iptables served as the host firewall for containment, and Nmap for post-incedent baseline check. The lab environment consisted of a Kali Linux attacker VM and a Metasploitable 2 target VM on an isolated host-only network. The Splunk alert was the primary detection mechanism that would trigger this response.

**Policies**

- Default credentials must be changed before any system goes live.
- SSH authentication logs are forwarded to the SIEM and retained for review.
- Any confirmed unauthorized login is escalated to the IR Lead immediately.

**Contact list (simulated roles)**

| Role            | Responsibility                       | Contact         |
|-----------------|--------------------------------------|-----------------|
| IR Lead         | Declares the incident, owns response | Simulated / Phone |
| System Owner    | Approves containment and recovery    | Simulated / Phone  |
| SOC Analyst     | Triage, log analysis, documentation  | Dashane James / via Email |
| Management      | Escalation and communication         | Simulated / Phone  |

<img width="638" height="152" alt="Screenshot 2026-09-24 212919" src="https://github.com/user-attachments/assets/07b2ab54-dfcc-4f73-89c7-d39cf4d6481a" />

*SSH Brute Force Detection alert, enabled and scheduled hourly, triggering when results > 0. The modified timestamp reflects a later schedule edit; the alert was created before the test.*

### 2. Detection Phase

The SSH Brute Force Detection alert in Splunk fired after the target's SSH authentication logs showed repeated login attempts from the attacker against 192.168.79.130 on port 22. The attack started at 12:54:28 and succeeded in under one second — the msfadmin account was cracked at 12:54:29 using a targeted wordlist. This was caught by the automated Splunk alert rather than manual discovery, proving the value of having pre-configured SIEM detection in place.

**Limitation:** the alert runs on an hourly schedule, so detection can lag an attack by up to 59 minutes. A production SOC would run this every few minutes or in real time.


### 3. Analysis: determine scope, accounts targeted, timeframe, and any succesful logins.

Log analysis revealed the following: the targeted account was msfadmin on 192.168.79.130, the attack came from 192.168.79.129, and it succeeded because default credentials were never changed. The attack duration was under one second (12:54:28 to 12:54:29) — the attacker clearly knew what credentials to try, which is consistent with prior reconnaissance. In a real incident the next steps would be checking for lateral movement and data exfiltration.


### 4. Containment: Write the firewall rule to block the attacker IP

The immediate action was blocking the attacker's IP at the firewall to cut off further access:

sudo iptables -A INPUT -s 192.168.79.129 -j DROP
sudo iptables -A OUTPUT -d 192.168.79.129 -j DROP

The msfadmin account was also flagged for an immediate password reset and SSH access was restricted to trusted IPs only. Blocking the attacker IP is *Containment* — it stops the attack but doesn't clean up the damage. That comes next.


### 5. Eradication: Listing what needs to be removed (New accounts, SSH keys, malware)

With the attacker blocked, the cleanup phase focused on removing everything they touched: the default msfadmin password was changed immediately, all active SSH sessions from the attacker IP were terminated, the authorized keys file was audited for any backdoors, and all system accounts and scheduled tasks were reviewed for unauthorized changes. The root cause was the default credentials left in place and were addressed by establishing a policy requiring credential changes on all systems before they go live.


### 6. Recovery: Describe recovery and verification process.

During the recovery process: if any integrity check had failed, the target would be rebuilt from a known good vm snapshot rather than cleaned in place.

During the verification process: I confirmed the attacker IP was fully blocked with no active sessions remaining, verified the new msfadmin password was working correctly, reviewed system logs for any unauthorized changes made during the compromise window, re-enabled normal SSH access from trusted IPs only, and ran a fresh Nmap scan to confirm the attack surface matched the pre-incident baseline. The system was declared recovered after all checks passed and Splunk monitoring was confirmed active.

### 7. Lessons Learned

Three things would have prevented this incident entirely:

1. No default credentials — msfadmin:msfadmin should never have been left in place. A simple policy requiring credential changes before any system goes live would have stopped this attack before it started.

2. SSH account lockout — there was no limit on login attempts, which let Medusa try passwords without any resistance. Locking an account after 5 failed attempts in 60 seconds would have blocked the attack even if the attacker had the right password in their wordlist.

3. Key-only SSH authentication — Disabling password logins removes password guessing as an attack path entirely.



**Findings:** 

The table above consolidates all of the findings reported during the Nmap --script vuln scan and categorizes each result by severity. The most significant finding was the vsFTPd 2.3.4 backdoor (CVE-2011-2523) on port 21, which was successfully exploited to obtain root-level access, making it the only Critical vulnerability confirmed through live exploitation. Several additional weaknesses were identified, including SQL injection, Telnet running without encryption, Slowloris denial-of-service susceptibility, and weak SSH cryptographic settings, all of which increase the attack surface of the target system. The SMB vulnerability checks also demonstrated that vulnerability scans can produce different outcomes: some scripts confirmed the system was not vulnerable, while others returned inconclusive results because the checks could not be completed. Overall, the scan illustrates the importance of reviewing every NSE script result individually, as findings may represent confirmed vulnerabilities, informational issues, successful mitigations, or inconclusive tests requiring additional investigation.

---

## 📊 Key Findings Summary

| Port/Service | Tool Used | Risk Level | Notes |



**Risk Levels:** 🔴 Critical | 🟠 High | 🟡 Medium | 🟢 Low


---

## 🗺️ MITRE ATT&CK Mapping

| Action Performed | ATT&CK Tactic | Technique ID | Technique Name |
|---|---|---|---|

**MITRE ATT&CK mapping**

| Technique                          | ID        |
|------------------------------------|-----------|
| Brute Force: Password Guessing     | T1110.001 |
| Valid Accounts: Local Accounts     | T1078.003 |
| Remote Services: SSH               | T1021.004 |


---

## 🛡️ Defensive Recommendations/Policies

Based on findings, the following remediations would be recommended in a real environment:


- Default credentials must be changed before any system goes live.
- SSH authentication logs are forwarded to the SIEM and retained for review.
- Any confirmed unauthorized login is escalated to the IR Lead immediately.

---

---

## 🔑 Technical Notes




# Any important commands or workarounds
[-sC]
[-Pn]


```

---

## 📌 About This Project

[1-2 sentences about how this fits into your overall portfolio and career goals.]




**Related repositories:**


---

## 👤 Author

**Dashane James**  
Senior Field Service Technician → Cybersecurity Analyst  
📍 Yonkers, NY  
🎓 B.S. Information Technology — SUNY Canton  
🏆 CompTIA Security+ | CySA+ (In Progress)  
🔗 [GitHub](https://github.com/Dashanejames1) | [Zero Trust Cyber Security Brand](https://www.instagram.com/zerotrust_cybersecurity)

---

*This repository is part of an active portfolio demonstrating hands-on cybersecurity skills. All lab work performed in isolated environments for educational purposes.*

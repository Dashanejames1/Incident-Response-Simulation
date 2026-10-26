# Incident-Response-Simulation


**Author:** Dashane James
**Lab Environment:** VMware Workstation | Kali Linux | Metasploitable 2
**Purpose:** Walk through a full incident response simulation using the NIST SP 800-61 framework phases.
**Status:** 🔵 Completed

---

## 📋 Overview

This repository documents a complete incident response simulation driven by a real attack. Using Kali Linux as the attacker machine, I ran Medusa to brute force SSH on a Metasploitable 2 target, compromising the default msfadmin account in under a second. A Splunk Enterprise alert detected the activity, and the response is documented across seven phases: preparation, detection, analysis, containment, eradication, recovery, and lessons learned. Findings are mapped to MITRE ATT&CK.

---

## 🧪 Lab Environment

| Component | Details |
|---|---|
| Hypervisor | VMware Workstation (Host-Only Network) |
| Attacker Machine | Kali Linux 2026.1, `192.168.79.129` |
| Target Machine | Metasploitable 2, `192.168.79.130` |
| Network Type | Host-Only (isolated, no internet exposure) |
| Host OS | Windows 11, ASUS Vivobook 14 |

> ⚠️ **Note:** All activity was performed in a controlled, isolated lab environment against deliberately vulnerable machines. No unauthorized access to live networks was performed.

---

## 🛠️ Tools Used

- **Splunk Enterprise (Free):** Log aggregation and alerting platform. The pre-configured SSH Brute Force Detection alert served as the primary detection mechanism for this incident.
- **TCPDump:** Packet capture tool used to record all network traffic during the attack. Capture file was imported into Splunk for forensic analysis.
- **iptables:** Linux host firewall used during the containment phase to block the attacker IP.
- **Nmap:** Used during the recovery phase to run a post-incident baseline scan and confirm the attack surface matched pre-incident conditions.
- **Medusa:** SSH brute force tool used to generate the attack traffic. Chosen over Hydra because Medusa supports legacy SSH algorithms required by Metasploitable's older SSH server.

---

## 🔬 Tasks / Assessments Performed

### 1. Preparation: Document the tools, policies, and contact list

Before the incident, Splunk was configured with an SSH Brute Force Detection alert, TCPDump was available for packet capture, iptables served as the host firewall for containment, and Nmap was on hand for post-incident baseline checks. The lab environment consisted of a Kali Linux attacker VM and a Metasploitable 2 target VM on an isolated host-only network. The Splunk alert was the primary detection mechanism that would trigger this response.

**Policies**

- Default credentials must be changed before any system goes live.
- SSH authentication logs are forwarded to the SIEM and retained for review.
- Any confirmed unauthorized login is escalated to the IR Lead immediately.

**Contact list (simulated roles)**

| Role | Responsibility | Contact |
|---|---|---|
| IR Lead | Declares the incident, owns response | Simulated / Phone |
| System Owner | Approves containment and recovery | Simulated / Phone |
| SOC Analyst | Triage, log analysis, documentation | Dashane James / via Email |
| Management | Escalation and communication | Simulated / Phone |

<img width="638" height="152" alt="Screenshot 2026-09-24 212919" src="https://github.com/user-attachments/assets/07b2ab54-dfcc-4f73-89c7-d39cf4d6481a" />

*SSH Brute Force Detection alert, enabled and scheduled hourly, triggering when results > 0. The modified timestamp reflects a later schedule edit; the alert was created before the test.*

### 2. Detection Phase

The SSH Brute Force Detection alert in Splunk fired after the target's SSH authentication logs showed repeated login attempts from the attacker against 192.168.79.130 on port 22. The attack started at 12:54:28 and succeeded in under one second, with the msfadmin account cracked at 12:54:29 using a targeted wordlist. This was caught by the automated Splunk alert rather than manual discovery, proving the value of having pre-configured SIEM detection in place before an attack occurs.

**Limitation:** The alert runs on an hourly schedule, so detection can lag an attack by up to 59 minutes. A production SOC would run this every few minutes or in real time.


### 3. Analysis: Determine scope, accounts targeted, timeframe, and any successful logins

Log analysis revealed the following: the targeted account was msfadmin on 192.168.79.130, the attack came from 192.168.79.129, and it succeeded because default credentials were never changed. The attack duration was under one second (12:54:28 to 12:54:29). The attacker clearly knew what credentials to try, which is consistent with prior reconnaissance. In a real incident the next steps would be checking for lateral movement and data exfiltration.


### 4. Containment: Write the firewall rule to block the attacker IP

The immediate action was blocking the attacker's IP at the firewall to cut off further access:

```bash
sudo iptables -A INPUT -s 192.168.79.129 -j DROP
sudo iptables -A OUTPUT -d 192.168.79.129 -j DROP
```

The msfadmin account was also flagged for an immediate password reset and SSH access was restricted to trusted IPs only. Blocking the attacker IP is Containment: it stops the attack but does not clean up the damage. That comes next.


### 5. Eradication: Listing what needs to be removed (new accounts, SSH keys, malware)

With the attacker blocked, the cleanup phase focused on removing everything they touched: the default msfadmin password was changed immediately, all active SSH sessions from the attacker IP were terminated, the authorized keys file was audited for any backdoors, and all system accounts and scheduled tasks were reviewed for unauthorized changes. The root cause, default credentials left in place, was addressed by establishing a policy requiring credential changes on all systems before they go live.


### 6. Recovery: Describe recovery and verification process

During recovery: if any integrity check had failed, the target would be rebuilt from a known good VM snapshot rather than cleaned in place.

During verification: I confirmed the attacker IP was fully blocked with no active sessions remaining, verified the new msfadmin password was working correctly, reviewed system logs for any unauthorized changes made during the compromise window, re-enabled normal SSH access from trusted IPs only, and ran a fresh Nmap scan to confirm the attack surface matched the pre-incident baseline. The system was declared recovered after all checks passed and Splunk monitoring was confirmed active.

### 7. Lessons Learned

Three controls would have prevented this incident entirely:

1. **No default credentials:** msfadmin:msfadmin should never have been left in place. A policy requiring credential changes before any system goes live would have stopped this attack before it started.

2. **SSH account lockout:** There was no limit on login attempts, which let Medusa try passwords without any resistance. Locking an account after 5 failed attempts in 60 seconds would have blocked the attack even if the attacker had the correct password in their wordlist.

3. **Key-only SSH authentication:** Disabling password logins removes password guessing as an attack path entirely. With key-only auth, a brute force tool has nothing to test against.

---

## 📊 Key Findings Summary

| Finding | Risk Level | Notes |
|---|---|---|
| Default credentials (msfadmin:msfadmin) in use | 🔴 Critical | Account cracked in under 1 second on first attempt using a 6-word targeted wordlist |
| No SSH account lockout policy | 🔴 Critical | Unlimited authentication attempts allowed with no rate limiting or temporary block |
| SSH brute force detected via Splunk alert | 🟠 High | Alert fired correctly but hourly scheduling means up to 59-minute detection lag |
| Attacker IP blocked via iptables | Remediated | `INPUT` and `OUTPUT` rules applied; all active sessions terminated |
| SSH authorized_keys audited | 🟢 Clean | No unauthorized keys found in the compromise window |

**Risk Levels:** 🔴 Critical | 🟠 High | 🟡 Medium | 🟢 Low

---

## 🗺️ MITRE ATT&CK Mapping

| Action Performed | ATT&CK Tactic | Technique ID | Technique Name |
|---|---|---|---|
| Medusa SSH brute force (targeted wordlist) | Credential Access | T1110.001 | Brute Force: Password Guessing |
| msfadmin:msfadmin accepted, SSH access gained | Initial Access | T1078.003 | Valid Accounts: Local Accounts |
| SSH session established from attacker IP | Lateral Movement | T1021.004 | Remote Services: SSH |

---

## 🛡️ Defensive Recommendations

Based on the findings from this incident, the following controls are recommended for any production environment running SSH-accessible systems:

- **Default credentials must be changed before any system goes live.** Automated scanning tools should verify no default credentials exist on any deployed system as part of the onboarding checklist.
- **SSH authentication logs must be forwarded to the SIEM and retained for review.** Logs that live only on the host are lost if the host is compromised or rebuilt.
- **Any confirmed unauthorized login must be escalated to the IR Lead immediately.** A detection that sits in a queue is the same as no detection.
- **Implement fail2ban or equivalent SSH lockout:** Configure SSH to lock accounts after 5 failed login attempts within 60 seconds. This stops brute force attacks before they succeed regardless of what is in the attacker's wordlist.
- **Upgrade to real-time SIEM alerting:** A brute force attack that succeeds in under a second cannot be adequately monitored with hourly alert scheduling.

---

## 🔑 Technical Notes

```bash
# Containment: block attacker IP at the firewall
sudo iptables -A INPUT -s 192.168.79.129 -j DROP
sudo iptables -A OUTPUT -d 192.168.79.129 -j DROP

# Verify the rule is applied
sudo iptables -L -n

# Post-incident baseline scan
sudo nmap -sT -Pn --disable-arp-ping -n 192.168.79.130

# Review active SSH sessions
who
last
w

# Audit SSH authorized keys for backdoors
cat ~/.ssh/authorized_keys
cat /root/.ssh/authorized_keys
```

**Splunk detection queries used during analysis:**

```
# Confirm brute force activity in Medusa log
index=main source="medusa_results.txt" "ACCOUNT FOUND"

# Port scan pattern from TCPDump capture
index=main source="lab_capture.txt" | rex field=_raw "(?<src_ip>\d+\.\d+\.\d+\.\d+)" | stats count by src_ip | where count > 100
```

---

## 📌 About This Project

This repository is the practical capstone of the SIEM, log analysis, and incident response phase. Every previous lab in this phase built toward this one: Splunk was installed and configured, SPL detection queries were written, real attack traffic was generated and detected, a second SIEM platform was set up, and a network IDS was deployed. This simulation ties it all together by walking through what actually happens when that detection work finds something.

The attack scenario, SSH brute force leading to a credential compromise, is one of the most common incidents in production SOC environments. Walking through the full NIST SP 800-61 lifecycle with a real attack rather than a hypothetical scenario demonstrates both the tools and the process that security analysts use to move from an alert to a contained, remediated, and documented incident.

**Related repositories (SIEM, log analysis, and incident response phase):**
- `Splunk-Free-Installation-and-Ingest-First-Logs`: Installed the Splunk SIEM that detected this incident
- `Splunk-Search-Language`: SPL syntax used to build the detection queries applied during the analysis phase
- `Generating-Attacks-and-Defending-in-Splunk`: Executed the SSH brute force and port scan that serve as the attack scenario in this simulation
- `Install-and-Understand-ELK-Stack`: Installed the ELK Stack as a second SIEM platform to complement the Splunk detection workflow
- `Install-Snort-IDS-and-Write-Detection-Rules`: Network-layer detection that would catch the reconnaissance preceding this attack
- `Incident-Response-Report`: Formal written incident report covering this same event with full timeline, impact assessment, and control recommendations

---

## 👤 Author

**Dashane James**
Senior Field Service Technician → Cybersecurity Analyst
📍 Yonkers, NY
🎓 B.S. Information Technology, SUNY Canton
🏆 CompTIA Security+
🔗 [GitHub](https://github.com/Dashanejames1) | [Zero Trust Cyber Security Brand](https://www.instagram.com/zerotrust_cybersecurity)

---

*This repository is part of an active portfolio demonstrating hands-on cybersecurity skills. All lab work performed in isolated environments for educational purposes.*

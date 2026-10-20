# Summit Federal Bank — Cybersecurity Capstone

**Build · Harden · Monitor · Respond**

A four-week capstone in which a two-person team built a simulated bank's environment in Azure, hardened it to PCI DSS v4.0.1, monitored it, and responded to a live, instructor-injected attack. The project ended with a presentation to bank leadership.

> All data in this project is synthetic. No real customer, cardholder or employer data was used. Credentials, hashes of secrets and environment-specific identifiers have been removed.

---

## Team

| Name | Role |
| --- | --- |
| **Carter Yanosick** | Incident Commander · Windows Server, Active Directory, file services, compliance (PCI DSS) |

---

## Contents

- [Environment](#environment)
- [Hardening](#hardening)
- [Incident: October 8](#incident-october-8)
- [Lessons learned](#lessons-learned)
- [PCI DSS mapping](#pci-dss-mapping)
- [Course takeaways](#course-takeaways)
- [Repository structure](#repository-structure)
- [Skills demonstrated](#skills-demonstrated)

---

## Environment

| Component | Details |
| --- | --- |
| **Domain controller** | Windows Server 2022 Datacenter: Active Directory (`summitfed.local`), DNS, internal Certificate Authority, customer records file share |
| **Web host** | Ubuntu Linux running Apache and a deliberately vulnerable web app (DVWA) over TLS |
| **Network** | Isolated Azure subnet behind a Network Security Group; no public IPs; admin access only through Azure Bastion |
| **Constraint** | Team accounts had **Reader-only** access in Azure, so no VM snapshots, Run Command or NIC changes. Every rollback had to be built inside the guest OS. |

**Defense in depth, three layers:**
1. **Azure NSG** at the subnet boundary: Bastion in, own subnet allowed, all other teams denied
2. **Windows Firewall** on the DC: scoped rules per port and source
3. **ufw** on the web host: SSH from Bastion only, web traffic from the bank subnet

---

## Hardening

### Domain controller

| Control | Before | After | PCI DSS |
| --- | --- | --- | --- |
| Inbound firewall allow rules | 120, most from any address | **23**, each scoped to a source and purpose | 1.2, 1.3 |
| Account lockout | Never (threshold 0) | **6 attempts, 30-minute lockout** | 8.3.4 |
| Minimum password length | 7 | **14**, complexity on, history 24 | 8.3.6, 8.3.7 |
| Audit policy | Mostly "No Auditing" | **19 advanced audit subcategories**, command-line and PowerShell Script Block logging, 1 GB Security log | 10.2 |
| Legacy protocols | Print Spooler, SMBv1, LLMNR, NetBIOS, NTLMv1 available | **Disabled or removed**; NTLMv2 only | 2.2.4, 8.3.2 |
| Admin accounts | One shared, instructor-issued admin | **Named admin accounts** per person; shared account kept as break-glass | 8.2.1 |
| Customer records share | Authenticated Users had **Modify** | **Role-based access** (AGDLP); IT has no data access; SMB encryption; access-based enumeration | 7.2, 4.2.1 |
| File auditing | None | **SACL** on the share: every read, write, delete and permission change logged | 10.2.1.1 |
| Backup | None | **Twice-daily shadow copies**; ransomware simulation restored and **verified 101/101 files by SHA-256** | 12.10.1 |

**Verified, not assumed:** an nmap scan of all 65,535 ports from the web host found only **4 reachable ports** on the DC. A follow-up check a week later caught **configuration drift** (17 firewall rules re-enabled and an unplanned IIS install), which was attributed through audit logs and remediated.

### Monitoring
Splunk forwarding was attempted (forwarder installed and configured) but the lab's indexer receiving port could not be opened, and SIEM was removed from scope. In its place:

- **`SFB-DailyCheck.ps1`** — a PowerShell detection script that checks the DC for failed-logon bursts, lockouts, break-glass use, account and admin-group changes, new services and tasks, log clearing, mass or ransomware-pattern file activity on the share, shadow copy deletion, firewall drift, new listening ports, Defender status and share integrity. Each finding is labeled ALERT, REVIEW or OK and saved as evidence.
- **Baseline comparison** of services, admin groups, scheduled tasks and firewall rules against the hardened state.

---

## Incident: October 8

An instructor-injected attack against the domain controller, run as a live exercise.

| Time | Attacker action | Team response |
| --- | --- | --- |
| 12:41 | Break-glass admin account logs on | **Detected at 12:42 (40 seconds)** by `SFB-DailyCheck.ps1`; SEV1 declared |
| 12:56 | Test malware (EICAR) downloaded | **Blocked and removed** by Microsoft Defender |
| 13:03 | Hidden admin account `svchelper` created, added to Administrators | Disabled, stripped of rights, removed |
| ~13:10–13:26 | Scheduled task `WindowsUpdateHelper` planted for persistence | Found by **baseline comparison** after its log record was erased |
| 13:26 | Security audit log cleared | Timeline already preserved **3 minutes earlier** in monitoring reports and evidence exports |

**Delivery:** every attacker action ran as **SYSTEM** through cloud management-plane access, so it never crossed the network and network controls never saw it.

**Recovery:** backdoor account and task removed; break-glass account reset and disabled; **every domain credential reset, including krbtgt**; integrity check confirmed **0 of 101 customer records modified**.

---

## Lessons learned

**Root cause:** compromised privileged access (cloud administrative access and the break-glass password) allowed the attacker to act as SYSTEM, which no control inside the server can block.

**What worked**
- Detection in 40 seconds from the team's own monitoring
- Named admin accounts plus process and PowerShell logging traced every action to an account and a minute
- Evidence preserved before the attacker wiped the log
- Baselines found what the logs could no longer show
- A tested backup meant recovery was never in doubt

**What we would change**
1. Least privilege and MFA on cloud administrative access, with alerts on remote command execution
2. Break-glass credentials in a sealed vault, rotated after any use, alerted on every logon
3. Real-time central logging so a wiped log loses nothing
4. All containment decisions routed through the Incident Commander
5. State-based checks (tasks, admin groups) on every monitoring run

About two minutes of file-access audit data was lost in the log wipe and is disclosed as a gap.

---

## PCI DSS mapping

| Requirement | Result |
| --- | --- |
| 3.5.1 Stored PAN unreadable | **Held** — card numbers truncated (first 6 / last 4) |
| 5.2, 5.3 Anti-malware | **Held** — EICAR blocked twice |
| 7.2 Least privilege | **Gap at the cloud layer** — SYSTEM-level execution bypassed in-guest controls |
| 8.2, 8.6 Account management, break-glass | **Gap** — break-glass credential known outside the team; reset and disabled |
| 10.2 Audit logging | **Held** — full attack chain captured |
| 10.3 Log protection, central copy | **Gap** — no central log server; compensated by scheduled checks and exports |
| 10.4.1 Log review | **Held** — detection within 1 minute |
| 11.5.2 Change detection | **Held** — baseline found hidden persistence |
| 12.10.1 Incident response | **Held** — playbooks executed, evidence preserved, recovery verified |




## Skills demonstrated

Windows Server and Active Directory hardening · Group Policy · Windows Firewall · Linux hardening · PKI and TLS · Azure networking (NSG, Bastion) · audit policy design · PowerShell detection scripting · incident response and command · digital forensics and evidence handling · backup and recovery testing · PCI DSS v4.0.1 control mapping · executive communication

---

*Capstone project, BC43, 2026. Built in an instructor-provided lab environment for training purposes.*

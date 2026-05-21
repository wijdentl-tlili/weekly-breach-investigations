# Storm-2949: Microsoft Entra ID Abused for Cloud-Wide Data Exfiltration — 2026-05-18

> **Weekly Breach Investigation**
> *Week 03 · Identity Compromise · Social Engineering · MFA Abuse · Cloud Lateral Movement · Financially Motivated*

---

## Executive Summary

A threat actor tracked by Microsoft as **Storm-2949** executed a methodical, multi-stage attack campaign against an unnamed organisation's Microsoft cloud environment, exploiting legitimate administrative features rather than malware, beginning with targeted social engineering against IT staff and senior leadership to hijack Microsoft Entra ID accounts via SSPR abuse.

Once inside, the attackers expanded from compromised identities into the full cloud stack: bulk-downloading files from OneDrive and SharePoint, pivoting into Azure Key Vault to extract dozens of secrets, modifying SQL firewall rules, exfiltrating storage account data, and deploying ScreenConnect on virtual machines for persistent remote control.

The campaign exfiltrated sensitive data across SaaS, PaaS, and IaaS layers ( including VPN configurations, database credentials, application secrets, and connection strings ) and was ultimately detected not by the victim organisation's own controls but by Microsoft Defender correlating signals across identity, cloud, and endpoint environments.

---

## Attack Timeline

| Date / Time (UTC) | Event |
|---|---|
| **Pre-attack** | Storm-2949 conducts reconnaissance on target organisation, selecting IT personnel and senior leadership as primary victims, indicating prior intelligence gathering. |
| **Phase 1 — Identity** | Attackers initiate Microsoft SSPR process targeting users while impersonating internal IT support, instructing victims to approve MFA prompts under the pretext of "routine account verification". Victims approve fraudulent MFA prompts, after which attackers reset passwords, remove existing authentication methods (phone, email, Microsoft Authenticator), and register their own devices as new MFA authenticators, locking out legitimate users and establishing persistent access. This process is repeated across multiple privileged accounts within the tenant. |
| **Phase 2 — Microsoft 365** | Using compromised accounts, attackers perform Microsoft Graph API queries to enumerate users and privileged identities across the Entra ID tenant. They then exfiltrate large volumes of data from OneDrive and SharePoint via the web interface, focusing on VPN configuration files and remote access documentation. |
| **Phase 3 — Azure & Infrastructure** | Attackers pivot into Azure using RBAC permissions inherited from compromised identities. They attempt App Service compromise via publishing profiles and Kudu interface, then shift to Azure Key Vault where they modify access policies and extract sensitive secrets (credentials and connection strings). These secrets are used to access production applications and modify credentials for persistence. Azure IMDS is abused for additional secret retrieval. |
| **Phase 3 — Continued (Infrastructure Abuse)** | Attackers modify SQL firewall rules to enable database access, exfiltrate data, then restore rules for anti-forensic purposes. Storage accounts are made publicly accessible, allowing retrieval of keys and SAS tokens. Custom Python scripts using Azure SDK are used for bulk data exfiltration. |
| **Phase 3 — Virtual Machines & Persistence** | VMAccess extension is used to create new administrator accounts on Azure VMs. The Run Command feature is abused for remote execution. ScreenConnect is deployed for persistent C2 access, and attempts are made to disable Microsoft Defender and clear Windows event logs. |
| **May 18, 2026** | Microsoft publishes a full incident report. Detection is attributed to Microsoft Defender XDR correlating signals across identity, cloud, and endpoint telemetry. |
> **Dwell time: Unknown** — Microsoft has not disclosed the compromise start date; the full timeline within the victim organisation has not been publicly confirmed. The multi-stage nature and post-compromise cleanup suggest an extended dwell.

---

## How the Attack Worked

Storm-2949 operated entirely without traditional malware for the initial access and cloud exfiltration phases. Every action used a legitimate Microsoft administrative feature — turned against the organisation that trusted it.

```
ATTACK SURFACE MAP
══════════════════════════════════════════════════════════

[PHASE 1 — IDENTITY COMPROMISE]

  Attacker
     │
     ├─ Initiates SSPR process for target user
     ├─ Impersonates IT support (phone/message)
     ├─ Target approves MFA prompt ──→ Attacker resets password
     │                                  Removes existing MFA methods
     │                                  Enrolls own device as authenticator
     │                                  Legitimate user LOCKED OUT
     └─ Repeated across multiple privileged accounts

[PHASE 2 — MICROSOFT 365]

  Compromised Entra ID Account
     │
     ├── Microsoft Graph API ──→ Enumerate users + privileged identities
     ├── OneDrive (web interface) ──→ ⚠️ Bulk download (thousands of files)
     │       └─ VPN configs, remote access procedures
     └── SharePoint ──→ ⚠️ Bulk file exfiltration

[PHASE 3 — AZURE INFRASTRUCTURE]

  Compromised Entra ID (RBAC inherited)
     │
     ├── Azure App Services ──→ ❌ Initial access failed
     │       └─ Pivoted to...
     ├── Azure Key Vault (Owner-level) ──→ ⚠️ Dozens of secrets extracted
     │       └─ Credentials, connection strings → production app access
     ├── Azure IMDS ──→ ⚠️ Token theft → Key Vault auth
     ├── Azure SQL ──→ ⚠️ Firewall rules modified → data exfiltrated
     │       └─ Firewall rules RESTORED (anti-forensic)
     ├── Azure Storage ──→ ⚠️ Public access enabled
     │       └─ Storage keys + SAS tokens retrieved
     │       └─ Python + Azure SDK bulk download
     └── Azure VMs ──→ ⚠️ VMAccess extension abused
             ├─ New admin accounts created
             ├─ Run Command → remote script execution
             ├─ ScreenConnect deployed (persistent C2)
             ├─ Attempted Defender disable
             └─ Windows event log clearing

WHAT WAS NOT AFFECTED (as reported)
══════════════════════════════════════
  ✅ Traditional endpoints (no malware deployed)
  ✅ On-premises infrastructure (cloud-only attack surface)
  ✅ Accounts NOT targeted by the social engineering campaign
```

**Key attacker behaviour:** After modifying SQL firewall rules to exfiltrate data, the attackers restored the original rules — a deliberate anti-forensic step to reduce detection probability and make the intrusion harder to reconstruct post-incident.

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Initial Access | Valid Accounts: Cloud Accounts (SSPR/MFA abuse) | [T1078.004](https://attack.mitre.org/techniques/T1078/004/) |
| Reconnaissance | Phishing: Spearphishing Voice (IT impersonation) | [T1598.004](https://attack.mitre.org/techniques/T1598/004/) |
| Execution | User Execution: Malicious Link (MFA prompt approval) | [T1204.001](https://attack.mitre.org/techniques/T1204/001/) |
| Persistence | Account Manipulation: Additional Cloud Credentials | [T1098.001](https://attack.mitre.org/techniques/T1098/001/) |
| Persistence | Account Manipulation: Device Registration (own MFA device) | [T1098.005](https://attack.mitre.org/techniques/T1098/005/) |
| Command and Control | Remote Access Tools (ScreenConnect) | [T1219](https://attack.mitre.org/techniques/T1219/) |
| Privilege Escalation | Valid Accounts: Cloud Accounts (RBAC inheritance) | [T1078.004](https://attack.mitre.org/techniques/T1078/004/) |
| Defense Impairment | Impair Defenses: Disable or Modify Tools (Defender) | [T1562.001](https://attack.mitre.org/techniques/T1562/001/) |
| Defense Impairment | Indicator Removal: Clear Windows Event Logs | [T1070.001](https://attack.mitre.org/techniques/T1070/001/) |
| Defense Impairment | Indicator Removal: Modify Cloud Compute Configurations (firewall rule restore) | [T1578](https://attack.mitre.org/techniques/T1578/) |
| Credential Access | Steal Web Session Cookie / Token (IMDS token theft) | [T1539](https://attack.mitre.org/techniques/T1539/) |
| Credential Access | Unsecured Credentials: Cloud Instance Metadata API | [T1552.005](https://attack.mitre.org/techniques/T1552/005/) |
| Discovery | Cloud Service Discovery (Graph API tenant enumeration) | [T1526](https://attack.mitre.org/techniques/T1526/) |
| Discovery | Permission Groups Discovery: Cloud Groups | [T1069.003](https://attack.mitre.org/techniques/T1069/003/) |
| Lateral Movement | Use Alternate Authentication Material (Key Vault secrets → app pivots) | [T1550](https://attack.mitre.org/techniques/T1550/) |
| Collection | Data from Cloud Storage (OneDrive, SharePoint, Storage accounts) | [T1530](https://attack.mitre.org/techniques/T1530/) |
| Collection | Data from Information Repositories (Key Vault secrets, SQL) | [T1213](https://attack.mitre.org/techniques/T1213/) |
| Command & Control | Application Layer Protocol (ScreenConnect C2) | [T1071](https://attack.mitre.org/techniques/T1071/) |
| Exfiltration | Transfer Data to Cloud Account (Azure SDK bulk download) | [T1537](https://attack.mitre.org/techniques/T1537/) |

---

## Detection Opportunities

### Log Sources

- **Microsoft Entra ID Sign-In Logs** — SSPR initiation events followed immediately by authentication method removal; MFA registration from an unrecognised device; impossible travel or new ASN login
- **Microsoft Entra ID Audit Logs** — Authentication method deletion events; new MFA device registration; password reset events not initiated by the account owner
- **Microsoft Graph API Activity Logs** — Bulk directory enumeration queries; Graph calls originating from a non-standard client application
- **OneDrive / SharePoint Unified Audit Logs** — Anomalous volume of file download events from a single account in a short window
- **Azure Activity Log** — Key Vault access policy modifications; SQL server firewall rule changes (especially short-lived changes that are quickly reverted); public access enabled on storage accounts; SAS token generation
- **Azure Monitor / VM Logs** — VMAccess extension invocations; Run Command executions; new local administrator account creation on VMs
- **Microsoft Defender for Endpoint** — ScreenConnect installation; Defender tamper protection events; Windows event log clearing (Event ID 1102 / 104)


### IOCs

| Type | Value | Description |
|---|---|---|
| IP Address | `176.123.4[.]44` | Attacker egress IP |
| IP Address | `91.208.197[.]87` | Attacker egress IP |
| IP Address | `185.241.208[.]243` | ScreenConnect C2 instance used by attacker |
| Tool | ScreenConnect (ConnectWise) | Remote monitoring tool — legitimate tool abused for persistence |
| Script | Custom Python + Azure SDK | Used for bulk Azure Storage data exfiltration |
| API | Microsoft Graph API | Used for tenant directory enumeration |
| Feature | Azure IMDS | Abused for token theft and Key Vault authentication |
| Threat actor | Storm-2949 | Microsoft tracking designation |

> All IP addresses defanged with `[.]` notation. Re-fang only within controlled TI platforms (MISP, VirusTotal, SIEM).

---

## Analyst Notes

**What I learned**
In this attack every single tool used (SSPR, MFA registration, Graph API, OneDrive, Key Vault, IMDS, VMAccess, Run Command, SAS tokens) is a legitimate Microsoft feature. There is no malware to signature-scan, no exploit code to reverse engineer. The entire attack surface is identity and configuration, not software. Detection requires behavioural correlation across multiple log sources — none of the individual events are inherently suspicious in isolation.

**What surprised me**
SQL firewall cleaning up is an advanced security operation technique which the vast majority of low-level threat actors tend not to employ. In particular, the actors manipulated rule tables, exfiltrated the data, and returned the rules to their original state – a tactic aimed at making it impossible for any forensic investigation to establish exactly what had been compromised. Such sophisticated anti-forensic activity is typical of nation-states’ operations, although the current classification of Storm-2949 is financially motivated.

---

## Who Is at Risk — Quick Check

| Condition | At risk? |
|---|---|
| Your organisation uses Microsoft Entra ID for cloud identity | ⚠️ Review — this is your attack surface |
| IT staff or senior leadership were recently asked to approve unexpected MFA prompts | ✅ Yes — treat as active compromise, investigate immediately |
| You have SSPR enabled without additional identity verification steps | ⚠️ High risk — review SSPR configuration now |
| Accounts have Owner-level permissions on Azure Key Vault | ⚠️ High risk — audit and reduce immediately |
| ScreenConnect or ConnectWise was unexpectedly installed on an Azure VM | ✅ Yes — assume compromise, isolate and investigate |
| You use Microsoft Defender XDR with cross-domain correlation enabled | ✅ Lower risk — this is how the attack was detected |
| You have Conditional Access blocking MFA registration from unknown networks | ✅ Lower risk — key control blocks persistence step |
| SQL Auditing logs are stored in an immutable Azure Storage account | ✅ Lower risk — attackers can clear Windows logs but not Azure-native audit |

---

## References

- [Microsoft Security Blog — How Storm-2949 turned a compromised identity into a cloud-wide breach](https://www.microsoft.com/en-us/security/blog/2026/05/18/storm-2949-turned-compromised-identity-into-cloud-wide-breach/) — primary vendor disclosure
- [BleepingComputer — Microsoft Self-Service Password Reset abused in Azure data theft attacks](https://www.bleepingcomputer.com/news/security/microsoft-self-service-password-reset-abused-in-azure-data-theft-attacks/)
- [GBHackers — Hackers Exploit Entra ID Accounts to Steal Microsoft 365, Azure Data](https://gbhackers.com/hackers-exploit-entra-id/)
- [Cyber Security News — Hackers Abuse Microsoft Entra ID Accounts to Exfiltrate Microsoft 365 and Azure Data](https://cybersecuritynews.com/hackers-abuse-microsoft-entra-id-accounts/)
- [MITRE ATT&CK](https://attack.mitre.org) — technique reference

---

<sub>Part of the <strong>Weekly Breach Investigation</strong> series · Investigating one real-world breach per week to build practical SOC analyst skills · <a href="https://attack.mitre.org">attack.mitre.org</a></sub>
# Mistic Backdoor — KongTuke IAB Campaign — 2026-06-24

> **Weekly Breach Investigation**
> *Week 08 · Initial Access Broker · DLL Sideloading · ClickFix · In-Memory Backdoor · Ransomware Enablement*

---

## Executive Summary

A financially motivated initial access broker tracked as KongTuke (also known as Woodgnat, 404 TDS, TAG-124, and Chaya_002) has been deploying a newly developed, fileless backdoor named Mistic against organizations in insurance, education, IT, and professional services since April 2026.

Mistic runs entirely in memory using DLL sideloading via a legitimate Microsoft endpoint security executable, actively masquerading as trusted tooling to avoid detection, and delivers victim network access to ransomware affiliates including Qilin, Akira, Rhysida, Black Basta, Interlock, and 8Base.

The backdoor was discovered not by the victim organizations but by researchers at Symantec and Zscaler ThreatLabz, who identified it deployed alongside ModeloRAT(another KongTuke tool) and confirmed it includes a self-destruct kill switch designed to erase all evidence of the intrusion.

---

## Attack Timeline

| Date | Event |
|---|---|
| **May 2024** | KongTuke begins operating a traffic distribution system (TDS) built on compromised WordPress sites, profiling visitors with injected JavaScript before pushing social engineering lures |
| **Early 2025** | KongTuke introduces ClickFix lure variants: original ClickFix → FileFix → CrashFix (fake ad-blocker extension that crashes browser and presents malicious "fix") |
| **Jan 2026** | Huntress first documents ModeloRAT, a Python RAT attributed to KongTuke, delivered via the CrashFix variant |
| **Feb 2026** | Microsoft flags a DNS-based ClickFix variant where KongTuke uses DNS TXT record lookups as a lightweight staging and signaling channel for payload delivery |
| **Apr 2026** | KongTuke pivots to fake Microsoft Teams IT helpdesk messages (Rapid7 / ReliaQuest report). Victim tricked into running PowerShell; script downloads a portable Python environment and launches ModeloRAT. Reconnaissance and credential harvesting follow. **Mistic first observed deployed in this campaign.** |
| **May 2026** | Zscaler ThreatLabz identifies Mistic (tracked as MLTBackdoor) as a payload in a multi-stage ClickFix chain. Notes BOF (Beacon Object File) loading as a key capability |
| **Jun 24, 2026** | Symantec and Carbon Black publish full attribution report linking Mistic firmly to KongTuke/Woodgnat. BleepingComputer and The Hacker News report concurrently. Full IOC set published by CybersecurityNews |

> **Dwell potential: Extended / unknown.** Mistic is designed for long-term, low-visibility persistence. No victim organisation is identified as having detected it internally; discovery was by external threat researchers across multiple incidents.

---

## How the Attack Worked

KongTuke does not launch ransomware itself, it sells access. The goal of every technique below is to get persistent, undetected footing inside a corporate network and then sell that access to ransomware affiliates.

### Delivery vectors (multiple, evolving)

```
Vector 1 — ClickFix / CrashFix (web-based):
  Compromised WordPress site (KongTuke TDS)
        ↓
  Visitor profiled via injected JavaScript
        ↓
  Social engineering lure served (ClickFix / FileFix / CrashFix variant)
  "Your browser has crashed. Run this command to fix it."
        ↓
  User pastes malicious command into Run/PowerShell
        ↓
  DNS TXT lookup OR direct download retrieves next-stage payload
        ↓
  WinPython / Node.js runtime downloaded → ModeloRAT executed
        ↓
  Reconnaissance + credential harvest → Mistic deployed

Vector 2 — Microsoft Teams (corporate vishing):
  Attacker impersonates IT helpdesk via fake Teams account
        ↓
  "We need you to run a security scan. Open PowerShell and paste this."
        ↓
  PowerShell script → portable Python environment → ModeloRAT
        ↓
  Reconnaissance + credential harvest → Mistic deployed
```

### Mistic infection chain (post-ModeloRAT)

```
MpExtMs.exe (LEGITIMATE Microsoft endpoint security binary)
        │
        └─── sideloads ──► version.dll  [MALICIOUS LOADER]
                                │
                                │  hooks GetModuleFileNameW + LoadLibraryW
                                │
                                └─► EndpointDlp.dll  [MISTIC BACKDOOR]
                                        │
                                        ├─ All execution in MEMORY (nothing written to disk)
                                        ├─ Polls C2 for commands
                                        ├─ Loads BOFs to expand capabilities dynamically
                                        ├─ Kill switch: can terminate + delete all files
                                        │
                                        └─► f.dll  [FAKE LOGIN SCREEN — .NET credential stealer]
                                                │
                                                └─ Harvests user credentials via spoofed prompt
```

### What was and was NOT affected

| Component | Status |
|---|---|
| Insurance sector organisations | ⚠️ Targeted — confirmed incidents |
| Education sector organisations | ⚠️ Targeted — confirmed incidents |
| IT / Professional Services firms | ⚠️ Targeted — confirmed incidents |
| Organisations using Microsoft Teams | ⚠️ At risk via Teams vishing vector |
| WordPress site visitors (via KongTuke TDS) | ⚠️ At risk via ClickFix/CrashFix vector |
| Systems where MpExtMs.exe loads unexpected DLLs | ⚠️ Indicator of active compromise |
| Organisations without EDR memory scanning | ⚠️ Blind to this threat — no disk artifacts |

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Initial Access | Drive-By Compromise (compromised WordPress TDS) | [T1189](https://attack.mitre.org/techniques/T1189/) |
| Initial Access | Phishing: Spearphishing via Service (Teams vishing) | [T1566.003](https://attack.mitre.org/techniques/T1566/003/) |
| Execution | User Execution: Malicious File (ClickFix PowerShell paste) | [T1204.002](https://attack.mitre.org/techniques/T1204/002/) |
| Execution | Command and Scripting Interpreter: PowerShell | [T1059.001](https://attack.mitre.org/techniques/T1059/001/) |
| Defense Evasion | Hijack Execution Flow: DLL Side-Loading | [T1574.002](https://attack.mitre.org/techniques/T1574/002/) |
| Defense Evasion | Masquerading: Match Legitimate Name or Location | [T1036.005](https://attack.mitre.org/techniques/T1036/005/) |
| Defense Evasion | Obfuscated Files or Information: Fileless Execution | [T1027.011](https://attack.mitre.org/techniques/T1027/011/) |
| Defense Evasion | Indicator Removal: File Deletion (kill switch) | [T1070.004](https://attack.mitre.org/techniques/T1070/004/) |
| Credential Access | Input Capture: GUI Input Capture (fake login screen) | [T1056.002](https://attack.mitre.org/techniques/T1056/002/) |
| Credential Access | Credentials from Password Stores | [T1555](https://attack.mitre.org/techniques/T1555/) |
| Persistence | (Designed for persistent C2 polling — no specific disk persistence mechanism confirmed) | — |
| Command & Control | Application Layer Protocol (C2 polling via HTTP/S) | [T1071.001](https://attack.mitre.org/techniques/T1071/001/) |
| Command & Control | Dynamic Resolution: DNS (DNS TXT staging channel) | [T1568.001](https://attack.mitre.org/techniques/T1568/001/) |
| Command & Control | Ingress Tool Transfer (BOF loading from C2) | [T1105](https://attack.mitre.org/techniques/T1105/) |
| Collection | Data from Local System | [T1005](https://attack.mitre.org/techniques/T1005/) |
| Impact | Financial Theft (access sold to ransomware affiliates) | [T1657](https://attack.mitre.org/techniques/T1657/) |

---

## Detection Opportunities

### Log Sources

- **EDR / memory telemetry** : process injection, reflective DLL loading, in-memory execution without disk backing. This is the primary detection surface; disk-based AV is ineffective.
- **Windows Event Log — 4688 (process creation)** : flag `MpExtMs.exe` spawning from unexpected parent processes or loading non-standard DLLs
- **Sysmon Event ID 7 (Image Loaded)** : `MpExtMs.exe` loading `version.dll` or `EndpointDlp.dll` from non-standard paths
- **DNS logs** : TXT record queries for non-infrastructure domains, especially following PowerShell execution
- **Microsoft Teams audit logs** : messages from accounts outside the corporate tenant containing PowerShell instructions or file links
- **Network flow / proxy logs** : periodic beaconing to C2 domains, especially those registered recently with `.top` / `.com` TLDs following unusual PowerShell activity
- **PowerShell Script Block Logging (Event 4104)** : commands downloading Python environments, running DNS TXT lookups, or executing encoded payloads


### IOCs

| Type | Value | Description |
|---|---|---|
| SHA-256 | `1e41c7bfaa6aa3b93b6cc024274a10e33f3e12fe7c98c1db387ef8927f9d1984` | Backdoor.Mistic — endpointdlp.dll |
| SHA-256 | `34d798a6c55e57ed0932b6499f4fbcb5454bdfca903307be101a0594b0ac07bc` | Credential stealer — f.dll (fake login screen) |
| SHA-256 | `3f797a639bc855bc6d5471f327924b62d10900ddec49b970eca6604142bbb4be` | Backdoor.Mistic — aeff97fe.msi |
| SHA-256 | `59e3c4cb06331b4f2d78a9a0592f3747e573bd01c5a7650c26361d1e25520712` | Loader — version.dll |
| SHA-256 | `8c935feec4bd05d5d918df308be417532fb42608fb989a08eab183e0ae699235` | Likely privilege escalation — n.dll |
| SHA-256 | `afd5f1ed45a9867daf3bc64152cef460a06b164c8183e490db39146d4749a82c` | Backdoor.Mistic — endpointdlp.dll (variant) |
| SHA-256 | `db972979d508e75fe730d3b72c2701470fbdaeaf8ebdd674744754fa44438ca5` | Backdoor.Mistic — endpointdlp.dll (variant) |
| SHA-256 | `f591275a8f014b29e567529d67c54eb7bb4473db1c38737d6bfd5b3d52c9344e` | Backdoor.Mistic — 48b47c0.msi |
| SHA-256 | `fb3630822b70bacb56aa4cec29b5a0e3e9acb3920809e70310a4003385a6d34a` | Backdoor.Mistic — endpointdlp.dll (variant) |
| IP | `142[.]93[.]242[.]144` | C2 network indicator |
| IP | `144[.]31[.]53[.]78` | C2 network indicator |
| IP | `198[.]13[.]159[.]44` | C2 network indicator |
| IP | `199[.]91[.]221[.]42` | C2 network indicator |
| Domain | `authorized-logins[.]net` | C2 domain |
| Domain | `b6w9m2z5x8q1v3k[.]top` | C2 domain |
| Domain | `carrolc[.]com` | C2 domain |
| Domain | `cj06y9v4xab[.]com` | C2 domain |
| Domain | `cwrtwright[.]com` | C2 domain |
| Domain | `defs[.]updater-worelos[.]com` | C2 domain |
| Domain | `ftps[.]upd-domain-goloro[.]com` | C2 domain |
| Domain | `grande-luna[.]top` | C2 domain |
| Domain | `human-check[.]top` | C2 domain |
| Domain | `mueleer[.]com` | C2 domain |
| Domain | `nano[.]upscale-kolo[.]com` | C2 domain |
| Domain | `oeannon[.]com` | C2 domain |
| Domain | `rotoa-upda-lo[.]com` | C2 domain |
| Domain | `sql-updater-service[.]com` | C2 domain |
| Domain | `thomphon[.]com` | C2 domain |
| Domain | `upd-domain-goloro[.]com` | C2 domain |
| Domain | `update[.]update-fall[.]com` | C2 domain |
| Domain | `updater-worelos[.]com` | C2 domain |
| Domain | `upscale-kolo[.]com` | C2 domain |
| Domain | `w3xasv14culvnqj[.]top` | C2 domain |
| URL | `hxxp://thomphon[.]com/update[.]msi` | Malware delivery URL |
| Malicious filename | `EndpointDlp.dll` | Mistic backdoor DLL |
| Malicious filename | `version.dll` | Mistic loader DLL |
| Loader binary (legitimate, abused) | `MpExtMs.exe` | Legitimate Microsoft binary used for sideloading |
| Threat actor aliases | KongTuke / Woodgnat / 404 TDS / Chaya_002 / LandUpdate808 / TAG-124 | All same actor |

---

## Recommended Mitigations

1. **Block the identified C2 IPs and domains at the perimeter immediately**
   Add all four C2 IPs (`142[.]93[.]242[.]144`, `144[.]31[.]53[.]78`, `198[.]13[.]159[.]44`, `199[.]91[.]221[.]42`) and all defanged domains above to DNS blocklists and firewall deny rules. Check historical DNS resolution logs for any of these domains in your environment — any hit is a potential Mistic infection.

2. **Enable and alert on DLL sideloading from Microsoft security binaries**
   Configure Sysmon Event ID 7 to log image loads. Alert specifically when `MpExtMs.exe` loads any DLL that is not signed by Microsoft or not located in `%SystemRoot%\System32`. This detection would have caught Mistic's loader stage regardless of filename obfuscation.

3. **Enable PowerShell Script Block Logging (Event 4104) and Transcription**
   KongTuke's entire delivery chain runs through PowerShell — DNS TXT lookups, Python runtime downloads, and payload execution all appear in script block logs. Without this visibility, the initial access stage is invisible. Set `HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging` → `EnableScriptBlockLogging = 1`.

4. **Restrict Microsoft Teams messages from external tenants**
   In the Microsoft Teams admin centre, configure External Access to block or require approval for messages from unknown or unverified external organisations. The Teams vishing vector requires the attacker's fake account to reach employees — this policy blocks that path at the protocol level.

5. **Deploy EDR with in-memory execution detection**
   Mistic leaves no disk artifacts. Traditional AV and file-based detection will not find it. EDR solutions capable of detecting reflective DLL injection, process hollowing, and executable memory regions without corresponding disk files are the only reliable detection control. Test your EDR against these techniques specifically.

6. **Block or monitor PowerShell downloading Python runtimes**
   A PowerShell script downloading and executing a portable Python environment (`WinPython`, `pythonw.exe`) is not normal user behaviour. Create an AppLocker or Windows Defender Application Control (WDAC) rule that blocks execution of Python binaries from user-writable temp or download directories — this severs the ModeloRAT deployment step.

7. **Hunt for historical KongTuke TDS WordPress indicators**
   KongTuke's WordPress TDS has been active since mid-2024. Run retrospective queries in your proxy and DNS logs for requests to WordPress sites that subsequently resolved DNS TXT records or downloaded `.msi` files. Any such pattern in the past 12 months warrants endpoint investigation on the requesting machine.

---

## Analyst Notes

**What I learned**

KongTuke's business model reframes how to think about the ransomware threat: the actual ransomware groups are not doing the hard part. An IAB gets access, sits quietly for weeks or months, and sells a clean entry point to the highest bidder. By the time ransomware deploys, Mistic has already been doing its job invisibly for an extended period. The threat intel gap ( knowing that KongTuke has been active since 2024 but only now seeing Mistic documented ) suggests there are likely earlier tools in this actor's arsenal that haven't been named yet.

**What surprised me**

The evolution of the social engineering delivery is genuinely impressive from a tradecraft perspective. ClickFix → FileFix → CrashFix → Teams IT helpdesk impersonation shows a group that actively monitors which lures get burned and rotates to new ones. The Teams vector is particularly dangerous because it exploits the implicit trust employees have in their internal IT communication channel — the attack surface is now any collaboration tool, not just email or the web.


---

## References

- [Symantec / Broadcom Threat Intelligence — New Mistic Backdoor and ModeloRAT](https://www.security.com/threat-intelligence/new-mistic-backdoor-modeloRAT) — primary technical report
- [BleepingComputer — Stealthy Mistic backdoor linked to ransomware access broker KongTuke](https://www.bleepingcomputer.com/news/security/stealthy-mistic-backdoor-linked-to-ransomware-access-broker-kongtuke/) — Jun 24, 2026
- [The Hacker News — New Mistic Backdoor Linked to KongTuke in ClickFix and ModeloRAT Campaigns](https://thehackernews.com/2026/06/new-mistic-backdoor-linked-to-kongtuke.html) — Jun 25, 2026
- [CybersecurityNews — Mistic Backdoor Blends With Microsoft Endpoint Security Tooling](https://cybersecuritynews.com/mistic-backdoor-blends-with-microsoft-endpoint-security/) — Jun 24, 2026 (full IOC table)
- [Zscaler ThreatLabz — Technical Analysis: MLTBackdoor](https://www.zscaler.com/blogs/security-research/technical-analysis-mltbackdoor) — technical deep-dive
- [MITRE ATT&CK](https://attack.mitre.org) — technique reference

---

<sub>Part of the <strong>Weekly Breach Investigation</strong> series · Investigating one real-world breach per week to build practical SOC analyst skills · <a href="https://attack.mitre.org">attack.mitre.org</a></sub>
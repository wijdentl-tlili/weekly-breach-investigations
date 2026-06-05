# FortiClient EMS CVE-2026-35616 — EKZ Infostealer via Trusted Management Infrastructure — 2026-05-27

> **Weekly Breach Investigation**
> *Week 04 · CVE Exploitation · Trusted Infrastructure Abuse · Credential Theft · Enterprise VPN*

---

## 1. Executive Summary

Threat actors exploited **CVE-2026-35616**, a critical pre-authentication API access bypass (CVSS 9.1) in Fortinet's FortiClient Enterprise Management Server, to gain unauthenticated remote code execution on the EMS server itself. Once inside, attackers abused FortiClient's own endpoint management pipeline to silently push a previously undocumented credential stealer — dubbed **EKZ Infostealer** — to every managed endpoint, disguising it as a legitimate Fortinet firmware update. Any organisation running an unpatched FortiClient EMS instance with internet-exposed management interfaces is potentially affected, with the attack surface confirmed at approximately 2,000 exposed instances globally at time of disclosure. Stolen browser credentials — including Chromium v20 AES-256 encrypted password databases from Chrome, Edge, and Firefox — were exfiltrated over HTTP to attacker-controlled infrastructure before local artifacts were wiped, leaving minimal forensic trace.

---

## 2. Attack Timeline

| Date | Event |
|---|---|
| **Feb 2026** | CVE-2026-35616 originally disclosed by Fortinet's product security team as CVE-2026-21643 |
| **Mar 24, 2026** | Defused honeypot network detects first in-the-wild exploitation activity |
| **Mar 28, 2026** | Defused publishes detection findings; Fortinet confirms active exploitation and issues emergency hotfixes for FortiClient EMS 7.4.5 and 7.4.6 |
| **Late Mar–Early Apr 2026** | CISA issues emergency directive ordering federal agencies to patch by end of that week. Shadowserver Foundation reports ~2,000 internet-exposed EMS instances globally |
| **May 2026** | Arctic Wolf Labs observes a fresh cluster of intrusions exploiting the same CVE — weeks after hotfixes were available — delivering EKZ Infostealer to managed endpoints |
| **May 27, 2026** | Arctic Wolf publishes full technical disclosure; BleepingComputer and The Hacker News report |

> **Patch gap / dwell window:** CVE publicly exploited from ~Mar 24. EKZ deployment campaign observed May 2026 — meaning organisations that had not applied the emergency hotfix remained exposed for **~6 weeks** of active exploitation before this campaign was detected.

---

## 3. How the Attack Worked

The attack abused FortiClient EMS's own trusted management channel as the delivery mechanism — no phishing, no social engineering of end users required.

```
ATTACK FLOW

[1] INITIAL ACCESS
Attacker sends specially crafted API request to FortiClient EMS
→ CVE-2026-35616: authentication header spoofed
→ Bypass grants unauthenticated access to EMS admin API

[2] PERSISTENCE & CONFIGURATION MANIPULATION
Attacker modifies EMS configuration:
  - Defers firmware upgrade reminders (prevents legitimate patching)
  - Modifies Remote Access Profile
  - Inserts malicious PowerShell into endpoint policy

[3] MALWARE DELIVERY (via trusted channel)
EMS pushes "update" to ALL managed endpoints
  - Payload: EKZ Infostealer EXE disguised as Fortinet firmware patch
  - Delivery: FortiClient component on endpoints launches PowerShell
  - PowerShell downloads EKZ → executes silently → 8-second delay (sandbox evasion not confirmed here, but consistent with known evasion)

[4] CREDENTIAL HARVEST (on each managed endpoint)
EKZ targets Chrome / Edge (Chromium):
  - Reads HKCU registry to locate browser installations
  - Copies os_crypt.app_bound_encrypted_key from Local State
  - Re-launches from browser Application\ directory (passes elevation service path validation)
  - Calls IElevator::DecryptData → obtains Chromium v20 AES-256 master key
  - Iterates browser profiles → decrypts SQLite credential databases

EKZ targets Firefox:
  - Locates profile directory, extracts logins.json and key4.db

[5] STAGING & EXFILTRATION
  - Harvested credentials written to log.txt in C:\ProgramData\
  - Exfiltrated over HTTP on a timed basis
  - Local artifacts removed post-exfiltration

WHAT WAS AND WAS NOT AFFECTED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FortiClient EMS server          → ⚠️ COMPROMISED (auth bypass → RCE)
All EMS-managed endpoints       → ⚠️ COMPROMISED (malicious policy pushed)
Browser credentials (Chrome/Edge/Firefox) → ⚠️ EXFILTRATED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Unmanaged / standalone endpoints → ✅ Not directly affected
FortiGate firewall appliances   → ✅ Not this CVE (separate product)
FortiClient app (standalone)    → ✅ Not the vulnerability vector
```

---

## 4. MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Initial Access | Exploit Public-Facing Application | [T1190](https://attack.mitre.org/techniques/T1190/) |
| Execution | Exploitation for Client Execution (via EMS policy) | [T1203](https://attack.mitre.org/techniques/T1203/) |
| Execution | Command and Scripting Interpreter: PowerShell | [T1059.001](https://attack.mitre.org/techniques/T1059/001/) |
| Defense Evasion | Masquerading: Match Legitimate Name or Location | [T1036.005](https://attack.mitre.org/techniques/T1036/005/) |
| Defense Evasion | Indicator Removal: File Deletion (post-exfil artifact wipe) | [T1070.004](https://attack.mitre.org/techniques/T1070/004/) |
| Defense Evasion | Trusted Developer Utilities Proxy Execution | [T1127](https://attack.mitre.org/techniques/T1127/) |
| Persistence | Modify Authentication Process (EMS config modification) | [T1556](https://attack.mitre.org/techniques/T1556/) |
| Credential Access | Credentials from Password Stores: Credentials from Web Browsers | [T1555.003](https://attack.mitre.org/techniques/T1555/003/) |
| Credential Access | Unsecured Credentials (log.txt staged in ProgramData) | [T1552](https://attack.mitre.org/techniques/T1552/) |
| Collection | Data from Local System | [T1005](https://attack.mitre.org/techniques/T1005/) |
| Exfiltration | Exfiltration Over C2 Channel (HTTP timed exfil) | [T1041](https://attack.mitre.org/techniques/T1041/) |

---

## 5. Detection Opportunities

### Log Sources

- **FortiClient EMS audit logs** — unauthenticated API access events; unexpected Remote Access Profile modifications; endpoint policy changes not initiated by a known admin account
- **Windows Event Log ID 4688** — PowerShell spawning from FortiClient process tree; unexpected child processes of `FortiClient.exe`
- **Sysmon Event ID 1 / 11** — EXE written to `C:\ProgramData\` from a FortiClient parent process; process creation with browser `Application\` path
- **Network / proxy logs** — HTTP POST to non-Fortinet destinations from `log.txt` reference; periodic timed outbound connections from managed endpoints
- **EDR telemetry** — process calling `IElevator::DecryptData`; browser credential database (Login Data SQLite) access from non-browser processes

### IOCs

| Type | Value |
|---|---|
| CVE | CVE-2026-35616 (CVSS 9.1 — pre-auth API bypass → RCE) |
| Malware family | EKZ Infostealer (undocumented prior to this campaign) |
| Staged artifact path | `C:\ProgramData\log.txt` |
| Malware masquerade | Fortinet firmware/endpoint update |
| Affected versions | FortiClient EMS 7.4.5, 7.4.6 (patched in 7.4.7 + emergency hotfixes) |
| Exposed instance count (at disclosure) | ~2,000 internet-facing EMS instances (Shadowserver) |
| Discovery source | Arctic Wolf Labs (May 2026 cluster); original exploitation detected by Defused honeypots (Mar 2026) |

> Note: Specific C2 domains and file hashes for EKZ were not publicly released at time of writing. Monitor Arctic Wolf Labs and AlienVault OTX for updates.

---

## 6. Recommended Mitigations

1. **Apply emergency hotfixes immediately — do not wait for 7.4.7**
   Fortinet released emergency hotfixes for FortiClient EMS 7.4.5 and 7.4.6 in late March 2026. Any organisation still running unpatched versions is actively exploitable. Apply the hotfix now; upgrade to 7.4.7 when available to receive the permanent fix.

2. **Remove FortiClient EMS management interfaces from the public internet**
   Shadowserver identified ~2,000 internet-exposed instances at disclosure. EMS management should only be accessible from internal networks or via a separately authenticated VPN jump host. No management plane should ever face the internet directly.

3. **Audit all FortiClient EMS endpoint policies and Remote Access Profiles immediately**
   Review every policy for unexpected PowerShell references, external download URLs, or profile modifications made by non-admin accounts. Any unrecognised change should be treated as a compromise indicator until proven otherwise.

4. **Hunt for EKZ Infostealer artifacts on all EMS-managed endpoints**
   Search for `log.txt` in `C:\ProgramData\`, unexpected EXEs in browser `Application\` directories, and Sysmon events showing non-browser processes accessing Chrome's `Login Data` or `Local State` files. Correlate with timed outbound HTTP from affected hosts.

5. **Rotate all credentials stored in browsers on managed endpoints**
   Any endpoint that was under EMS management during the exploitation window should be treated as credential-compromised. Rotate all browser-saved passwords, API keys, and tokens regardless of whether EKZ artifacts are found — the malware removes its own traces post-exfil.

6. **Enforce network segmentation between EMS server and endpoint policy delivery**
   The root damage amplifier here was that one compromised EMS server had unrestricted authority to push executable code to all managed endpoints. Implement change-approval workflows, policy signing, and alert on any policy containing script execution before it is deployed.

---

## 7. Analyst Notes

**What I learned**

This attack is a case study in how trust hierarchies become weapons. FortiClient EMS exists specifically to have authoritative control over endpoints — that's its job. When the attacker took over the EMS server, they didn't need to compromise each endpoint individually. They inherited the trust relationship that every managed endpoint already had with the server. One exploitation point, thousands of victims. Any centralised management platform — MDM, RMM, SIEM agents, EDR consoles — carries this same risk profile if the management server itself is exposed and unpatched.

**What surprised me**

The ~6-week gap between the first confirmed exploitation (March 24) and the Arctic Wolf cluster (May 2026) is alarming but not surprising — what's notable is that CISA issued an emergency directive and Shadowserver publicly mapped 2,000 exposed instances, and organisations still hadn't patched. The signal-to-action gap in enterprise patching is consistently wider than defenders assume, especially for internet-facing management infrastructure that teams treat as "internal."

---

## 8. Who Is at Risk — Quick Check

| Condition | At risk? |
|---|---|
| Running FortiClient EMS 7.4.5 or 7.4.6 with internet-exposed management interface | ✅ Yes — patch immediately, hunt for EKZ |
| Running FortiClient EMS 7.4.5 / 7.4.6, management interface internal-only | ⚠️ Lower risk but still vulnerable — patch immediately |
| Applied Fortinet emergency hotfix for 7.4.5 / 7.4.6 before May 2026 cluster | ⚠️ Patched against new exploitation, but review logs for earlier compromise window |
| Running FortiClient EMS 7.4.7 or later | ✅ Patched — confirm by reviewing changelog |
| Using FortiClient standalone (not EMS-managed) | ✅ Not the attack vector — not directly affected |
| Using FortiGate or other Fortinet appliances (not EMS) | ✅ Different products — not this CVE |

---

## 9. References

- [Arctic Wolf Labs — Technical disclosure (primary research source)](https://arcticwolf.com/resources/blog/forticlient-ems-exploited-via-cve-2026-35616-to-deliver-ekz-infostealer-disguised-as-a-fortinet-patch/)
- [BleepingComputer — Hackers exploit FortiClient EMS flaw to push infostealer malware](https://www.bleepingcomputer.com/news/security/hackers-exploit-forticlient-ems-flaw-to-push-infostealer-malware/)
- [The Hacker News — Threat Actors Exploit Critical FortiClient EMS Flaw](https://thehackernews.com/2026/05/threat-actors-exploit-critical.html)
- [Cybersecurity Dive — Critical flaw in FortiClient EMS under exploitation](https://www.cybersecuritydive.com/news/critical-flaw-forticlient-ems-exploitation/816699/)
- [MITRE ATT&CK](https://attack.mitre.org) — Technique reference
- [Shadowserver Foundation](https://shadowserver.org) — Exposed instance enumeration

---

<sub>Part of the <strong>Weekly Breach Investigation</strong> series · Investigating one real-world breach per week to build practical SOC analyst skills · <a href="https://attack.mitre.org">attack.mitre.org</a></sub>
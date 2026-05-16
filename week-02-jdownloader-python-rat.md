# JDownloader Site Hacked — Python RAT via Poisoned Installers — 2026-05-07

> *Week 02 · Software Supply Chain · CMS Exploitation · Python RAT · Credential Theft*

---

## Executive Summary

Attackers exploited an unpatched vulnerability in the JDownloader website's content management system to silently redirect official Windows and Linux installer download links to malicious third-party files containing a heavily obfuscated Python-based remote access trojan.

Users who downloaded the Windows "Alternative Installer" or the Linux shell installer from jdownloader.org between **May 6 and May 7, 2026** unknowingly installed a full-featured RAT giving attackers persistent remote control over their machines.

The compromise was discovered by a Reddit user whose antivirus flagged the download ( not by the vendor ) and the site was taken offline within 18 minutes of the report; affected users are advised to perform a full OS reinstall as antivirus scans alone cannot guarantee removal of all persistence mechanisms.

---

## Attack Timeline

| Date / Time (UTC) | Event |
|---|---|
| **May 5, 2026 — 23:55 UTC** | Attackers test their CMS exploit on a low-traffic page as a dry run |
| **May 6, 2026 — 00:01 UTC** | Live installer download links on jdownloader.org successfully modified to redirect to attacker-controlled servers |
| **May 6–7, 2026** | Users downloading the Windows "Alternative Installer" or Linux shell installer receive trojanized files signed by spoofed publishers "Zipline LLC" and "The Water Team" instead of the legitimate "AppWork GmbH" |
| **May 7, 2026 — 17:06 UTC** | Reddit user **PrinceOfNightSky** reports suspicious publisher names and Microsoft Defender alerts on the downloaded installer |
| **May 7, 2026 — 17:24 UTC** | JDownloader development team takes the site fully offline for investigation — **18 minutes after the Reddit report** |
| **May 8–9, 2026 (night, UTC)** | After full security analysis and remediation, jdownloader.org restored with verified clean installer links and hardened CMS configuration |
| **May 9, 2026** | Public disclosure by JDownloader developers. |

> **Dwell time: ~41 hours** (00:01 May 6 → 17:24 May 7). Any user who downloaded during this window is at risk.

---

## How the Attack Worked

The attackers did **not** modify JDownloader's actual software or its internal update system. Instead they exploited a CMS vulnerability to change only the *targets* of specific download links on the website, pointing them at attacker-controlled servers hosting malicious files.

```
jdownloader.org (CMS exploited)
        │
        ├── "Alternative Installer" link (Windows) → ⚠️ REDIRECTED to attacker server
        ├── Linux shell installer link             → ⚠️ REDIRECTED to attacker server
        │
        ├── Main JAR package                        → ✅ Clean (not modified)
        ├── In-app update system (RSA-signed)        → ✅ Clean (cryptographically verified)
        ├── macOS installer                         → ✅ Clean
        ├── Flatpak / Winget / Snap packages        → ✅ Clean
        └── Primary Windows installer               → ✅ Clean
```

**Windows payload:** Trojanized installer acting as a loader for a heavily obfuscated Python-based RAT. Modular bot framework allowing attackers to execute arbitrary Python code delivered from C2 servers. 8-minute delay before payload activation (evasion tactic). Signed by spoofed publishers "Zipline LLC" / "The Water Team."

**Linux payload:** Injected shell script downloaded additional ELF malware, installed a SUID-root launcher, and disguised the payload as `/usr/libexec/upowerd` to blend in with legitimate system processes and survive reboots.

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Initial Access | Supply Chain Compromise: Software Distribution | [T1195.002](https://attack.mitre.org/techniques/T1195/002/) |
| Execution | User Execution: Malicious File | [T1204.002](https://attack.mitre.org/techniques/T1204/002/) |
| Stealth | Masquerading: Match Legitimate Name or Location | [T1036.005](https://attack.mitre.org/techniques/T1036/005/) |
| Defense Impairment | Subvert Trust Controls: Code Signing | [T1553.002](https://attack.mitre.org/techniques/T1553/002/) |
| Stealth | Obfuscated Files or Information | [T1027](https://attack.mitre.org/techniques/T1027/) |
| Persistence | Create or Modify System Process (Linux SUID launcher) | [T1543](https://attack.mitre.org/techniques/T1543/) |
| Privilege Escalation | Abuse Elevation Control: SUID/SGID (Linux) | [T1548.001](https://attack.mitre.org/techniques/T1548/001/) |
| Command & Control | Application Layer Protocol | [T1071](https://attack.mitre.org/techniques/T1071/) |
| Credential Access | Credentials from Password Stores | [T1555](https://attack.mitre.org/techniques/T1555/) |
| Collection | Data from Local System | [T1005](https://attack.mitre.org/techniques/T1005/) |

---

## IOCs

| Type | Value |
|---|---|
| C2 endpoint (Windows) | `parkspringshotel[.]com/m/Lu6aeloo.php` |
| C2 endpoint (Windows) | `auraguest[.]lk/m/douV2quu.php` |
| Malicious publisher name | `Zipline LLC` |
| Malicious publisher name | `The Water Team` |
| Linux malicious filename | `JDownloader2Setup_unix_nojre.sh` |
| Linux file size | `7,934,496 bytes` |
| Linux SHA256 | `6d975c05ef7a164707fa359284a31bfe0b1681fe0319819cb9e2c4eec2a1a8af` |
| Windows 'JDownloader2Setup_windows-amd64_v11_0_30.exe' SHA256 | `fb1e3fe4d18927ff82cffb3f82a0b4ffb7280c85db5a8a8b6f6a1ac30a7e7ed9` |
| Windows 'JDownloader2Setup_windows-amd64_v17_0_18.exe' SHA256 | `04cb9f0bca6e0e4ed30bc92726590724bf60938440b3825252657d1b3af45495` |
| Windows 'JDownloader2Setup_windows-amd64_v1_8_0_482.exe' SHA256 | `5a6636ce490789d7f26aaa86e50bd65c7330f8e6a7c32418740c1d009fb12ef3` |
| Windows 'JDownloader2Setup_windows-amd64_v21_0_10.exe' SHA256 | `32891c0080442bf0a0c5658ada2c3845435b4e09b114599a516248723aad7805` |
| Windows 'JDownloader2Setup_windows-x86_v11_0_29.exe' SHA256 | `de8b2bdfc61d63585329b8cfca2a012476b46387435410b995aeae5b502bd95e` |
| Windows 'JDownloader2Setup_windows-x86_v17_0_17.exe' SHA256 | `e4a20f746b7dd19b8d9601b884e67c8166ea9676b917adea6833b695ba13de16` |
| Windows 'JDownloader2Setup_windows-x86_v1_8_0_472.exe' SHA256 | `4ff7eec9e69b6008b77de1b6e5c0d18aa717f625458d80da610cb170c784e97c` |
| Linux disguised payload path | `/usr/libexec/upowerd` |
| Legitimate publisher (clean) | `AppWork GmbH` |


---

## Recommended Mitigations

1. **Full OS reinstall for anyone who ran the affected installers**
   Antivirus scans, including Malwarebytes and Windows Defender Offline, have returned no detections on confirmed-infected machines. The RAT's persistence mechanisms are robust enough to survive standard remediation. Full wipe and reinstall is the only reliable recovery path.

2. **Rotate all credentials on a separate, clean device**
   As the RAT had full arbitrary code execution capability, any credentials stored in browsers, password managers, or environment variables on the affected device must be treated as compromised. Reset all passwords and revoke any API keys or tokens before accessing sensitive accounts.

3. **Verify installer digital signatures before execution — always**
   Right-click → Properties → Digital Signatures tab. Legitimate JDownloader installers are signed by **AppWork GmbH** exclusively. Any other publisher name or a missing signature is a hard stop — do not execute.

4. **Prefer in-app updates over website re-downloads**
   JDownloader's built-in updater is RSA-signed and cryptographically verified independently of the website. It was not affected by this attack. Where an in-app update path exists, always prefer it over downloading a fresh installer from a website.

5. **Block the identified C2 domains at the perimeter**
   Add `parkspringshotel[.]com` and `auraguest[.]lk` to DNS blocklists and firewall deny rules. Monitor for any historical DNS resolution of these domains in your environment to identify potentially compromised hosts.

6. **Apply CMS security hardening and patch management**
   The root cause was an unpatched CMS vulnerability that allowed unauthenticated modification of access control lists. Organisations running public-facing CMS platforms should enforce: regular patching cycles, principle of least privilege for CMS accounts, WAF rules for ACL-modification endpoints, and immutable infrastructure patterns where possible.

---

## Analyst Notes

**What I learned**

This attack illustrates one of the most dangerous properties of supply chain attacks: the victim's own trust does the attacker's work for them. A user downloading software from an official website has every reason to trust what they receive, they don't need to be phished or socially engineered. The attacker only needed to compromise the *pointer* (the download link), not the actual software. CMS security is now a critical part of the software supply chain, and most organisations treat it as a web admin problem rather than a security problem.

**What surprised me**

The 8-minute payload activation delay is a deliberate sandbox evasion technique, automated malware analysis sandboxes typically run samples for 2–5 minutes before making a verdict. By waiting 8 minutes, the RAT bypassed most automated initial analysis. The fact that Malwarebytes and Windows Defender Offline both returned clean results on confirmed-infected machines is particularly alarming, it means defenders cannot rely on AV as a confidence signal for this malware class.


---

## Who Is at Risk — Quick Check

| Condition | At risk? |
|---|---|
| Downloaded "Alternative Installer" from jdownloader.org on **May 6–7, 2026** AND ran it | ✅ Yes — full OS reinstall recommended |
| Downloaded but did NOT execute the file | ⚠️ Delete the file immediately without running it |
| Used JDownloader's in-app update system | ✅ No — RSA-signed, not affected |
| Downloaded the primary Windows installer (non-alternative) | ✅ No — not modified |
| Downloaded macOS installer | ✅ No — not affected |
| Used Flatpak / Winget / Snap | ✅ No — not affected |
| Downloaded before May 6 or after May 7 | ✅ No — outside the compromise window |

---

## References

- [BleepingComputer — JDownloader site hacked to replace installers with Python RAT malware](https://www.bleepingcomputer.com/news/security/jdownloader-site-hacked-to-replace-installers-with-python-rat-malware/)
- [JDownloader official incident report](https://jdownloader.org) — vendor disclosure
- [GBHackers — JDownloader Hack Spreads New Python RAT](https://gbhackers.com/jdownloader-hack/)
- [Security Affairs — Official JDownloader site served malware](https://securityaffairs.com/191920/malware/official-jdownloader-site-served-malware-to-windows-and-linux-users.html)

---

<sub>Part of the <strong>Weekly Breach Investigation</strong> series · Investigating one real-world breach per week to build practical SOC analyst skills · <a href="https://attack.mitre.org">attack.mitre.org</a></sub>
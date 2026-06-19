# FortiBleed — Global Fortinet Credential Exposure Campaign — 2026-06-17

> **Weekly Breach Investigation**
> *Week 07 · Credential Harvesting · Edge Device Compromise · Critical Infrastructure · Financially Motivated*

---

## Executive Summary

A Russian-speaking cybercrime group ran a large-scale, automated campaign (dubbed **FortiBleed**) that combined credential stuffing, password cracking, and SSL VPN traffic interception against internet-facing Fortinet FortiGate firewalls, building a continuously updated database of verified working administrator and VPN credentials.

Organisations across **194 countries** were affected, with confirmed victims spanning telecommunications, government, healthcare, finance, energy, and named enterprises including Foxconn, Samsung, Siemens, PwC, Accenture, and Comcast; estimates of compromised devices range from **30,791 verified** (SOCRadar) to **~75,000–86,000** (Hudson Rock/Beaumont, SOCRadar follow-up).

The campaign gives attackers a direct foothold into victim perimeter networks and a credential-harvesting pivot point into internal Active Directory environments, prompting emergency advisories from the UK NCSC, US CISA, and Australia's ACSC within 24 hours of public disclosure.

---

## Attack Timeline

| Date | Event |
|---|---|
| **~Apr 2026** | Estimated start of campaign activity based on credential-database freshness (exact start unconfirmed — operation was already mature when discovered) |
| **Throughout campaign** | Attacker group runs continuous automated loop: scan internet-facing FortiGate devices → credential-stuff/brute-force/dictionary-attack SSL VPN portals → intercept and crack authentication via adversary-in-the-middle interception and GPU-cluster hash cracking → feed newly harvested credentials back into the scanning loop |
| **Jun 11–15, 2026** | Security researcher Volodymyr "Bob" Diachenko first flags credential harvesting activity in a LinkedIn post |
| **Jun 16, 2026** | SOCRadar discovers an exposed, unsecured operational server belonging to the threat group gaining visibility into tooling, victim database, automation infrastructure, and the verified credential repository |
| **Jun 17, 2026** | Public disclosure. SOCRadar reports 30,791+ verified working credentials across 194 countries; Beaumont/Hudson Rock joint analysis estimates ~75,000 affected devices (~50% of all internet-facing FortiGate firewalls visible on Shodan); Arctic Wolf publishes technical bulletin |
| **Jun 18, 2026** | UK NCSC, US CISA, and Australian ACSC each issue formal advisories within hours of each other. SOCRadar follow-up raises the device count to **86,644**. Reports confirm 5,616 credential entries tied to telecom organisations and 591 to government entities across 111 domains |

> **Dwell time: Unknown / extended** the credential database had the "recognizable fingerprint" of a mature, continuously-curated operation organised by sector, country, and company revenue tier at time of discovery, meaning the harvesting had been running for an unconfirmed but substantial period before any defender became aware of it. This is a *discovery-of-an-active-operation* incident, not a single point-in-time breach, the dwell time question remains open as of publication.

---

## How the Attack Worked

This was **not** a Fortinet zero-day and **not** a breach of Fortinet itself. It was a large-scale abuse of weak credential hygiene and a historical password-hashing weakness across tens of thousands of independently-managed customer devices.

```
Internet-facing FortiGate firewalls / SSL VPN gateways (customer-managed)
        │
        ├── Credential stuffing / brute-force / dictionary attacks against management interfaces (ports 443, 4443, 8443, 10443)
        │ 
        │
        ├── Adversary-in-the-middle interception of SSL VPN authentication
        │
        ├── Configuration file extraction → stored password hashes recovered
        │   └── Legacy SHA-256-with-Salt hashes (pre-FortiOS 7.2.11/7.4.8/7.6.1 or not yet re-hashed post-upgrade) → cracked via GPU cluster
        │       
        │
        ▼
   Verified working admin/VPN credential added to attacker database
   (organised by country, sector, company revenue — resold/used for access)
        │
        ▼
   Compromised device used as a "listening post" → sniffs further
   credentials from passing traffic → feeds back into the loop
        │
        ▼
   Pivot into internal network → Active Directory credential harvesting
   and lateral movement
```

**What was NOT exploited:** No confirmed Fortinet RCE or auth-bypass zero-day has been tied to FortiBleed as of publication. Leading theories for initial credential acquisition include credential reuse against exposed management interfaces, exploitation of known (already-patched) Fortinet vulnerabilities such as CVE-2026-24858, or infostealer malware harvesting admin credentials directly from operator machines, but as of publication **no source has definitively confirmed the initial access vector**.

**Root cause enabling the cracking step:** Fortinet introduced PBKDF2-based password hashing for administrator accounts in FortiOS 7.2.11, 7.4.8, and 7.6.1, replacing legacy SHA-256 storage. However, after upgrading FortiOS, existing administrator passwords **remain stored as SHA-256 hashes** until that specific administrator logs in post-upgrade meaning a large population of devices kept crackable legacy hashes long after "patching."

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Reconnaissance | Active Scanning: Vulnerable Devices | [T1595.002](https://attack.mitre.org/techniques/T1595/002/) |
| Initial Access | Valid Accounts (cracked/stuffed admin & VPN credentials) | [T1078](https://attack.mitre.org/techniques/T1078/) |
| Initial Access | External Remote Services (FortiGate SSL VPN) | [T1133](https://attack.mitre.org/techniques/T1133/) |
| Credential Access | Brute Force: Password Cracking | [T1110.002](https://attack.mitre.org/techniques/T1110/002/) |
| Credential Access | Brute Force: Credential Stuffing | [T1110.004](https://attack.mitre.org/techniques/T1110/004/) |
| Credential Access | Adversary-in-the-Middle | [T1557](https://attack.mitre.org/techniques/T1557/) |
| Credential Access | Unsecured Credentials (legacy SHA-256 config hashes) | [T1552](https://attack.mitre.org/techniques/T1552/) |
| Discovery | Network Service Discovery (mass internet scanning) | [T1046](https://attack.mitre.org/techniques/T1046/) |
| Collection | Network Sniffing (compromised devices as listening posts) | [T1040](https://attack.mitre.org/techniques/T1040/) |
| Lateral Movement | Use Alternate Authentication Material | [T1550](https://attack.mitre.org/techniques/T1550/) |
| Impact | Financial Theft (credential database curated for resale) | [T1657](https://attack.mitre.org/techniques/T1657/) |

---

## Detection Opportunities

### Log Sources
- **FortiGate VPN and system event logs** : SSL VPN login events, especially outside normal business hours or with anomalous client characteristics
- **FortiGate admin authentication logs** : login attempts against management interfaces (ports 443, 4443, 8443, 10443)
- **Active Directory authentication logs** : anomalous logins shortly following any suspected FortiGate compromise (lateral movement indicator)
- **Configuration backup audit trail** : any unscheduled or unauthorised configuration export from a super_admin account


### IOCs

| Type | Value |
|---|---|
| Campaign name | FortiBleed |
| Threat actor | Russian-speaking, multi-operator cybercrime group (unattributed to a named APT) |
| Scale (SOCRadar initial) | 30,791 verified working credentials |
| Scale (SOCRadar follow-up, Jun 18) | 86,644 compromised devices |
| Scale (Hudson Rock/Beaumont) | ~73,932 unique firewall URLs / ~75,000 devices (~50% of internet-facing FortiGate on Shodan) |
| Affected sectors (highest) | Telecommunications (5,616 credential entries); Government (591 entries / 111 domains) |
| Targeted ports | 443, 4443, 8443, 10443 |
| Cracking infrastructure | Hashtopolis-managed GPU cluster (reported 45-GPU scale in some analyses) |
| Named affected organisations | Foxconn, Samsung, Siemens, PwC, Accenture, Comcast *(per Nopal Cyber threat-hunting advisory; treat as third-party reporting pending individual confirmation)* |
| Geographic concentration | India and United States represent roughly one-third of identified compromises |
| Suspected linked CVE | CVE-2026-24858 (FortiCloud SSO SAML auth bypass, CVSS 9.8) — suspected, not confirmed, as a contributing initial-access vector |

> **No file hashes or C2 domains have been published for this campaign as of the reporting window** : this is a credential-database exposure/harvesting story, not a malware-payload story, so the IOC set is access-credentials and infrastructure-pattern based rather than file-based.

---

## Recommended Mitigations

1. **Rotate all FortiGate admin and SSL VPN credentials immediately**
   Treat every internet-facing FortiGate device as potentially compromised regardless of patch level. This is the unanimous first recommendation from NCSC, CISA, ACSC, and Arctic Wolf.

2. **Enforce PBKDF2 hashing and force admin re-login post-upgrade**
   Upgrading FortiOS alone does not re-hash existing administrator passwords. Every admin must log in at least once post-upgrade to trigger PBKDF2 conversion, or a super_admin must manually reset remaining accounts. On FortiOS v7.2.x/v7.4.x, additionally enable `login-lockout-upon-weaker-encryption` under `system password-policy` to fully purge lingering SHA-256 hashes from the hidden `old-password` field.

3. **Enforce MFA on all administrative and remote-access accounts**
   Phishing-resistant MFA neutralises the value of a cracked or stuffed credential even if the password itself is compromised.

4. **Remove management interfaces from public internet exposure**
   Restrict FortiGate management interface access to trusted internal networks only, regardless of vendor, this is a best practice that would have prevented external scanning and brute-forcing entirely.

5. **Audit for IOCs before any factory reset**
   Per NCSC guidance: check for unauthorised account creation and unexpected log file activity, and collect forensic artefacts (configuration backups, logs) **before** performing a factory reset, since reset can destroy evidence needed for incident response.

6. **Cross-reference exposure using public checking tools**
   Use Hudson Rock's FortiBleed Checker to determine whether your organisation's domains/IP ranges appear in the leaked credential dataset.

7. **Review Active Directory for lateral movement from edge devices**
   Given the confirmed pivot pattern (FortiGate compromise → AD credential harvesting), any organisation with a previously vulnerable FortiGate device should specifically audit AD authentication logs for activity sourced from that device's subnet.

---

## Analyst Notes

**What I learned**
This isn't a single breach. It's the discovery of an active, automated harvesting *operation* that had clearly been running and curating its database for some time before anyone outside the attacker group knew it existed. The "victim" framing is unusual here: tens of thousands of independently-managed organisations are all simultaneously exposed by the same root cause (legacy password hashing + internet-facing management interfaces) rather than by a single supply-chain pivot point. It's a reminder that edge devices ( firewalls, VPN gateways ) are not just perimeter controls; they're often the front door to the entire internal identity infrastructure behind them.

**What I'd investigate with internal access**
I'd want to pull our own FortiGate device inventory and cross-reference upgrade history against admin login history specifically, identify any device that was upgraded to FortiOS 7.2.11+ but where the admin account has not logged in since, since that's a silent, invisible exposure that wouldn't show up in a simple "are we patched" check.

---

## Who Is at Risk — Quick Check

| Condition | At risk? |
|---|---|
| Internet-facing FortiGate firewall or SSL VPN gateway, admin password unchanged in 12+ months | ✅ Yes — rotate immediately and check for IOCs |
| FortiOS upgraded to 7.2.11/7.4.8/7.6.1+ but admin has not logged in since upgrade | ✅ Yes — password is still stored as crackable SHA-256, not PBKDF2 |
| Management interface (ports 443/4443/8443/10443) reachable from the public internet | ✅ Yes — primary attack surface for this campaign |
| MFA not enforced on FortiGate admin or SSL VPN accounts | ✅ Yes — credential alone is sufficient for access |
| Management interface restricted to internal/trusted networks only | ⚠️ Reduced risk — still rotate credentials as a precaution |
| MFA enforced on all admin and VPN accounts | ⚠️ Reduced risk — still audit for prior compromise |
| Organisation's domain confirmed absent from Hudson Rock's FortiBleed Checker | ⚠️ No confirmed exposure — continue monitoring, campaign is still active |

---

## References

- [NCSC — Advice following global targeting of Fortinet firewalls and VPN gateways](https://www.ncsc.gov.uk/news/advice-following-global-targeting-of-fortinet-firewalls-and-vpn-gateways) — UK national authority advisory
- [Cyber.gov.au (ACSC) — Reported widespread credential exposure affecting Fortinet Firewalls and VPN Gateways](https://www.cyber.gov.au/about-us/view-all-content/Reported-widespread-credential-exposure-affecting-Fortinet-Firewalls-and-VPN-Gateways) — Australian national authority advisory
- [Arctic Wolf — Active FortiBleed Campaign Impacting Fortinet Devices Across 194 Countries](https://arcticwolf.com/resources/blog/active-fortibleed-campaign-impacting-fortinet-devices-across-194-countries/) — technical analysis and remediation guidance
- SOCRadar — original threat infrastructure discovery and credential-count reporting
- Hudson Rock / Kevin Beaumont (DoublePulsar) — independent device-count estimation and FortiBleed Checker tool
- [MITRE ATT&CK](https://attack.mitre.org) — technique reference

---
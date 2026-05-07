# Vercel OAuth Supply Chain Breach — 2026-04-19

> **Weekly Breach Investigation**
> *Week 01 · Supply Chain · OAuth Abuse · Financially Motivated*

---

## Executive Summary

An attacker infected a Context.ai employee with **Lumma Stealer** malware via a malicious download in February 2026, then used stolen OAuth tokens to hijack a Vercel employee's Google Workspace account and pivot into Vercel's internal systems.

A limited subset of Vercel customers whose non-sensitive environment variables were stored in plaintext were affected, along with any downstream systems whose credentials appeared in those variables.

Internal database contents and customer environment variables were exfiltrated and listed for sale on **BreachForums** by a threat actor using the ShinyHunters persona, with an asking price of **$2 million** alongside a direct ransom demand.

---

## Attack Timeline

| Date | Event |
|---|---|
| **Feb 2026** | Context.ai employee downloads Roblox game exploit scripts → Lumma Stealer installs silently, harvesting credentials, AWS keys, and browser tokens |
| **Feb–Mar 2026** | Attacker accesses Context.ai's AWS environment using stolen credentials. OAuth tokens belonging to Context.ai users — including a Vercel employee — exfiltrated |
| **Mar 2026** | Context.ai detects unauthorized AWS access and stops it — but does not identify the OAuth token exfiltration. Chrome extension subsequently removed from Chrome Marketplace |
| **Mar–Apr 2026** | Attacker uses stolen OAuth token to take over Vercel employee's Google Workspace account (employee had granted "Allow All" scopes to Context.ai's OAuth app). Pivot into Vercel internal systems — non-sensitive environment variables enumerated via Vercel product API and exfiltrated |
| **Apr 19, 2026** | Vercel publishes security bulletin. CEO Guillermo Rauch confirms attack chain on X. ShinyHunters persona posts stolen database on BreachForums for $2M. Ransom demand of $2M also made directly to Vercel |

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Initial Access | Supply Chain Compromise (via Context.ai) | [T1195](https://attack.mitre.org/techniques/T1195/) |
| Initial Access | Valid Accounts (stolen OAuth token) | [T1078](https://attack.mitre.org/techniques/T1078/) |
| Execution | User Execution (employee downloads malicious script) | [T1204](https://attack.mitre.org/techniques/T1204/) |
| Credential Access | Steal Web Session Cookie / OAuth Token | [T1539](https://attack.mitre.org/techniques/T1539/) |
| Credential Access | Unsecured Credentials (plaintext env vars) | [T1552](https://attack.mitre.org/techniques/T1552/) |
| Lateral Movement | Use Alternate Authentication Material (OAuth token) | [T1550](https://attack.mitre.org/techniques/T1550/) |
| Discovery | Cloud Service Discovery (Vercel API enumeration) | [T1526](https://attack.mitre.org/techniques/T1526/) |
| Exfiltration | Transfer Data to Cloud Account | [T1537](https://attack.mitre.org/techniques/T1537/) |
| Impact | Financial Theft / Extortion ($2M demand + sale) | [T1657](https://attack.mitre.org/techniques/T1657/) |

---

## Detection Opportunities

### Log Sources
- **Google Workspace Admin Audit Logs** — OAuth app grant events and login events using non-standard client IDs
- **Vercel access logs** — bulk environment variable read events from service accounts
- **EDR / Windows Event Log ID 4688** — process creation on Context.ai employee endpoint

### Detection Rules

```
# Rule 1 — Unauthorized OAuth app grant
ALERT when:
  event = OAuth grant
  AND scope includes "drive" OR "mail"
  AND app_id NOT IN approved_allowlist
→ block + alert immediately

# Rule 2 — Bulk env var enumeration
ALERT when:
  Vercel API env var read events exceed baseline
  within any 10-minute window

# Rule 3 — Infostealer behaviour (endpoint)
ALERT when:
  process reads browser credential store
  AND process is NOT a browser binary
  AND outbound network connection follows within 60s
```

### IOCs

| Type | Value |
|---|---|
| OAuth App Client ID | `110671459871-30f1spbu0hptbs60cb4vsmv79i7bbvqj.apps.googleusercontent.com` |
| Threat actor persona | ShinyHunters *(actual group denied involvement — persona may be misappropriated)* |
| Sale venue | BreachForums |
| Malware family | Lumma Stealer |

---

## Recommended Mitigations

1. **Audit and revoke unknown OAuth app grants immediately**
   In Google Workspace Admin Console, review every third-party app authorization. Revoke any app that is unknown, unused, or holds excessive scopes (Drive, Mail). Check specifically for the IOC client ID above.

2. **Enforce OAuth app allowlisting in Google Workspace**
   Require admin approval before any OAuth app can request access — regardless of user role. This would have blocked the Context.ai "Allow All" grant at the source.

3. **Mark all credentials as sensitive in Vercel and equivalent platforms**
   Non-sensitive variables are stored in plaintext and readable to any actor with internal access. Review every environment variable and encrypt regardless of perceived sensitivity level.

4. **Rotate all secrets stored in connected platform env vars**
   Any organisation that used the Context.ai OAuth app should rotate all secrets stored as environment variables on any connected platform — treat them as compromised.

5. **Deploy infostealer endpoint defences**
   Enforce application allowlisting and endpoint detection that flags infostealer behavioural patterns — credential vault access, browser data exfiltration, rapid outbound connections following credential reads.

6. **Conduct third-party SaaS security reviews before onboarding AI tools**
   Every OAuth token granted to an AI productivity tool is a potential lateral movement path. Require security review for all tools requesting enterprise Workspace scopes before employee use is permitted.

---

## Analyst Notes

**What I learned**
OAuth is the new lateral movement vector. This attack required zero novel exploit code — just a stolen token and knowledge of an API surface. The attacker didn't break into Vercel; they walked in through a door a Vercel employee propped open by granting broad permissions to a third-party AI tool. Supply chain trust is now one of the most critical attack surfaces in cloud security.

**What surprised me**
Hudson Rock possessed the compromised credential data from Context.ai over a month before Vercel confirmed the breach. Threat intelligence vendors had visibility into the compromise while victim organisations were still unaware — a ~2 month dwell time that no internal detection caught. The gap between infostealer infection and breach discovery is consistently larger than defenders expect.

---

## References

- [Vercel Security Bulletin](https://vercel.com/security) — official vendor disclosure

- [BleepingComputer](https://bleepingcomputer.com) — breach narrative and timeline
- [Hudson Rock Research](https://hudsonrock.com/research) — infostealer credential intelligence
- [MITRE ATT&CK](https://attack.mitre.org) — technique reference

---
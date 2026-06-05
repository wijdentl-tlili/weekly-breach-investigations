# JINX-0164: Fake Recruiter Lures, AUDIOFIX macOS RAT, and CI/CD Hijacking — 2026-05-27

> **Weekly Breach Investigation**
> *Week 04 · Threat Actor Profile · Social Engineering · macOS Malware · CI/CD Supply Chain · Cryptocurrency Theft*

---

## 1. Executive Summary

**JINX-0164** is a previously undocumented, financially motivated threat actor — first reported by Wiz Research on May 27, 2026 — that has been systematically targeting cryptocurrency organisations since at least mid-2025 using LinkedIn-based fake recruiter social engineering, a custom Python-based macOS RAT named **AUDIOFIX**, and deep targeting of CI/CD infrastructure to propagate malware via compromised code repositories. In a landmark early-2026 intrusion, the actor progressed from a single LinkedIn message to full CI/CD compromise within two weeks, exfiltrating cryptocurrency wallet credentials, cloud API keys, and active session tokens across Discord, Slack, and Telegram. In at least one confirmed case, JINX-0164 extended the attack to a supply chain compromise — publishing a trojanised version of the `@velora-dex/sdk` npm package — turning compromised development infrastructure into a propagation vector targeting downstream users.

---

## 2. Attack Timeline

| Date | Event |
|---|---|
| **Mid-2025** | JINX-0164 assessed as first active; earliest linked campaigns begin targeting cryptocurrency developers |
| **Late 2025 – Early 2026** | StepSecurity and iru independently report separate incidents later attributed to JINX-0164 (trojanised `@velora-dex/sdk` npm package; MiniRAT campaign) |
| **Feb 2026** | Reddit user publicly reports receiving a fake BitGet recruiter approach via LinkedIn — one of several linked social engineering incidents |
| **Early 2026 (landmark intrusion)** | JINX-0164 contacts crypto developer via LinkedIn impersonating a business partner → virtual meeting invite → AUDIOFIX downloaded and executed → two-week progression from initial access to CI/CD compromise |
| **May 27, 2026** | Wiz CIRT and Wiz Research publish full attribution and technical analysis, naming the cluster JINX-0164 |

> **Dwell time (landmark intrusion): ~2 weeks** from initial LinkedIn contact to CI/CD compromise. The actor operated at a deliberate, patient pace — building trust before deploying malware.

---

## 3. How the Attack Worked

JINX-0164 operated across five distinct stages, each building on the last. The entire chain required no CVE exploitation — only social trust, a macOS RAT, and knowledge of developer workflows.

```
STAGE 1 — SOCIAL ENGINEERING (LinkedIn)
────────────────────────────────────────
Attacker creates / hijacks credible LinkedIn profile
  - Established connections, relevant employment history, crypto industry alignment
  - Poses as business partner or recruiter (e.g. impersonating BitGet)
  - Proposes virtual meeting → sends invite link

STAGE 2 — MALWARE DELIVERY (fake meeting platform)
────────────────────────────────────────────────────
Invite links to lookalike domain mimicking Microsoft Teams / Slack / Aircall
  - Victim clicks → prompted to download "meeting client"
  - Bash script executes → downloads AUDIOFIX from:
    apple.driver-store[.]com
  - AUDIOFIX saved as "ChromeUpdater", masquerades as system process "coreaudiod"
  - Executed via launchctl → persistent on macOS
  - C2 communication over HTTPS
  - XOR-encoded password phishing drops result in ~/.zsh_cache

STAGE 3 — CREDENTIAL HARVEST (endpoint)
────────────────────────────────────────
AUDIOFIX exfiltrates:
  ✗ macOS Keychain files
  ✗ Browser-stored credentials
  ✗ Local admin credentials
  ✗ SSH keys + config files + console history
  ✗ 26 cryptocurrency wallet browser extensions
  ✗ 2 desktop cryptocurrency wallet applications
  ✗ AWS / GCP / Azure keys + Cloudflare API tokens
  ✗ GitHub tokens
  ✗ Active Discord / Slack / Telegram sessions

STAGE 4 — CI/CD LATERAL MOVEMENT
────────────────────────────────────────
GitHub tokens used to access internal repositories
  - nord-stream (open-source tool) used to exfiltrate GitHub Actions Secrets
  - AUDIOFIX payload injected into internal repos using 3 evasion tactics:
    (a) Developer impersonation: committer name/email fields forged
    (b) Direct to main: pushed to unprotected main branch directly
    (c) Branch hijacking: injected into active branches when main was protected
  - Other developers pulling + building → infected in turn

STAGE 5 — SUPPLY CHAIN IMPACT
────────────────────────────────────────
@velora-dex/sdk npm package trojanised → published to public registry
  → downstream developers installing the package also infected

WHAT WAS AND WAS NOT AFFECTED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
macOS developer endpoints                    → ⚠️ COMPROMISED
Internal GitHub repositories / CI/CD         → ⚠️ COMPROMISED
GitHub Actions Secrets                       → ⚠️ EXFILTRATED
Crypto wallets / browser extensions         → ⚠️ TARGETED
npm ecosystem (@velora-dex/sdk)              → ⚠️ COMPROMISED (supply chain)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Windows infrastructure (no confirmed cases)  → ✅ Not confirmed (infrastructure hints suggest possible future targeting)
Cloud environments (AWS/GCP/Azure directly)  → ✅ Limited — actor pivoted to CI/CD instead of cloud
```

---

## 4. MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Initial Access | Phishing: Spearphishing via Service (LinkedIn) | [T1566.003](https://attack.mitre.org/techniques/T1566/003/) |
| Initial Access | Supply Chain Compromise: Compromise Software Supply Chain (npm) | [T1195.002](https://attack.mitre.org/techniques/T1195/002/) |
| Execution | User Execution: Malicious File | [T1204.002](https://attack.mitre.org/techniques/T1204/002/) |
| Execution | Command and Scripting Interpreter: Unix Shell (bash dropper) | [T1059.004](https://attack.mitre.org/techniques/T1059/004/) |
| Persistence | Create or Modify System Process: Launch Daemon (launchctl) | [T1543.004](https://attack.mitre.org/techniques/T1543/004/) |
| Defense Evasion | Masquerading: Match Legitimate Name or Location (coreaudiod / ChromeUpdater) | [T1036.005](https://attack.mitre.org/techniques/T1036/005/) |
| Defense Evasion | Obfuscated Files or Information (XOR encoding) | [T1027](https://attack.mitre.org/techniques/T1027/) |
| Defense Evasion | Forge Web Credentials: Web Cookies (session token theft) | [T1606.001](https://attack.mitre.org/techniques/T1606/001/) |
| Credential Access | Credentials from Password Stores: Keychain (macOS Keychain) | [T1555.001](https://attack.mitre.org/techniques/T1555/001/) |
| Credential Access | Credentials from Password Stores: Credentials from Web Browsers | [T1555.003](https://attack.mitre.org/techniques/T1555/003/) |
| Credential Access | Unsecured Credentials: Private Keys (SSH keys) | [T1552.004](https://attack.mitre.org/techniques/T1552/004/) |
| Discovery | Cloud Service Discovery (AWS/GCP/Azure/GitHub token harvest) | [T1526](https://attack.mitre.org/techniques/T1526/) |
| Lateral Movement | Use Alternate Authentication Material: Application Access Token (GitHub token) | [T1550.001](https://attack.mitre.org/techniques/T1550/001/) |
| Lateral Movement | Internal Spearphishing via compromised repo commits | [T1534](https://attack.mitre.org/techniques/T1534/) |
| Collection | Data from Local System | [T1005](https://attack.mitre.org/techniques/T1005/) |
| Collection | Email Collection / Messaging (Discord, Slack, Telegram sessions) | [T1114](https://attack.mitre.org/techniques/T1114/) |
| Command & Control | Application Layer Protocol: Web Protocols (HTTPS C2) | [T1071.001](https://attack.mitre.org/techniques/T1071/001/) |
| Command & Control | Proxy: Multi-hop Proxy (Mullvad / Astrill / ExpressVPN) | [T1090.003](https://attack.mitre.org/techniques/T1090/003/) |
| Impact | Financial Theft (cryptocurrency wallet exfiltration) | [T1657](https://attack.mitre.org/techniques/T1657/) |

---

## 5. Detection Opportunities

### Log Sources

- **LinkedIn / HR / recruitment logs** — unsolicited outreach from unknown accounts offering virtual meetings; accounts deleted shortly after contact
- **macOS Unified Log / Endpoint telemetry** — `launchctl` loading new agents from unexpected paths; process named `coreaudiod` or `ChromeUpdater` not matching known system binary hashes; bash scripts downloading from external domains
- **Network / proxy logs** — DNS resolution or HTTPS connections to `apple.driver-store[.]com` or similar driver-store spoofing domains; connections routed through Mullvad, Astrill, or ExpressVPN exit nodes
- **GitHub audit logs** — `git push` events from previously inactive or unexpected IP addresses; commit author / committer email mismatch (unsigned or `unverified` badge); GitHub Actions Secrets accessed by non-standard workflows
- **macOS Keychain access logs** — non-system processes reading Keychain entries; `~/.zsh_cache` file creation (XOR-encoded credential staging)
- **CI/CD pipeline logs** — nord-stream tool signatures; unexpected workflow runs accessing secrets

### IOCs

| Type | Value |
|---|---|
| Threat actor | JINX-0164 (Wiz Research designation, first reported May 27, 2026) |
| Malware family | AUDIOFIX (Python-based macOS RAT + infostealer) |
| Malware masquerade names | `coreaudiod`, `ChromeUpdater` |
| Malware execution method | `launchctl` (macOS LaunchAgent persistence) |
| C2 dropper domain | `apple.driver-store[.]com` |
| Potential Windows infrastructure | `windows.driver-store[.]com` (observed, no confirmed Windows victims) |
| Credential staging path | `~/.zsh_cache` (XOR-encoded) |
| CI/CD exfil tool | `nord-stream` (open-source, used legitimately; presence is suspicious in this context) |
| Trojanised npm package | `@velora-dex/sdk` (supply chain vector) |
| VPN providers used by actor | Mullvad VPN · Astrill VPN · ExpressVPN |
| Cryptocurrency wallets targeted | 26 browser extension wallets + 2 desktop wallet applications |
| Cloud credentials targeted | AWS · GCP · Azure · Cloudflare API tokens · GitHub tokens |
| Communication sessions targeted | Discord · Slack · Telegram |
| Sector targeted | Cryptocurrency organisations / software developers |
| Attribution confidence | Financially motivated, likely state-nexus (North Korean TTPs noted, no confirmed attribution) |

---

## 6. Recommended Mitigations

1. **Enable GitHub Vigilant Mode organisation-wide immediately**
   Vigilant Mode flags commits where the GPG signing key owner does not match the listed commit author — this is exactly what caught the developer impersonation in the landmark intrusion. It is off by default. Enable it at the organisation level in GitHub settings → Code security and analysis.

2. **Require signed commits and branch protection rules on all repositories**
   Enforce GPG-signed commits on `main` and all release branches. Combine with branch protection rules requiring at least one reviewer approval and status checks before merge. This eliminates the "direct to main" and "branch hijacking" lateral movement paths JINX-0164 used.

3. **Restrict GitHub Actions Secrets access and audit all workflow permissions**
   Limit which workflows can access repository and organisation secrets. Audit all `GITHUB_TOKEN` permission scopes — default permissions should be read-only. Review recent workflow runs for any access to secrets from unexpected actors or IPs.

4. **Establish a recruiter verification protocol for employees**
   JINX-0164 specifically targets developers via LinkedIn. Brief all technical staff: any unsolicited virtual meeting from an external contact, especially one using a third-party conferencing link rather than a company-hosted link, must be verified through an independent channel before the link is clicked. Never install software prompted by a meeting invite from an unverified contact.

5. **Monitor for macOS LaunchAgent persistence outside known-good paths**
   Deploy macOS EDR or Endpoint Security Framework (ESF) rules alerting on any `launchctl load` event where the plist path is not in a known-good system directory. AUDIOFIX persists via this mechanism — catching the persistence event is the earliest reliable detection point.

6. **Treat VPN exit node logins to developer tooling as high-risk signals**
   Log the source IPs for all GitHub, AWS, GCP, and cloud console logins. Cross-reference against known VPN exit node IP ranges (Mullvad, Astrill, ExpressVPN publish these). An authenticated session to a developer platform from a VPN exit node not on a corporate VPN allowlist should trigger a session review immediately.

---

## 7. Analyst Notes

**What I learned**

JINX-0164 understood that the real value in a cryptocurrency developer's laptop is not the laptop — it's the keys. Every credential, token, and session on that machine is a door into something more valuable: the code, the pipelines, the wallets. The attack path from LinkedIn message to CI/CD compromise in two weeks shows a group that has mapped the developer workflow and knows exactly which pivot to make at each stage. The use of nord-stream to automate GitHub Actions Secret exfiltration is particularly notable — this is a red team tool being used operationally. The actor is technically capable and operationally patient.

**What surprised me**

The actor's restraint in the cloud environment. After harvesting AWS, GCP, and Azure keys, JINX-0164 made limited use of cloud resources — no widespread enumeration, no ransomware, no data destruction. They went straight for the development infrastructure. This suggests the actor understands that cloud activity generates high-fidelity alerts in mature environments, while repository commits (especially in less mature organisations) often go unmonitored for weeks. They chose the stealthier path, not the most obvious one.

---

## 8. Who Is at Risk — Quick Check

| Condition | At risk? |
|---|---|
| macOS developer at a cryptocurrency / blockchain organisation | ✅ Yes — primary target profile; apply all mitigations |
| Received unsolicited LinkedIn meeting invite from crypto-adjacent recruiter | ⚠️ Verify the contact independently before clicking any link |
| Installed software from a virtual meeting invite link (external domain) | ✅ Yes — assume AUDIOFIX; check for ChromeUpdater / coreaudiod processes and ~/.zsh_cache |
| Used or installed `@velora-dex/sdk` npm package (any version from late 2025) | ✅ Yes — supply chain vector; rotate all secrets from that environment |
| Developer who pulled / built from internal repos during the compromise window | ⚠️ Potentially — check for unverified commits in your build history |
| Windows-based developer at a crypto org | ⚠️ No confirmed victims but actor infrastructure (`windows.driver-store[.]com`) suggests active development of Windows capability |
| Not in cryptocurrency / blockchain sector | ✅ Lower risk — JINX-0164 is sector-focused; not a broad campaign |

---

## 9. References

- [Wiz Research — Commit to Compromise: A New Threat Actor Targeting the Cryptocurrency Industry (primary technical source)](https://www.wiz.io/blog/threat-actors-target-crypto-orgs)
- [The Hacker News — JINX-0164 Targets Cryptocurrency Firms with Fake Recruiter Lures and macOS Malware](https://thehackernews.com/2026/05/jinx-0164-targets-cryptocurrency-firms.html)
- [Infosecurity Magazine — New Threat Actor Jinx-0164 Targets Crypto Developers on macOS](https://www.infosecurity-magazine.com/news/jinx-0164-crypto-developers-macos/)
- [StepSecurity — @velora-dex/sdk compromise disclosure](https://www.stepsecurity.io/blog/velora-dex-sdk-compromised-on-npm-malicious-version-drops-macos-backdoor-via-launchctl-persistence)
- [MITRE ATT&CK](https://attack.mitre.org) — Technique reference

---

<sub>Part of the <strong>Weekly Breach Investigation</strong> series · Investigating one real-world breach per week to build practical SOC analyst skills · <a href="https://attack.mitre.org">attack.mitre.org</a></sub>
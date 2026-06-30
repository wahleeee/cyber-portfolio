# SOC Analyst Training — SIEM Alerts & Phishing Email Investigation

Blue-team training notes and a reusable **phishing-alert investigation playbook**, distilled from four foundational [LetsDefend](https://app.letsdefend.io/) courses. This repo documents both the **core concepts** behind SOC operations and the **end-to-end workflow** a Tier-1 analyst follows when a phishing alert fires in a SIEM — from taking ownership of the alert, through IOC analysis, to containment and closing the case.

---

## Trainings completed

| Course | Level | Focus | Reference |
|---|---|---|---|
| **SOC Fundamentals** | Beginner | How a SOC operates: roles, SIEM/EDR/SOAR, log management, threat intel | [link](https://app.letsdefend.io/training/lessons/soc-fundamentals) |
| **Introduction to SIEM Alerts** | Beginner | Anatomy of a SIEM alert and the analyst triage loop | [link](https://app.letsdefend.io/training/lesson_detail/introduction-to-siem-alerts-8c12a41d) |
| **VirusTotal for SOC Analysts** | Medium | File/URL reputation analysis and IOC pivoting | [link](https://app.letsdefend.io/training/lessons/virustotal-for-soc-analysts) |
| **Phishing Email Analysis** | Beginner | Header, static, and dynamic analysis of suspicious email | [link](https://app.letsdefend.io/training/lessons/phishing-email-analysis) |

**Modules covered across the courses:** Introduction to SOC · SOC types & roles · analyst responsibilities · the SIEM–analyst relationship · log management · EDR · SOAR · threat intelligence feeds · common analyst mistakes · file/URL analysis with VirusTotal · IOC searching · email header analysis · static analysis · dynamic analysis.

---

## Core concepts from the training

**Log management** — Collecting logs from across the environment (web server, OS, firewall, proxy, EDR, and more) into a single place so events can be searched and correlated. *You can't detect what you don't collect — this is the foundation everything else sits on.*

**SIEM (Security Information & Event Management)** — A security solution that performs **real-time logging and correlation of events** across the environment for the purpose of detecting security threats. SIEM products do a lot, but the features that matter most to a SOC analyst are the ones that **collect and filter data and raise alerts** on suspicious activity.

**SOAR (Security Orchestration, Automation & Response)** — Connects an environment's security tools so they work together and **automates repetitive analyst tasks**. Example: automatically looking up the source IP of a SIEM alert on VirusTotal so the analyst doesn't have to do it by hand — cutting down workload and response time.

**Threat intelligence feed** — A stream of known-bad indicators (malware **hashes**, **C2 domains/IPs**) used to enrich and validate alerts. Sources include VirusTotal and Cisco Talos Intelligence.

**Endpoint telemetry — Sysmon (Sysinternals)** — A Windows tool that generates detailed endpoint event logs (process creation, network connections, file/registry changes). That telemetry feeds the SIEM and gives analysts the granularity needed to reconstruct what actually happened on a host. *(Closely related: EDR provides detection and response at the endpoint.)*

**Wireshark** — Packet-capture and deep-packet-inspection tool for analyzing network traffic at the protocol level during an investigation.

---

## Common SOC analyst mistakes

Pitfalls to actively guard against (from the *Common Mistakes* module). Knowing these is the difference between running a checklist and actually reasoning about a case:

- **Over-reliance on VirusTotal results** — treating a detection score as the final verdict instead of one input. A low or zero score does **not** prove a file is safe.
- **Hasty analysis of malware in a sandbox** — cutting detonation short and missing delayed or conditional behavior (sleep timers, geofencing, behavior that only triggers on user interaction).
- **Inadequate log analysis** — failing to dig into the logs to confirm what truly happened, e.g. whether a user actually clicked a link or a process actually executed.
- **Overlooking VirusTotal dates** — ignoring the *first-seen* / *last-analysis* dates. A sample "first seen today" tells a very different story from one that's been known for years, and stale results can mislead.

---

## Phishing alert investigation playbook

The workflow follows the standard SOC triage loop:

> **Detect → Take ownership → Analyze → Determine impact → Contain → Document → Close**

Each step lists *what to do* and *why it matters*.

### 1. Take ownership of the alert
Claim the alert in the investigation/monitoring channel before doing anything else.
*Why:* it prevents duplicate work and creates a clear chain of custody. From here on, the alert is yours until you reach a verdict.

### 2. Open and read the email
Click into the email and review the **body, sender information, subject, and any attachments**; scroll down to confirm whether attachments are present.
*Why:* the body and sender are your first read on intent — urgency, threats, payment/credential requests, and lookalike sender names are classic phishing tells. This is the start of **static analysis** (observing, not executing).

### 3. Parse the email — extract the facts
Pull the core metadata into your case notes by answering:

- [ ] **When was it sent?** (timestamp)
- [ ] **What is the originating SMTP address / sending mail server?**
- [ ] **What is the sender (From) address?**
- [ ] **What is the recipient address?**
- [ ] **Is the message content suspicious?** (tone, intent, lures)
- [ ] **Are there any attachments or URLs to analyze?**

*Why:* these answers are the backbone of the case. The **header / SMTP details** matter because the display name and the actual sending server often disagree in spoofed mail — comparing the originating IP, envelope sender, and `From` address is how you separate a legitimate sender from an impersonation.

### 4. Analyze the IOCs (URLs and attachments)
Analyze any URLs or attachments using trusted threat-intel and sandbox tooling (see [Tools & technologies](#tools--technologies)). **Cross-check across at least two sources** rather than trusting a single score.

For **password-protected attachments**, detonate inside an **isolated analysis VM / sandbox** — never on your host. The archive password is usually in the email body (an evasion trick to slip past mail-gateway scanning). Extract it in the VM, hash the file, and submit the hash/sample.

*Why:* this is **dynamic analysis** — you confirm malicious behavior by observing detonation (network callbacks, dropped files, persistence) instead of guessing. Letting the sandbox run long enough avoids the "hasty sandbox analysis" trap.

> ⚠️ **Safety rule:** treat every attachment and link as live. Detonation happens only in a sandbox you can throw away.

### 5. Determine whether the email reached the user
Return to the **Alert Details** page and find the **Device Action** section — what the security stack (mail gateway / EDR) did with the message. Look for a delivery status such as **Allowed**, **Quarantined**, or **Deleted**, then answer the playbook question:
- Reached the inbox → **Delivered**
- Blocked, quarantined, or removed in transit → **Not Delivered**

*Why:* delivery status defines the **blast radius**. "Delivered" means a user *could* have interacted with it; "Not Delivered" means the controls already contained it.

### 6. Assess impact — did the user actually interact?
If the mail was delivered, check the logs to see whether the user **clicked the URL or opened the attachment** (proxy logs, EDR/endpoint telemetry, mail logs).
*Why:* this is the line between *exposed* and *compromised*. Delivered ≠ clicked ≠ executed — and skipping this check is exactly the "inadequate log analysis" mistake. Confirmed interaction escalates the case toward incident response.

### 7. Contain — remove the email from the recipient
If delivered, purge it from the mailbox:
1. Go to the **Email Security** tab where the message was found.
2. **Search** for the specific malicious email.
3. Open it and click **Delete** (top-right of the panel) to remove it from the recipient's mailbox.

*Why:* containment removes the threat from reach. If the campaign hit multiple users, repeat for every affected mailbox.

### 8. Document and close
Record the verdict (**True Positive** vs **False Positive**), the supporting evidence (parsed headers, IOC results, delivery status, interaction findings, containment actions), and escalate to incident response if a click, credential entry, or execution is confirmed.
*Why:* clean documentation turns one alert into reusable detection logic and lets the next analyst or an auditor reconstruct exactly what happened and why.

---

## Tools & technologies

| Tool | Category | What it's used for |
|---|---|---|
| [Sysmon (Sysinternals)](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) | Endpoint telemetry | Detailed Windows event logs (process, network, file) that feed the SIEM |
| [Wireshark](https://www.wireshark.org/) | Network analysis | Inspect captured network traffic at the protocol level |
| [ANY.RUN](https://any.run/) | Sandbox (dynamic) | Interactive detonation — watch processes, network, and dropped files live |
| [VMRay](https://www.vmray.com/) | Sandbox (dynamic) | Automated malware analysis and behavior reporting |
| [Joe Sandbox](https://www.joesandbox.com/) | Sandbox (dynamic) | Automated deep malware analysis |
| [Hybrid Analysis](https://www.hybrid-analysis.com/) | Sandbox (dynamic) | Free automated analysis (CrowdStrike Falcon Sandbox) |
| [VirusTotal](https://www.virustotal.com/) | Reputation / threat intel | Scan files, URLs, hashes, domains, IPs; pivot on IOCs across engines |
| [Cisco Talos Intelligence](https://talosintelligence.com/) | Reputation / threat intel | Reputation lookups and threat-intel feeds (hashes, domains, IPs) |
| [URLhaus](https://urlhaus.abuse.ch/) | Malicious-URL database | Check whether a URL is a known malware-distribution host (abuse.ch) |
| [urlscan.io](https://urlscan.io/) | URL / web analysis | Render a URL safely; inspect screenshots, DOM, redirects, requests |
| [Browserling](https://www.browserling.com/) | URL / web analysis | Quickly and safely open a suspicious link in a remote browser |

*Reminder from the training: a low detection count is a data point, not a verdict — corroborate across sources and always check first-seen / last-analysis dates before deciding.*

---

## Key takeaways

- **Own the alert first.** Triage discipline (claim → investigate → verdict → close) keeps the SOC queue coherent.
- **Static before dynamic.** Read headers and content safely before you ever detonate anything.
- **Headers don't lie the way display names do.** Comparing the originating SMTP server, envelope sender, and `From` address is the core skill in spotting spoofed mail.
- **Tools are inputs, not verdicts — and check the dates.** VirusTotal/Talos/URLhaus inform the call; the analyst's reasoning makes it.
- **Delivered ≠ clicked ≠ compromised.** Confirm actual impact in the logs; this is what separates an exposure from an incident.
- **Contain in disposable environments.** Sandboxes and VMs are what separate analysis from infection.

---

## References

- LetsDefend — [SOC Fundamentals](https://app.letsdefend.io/training/lessons/soc-fundamentals)
- LetsDefend — [Introduction to SIEM Alerts](https://app.letsdefend.io/training/lesson_detail/introduction-to-siem-alerts-8c12a41d)
- LetsDefend — [VirusTotal for SOC Analysts](https://app.letsdefend.io/training/lessons/virustotal-for-soc-analysts)
- LetsDefend — [Phishing Email Analysis](https://app.letsdefend.io/training/lessons/phishing-email-analysis)

# Wazuh SOC Home Lab — SIEM, IDS/IPS & Firewall on Proxmox

> An end-to-end Security Operations Center (SOC) lab built on **Proxmox VE**, integrating the **Wazuh** SIEM/XDR platform, **Suricata** as an inline IDS/IPS, and a **pfSense** firewall — with full network segmentation, endpoint telemetry, and threat-intelligence enrichment.

![Platform](https://img.shields.io/badge/Platform-Proxmox%20VE-E57000)
![SIEM](https://img.shields.io/badge/SIEM-Wazuh-005792)
![IDS](https://img.shields.io/badge/IDS%2FIPS-Suricata-EE2A24)
![Firewall](https://img.shields.io/badge/Firewall-pfSense-212121)

---

## Table of Contents

- [Overview](#overview)
- [Lab Architecture](#lab-architecture)
- [Network Design](#network-design)
- [Component Build Notes](#component-build-notes)
  - [1. pfSense Firewall](#1-pfsense-firewall)
  - [2. Wazuh Manager](#2-wazuh-manager)
  - [3. Suricata Inline IDS/IPS Sensor](#3-suricata-inline-idsips-sensor)
  - [4. Windows 10 Endpoint](#4-windows-10-endpoint)
- [Integrations](#integrations)
  - [pfSense → Wazuh (Syslog)](#pfsense--wazuh-syslog)
  - [Suricata → Wazuh (EVE JSON)](#suricata--wazuh-eve-json)
  - [VirusTotal Threat Intelligence](#virustotal-threat-intelligence)
  - [File Integrity Monitoring (FIM)](#file-integrity-monitoring-fim)
  - [Sysmon Ingestion](#sysmon-ingestion)
- [Attack Simulation & Detection](#attack-simulation--detection)
- [Engineering Challenges & Lessons Learned](#engineering-challenges--lessons-learned)
- [Skills Demonstrated](#skills-demonstrated)
- [References](#references)

---

## Overview

This project demonstrates the design, deployment, and operation of a functional mini-SOC built entirely in a virtualized homelab. The goal was to replicate the core detection-and-response stack a security analyst works with daily: a **firewall** generating network logs, a **network IDS/IPS** inspecting traffic inline, a **SIEM** correlating events from every source, and **Windows endpoint telemetry** feeding host-based detections.

Unlike a flat "all VMs on one bridge" lab, this build emphasizes **realistic network segmentation** — traffic is forced through a firewall and an inline intrusion-prevention sensor before reaching protected hosts, mirroring how layered defenses are deployed in production environments.

**What this lab does:**

- Collects and correlates logs from agents, Suricata, and pfSense in a single Wazuh dashboard
- Inspects all east-west and north-south traffic to the protected segment via an inline Suricata sensor
- Enriches alerts with VirusTotal reputation data for faster triage
- Monitors Windows endpoints for file integrity changes, authentication events, and process activity
- Detects simulated attacks (e.g., SSH brute force) end-to-end, from network packet to dashboard alert

---

## Lab Architecture

The lab is built on **Proxmox VE** (type-1 hypervisor) and consists of the following components:

| Component | Role | OS / Platform |
|-----------|------|---------------|
| **Wazuh Server** | Central Manager, Indexer, and Dashboard. Collects/correlates logs from agents, Suricata, and pfSense. | Wazuh OVA (Linux) |
| **pfSense Firewall** | Network security gateway. Segments the lab and forwards firewall logs to Wazuh. | pfSense (FreeBSD) |
| **Suricata Sensor** | Inline IDS/IPS bridging two segments. Inspects all traffic and ships alerts to Wazuh. | Linux (Debian/Ubuntu-based) |
| **Windows 10 Endpoint** | Protected host running the Wazuh Agent, Sysmon, and FIM. Acts as the attack target. | Windows 10 |
| **Attacker** | Used to simulate threats (brute force, recon). | Kali Linux |
| **VirusTotal** | External threat-intelligence enrichment via API. | Cloud (API) |

> **Note on platform choice:** This lab was built on **Proxmox** (not VMware) using Linux-bridge networking. Suricata runs on a **dedicated Linux VM in inline mode** rather than on Windows with Npcap — the Linux AF_PACKET path is the industry-standard approach for inline IPS and avoids the fragility of the WinDivert/Npcap stack on Windows.

---

## Network Design

The defining feature of this lab is its **three-bridge segmented topology**. Each Proxmox Linux bridge acts as a virtual switch, and traffic is deliberately routed through security controls before reaching protected assets.

| Bridge | Purpose | Physical NIC | Notes |
|--------|---------|--------------|-------|
| `vmbr0` | **WAN** (uplink) | `nic0` (wired) | Bridged to the physical NIC; reaches the home network / internet. Host management IP lives here (`192.168.0.x`). |
| `vmbr1` | **Transit** (pfSense LAN ↔ sensor) | *none* | Virtual-only bridge. Connects pfSense's LAN interface to the inline sensor's outside leg. |
| `vmbr2` | **Protected LAN** | *none* | Virtual-only bridge. Protected hosts (Win10, Kali target) live here, behind both pfSense and Suricata. |

### Traffic Flow

```
Internet
   │
[ vmbr0 ]  ── WAN ──►  pfSense  ── LAN ──►  [ vmbr1 ]
                                                │
                                    Suricata Inline Sensor
                                    (bridges vmbr1 ↔ vmbr2)
                                                │
                                           [ vmbr2 ]
                                                │
                              ┌─────────────────┼─────────────────┐
                          Win10 Endpoint    Wazuh Agent       Kali (target)
```

**Why this matters:** Because every protected host sits on `vmbr2`, *all* of its traffic must traverse the Suricata inline sensor to reach pfSense and the internet. This means Suricata sees 100% of the traffic to/from the protected segment — true inline inspection, not passive port mirroring. If the sensor drops a malicious packet in IPS mode, it never reaches the target.

### pfSense Interface Assignment

| pfSense Interface | Driver | Bridge | IP |
|-------------------|--------|--------|-----|
| **WAN** | `em0` | `vmbr0` | DHCP from home network (`192.168.0.x`) |
| **LAN** | `em1` | `vmbr1` | Static `192.168.1.1/24` |

> The Intel **E1000** NIC model (driver `em`) was chosen over VirtIO for the pfSense VM to avoid a known FreeBSD/VirtIO DHCP quirk. Interfaces were verified by **MAC address** rather than assuming `em0 = WAN`, to guarantee correct WAN/LAN mapping.

---

## Component Build Notes

### 1. pfSense Firewall

pfSense provides the network boundary and generates firewall logs for Wazuh.

- **WAN (`em0`)** → `vmbr0`, pulls a DHCP lease from the upstream home router.
- **LAN (`em1`)** → `vmbr1`, static `192.168.1.1/24`, serves DHCP to the downstream segment.
- Default credentials changed immediately on first login.
- **Management hardening:** the web GUI is reachable only from the LAN side; no WAN allow-rule is added to the management interface.

> **Design context:** This pfSense sits *behind* the home router (double-NAT), so its WAN is on a private IP. This is intentional and correct for a segmentation-focused lab — the home router remains the true internet edge, and the lab segment is fully isolated from personal devices.

### 2. Wazuh Manager

Deployed from the official Wazuh **OVA**, running the Manager, Indexer, and Dashboard on a single VM.

- Accessed via `https://<wazuh-ip>` after confirming all three services are active:
  ```bash
  systemctl status wazuh-manager
  systemctl status wazuh-indexer
  systemctl status wazuh-dashboard
  ```
- Lives on the protected segment so all agents reach it over the LAN.

### 3. Suricata Inline IDS/IPS Sensor

A dedicated Linux VM acts as a **bump-in-the-wire** between `vmbr1` and `vmbr2`.

**Two NICs:**
- `ens18` → `vmbr1` (outside leg, toward pfSense)
- `ens19` → `vmbr2` (inside leg, toward protected hosts)

**Install (Debian/Ubuntu-based):**
```bash
sudo apt update
sudo apt install -y suricata
suricata -V
```

> **No Npcap required.** Linux provides packet capture natively in the kernel (AF_PACKET / NFQUEUE). The Npcap dependency only exists on the Windows port of Suricata.

**Key configuration fixes applied:**
- The default `suricata.yaml` ships with `interface: eth0`, which does not exist on a Proxmox VM. All references were updated to the real interface:
  ```bash
  sudo sed -i 's/eth0/ens18/g' /etc/suricata/suricata.yaml
  ```
- Ruleset pulled and config validated before starting:
  ```bash
  sudo suricata-update
  sudo suricata -T -c /etc/suricata/suricata.yaml -v   # ~50,000 rules loaded
  sudo systemctl reset-failed suricata
  sudo systemctl restart suricata
  ```

**Operating mode:** Started in **IDS (alert-only)** mode first to confirm the bridge passes traffic and alerts are generated, *before* flipping to inline IPS/drop mode. This avoids stranding the protected segment behind a misconfigured drop policy.

### 4. Windows 10 Endpoint

A Windows 10 VM on `vmbr2` serves as the protected host and attack target.

- Runs the **Wazuh Agent**, **Sysmon**, and **File Integrity Monitoring**.
- Pulls a `192.168.1.x` lease from pfSense and routes out through the Suricata sensor.
- Because it sits behind the inline sensor, it yields **dual telemetry**: host-based events (Wazuh agent) *and* network-based detections (Suricata) for the same activity — exactly the cross-source correlation a SOC analyst performs.

---

## Integrations

### pfSense → Wazuh (Syslog)

**On pfSense:** `Status → System Logs → Settings` → enable **Remote Logging**, point at the Wazuh Manager IP, port `514`, forwarding System / Firewall / DHCP / DNS categories.

**On the Wazuh Manager** (`/var/ossec/etc/ossec.conf`):
```xml
<ossec_config>
  <remote>
    <connection>syslog</connection>
    <port>514</port>
    <protocol>udp</protocol>
    <allowed-ips>YOUR_PFSENSE_IP</allowed-ips>
    <local_ip>YOUR_WAZUH_SERVER_IP</local_ip>
  </remote>
</ossec_config>
```

**Custom decoder** (`/var/ossec/etc/decoders/pfsense-custom-decoder.xml`) — extracts source/dest IP, protocol, and ports from pfSense's `filterlog` CSV format.

**Custom rules** (`/var/ossec/etc/rules/pfsense-custom-rules.xml`) — assign severity by event type:

| Rule ID | Level | Trigger |
|---------|-------|---------|
| `100111` | 3 (Info) | Generic pfSense log received |
| `100112` | 3 (Info) | Successful login |
| `100113` | 5 (Notice) | Allowed traffic |
| `100114` | 7 (Warning) | Blocked traffic |
| `100115` | 10 (High) | Authentication error |

> Custom rules/decoders go in `/var/ossec/etc/` (survives upgrades), **never** in `/var/ossec/ruleset/` (overwritten on update). Custom rule IDs use the `100000+` range to avoid collisions with built-ins.

### Suricata → Wazuh (EVE JSON)

The Wazuh agent on the sensor reads Suricata's EVE output. In the agent's `ossec.conf`:
```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```
Wazuh has a built-in decoder for Suricata's JSON, so alert fields (`data.alert.signature`, `data.src_ip`, etc.) are parsed automatically. Restart with `sudo systemctl restart wazuh-agent`.

**Verify in dashboard:** *Threat Hunting* → filter `data.alert.signature` **exists**, or `rule.groups` **is** `ids`.

### VirusTotal Threat Intelligence

Enriches alerts with multi-engine reputation data. Configured on the **Wazuh Manager** (integrations run on the manager, not the agent):
```xml
<integration>
  <name>virustotal</name>
  <api_key>YOUR_API_KEY</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```
Paired with a monitored directory in `<syscheck>` so that any new file triggers an automatic hash lookup against VirusTotal.

### File Integrity Monitoring (FIM)

Tracks file/directory changes in real time. Within the **existing** `<syscheck>` block on the Windows agent (`C:\Program Files (x86)\ossec-agent\ossec.conf`):
```xml
<directories check_all="yes" report_changes="yes" realtime="yes">C:\Users\<user>\Desktop</directories>
```
Validated by creating, modifying, and deleting a test file — each action produced a distinct Wazuh alert (added → level 5, modified/deleted → level 7).

> **Lesson learned:** Adding a *second* `<syscheck>` block (instead of adding directories inside the existing one) is a common cause of agent startup failure. FIM config is OS-specific — Windows paths and `<windows_registry>` keys go on the **Windows** agent; Linux paths go on Linux agents.

### Sysmon Ingestion

Sysmon provides deep process, network, and registry telemetry on Windows. After installing Sysmon with a configuration (e.g., SwiftOnSecurity's config), the agent ingests its event channel:
```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

---

## Attack Simulation & Detection

**Scenario: SSH Brute Force** (Kali → Windows OpenSSH Server)

```bash
# On the attacker (Kali)
hydra -l <user> -P passwords.txt ssh://<target_ip>
```

**Detection in Wazuh** (*Threat Hunting*):
- Multiple **failed logon** events surfaced as **Event ID 4625**
- Mapped by Wazuh to severity level 5, with a rule firing repeatedly (indicating sustained activity, not an isolated failure)
- Key fields parsed: `data.win.system.eventID = 4625`, `logonType = 8` (NetworkCleartext, typical of SSHD), `subStatus = 0xc0000064` (unknown username / bad password), `processName = sshd.exe`

This demonstrates the full pipeline: an attack on the network is observed by Suricata (network layer) *and* logged by the Windows host (endpoint layer), then correlated and surfaced in a single SIEM view.

**Defensive countermeasures documented:** strong/unique passwords, MFA, login-activity monitoring, and rate-limiting / account lockout after repeated failures.

---

## Engineering Challenges & Lessons Learned

This section captures the real troubleshooting work behind the build — the part that doesn't show up in a clean architecture diagram but reflects actual operational skill.

**1. pfSense WAN would not pull a DHCP lease.**
Symptom: WAN interface "active" but no IPv4 address; Netgate connectivity check looped the installer. Root cause was a combination of (a) a **leftover third vNIC** attached to the wrong bridge, causing pfSense to auto-assign WAN to a dead-end bridge, and (b) the VirtIO NIC model misbehaving with FreeBSD. Resolved by removing the stray NIC, switching to the **Intel E1000** model, and verifying WAN/LAN mapping by **MAC address**. Diagnosed methodically using `ifconfig`, `dhclient -d`, and confirming the Proxmox host itself could reach the internet on the same physical NIC.

**2. Accidentally bridging a Wi-Fi NIC.**
An early attempt attached `vmbr1` to the wireless NIC. Wi-Fi NICs generally **cannot be bridged** (the AP drops frames from VM MAC addresses), which silently broke connectivity. Lesson: only wired NICs are viable for VM bridging; the LAN bridge should be virtual-only.

**3. Suricata service crash-looping on startup.**
`systemd` reported "start request repeated too quickly." The underlying error (`af-packet: eth0: No such device`) was a hardcoded default interface that didn't match the VM's real NIC names. Fixed by updating `suricata.yaml` and clearing the failed state with `systemctl reset-failed`.

**4. Distinguishing "no data" from "wrong filter" in the dashboard.**
An empty dashboard view looked identical whether the data pipeline was broken or the filter was simply wrong. Learned to verify end-to-end: confirm `event_type:alert` lines exist in `eve.json` on the sensor first, *then* tune the dashboard filter — rather than assuming the integration was broken.

**5. XML config discipline.**
Multiple agent-startup failures traced back to malformed `ossec.conf` — a duplicate `<syscheck>` block and pasted content with mismatched tags. Adopted a workflow of validating with `xmllint --noout` before every service restart.

> **Takeaway:** The most valuable part of this lab wasn't the happy path — it was learning to isolate faults across the hypervisor, firewall, IDS, and SIEM layers using the right diagnostic tool at each level.

---

## Skills Demonstrated

- **SIEM administration** — Wazuh Manager/Indexer/Dashboard deployment, custom decoders & rules, multi-source log correlation
- **Network IDS/IPS** — Suricata inline (AF_PACKET) deployment, ruleset management, IDS→IPS progression
- **Firewall & network segmentation** — pfSense configuration, multi-bridge topology, double-NAT design, management-plane hardening
- **Endpoint security** — Wazuh agent enrollment, File Integrity Monitoring, Sysmon telemetry, Windows event analysis
- **Threat intelligence** — VirusTotal API enrichment for alert triage
- **Detection engineering** — attack simulation (SSH brute force) and mapping observed telemetry (Event ID 4625) to detections
- **Virtualization & troubleshooting** — Proxmox VM/networking administration, systematic fault isolation across the stack

---

## References

- [Wazuh Documentation](https://documentation.wazuh.com/)
- [Suricata Documentation](https://docs.suricata.io/)
- [pfSense Documentation](https://docs.netgate.com/pfsense/)
- Project repository: [github.com/wahleeee/cyber-portfolio](https://github.com/wahleeee/cyber-portfolio)

---

*Built as a hands-on portfolio project to develop and demonstrate SOC analyst and endpoint-security skills.*

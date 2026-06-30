# Pi-hole Network DNS Sinkhole — LAB-03

**Status:** Complete · **Platform:** Proxmox VE · Pi-hole · Ubuntu 22.04 LXC

A network-wide DNS sinkhole. Pi-hole becomes the DNS authority for the whole LAN — every device is pointed at it, and it decides per query whether a domain is allowed to resolve or gets dropped into a sinkhole. Ads and trackers are removed across every device with no client-side software, known malware/phishing domains are refused, and every DNS request the network makes becomes visible in one dashboard.

---

## Contents

- [What this lab does](#what-this-lab-does)
- [What is Pi-hole & DNS sinkholing](#what-is-pi-hole--dns-sinkholing)
- [Architecture](#architecture)
- [Network design](#network-design)
- [Build](#build)
- [Configuration & rollout](#configuration--rollout)
- [Hardening](#hardening)
- [Validation](#validation)
- [Integration with the SOC lab (LAB-01)](#integration-with-the-soc-lab-lab-01)
- [Challenges & lessons learned](#challenges--lessons-learned)
- [Skills demonstrated](#skills-demonstrated)
- [References](#references)

---

## What this lab does

- Runs Pi-hole in an **unprivileged LXC container** on Proxmox with a fixed static IP.
- Blocks ad, tracker, and malicious domains **network-wide** via DNS sinkholing — no per-device agents.
- Forwards legitimate queries to a chosen upstream resolver, with **DNSSEC** validation enabled.
- Distributes Pi-hole as the LAN's DNS server at the **DHCP layer**, so new devices are protected automatically.
- Surfaces a live dashboard of total queries, percentage blocked, top clients, and top blocked domains.

**Why an LXC and not a VM?** Pi-hole is a small DNS workload that doesn't need its own kernel. A system container shares the Proxmox host kernel, so it boots in seconds and uses a fraction of the RAM and disk of a full VM — the right trade-off for an always-on network service.

---

## What is Pi-hole & DNS sinkholing

Almost everything a device does online starts with a **DNS lookup**: before a browser can load `github.com`, it asks a DNS server "what IP is that?" Ads and trackers work the same way — a page tries to pull content from `ads.tracker-example.com`, which also needs a lookup first.

Pi-hole inserts itself as that DNS server. For every query, it checks the domain against its blocklists (**gravity**). If the domain is a known ad server, tracker, or malware host, Pi-hole **sinkholes** it — returning a non-routable answer (`0.0.0.0`) so the connection is never made. If the domain isn't blocked, Pi-hole forwards the query upstream and returns the answer. Because this happens at the DNS layer, it protects **every device on the network at once**.

```
   CLIENT DEVICE                  PI-HOLE  (LXC)                 UPSTREAM DNS
        │                             │                       (1.1.1.1 / 9.9.9.9)
        │  "ads.tracker.com ?"        │
        ├────────────────────────────►│
        │                  on a blocklist?  ── YES ──►  return 0.0.0.0  (sinkhole)
        │◄────────── 0.0.0.0 ─────────┤                 ✗ ad never loads
        │
        │  "github.com ?"             │
        ├────────────────────────────►│
        │                  on a blocklist?  ── NO ───►  ├──── forward query ───►│
        │                             │                 │◄──── 140.82.x.x ──────┤
        │◄──────── 140.82.x.x ────────┤
        │  ✓ site loads normally
```

This is the homelab-scale version of an enterprise **DNS firewall / Protective DNS** service.

---

## Architecture

| Component | Role | OS / Platform |
|-----------|------|---------------|
| **Proxmox Host** | Type-1 hypervisor hosting the container. | Proxmox VE (Debian) |
| **Pi-hole Container** | Network-wide DNS sinkhole and resolver; holds blocklists, query log, admin console. | Ubuntu 22.04 LXC |
| **DHCP Server** | Hands out the Pi-hole IP as the DNS server to every client. | pfSense / home router |
| **Upstream Resolver** | Resolves any domain Pi-hole permits (DNSSEC enabled). | Cloudflare `1.1.1.1` |
| **LAN Clients** | Every device that uses Pi-hole for DNS. | Mixed |

---

## Network design

For Pi-hole to protect the whole network: it needs a **fixed address** that never changes, and the **DHCP server** must advertise that address as the DNS server to every device.

| Setting | Value (example) | Why |
|---------|-----------------|-----|
| Container IP | `192.168.1.10/24` (static) | The DNS server address is handed to every client; if it changed, all DNS resolution would break. |
| Gateway | `192.168.1.1` | The LAN router / pfSense LAN interface from LAB-01. |
| Upstream DNS | `1.1.1.1` (Cloudflare) | Where Pi-hole forwards anything it doesn't block. DNSSEC enabled. |
| DHCP DNS handout | Primary `192.168.1.10` · Secondary `1.1.1.1` | Pi-hole primary; Cloudflare secondary for failover (trade-off below). |

```
          Internet
             │
        [ Router / pfSense ]  ──  DHCP: DNS primary = 192.168.1.10, secondary = 1.1.1.1
             │
        ─────┴─────────────────────────────  LAN  ─────────────────────────
             │                │                │                │
        192.168.1.10      Laptop            Phone           IoT / TV
        ┌───────────┐    (DNS → .10)      (DNS → .10)      (DNS → .10)
        │  Pi-hole  │◄──── all DNS queries from every client ───────────────
        │   (LXC)   │
        └─────┬─────┘
              │  permitted queries forwarded upstream (DNSSEC)
              ▼
        Cloudflare  1.1.1.1
```

**Trade-off:** Pi-hole is a single point of failure for DNS, which is why this build adds **Cloudflare as a secondary** resolver — if the container is down, name resolution still works (at the cost of some queries bypassing the filter; see rollout).

---

## Build

### 1. Create the LXC container

In the Proxmox UI, download a container template, then create the container:

- **Template:** `local` storage → *CT Templates* → *Templates* → download **Ubuntu 22.04**.
- **Type:** **unprivileged** container (secure default — the container's root is not the host's root).
- **Resources:** 1–2 CPU cores, **1 GB** RAM, **8 GB** disk (headroom for updates and extra blocklists).
- **Network:** bridge to the LAN, assign a **static IPv4** (`192.168.1.10/24`), gateway `192.168.1.1`.
- **DNS:** set a temporary upstream (`1.1.1.1`) so the container can reach the internet to install.

### 2. Run the Pi-hole installer

Access the container via the Proxmox console or SSH, log in as `root`, update, then run the official installer:

```bash
# update the container first
apt update && apt upgrade

# curl is used to fetch the install script
apt install curl

# download and run the official Pi-hole installer
curl -sSL https://install.pi-hole.net | bash
```

The wizard walks through: confirming the **network interface** (`eth0`), picking an **upstream DNS provider**, accepting the default **blocklist**, installing the **web admin interface** + **lighttpd**, confirming the **static IP**, and choosing query-logging / privacy level. At the end it prints the **admin URL** and a **generated admin password** — record both.

Change the admin password any time from the container shell:

```bash
pihole -a -p
```

### 3. Upstream resolver — Cloudflare

This build uses **Cloudflare (`1.1.1.1`)** as the upstream — low latency, a privacy-focused no-logging stance, and **DNSSEC** support (enabled in hardening). For context:

| Provider | Address | Notable for |
|----------|---------|-------------|
| **Cloudflare** *(chosen)* | `1.1.1.1` | Privacy-focused, very low latency, DNSSEC support. |
| Quad9 | `9.9.9.9` | Adds its own malicious-domain blocking — a second security layer. |
| Google | `8.8.8.8` | Highly reliable, globally distributed. |

Cloudflare also serves as the network's **secondary DNS server** for failover (see rollout).

---

## Configuration & rollout

### Point the network at Pi-hole (DHCP)

Devices have to actually *use* Pi-hole. The network-wide way is to change the **DNS server DHCP hands out**:

- **pfSense (LAB-01):** `Services → DHCP Server → LAN` → set **DNS Servers** to `192.168.1.10` (primary) and `1.1.1.1` (secondary) → save.
- **Home router:** find LAN / DHCP settings and set the DNS servers to the Pi-hole IP (primary) and Cloudflare (secondary).
- **Per-device (testing):** manually set one device's DNS to the Pi-hole IP before rolling out network-wide.

Existing devices cache DNS settings — force the change to validate it:

```bash
# Windows — renew DHCP + flush DNS cache
ipconfig /release && ipconfig /renew
ipconfig /flushdns

# Linux / macOS — confirm the resolver in use
nslookup github.com
```

### Access the admin console

Browse to `http://192.168.1.10/admin` and log in. The dashboard shows total queries, percent blocked, queries over time, top clients, and top blocked/permitted domains. The **Query Log** shows DNS requests in real time and flags which were blocked by gravity.

---

## Hardening

- **DNSSEC** — enabled under `Settings → DNS → Use DNSSEC`, paired with the Cloudflare upstream (which supports DNSSEC). Cryptographically validates upstream answers against spoofing / cache poisoning.
- **Blocklists** — see the dedicated section below; the build runs StevenBlack's unified list plus the Firebog collection, compiled into gravity with `pihole -g`.
- **Conditional forwarding** — `Settings → DNS → Conditional forwarding`, pointed at the router + local domain, so the dashboard shows hostnames instead of IPs.
- **Patch & lock down** — update with `pihole -up` and `apt`; keep the admin interface LAN-only (never expose ports 80/443 to the internet); set a strong admin password with `pihole -a -p`.
- **Future enhancement** — add **DNS-over-HTTPS** (`cloudflared`) to encrypt queries leaving Pi-hole to the upstream.

### Blocklists in use

Starting from **StevenBlack's unified hosts** (ads + malware) and layering the **Firebog** collection ([firebog.net](https://firebog.net/) — a curated directory of blocklists graded by safety; the "ticked"/green lists are the low-false-positive set recommended for everyday use). Active adlists:

| List | Source | Focus |
|------|--------|-------|
| **StevenBlack/hosts** | `raw.githubusercontent.com/StevenBlack` | Unified ads + malware — the default base list. |
| KADhosts | `PolishFiltersTeam/KADhosts` | Ads, fraud, scam & phishing. |
| add.Spam | `FadeMind/hosts.extras` | Spam / scam host domains. |
| w3kbl | `v.firebog.net/hosts/static` | Tracking & malicious (Matt's list). |
| AdAway | `adaway.org/hosts.txt` | Mobile advertising. |
| AdGuard DNS | `v.firebog.net/hosts/AdguardDNS.txt` | Ads & tracking. |
| Admiral | `v.firebog.net/hosts/Admiral.txt` | Anti-adblock & ad domains. |
| anudeepND adservers | `anudeepND/blacklist` | Ad-server domains. |
| Disconnect simple_ad | `lists.disconnect.me/simple_ad.txt` | Advertising domains. |
| EasyList | `v.firebog.net/hosts/Easylist.txt` | The classic ad-blocking list. |

This is the first page of the adlist config; the full set spans three pages of mostly Firebog-sourced lists. **Over-blocking scales with list count** — when a legitimate site breaks, read the Query Log, find the blocked domain, and whitelist that one domain rather than disabling a whole list.

---

## Validation

```bash
# a blocked domain should come back as 0.0.0.0
nslookup doubleclick.net

# a normal domain should resolve to a real address
nslookup github.com
```

1. **Blocked domain returns the sinkhole** (`0.0.0.0`, not a real IP).
2. **Normal domain resolves fine** — confirms Pi-hole forwards permitted queries.
3. **Dashboard reflects the traffic** — query count climbs, percent-blocked is non-zero, the blocked lookup appears in the Query Log tagged by gravity.

If a blocked domain still resolves to a real IP, the client isn't using Pi-hole yet — check that DHCP hands out only the Pi-hole address and that the client renewed its lease and flushed its cache.

---

## Integration with the SOC lab (LAB-01)

Because the segmented network from LAB-01 already runs **pfSense as the DHCP server**, Pi-hole slots in as the LAN's DNS authority and its query data becomes telemetry for the SOC stack:

- **DNS as detection signal** — Pi-hole logs *every* domain requested. Spikes in blocked lookups or repeated requests to algorithmically-generated domains are classic indicators of malware beaconing / C2.
- **Forward to Wazuh** — ship the Pi-hole query log to the LAB-01 Wazuh SIEM, turning DNS activity into searchable, alertable events alongside firewall and endpoint telemetry.
- **Layered DNS defense** — pfSense controls routing, Pi-hole controls name resolution, a malicious-domain-aware upstream (Quad9) adds a third independent check.

---

## Challenges & lessons learned

1. **Ads still appearing on some devices** — usually a second DNS server reaching the client (a DHCP fallback, or browser-level DNS-over-HTTPS in Chrome/Firefox routing past Pi-hole). Pi-hole must be the *only* resolver the client can reach, and browser DoH should be disabled.
2. **The static-IP requirement** — if the container pulls a different DHCP address, every device pointed at the old IP loses DNS. Pin a static IP (or reserved lease) and verify gateway/subnet in the LXC network config.
3. **Clients not picking up the new DNS** — devices held a cached lease and kept using the old resolver. Validate by forcing a lease renewal and flushing the DNS cache rather than assuming the change took effect.
4. **DNS loop / upstream pointing at itself** — misconfiguring the upstream so Pi-hole queries itself causes failures/timeouts. The upstream must be external, never the Pi-hole's own address.
5. **Over-blocking from extra lists** — aggressive blocklists broke legitimate sites. Read the Query Log, identify the blocked domain, and whitelist that one domain — don't disable blocking globally.

**Takeaway:** Pi-hole is simple to install but easy to under-deploy. The real skill is making it the network's *single, authoritative* resolver and proving — with lookups and the query log — that traffic actually flows through it.

---

## Skills demonstrated

| Area | What it covers |
|------|----------------|
| **Containerization** | Proxmox LXC deployment — unprivileged container, resource sizing, static network config. |
| **DNS administration** | Authoritative LAN resolver, upstream selection, conditional forwarding, query-log analysis. |
| **Network security** | DNS sinkholing of ad/tracker/malware domains, protective-DNS layering, DNSSEC validation. |
| **Network architecture** | DHCP-level DNS rollout, single-resolver enforcement, single-point-of-failure trade-offs. |
| **Linux administration** | Ubuntu container management, package updates, Pi-hole CLI (`pihole -g`, `-up`, `-a`). |
| **Hardening** | Admin-interface restriction, credential management, blocklist tuning, patch discipline. |

---

## References

- [Pi-hole Documentation](https://docs.pi-hole.net/)
- [VirtualizationHowto — How to Install Pi-hole on Proxmox](https://www.virtualizationhowto.com/2023/04/how-to-install-pi-hole-on-proxmox/)
- [Proxmox VE — Linux Container (LXC) documentation](https://pve.proxmox.com/wiki/Linux_Container)
- [StevenBlack/hosts — curated blocklist](https://github.com/StevenBlack/hosts)

---

*LAB-03 · built & documented by wahle*

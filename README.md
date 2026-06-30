# Cyber Portfolio

This is where I host my cybersecurity projects — hands-on labs, write-ups, and tools I build as I work toward a SOC analyst role.

Each project lives in its own folder with its own documentation. New work is added here over time.

## About

Help Desk Technician II and aspiring SOC analyst, focused on detection engineering, SIEM, network security, and endpoint management.

**Certifications:** B.S. Cybersecurity · Security+ · AZ-104 · AZ-900 · SC-200 (in progress)

## Projects

| Project | Description | Status |
|---------|-------------|--------|
|[Wazuh SOC Home Lab](./wazuh-soc-files/README.md) | End-to-end mini-SOC on Proxmox: Wazuh SIEM, inline Suricata IDS/IPS, pfSense firewall, VirusTotal enrichment, FIM, and Sysmon — detecting an attack from network packet to dashboard alert. | Complete |
|[SIEM Alerts & Phishing Email Investigation](./phishing-investigation/README.md) | Tier-1 SOC analyst playbook for triaging phishing alerts in a SIEM — alert ownership, email header / static / dynamic IOC analysis, blast-radius assessment, mailbox containment, and case documentation. Distilled from four LetsDefend courses, with the core SOC concepts and common analyst mistakes. | Complete |
|[Pi-hole Network DNS Sinkhole](./pihole-files/README.md) | Network-wide ad, tracker, and malware-domain blocking in an unprivileged LXC container on Proxmox: Pi-hole as the LAN's DNS authority, with a DNSSEC-validated Cloudflare upstream, the Firebog blocklist collection, and DHCP-level rollout. | Complete |
More projects coming soon.

## Connect

- GitHub: [@wahleeee](https://github.com/wahleeee)

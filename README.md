# Homelab Security Operations Environment

A production-style security monitoring environment built on consumer hardware, designed to simulate enterprise blue team infrastructure. The project covers network segmentation, intrusion detection, log aggregation, and threat visibility using open-source tooling.

---

## Architecture Overview

```
ISP / ONT (Public IP)
        │
        ▼
┌─────────────────────────────┐
│     OPNsense (Optiplex #1)  │
│   Router · Firewall · IDS   │
│     Suricata · WireGuard    │
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│   Cisco Catalyst 1200       │
│   Managed Switch            │
│   Mgmt: 192.168.99.3        │
└──┬──┬──┬──┬──┬──────────────┘
   │  │  │  │  │
```

| Port | VLAN | Subnet | Description |
|------|------|--------|-------------|
| gi3  | 10   | 192.168.10.0/24 | Main workstation (untagged) |
| gi1  | 20   | 192.168.20.0/24 | WiFi — ISP router bridged mode |
| gi5  | 30   | 192.168.30.0/24 | Wired desktops |
| gi6  | 40   | 192.168.40.0/24 | ELK stack VM (tagged via Proxmox) |
| gi6  | 99   | 192.168.99.0/24 | Management (Proxmox, switch) |

---

## Hardware

| Device | Role | OS |
|--------|------|----|
| Dell Optiplex (SFF) | Router / Firewall / IDS | OPNsense (bare metal) |
| Dell Optiplex Micro | Hypervisor | Proxmox VE |
| Cisco Catalyst 1200 | Managed switch | — |
| ISP Router | WiFi AP (bridged) | — |

---

## Network Security

### OPNsense
- Stateful firewall with per-VLAN rule sets
- Inter-VLAN traffic denied by default; explicitly permitted where required
- WireGuard VPN for remote access
- DNS resolver with local overrides

### Suricata IDS
- Monitoring on LAN interface and VLAN interfaces
- ET Open ruleset with custom alert tuning
- Alerts forwarded to ELK stack via Logstash
- Known detection gap documented: traffic destined for OPNsense's own interfaces bypasses Suricata inspection (host-based IDS limitation — planned mitigation: dedicated SPAN port sensor)

### VLAN Segmentation
- Management plane (VLAN99) isolated from user traffic
- Switch management interface migrated off default VLAN1 to VLAN99
- WiFi devices isolated to VLAN20; no inter-VLAN access to wired segments
- ELK stack on dedicated VLAN40, accessible only from management VLAN

---

## SIEM — ELK Stack (Ubuntu Server VM · VLAN40)

### Stack
- **Elasticsearch** — log storage and indexing
- **Logstash** — ingestion, parsing, enrichment pipeline
- **Kibana** — dashboards and alert visualization

### Logstash Pipeline
- Ingests Suricata EVE JSON logs from OPNsense
- GeoIP enrichment on source/destination IPs
- Severity normalization across alert categories
- Separate index routing by log type (alerts, DNS, flows, HTTP)

### Kibana Dashboards
- Suricata alert volume over time
- Top talkers by source IP and destination country (GeoIP map)
- Alert severity breakdown
- Firewall traffic analysis — allowed vs blocked by VLAN

---

## Detection Coverage

| Scenario | Detected | Method |
|----------|----------|--------|
| Nmap SYN scan against LAN hosts | ✅ | Suricata ET rules |
| Nmap service/version scan (-sV) against VLAN hosts | ✅ | Suricata ET rules |
| Nmap scan against OPNsense own IPs | ❌ | Known gap — host-based IDS blind spot |
| Inter-VLAN lateral movement attempts | ✅ | Firewall logs + Suricata |
| GeoIP — traffic to/from anomalous countries | ✅ | Logstash GeoIP + Kibana |

---

## MITRE ATT&CK Coverage

| Tactic | Technique | Detection |
|--------|-----------|-----------|
| Discovery | T1046 — Network Service Scanning | Suricata nmap signatures |
| Discovery | T1018 — Remote System Discovery | Suricata + firewall logs |
| Lateral Movement | T1021 — Remote Services | Inter-VLAN firewall rules |
| Command & Control | T1572 — Protocol Tunneling | WireGuard monitoring |

---

## Planned Improvements

- [ ] SPAN port configuration on Cisco Catalyst 1200
- [ ] Dedicated IDS sensor NIC on Proxmox host (Intel I350/I210)
- [ ] Suricata instance on SPAN interface to close OPNsense blind spot
- [ ] Additional VLAN for IoT device isolation
- [ ] VLAN-capable AP (TP-Link Omada or Ubiquiti UniFi) for per-SSID VLAN tagging
- [ ] Zeek for protocol-level network visibility alongside Suricata
- [ ] Elastic Certified Analyst certification path

---

## Certifications & Context

- CompTIA Security+ (2025)
- CompTIA CySA+ (in progress)
- Active T4 Public Trust clearance (adjudicated 2025)
- Day-to-day sysadmin experience in a FISMA-compliant federal environment (VA)

---

## Tools & Technologies

`OPNsense` `Suricata` `WireGuard` `Proxmox` `Elasticsearch` `Logstash` `Kibana` `Cisco IOS` `VLAN` `Active Directory` `Entra ID` `PowerShell` `Linux` `MITRE ATT&CK`

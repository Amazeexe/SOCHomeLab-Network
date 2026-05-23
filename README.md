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
│   Cisco 3560CX              │
│   Managed Switch            │
│   IOS 15.2(7)E14            │
│   Mgmt: 192.168.99.3        │
└──┬──┬──┬──┬──┬──────────────┘
   │  │  │  │  │
```

| Port | VLAN | Subnet | Description |
|------|------|--------|-------------|
| Gi0/1 | trunk | all | OPNsense uplink |
| Gi0/2 | 10 | 192.168.10.0/24 | Main workstation |
| Gi0/3 | 20 | 192.168.20.0/24 | WiFi — ISP router bridged mode |
| Gi0/4 | 30 | 192.168.30.0/24 | Wired desktops |
| Gi0/5 | 40/99 | 192.168.40.0/24 / 192.168.99.0/24 | Proxmox (VLAN99 native, VLAN40 tagged) |

---

## Hardware

| Device | Role | OS |
|--------|------|----|
| Dell Optiplex SFF | Router / Firewall / IDS | OPNsense (bare metal) |
| Dell Optiplex Micro | Hypervisor | Proxmox VE |
| Cisco Catalyst 3560CX | Managed switch | IOS 15.2(7)E14 |
| ISP Router | WiFi AP (bridged mode) | — |

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
- Management plane (VLAN99) isolated from all user traffic
- Switch management interface migrated off default VLAN1 to VLAN99
- VLAN1 disabled — SVI shut down, removed from trunk allowed list
- WiFi devices isolated to VLAN20; no inter-VLAN access to wired segments
- ELK stack on dedicated VLAN40, accessible only from management VLAN
- Proxmox host on VLAN99 (native/untagged), Ubuntu VM tagged to VLAN40 via Proxmox virtual bridge

### Switch Hardening
- Default VLAN1 disabled and removed from trunk
- Management VLAN99 restricts access via OPNsense firewall rules
- SSH enabled with modern key exchange (IOS 15.2(7)E14)
- Local user authentication enforced on all VTY lines (`login local`)
- Portfast enabled only on end-host access ports

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

## Switch Migration — Catalyst 1200 to Cisco 3560CX

Replaced the Cisco Catalyst 1200 (limited CLI, no standard IOS) with a Cisco Catalyst 3560CX running full IOS. Performed a zero-downtime migration by pre-configuring the new switch before any cable changes.

### Migration process
- Captured full running config from Catalyst 1200 (`show vlan brief`, `show interfaces status`, `show running-config`)
- Translated config to standard IOS syntax on the 3560CX
- Verified all VLAN assignments, trunk config, and management IP before swapping cables
- Swapped physical connections one at a time and confirmed connectivity after each

### IOS upgrade
- Upgraded from `15.2(7)E` (2019) to `15.2(7)E14` (2026) via USB flash transfer
- Deleted old image from flash to free space, transferred new image, set boot variable, reloaded
- Upgrade resolved legacy SSH key exchange limitations — modern SSH negotiation now works without flags

### Key IOS commands used
```
! VLAN config
vlan 10
 name MYPC

! Trunk port to OPNsense
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,99

! Proxmox trunk port
interface GigabitEthernet0/5
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 40,99
 spanning-tree portfast

! Management SVI
interface vlan 99
 ip address 192.168.99.3 255.255.255.0
 no shutdown

! Disable VLAN1
interface vlan 1
 shutdown

! Authentication
username ******* privilege 15 secret <password>
line vty 0 15
 login local
 transport input ssh

! IOS upgrade via USB
copy usbflash0:c3560cx-universalk9-mz.152-7.E14.bin flash:
boot system flash:c3560cx-universalk9-mz.152-7.E14.bin
```

---

## Planned Improvements

- [ ] SPAN port configuration on Cisco 3560CX
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

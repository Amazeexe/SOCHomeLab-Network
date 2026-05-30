# Homelab Security Operations Environment

A production-style security monitoring environment built on consumer hardware, designed to simulate enterprise blue team infrastructure. The project covers network segmentation, intrusion detection, log aggregation, vulnerability management, and threat visibility using open-source tooling.

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
| Dell Optiplex Micro | Hypervisor (Node 1) | Proxmox VE |
| Machinist X99 (Xeon E5-2690v4, 32GB ECC) | Hypervisor (Node 2 — AD/Windows lab) | Proxmox VE |
| Cisco Catalyst 3560CX | Managed switch | IOS 15.2(7)E14 |
| ISP Router | WiFi AP (bridged mode) | — |

---

## Network Security

### OPNsense
- Stateful firewall with per-VLAN rule sets
- Inter-VLAN traffic denied by default; explicitly permitted where required
- WireGuard VPN for remote access (split-tunnel routing to all internal VLANs)
- Unbound DNS resolver with local overrides and query logging enabled

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
- **Elasticsearch** — log storage and indexing (TLS 1.3 only)
- **Logstash** — ingestion, parsing, enrichment pipeline (TLS 1.3 only on beats input)
- **Kibana** — dashboards and alert visualization

### Log Sources (Filebeat filestream → Logstash → Elasticsearch)
- **Suricata** EVE JSON — IDS alerts and flow events
- **Firewall** — OPNsense filterlog (dissect-parsed)
- **DNS** — Unbound resolver query logs (client IP, query name, record type)
- **DHCP** — Dnsmasq lease events (MAC, assigned IP, hostname, VLAN)
- **Authentication** — OPNsense web UI access logs (nginx)
- **Linux auth** — Ubuntu host SSH/auth events via Elastic Agent

### Logstash Pipeline
- Tag-based routing — each log source routed to its own index by Filebeat tags
- GeoIP enrichment on source/destination IPs
- Severity normalization across Suricata alert categories
- Grok parsing for DNS, DHCP, and auth log formats
- All custom indices governed by a 30-day ILM policy for storage management

### Index Lifecycle Management
- `suricata-alerts-*`, `suricata-logs-*`, `firewall-logs-*`
- `dns-logs-*`, `dhcp-logs-*`, `opnsense-auth-*`
- All on a 30-day delete policy to bound storage growth on the single-node cluster

### Kibana Dashboards
- Suricata alert volume over time
- Top talkers by source IP and destination country (GeoIP map)
- Alert severity breakdown
- Firewall traffic analysis — allowed vs blocked by VLAN
- DNS query analytics (replaces Pi-hole native UI)

---

## Vulnerability Management — Greenbone / OpenVAS

A dedicated Greenbone Community Edition (OpenVAS) instance runs in Docker on a separate VM (VLAN40), providing authenticated and unauthenticated vulnerability scanning across all network segments.

### Deployment
- Greenbone Community Edition via official Docker Compose containers
- Default `admin` account locked out; dedicated admin user provisioned via `gvmd` CLI
- Scan targets defined per-VLAN across the full address space

### Workflow demonstrated
- Full and Fast authenticated scan across all five VLAN ranges
- Findings triaged by CVSS severity
- Remediation performed and verified by rescan (full vulnerability management loop)

### Example remediation — ELK stack TLS
- **Finding:** SSL/TLS Renegotiation DoS (CVE-2011-1473) on Elasticsearch (9200) and Logstash (5044); deprecated TLS 1.0/1.1 detected
- **Action:** Restricted both services to TLS 1.3 only via `supported_protocols` configuration
- **Verification:** Confirmed via `nmap --script ssl-enum-ciphers` that only TLS 1.3 is negotiated; renegotiation vector eliminated by protocol design
- **Outcome:** Findings cleared on rescan with no loss of pipeline functionality

---

## Detection Coverage

| Scenario | Detected | Method |
|----------|----------|--------|
| Nmap SYN scan against LAN hosts | ✅ | Suricata ET rules |
| Nmap service/version scan (-sV) against VLAN hosts | ✅ | Suricata ET rules |
| Nmap scan against OPNsense own IPs | ❌ | Known gap — host-based IDS blind spot |
| Inter-VLAN lateral movement attempts | ✅ | Firewall logs + Suricata |
| GeoIP — traffic to/from anomalous countries | ✅ | Logstash GeoIP + Kibana |
| DNS-based exfiltration / tunneling indicators | 🔧 | DNS query logs (detection rules in progress) |
| New / unknown device on network | 🔧 | DHCP lease logs (detection rules in progress) |

---

## MITRE ATT&CK Coverage

| Tactic | Technique | Detection |
|--------|-----------|-----------|
| Discovery | T1046 — Network Service Scanning | Suricata nmap signatures |
| Discovery | T1018 — Remote System Discovery | Suricata + firewall logs |
| Lateral Movement | T1021 — Remote Services | Inter-VLAN firewall rules |
| Command & Control | T1572 — Protocol Tunneling | WireGuard monitoring |
| Command & Control | T1071.004 — DNS | DNS query logging (rule in progress) |
| Initial Access | T1200 — Hardware Additions | DHCP new-device monitoring (rule in progress) |

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

- [ ] Windows Server 2022 Domain Controller (Proxmox Node 2) with joined endpoints
- [ ] Sysmon deployment via GPO across domain endpoints, shipped to ELK
- [ ] Kali Linux attack VM for purple-team telemetry generation
- [ ] Detection rules in Kibana — Sigma-authored, KQL/EQL-converted, MITRE-tagged
- [ ] Documented end-to-end alert investigation writeups
- [ ] Two-node Proxmox cluster with QDevice quorum on OPNsense
- [ ] SPAN port configuration on Cisco 3560CX
- [ ] Dedicated IDS sensor NIC on Proxmox host (Intel I350/I210)
- [ ] Suricata instance on SPAN interface to close OPNsense blind spot
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

`OPNsense` `Suricata` `WireGuard` `Proxmox` `Elasticsearch` `Logstash` `Kibana` `Greenbone/OpenVAS` `Docker` `Cisco IOS` `VLAN` `Active Directory` `Entra ID` `PowerShell` `Linux` `MITRE ATT&CK`

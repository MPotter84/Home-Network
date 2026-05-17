# Home Network Configuration
**Platform:** Ubiquiti UniFi  
**Controller:** UniFi Dream Machine (UDM) — Gateway + Built-in AP  
**Firmware:** UniFi OS 5.0.16 / Network Application 10.3.58  
**Uptime:** ~4+ days continuous operation  

---

## 1. Internet / WAN

| Property | Value |
|---|---|
| ISP Type | Residential fiber provider |
| WAN Interface | Port 5 (dedicated WAN port) |
| IPv4 Connection | DHCP (ISP-assigned) |
| IPv6 | Disabled |
| Expected Download | ~775 Mbps |
| Expected Upload | ~891 Mbps |
| Avg. Latency | ~5 ms |
| DNS Primary | Cloudflare (1.1.1.1) |
| DNS Secondary | Google (8.8.8.8) |
| UPnP | Enabled (Secure Mode + NAT-PMP) |
| Flow Control | Disabled |
| Automatic Speed Test | Daily at 5:00 AM |
| IPv6 | Disabled |

---

## 2. Hardware Topology

### Gateway
- **Device:** UniFi Dream Machine (UDM)
- **Role:** Combined router, firewall, switch, and AP controller
- **Built-in Radio:** 2.4 GHz (ch. 1, 20 MHz, 2×2 WiFi 4) + 5 GHz (ch. 149, 80 MHz, 4×4 WiFi 5)
- **WAN Uplink:** GbE to ISP modem/ONT
- **Memory Usage:** ~92% (active routing + IDS/IPS + UniFi controller)
- **Load Average:** 5.15 / 4.06 / 2.82

### Wireless Mesh Access Points (3× UniFi AC Mesh)
All three APs connect to the gateway via **wireless mesh uplink** (no PoE/cable runs required). Each operates on the same radio channels as the gateway for seamless roaming.

| AP Role | Location Intent | 2.4 GHz Ch. | 5 GHz Ch. | Connected Clients |
|---|---|---|---|---|
| Primary Gateway AP | Central/main floor | 1 (20 MHz) | 149 (80 MHz) | 7 |
| Mesh AP — Spare Room | Extended coverage, upper area | 6 (20 MHz) | 149 (80 MHz) | 3 |
| Mesh AP — Basement | Below-grade coverage | 1 (20 MHz) | 149 (80 MHz) | 1 |
| Mesh AP — Garage | Outbuilding/detached coverage | 1 (20 MHz) | 149 (80 MHz) | 0 |

**Mesh Configuration:**
- Wireless Meshing: Enabled
- Mesh Monitor: Gateway
- UniFi Auto-Link: Enabled
- WiFiMan Support: Enabled
- Default Channel Widths: 2.4 GHz → 20 MHz, 5 GHz → 40 MHz, 6 GHz → 160 MHz
- WiFi Speed Mode: Conservative (prioritizes stability over raw throughput)

---

## 3. Network Segmentation (VLANs)

Seven logical networks are defined, each with its own VLAN ID, subnet, and DHCP scope. The default security posture is **Block All** — inter-VLAN traffic is denied unless explicitly permitted by firewall rules.

| Network Name | Purpose / Intent | VLAN ID | Subnet | DHCP | Leases In Use |
|---|---|---|---|---|---|
| Trusted | Primary trusted devices — personal computers, phones, media devices | 1 (native) | /24 | Server (UDM) | 17 / 249 |
| Work Computers | Managed work/professional endpoints — isolated from home IoT | 2 | /24 | Server (UDM) | 0 / 249 |
| IoT (wired) | IoT backbone — managed by third-party gateway for legacy device support | 4 | — (no UDM DHCP) | Third-party gateway | — |
| IoT WLAN | Wireless IoT devices — smart home sensors, cameras, smart plugs | 30 | /24 | Server (UDM) | 6 / 249 |
| Lab | Homelab / self-hosted infrastructure — servers, containers, dev machines | 20 | /24 | Server (UDM) | 9 / 249 |
| Security Cameras | Dedicated surveillance VLAN — cameras isolated from all internal traffic | 99 | /24 | Server (UDM) | 0 / 249 |
| HoneyPot | Deception network — attracts and audits unauthorized lateral movement | 25 | /24 | Server (UDM) | 0 / 249 |

**Global Network Settings:**
- Default Security Posture: **Block All** (explicit allow rules required)
- Gateway mDNS Proxy: **Auto** (cross-VLAN service discovery where needed)
- IGMP Snooping: **Enabled** on IoT VLAN (multicast optimization)
- Spanning Tree: **RSTP**
- Rogue DHCP Server Detection: **Enabled**
- RADIUS Authentication: UDM local credentials

---

## 4. Wireless Networks (SSIDs)

Four SSIDs are defined across all APs. SSIDs are anonymized here by function.

| # | Purpose | Mapped VLAN | Bands | Security |
|---|---|---|---|---|
| 1 | Primary trusted household wireless | Native (Trusted) | 2.4 GHz + 5 GHz | WPA2 |
| 2 | IoT device wireless (primary) | IoT WLAN (30) | 2.4 GHz + 5 GHz | WPA2 |
| 3 | IoT device wireless (legacy/2.4GHz only) | IoT WLAN (30) | 2.4 GHz only | WPA2 |
| 4 | Trusted secondary / cross-device | Native (Trusted) | 2.4 GHz + 5 GHz | WPA2/WPA3 |

**Active Clients by SSID:** ~7 on trusted networks, ~4 on IoT WLAN, 0 on legacy IoT.

---

## 5. Firewall & Network Policy

### 5.1 Internet Firewall Rules (WAN → LAN)
Using the UniFi traffic & firewall rules engine (legacy mode — Zone-Based Firewall upgrade available but not yet applied).

| Rule | Action | Direction | Notes |
|---|---|---|---|
| Allow Established/Related Traffic | Accept | Internet In | Stateful connection tracking |
| Block Invalid Traffic | Drop | Internet In | Malformed/unexpected state packets |
| Allow Port Forward — Game Server | Accept | Internet In | TCP/UDP to game server host on custom port |
| Block All Other Traffic | Drop | Internet In | Default deny |
| Allow Established/Related Traffic | Accept | Internet Local | Router itself — stateful |
| Block Invalid Traffic | Drop | Internet Local | Router itself — invalid state |
| Block All Other Traffic | Drop | Internet Local | Default deny for router-bound traffic |

### 5.2 LAN Firewall Rules (Inter-VLAN Segmentation) — 40 Rules

The LAN ruleset enforces strict inter-VLAN segmentation. All rules are prefixed `AUDIT:` for logging/visibility, and follow a consistent pattern per VLAN.

**HoneyPot VLAN — Outbound Audit (visibility-only, allowed to all for deception):**
- AUDIT: Allow HoneyPot → Trusted
- AUDIT: Allow HoneyPot → Work Computers
- AUDIT: Allow HoneyPot → IoT WLAN
- AUDIT: Allow HoneyPot → Lab
- AUDIT: Allow HoneyPot → Security Cameras
- AUDIT: Block ALL Internal → HoneyPot (nothing initiates into the honeypot)

**IoT WLAN — Restricted Egress:**
- Allow IoT → Gateway DNS/NTP only (TCP/UDP)
- Allow IoT → Trusted (return traffic only — established/related)
- Block IoT → Trusted (new connections)
- Block IoT → Work Computers
- Block IoT → Lab
- Block IoT → HoneyPot

**Work Computers — Restricted Egress:**
- Allow Work → Gateway DNS/NTP only
- Block Work → IoT VLAN
- Block Work → Lab
- Block Work → HoneyPot

**Lab — Semi-Trusted Egress:**
- Allow Lab → Gateway DNS/NTP only
- Allow Lab → Trusted (return traffic)
- Block Lab → Trusted (new connections)
- Block Lab → IoT VLAN
- Block Lab → HoneyPot

**Trusted — Controlled Access:**
- Allow Trusted → Lab (full access)
- Block Trusted → HoneyPot
- Allow Trusted → Security Cameras
- Allow Security Cameras → Trusted (return traffic)
- Block ALL → Security Cameras (default deny for camera VLAN)

**System-generated baseline rules (auto-applied per subnet):**
- Isolate IPv4 traffic from Security Cameras to 6 other networks (Drop)
- Per-subnet Allow LAN In/Out rules for all 6 routed subnets (ensuring intra-VLAN routing functions)

### 5.3 Port Forwarding

| Service | Protocol | External Port | Internal Destination | Interface |
|---|---|---|---|---|
| Game Server (Minecraft) | TCP/UDP | 25565 | Lab VLAN host | WAN1 (Internet 1) |

---

## 6. DNS — Internal Resolution (Split-Horizon)

A set of internal DNS Host (A) records is configured via the Policy Engine to resolve internal service FQDNs without leaving the network. This enables private, low-latency access to self-hosted services by their domain names rather than IP addresses.

**Record types in use:** DNS Host (A) — Resolve action  
**Services with internal DNS records (count: 8):**

| Service Category | Resolution Intent |
|---|---|
| Game/app server | Internal host for self-hosted game backend |
| Automation platform | Node-based workflow automation engine |
| Development environment | Dev tooling / IDE-accessible workspace |
| Data layer | Database or API backend for internal apps |
| Monitoring/observability | Internal metrics and alerting dashboard |
| Version control | Self-hosted Git server |
| Secrets management | Credential vault / secrets store |
| AI services | Self-hosted or local AI inference endpoint |

All 8 records resolve to hosts on the Lab VLAN. Queries from any network source are resolved internally before reaching the public internet.

---

## 7. Security Stack

### 7.1 CyberSecure — Threat Protection

| Feature | Configuration |
|---|---|
| Intrusion Prevention (IDS/IPS) | **Enabled** — Notify and Block mode |
| Signature Database | Updated May 16, 2026 at 8:02 PM |
| Protected Networks | Trusted, IoT WLAN, Lab, Work Computers, Security Cameras (5 of 7 VLANs) |
| Botnet & Threat Intelligence | Max sensitivity (3/3) |
| Viruses, Malware & Spyware | Max sensitivity (3/3) |
| Hacking & Exploits | Max sensitivity (3/3) |
| Peer-to-Peer & Dark Web | Max sensitivity (3/3) |
| Protocol Vulnerabilities | Enabled (1/1 active rule) |
| Detection Exclusion | Gateway IP excluded to prevent false positives |

### 7.2 Region Blocking

Inbound and outbound traffic is blocked for 5 high-risk geographic regions:

- North Korea
- Russia
- China
- Iran
- Belarus

Direction: Both inbound and outbound  
Mode: Block

### 7.3 Encrypted DNS

| Setting | Value |
|---|---|
| Mode | Auto |
| Providers | Cloudflare + Google (DoH/DoT) |

DNS traffic from all clients is encrypted at the resolver level, preventing ISP-level DNS inspection.

### 7.4 HoneyPot Network

A dedicated deception VLAN (`HoneyPot`, VLAN 25) is configured with a monitored subnet. Any lateral movement attempting to reach this network from internal VLANs is audited. Firewall rules allow the HoneyPot to appear reachable (for deception) but block all unsolicited inbound traffic from other segments.

- Honeypot host IP: defined within the /24 subnet
- CyberSecure integration: active

### 7.5 Content Filtering

| Rule Name | Scope | Filter Type | Schedule |
|---|---|---|---|
| Block Malicious — All Networks | All devices (~102 clients/endpoints) | Ad Block | Always |

### 7.6 Traffic Logging

| Setting | Value |
|---|---|
| NetFlow (IPFIX) | Disabled |
| Activity Logging (Syslog) | Internally stored on UDM |
| Data Retention | 365 days |
| SNMP Monitoring | Disabled |
| Logging Level | Auto |

---

## 8. VPN

| Feature | Status |
|---|---|
| Teleport (Ubiquiti remote access VPN) | **Enabled** — link-based guest invite supported |
| VPN Server (WireGuard/L2TP) | Configured, no active tunnels |
| VPN Client | Not configured |
| Site-to-Site VPN | Not configured |

---

## 9. Active Client Summary

**Total clients: ~24 active (19 online, 5 offline)**

| VLAN | Connection Type | Approximate Device Mix |
|---|---|---|
| Trusted | WiFi 5 (5 GHz, 80 MHz) + GbE | Apple laptops, iPhones, Apple Watch, smart TV, gaming laptop |
| IoT WLAN | WiFi 4 (2.4 GHz, 20 MHz) | Smart home sensors (Eufy, Tstat), 3D printer (Bambu Lab), smart plugs |
| Lab | GbE (wired) | Proxmox servers (multiple VMs), Intel NUC, self-hosted services |
| Work Computers | GbE | MSI workstation |
| Security Cameras | (no active clients) | — |

**WiFi Generation Breakdown:**
- WiFi 4: 5 clients
- WiFi 5: 6 clients
- GbE wired: 8 clients

**Vendor Distribution:** Proxmox/server infra (6), Apple (6), Eufylife (2), Others (2), Bambu Lab (1), Intel (1), MSI (1), Resideo (1)

---

## 10. High Availability & Redundancy

| Feature | Status |
|---|---|
| High Availability (HA) | Available (not configured — single gateway) |
| Dynamic Routing | Available (not configured) |
| WAN Failover / Second WAN | Not configured (single ISP uplink) |
| Spanning Tree (RSTP) | Enabled on all switch ports |

---

## Design Philosophy

This network is built around **zero-trust inter-VLAN segmentation** with explicit allow rules for each traffic flow. Key design principles:

1. **Default deny everything** — the `Block All` security posture means no cross-segment traffic flows unless a rule explicitly permits it.
2. **Purpose-built VLANs** — IoT, work, lab, cameras, and honeypot traffic are fully isolated from the trusted household network.
3. **IoT containment** — IoT devices can only reach the gateway for DNS/NTP; they cannot initiate connections to any other internal network.
4. **Lab isolation with trusted reach** — the homelab can be accessed from the Trusted network, but lab hosts cannot initiate connections back to trusted devices.
5. **Active deception** — a honeypot VLAN with a live host acts as a canary for unauthorized lateral movement.
6. **Layered security** — IDS/IPS, region blocking, encrypted DNS, and content filtering operate at the gateway level, applying to all VLANs simultaneously.
7. **Internal DNS resolution** — all self-hosted services resolve via split-horizon DNS, keeping internal traffic internal and enabling human-readable service names.
8. **Wireless mesh** — three satellite APs extend coverage to all areas of the home including outbuildings, all over wireless backhaul with zero new cabling.

# 🏠 Home Lab

A self-built home lab used as a hands-on learning platform for cybersecurity, networking, system administration, and self-hosted services. Built around enterprise-grade Cisco Meraki networking hardware, a 3-node Proxmox cluster, and a TrueNAS SCALE storage server — all designed, deployed, and operated as personal learning infrastructure.

![Networking](https://img.shields.io/badge/Networking-Cisco%20Meraki-1BA0D7)
![Virtualisation](https://img.shields.io/badge/Virtualisation-Proxmox%20VE-E57000)
![Storage](https://img.shields.io/badge/Storage-TrueNAS%20SCALE-0095D5)
![Security](https://img.shields.io/badge/SIEM-Wazuh-4C9A2A)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

## Table of Contents

- [1. Overview](#1-overview)
  - [1.1 Objectives](#11-objectives)
  - [1.2 Skills Demonstrated](#12-skills-demonstrated)
- [2. Network Architecture](#2-network-architecture)
  - [2.1 Physical Topology](#21-physical-topology)
  - [2.2 Device Connectivity](#22-device-connectivity)
  - [2.3 Wireless Architecture](#23-wireless-architecture)
  - [2.4 VLAN Design](#24-vlan-design)
  - [2.5 Firewall Policy](#25-firewall-policy)
  - [2.6 VPN & Remote Access](#26-vpn--remote-access)
- [3. Hardware](#3-hardware)
  - [3.1 Proxmox Nodes — Specifications](#31-proxmox-nodes--specifications)
- [4. Storage — TrueNAS SCALE](#4-storage--truenas-scale)
  - [4.1 Storage Pools](#41-storage-pools)
  - [4.2 Network Shares](#42-network-shares)
  - [4.3 Backup Strategy (Current State)](#43-backup-strategy-current-state)
- [5. Services & Applications](#5-services--applications)
  - [5.1 Proxmox Workloads](#51-proxmox-workloads)
  - [5.2 TrueNAS Applications](#52-truenas-applications)
- [6. Troubleshooting & Problem Solving](#6-troubleshooting--problem-solving)
- [7. Future Roadmap](#7-future-roadmap)
- [8. Appendix](#8-appendix)
  - [8.1 Technologies & Tools Referenced](#81-technologies--tools-referenced)

---

## 1. Overview

<!--
  Optional hero image / rack overview photo goes here.
  Add the image file to: assets/lab-overview.jpg
  Then embed it with:
  ![Lab Overview](assets/lab-overview.jpg)
-->

This document provides a comprehensive technical overview of a self-built home lab designed and maintained as a platform for hands-on learning in cybersecurity, networking, system administration, and self-hosted services. The environment is built around enterprise-grade Cisco Meraki networking hardware, a Proxmox virtualisation cluster, and a TrueNAS SCALE storage server — all managed and operated as a personal learning infrastructure.

### 1.1 Objectives

- Develop practical, hands-on skills in networking, virtualisation, storage, and security operations.
- Build and operate a production-grade self-hosted services environment, replacing reliance on commercial cloud platforms.
- Experiment with new technologies, configurations, and tools in a safe, isolated environment before applying them in professional or academic contexts.
- Simulate real-world enterprise infrastructure at a small scale, including network segmentation, access control, monitoring, and incident response.

### 1.2 Skills Demonstrated

- Enterprise network design and configuration (Cisco Meraki MX, MS, MR)
- VLAN segmentation, inter-VLAN routing, and access control policy
- Firewall rule design and outbound traffic policy
- Proxmox VE: VM and LXC container lifecycle management
- TrueNAS SCALE: ZFS pool design, dataset management, NFS/SMB sharing
- Docker and containerised application deployment
- Reverse proxy configuration (Nginx Proxy Manager) with Cloudflare DNS integration
- Network-wide DNS filtering and ad-blocking (AdGuard Home)
- Security monitoring and SIEM deployment (Wazuh)
- Observability stack deployment (Prometheus, Grafana, Netdata, Uptime Kuma)
- VPN configuration (Cisco Secure Client, Tailscale)
- Troubleshooting across networking, Linux, DNS, NFS, and Docker layers

---

## 2. Network Architecture

### 2.1 Physical Topology

The physical network follows a simple three-tier hierarchy designed for clarity, manageability, and separation of concerns. All traffic flows through the Meraki MX67W, which acts as the sole gateway and enforces all security policies.

**Traffic flow — Primary Site:**
`ISP → NBN Modem → Cisco Meraki MX67W (Firewall/Router) → Cisco Meraki MS350-24P (Core Switch) → Devices/Servers (MR56, TrueNAS, Proxmox)`

**Traffic flow — Secondary Site:**
`Site-to-Site VPN from Primary Site MX → Secondary Site MX → TrueNAS Backup Server (downstream of Secondary Site MX)`

<!--
  Network topology diagram goes here.
  Add the image file to: assets/network-diagram.png
  Then embed it with:
  ![Network Topology Diagram](assets/network-diagram.png)
-->

### 2.2 Device Connectivity

**Primary Site**

| Device | Switch Port | VLAN | Connection Type |
|---|---|---|---|
| Cisco Meraki MX67W | Port 1 (Uplink) | – | — |
| Cisco Meraki MR56 (AP) | Port 14 | 100 (Native) + All | Trunk — AP receives all VLANs; broadcasts per SSID mapping |
| TrueNAS Server | Ports 6 & 7 | 20 (Servers) | Link Aggregation (LACP) — 2 × 1 Gbps active-active uplink |
| Proxmox Node 1 (HP) | Port 10 | 20 (Servers) | Access port — single uplink on Servers VLAN |
| Proxmox Node 2 (Lenovo) | Port 11 | 20 (Servers) | Access port — single uplink on Servers VLAN |
| Proxmox Node 3 (Dell) | Port 14 | 20 (Servers) | Access port — single uplink on Servers VLAN |
| Samsung TV | Port 4 on MX | 60 (IoT) | Access port — direct wired IoT device |

**Secondary Site**

| Device | Switch Port | VLAN | Connection Type |
|---|---|---|---|
| Cisco Meraki MX64W | Port 1 (Uplink) | – | — |
| CCTV | Port 2 | 200 (Camera) | Access — link to DVR |
| TrueNAS Backup Server | Port 3 | 201 (Backup Servers) | Access — single link to backup server |
| Printer | Port 4 | 202 (Test) | Access — link to printer |

### 2.3 Wireless Architecture

The Meraki MR56 (Wi-Fi 6) broadcasts four SSIDs, each mapped to a dedicated VLAN to enforce traffic segregation at the wireless layer:

- **SSID 1** → VLAN 30 (My Devices): Personal trusted devices only.
- **SSID 2** → VLAN 40 (Others Devices): Family and trusted guests.
- **SSID 3** → VLAN 50 (Guest): Fully isolated guest access.
- **SSID 4** → VLAN 80 (Test): Temporary SSID for testing and experimentation.

### 2.4 VLAN Design

The network is segmented into eight VLANs. Each VLAN serves a clearly defined trust zone, limiting the blast radius of any compromise and enforcing the principle of least privilege at the network level.

**Primary Site**

| ID | Name | Interface IP | DNS | Purpose |
|---|---|---|---|---|
| 20 | Servers | 172.17.20.1/26 | 172.17.20.3 (AdGuard) | Hosts TrueNAS and Proxmox. Only VLAN with direct upstream DNS to prevent AdGuard from interfering with service resolution. |
| 30 | My Devices | 192.168.30.1/29 | 172.17.20.3 (AdGuard) | Personal devices (PC, phone, tablet). Small /29 subnet intentionally sized for trusted personal use only. |
| 40 | Others Devices | 192.168.40.1/28 | 172.17.20.3 (AdGuard) | Family members with AdGuard DNS filtering. |
| 50 | Guest | 192.168.50.1/25 | 172.17.20.3 (AdGuard) | Isolated guest Wi-Fi. Larger subnet for visitor capacity. DNS filtered via AdGuard. |
| 60 | IoT | 192.168.60.1/27 | 172.17.20.3 (AdGuard) | Smart home and IoT devices isolated from trusted VLANs. |
| 80 | Test | 192.168.80.1/24 | Upstream (ISP) | Experimental and temporary deployments. Upstream DNS for full connectivity during testing. |
| 88 | Cybersecurity Capstone | 192.168.88.254/24 | Upstream (ISP) | Dedicated VLAN for university project. Outbound firewall rule blocks all inter-VLAN traffic. Team access via Tailscale VPN. |
| 100 | Management | 10.0.0.1/29 | Upstream (ISP) | Network infrastructure management only (switch, AP, servers). Tightly scoped /29 subnet. |

### 2.5 Firewall Policy

Firewall rules are applied on the Meraki MX67W. The primary custom rule enforces strict isolation for the Cybersecurity Capstone project:

> **Rule:** Block all traffic from VLAN 88 (Cybersecurity Capstone — 192.168.88.0/24) to all other VLANs.

**Rationale:** University project team members were granted access to a server on VLAN 88 via Tailscale VPN. This firewall rule ensures external collaborators — and any traffic originating from that VLAN — cannot traverse into the home network or reach any other internal resources.

### 2.6 VPN & Remote Access

- **Cisco Secure Client VPN** — Used for personal remote access to all internal resources and services. Terminates on the Meraki MX67W.
- **Meraki Site-to-Site VPN** — Connects the Primary Site MX and Secondary Site MX.
- **Tailscale VPN** — Deployed on the VLAN 88 server to provide controlled remote access for university project team members without exposing the broader home network.

---

## 3. Hardware

<!--
  Physical hardware / rack photo goes here.
  Add the image file to: assets/hardware-rack.jpg
  Then embed it with:
  ![Hardware Rack](assets/hardware-rack.jpg)
-->

| Role | Device | Description |
|---|---|---|
| Firewall / Router | Cisco Meraki MX67W | Dual-WAN capable security appliance and router. Handles NAT, firewall rules, VLAN routing, Cisco Secure Client VPN, and site-to-site VPN. Centralised policy management via Meraki Dashboard. |
| Core Switch | Cisco Meraki MS350-24P | 24-port PoE+ managed switch. Carries all VLANs as trunks to downstream devices. Powers the wireless access point via PoE. Connected to the MX67W on Management VLAN 100 (10.0.0.2/29). |
| Wireless Access Point | Cisco Meraki MR56 | Wi-Fi 6 access point. Broadcasts three production SSIDs mapped to VLANs 30, 40, and 50, plus a fourth SSID for testing. Connected on Management VLAN 100 (10.0.0.3/29). |
| NAS / Storage Server | TrueNAS SCALE | Dedicated network-attached storage server. Hosts all application data, databases, and network shares. Connected via link aggregation (ports 6 and 7) on VLAN 20 (172.17.20.2/26). |
| Virtualisation Node 1 | Proxmox VE (HP ProDesk Mini) | Cluster member for HA, running AdGuard Home, Wazuh, Netdata, Uptime Kuma, Docker/Nginx Proxy Manager, and other services. |
| Virtualisation Node 2 | Proxmox VE (Lenovo ThinkCentre M900 Tiny) | Cluster member for HA, running AdGuard Home, Wazuh, Netdata, Uptime Kuma, Docker/Nginx Proxy Manager, and other services. |
| Virtualisation Node 3 | Proxmox VE (Dell OptiPlex SFF) | Cluster member for HA, running AdGuard Home, Wazuh, Netdata, Uptime Kuma, Docker/Nginx Proxy Manager, and other services. |

### 3.1 Proxmox Nodes — Specifications

**Node 1 — HP ProDesk 600 G3 Mini**

| Component | Specification |
|---|---|
| CPU | Intel Core i5-7600T (4 Cores / 4 Threads, 2.8 GHz base) |
| RAM | 32 GB DDR4 |
| Storage | 512 GB NVMe + 500 GB HDD |
| Platform | Proxmox Virtual Environment (VE) |
| Network | 1 × 1 Gbps onboard NIC — VLAN 20 (Servers) |

**Node 2 — Lenovo ThinkCentre M900 Tiny**

| Component | Specification |
|---|---|
| CPU | Intel Core i5-6500T (4 Cores / 4 Threads, 2.5 GHz base) |
| RAM | 16 GB DDR4 |
| Storage | 256 GB M.2 SSD |
| Platform | Proxmox Virtual Environment (VE) |
| Network | 1 × 1 Gbps onboard NIC — VLAN 20 (Servers) |

**Node 3 — Dell OptiPlex 9020 SFF**

| Component | Specification |
|---|---|
| CPU | Intel Core i5-4570 (4 Cores / 4 Threads, 3.2 GHz base) |
| RAM | 16 GB DDR3 |
| Storage | 512 GB 2.5" SATA SSD |
| Platform | Proxmox Virtual Environment (VE) |
| Network | 1 × 1 Gbps onboard NIC — VLAN 20 (Servers) |

---

## 4. Storage — TrueNAS SCALE

The TrueNAS SCALE server acts as the central storage layer for the home lab, running ZFS-based storage pools to provide redundancy, data integrity, and flexible dataset management. It is connected to the core switch via link aggregation (LACP on ports 6 and 7), providing a 2 × 1 Gbps active-active uplink for improved throughput and redundancy.

| Component | Specification |
|---|---|
| CPU | Intel Core i7-7700 (4 Cores / 8 Threads, 3.6 GHz base) |
| RAM | 32 GB DDR4 |
| GPU | NVIDIA Quadro P620 2GB GDDR5 |
| Network | 2 × 1 Gbps onboard NIC — VLAN 20 (Servers) |
| Motherboard | Gigabyte GA-Z270P-D3 |
| Platform (OS) | TrueNAS Community Edition |
| Form Factor | Custom 2U Server Case |

<!--
  TrueNAS storage pool / dashboard screenshot goes here.
  Add the image file to: assets/truenas-pools.png
  Then embed it with:
  ![TrueNAS Storage Pools](assets/truenas-pools.png)
-->

### 4.1 Storage Pools

| Pool | Drives | Redundancy | Datasets | Purpose |
|---|---|---|---|---|
| Boot Pool | 2 × 128 GB (NVMe + SSD) | RAID 1 Mirror | TrueNAS OS | Operating system redundancy |
| App Config / DB | 2 × 256 GB NVMe | RAID 1 Mirror | Immich DB, Nextcloud DB, Paperless DB, Vaultwarden, Scrutiny | High-speed read/write for databases and app config |
| App Data | 2 × 4 TB HDD | RAID 1 Mirror | Immich photos, Nextcloud files, Paperless documents | Bulk storage for application data |
| Backups / Shares | 3 × 3 TB HDD | RAID Z1 (RAID 5 equiv.) | Windows SMB share, Proxmox NFS backup | Fault-tolerant backup and network share storage |

### 4.2 Network Shares

- **SMB Share** (`backups/windows-smb`) — Provides Windows-compatible file sharing for backup and file access. Also used for backing up iOS devices (iPhone/iPad).
- **NFS Share** (`backups/proxmox-nfs`) — Exposes a dedicated backup dataset to the Proxmox node via NFS, enabling automated Proxmox backup jobs (VZDump) to write directly to TrueNAS storage.

### 4.3 Backup Strategy (Current State)

The current backup posture provides storage redundancy through RAID/ZFS mirroring across all pools, but does not yet constitute a full backup strategy. RAID protects against drive failure, not against accidental deletion, ransomware, or site-level events.

- Datasets are mirrored using ZFS RAID 1 and RAID 5.
- Proxmox VM backups are scheduled via VZDump and written to the TrueNAS NFS share.
- A 3-2-1 backup strategy is planned as a future phase (see [Section 7](#7-future-roadmap)).

---

## 5. Services & Applications

All services are deployed across three platforms: LXC containers on Proxmox, Docker containers inside a dedicated Ubuntu VM on Proxmox, and native applications on TrueNAS SCALE. External access to internal services is handled exclusively through Nginx Proxy Manager, using Cloudflare-managed domain names and automated SSL/TLS certificate provisioning.

<!--
  Homarr dashboard / service overview screenshot goes here.
  Add the image file to: assets/homarr-dashboard.png
  Then embed it with:
  ![Homarr Dashboard](assets/homarr-dashboard.png)
-->

| Service | Deployment | Description |
|---|---|---|
| AdGuard Home | LXC Container | Network-wide DNS-based ad and tracker blocking. Primary DNS resolver for all VLANs except Servers, Management, Test, and Cybersecurity VLANs. |
| Nginx Proxy Manager | Docker Container | Reverse proxy for all internal services. Manages SSL/TLS certificates via Cloudflare DNS challenge. Enables domain-based routing without exposing ports. |
| Uptime Kuma | Docker (Ubuntu VM) | Service health monitoring and alerting. Tracks availability of all internal services with configurable notification channels. |
| Homarr | Docker (Ubuntu VM) | Centralised home lab dashboard — single pane of glass for accessing all services and monitoring status. |
| Wazuh | Docker (Ubuntu VM) | Security Information and Event Management (SIEM). Collects, correlates, and analyses security events across the environment. |
| Netdata | Docker (Ubuntu VM) | Real-time system performance monitoring — granular metrics for CPU, memory, disk I/O, and network across nodes. |
| Immich | TrueNAS App | Self-hosted photo and video backup platform. Replaces reliance on cloud photo services. Data stored on the dedicated 4 TB HDD pool. |
| Nextcloud | TrueNAS App | Self-hosted cloud storage and collaboration. File sync, sharing, and document access across all devices. |
| Paperless-ngx | TrueNAS App | Document management system with OCR. Digitises, indexes, and archives physical and digital documents. |
| Vaultwarden | TrueNAS App | Self-hosted Bitwarden-compatible password manager. Stores all credentials locally rather than relying on a third-party cloud. |
| Scrutiny | TrueNAS App | HDD/SSD S.M.A.R.T. monitoring. Tracks disk health and provides early warning of potential drive failures. |

### 5.1 Proxmox Workloads

**LXC Containers**
- **AdGuard Home** — Network-wide DNS resolver and ad/tracker blocker. All client VLANs point to this service as their primary DNS server.
- **Nginx Proxy Manager** — Reverse proxy and SSL termination. Manages Cloudflare DNS challenge-based certificate issuance for all internal services. *(Not yet in an LXC container — pending migration to the new server with more RAM and storage.)*

**Ubuntu VM (Docker Host)**

A dedicated Ubuntu virtual machine serves as the Docker host for all monitoring and operational services. Docker Compose is used for service management.

- Uptime Kuma — Availability monitoring
- Homarr — Unified home lab dashboard
- Wazuh — SIEM and security event management
- Netdata — Real-time system metrics
- Prometheus — Metrics scraping and time-series storage
- Grafana — Metrics visualisation and dashboarding

### 5.2 TrueNAS Applications

- Immich — Photo and video management
- Nextcloud — Cloud storage and file synchronisation
- Paperless-ngx — Document management with OCR
- Vaultwarden — Password manager (Bitwarden-compatible)
- Scrutiny — Disk health monitoring (S.M.A.R.T.)
- Proxmox Backup Server (as a container) — Backs up Proxmox cluster services

---

## 6. Troubleshooting & Problem Solving

The following section documents real issues encountered during the build and operation of the home lab. Each entry describes the problem, root cause analysis, and resolution steps taken. These troubleshooting experiences represent a core part of the practical learning this lab provides.

### TS-01 — SSID Client Isolation Blocking LAN Access

| | |
|---|---|
| **Affected System** | Cisco Meraki MR56 / VLAN 30 (My Devices) SSID |
| **Symptom** | All internal service dashboards and UIs were inaccessible from personal devices connected to the SSID mapped to VLAN 30, despite correct IP addresses and internet connectivity. |
| **Root Cause** | The "Block clients from accessing the LAN" option was enabled on the SSID in the Meraki dashboard, preventing wireless clients from communicating with wired network resources while still allowing internet access. |
| **Diagnosis** | A packet capture taken directly from the Meraki MX67W dashboard revealed that packets from the wireless client were not reaching the destination server — confirming the block was occurring at the wireless layer, not the firewall. |
| **Resolution** | Disabled the "Block clients from accessing the LAN" setting on the affected SSID. LAN access was restored immediately. |
| **Key Learning** | Packet captures are an effective first-response diagnostic tool. Client isolation (SSID-level) and firewall rules (MX-level) can silently drop traffic in ways that look identical to the client — always verify wireless-level settings before escalating to firewall rule investigation. |

### TS-02 — DNS Resolution Failure on Ubuntu VM

| | |
|---|---|
| **Affected System** | Ubuntu VM on Proxmox (Docker host) |
| **Symptoms** | `apt update` failed with "Temporary failure resolving". Docker image pulls failing. Homarr dashboard loading very slowly. General delays for all outbound connections from the VM. |
| **Root Cause** | The VM was configured with a static IP via Cockpit (NetworkManager), but no DNS servers were explicitly set. systemd-resolved had no upstream resolvers assigned to the active interface (`ens18`), causing all DNS queries to fail silently. `/etc/resolv.conf` pointed to the local stub (127.0.0.53), which had nothing to forward queries to. |
| **Diagnosis** | `resolvectl status` showed "Current Scopes: none" with no DNS servers listed. `ping 8.8.8.8` succeeded (IP routing OK); `ping google.com` failed (DNS broken) — proving the issue was DNS-only, not general connectivity. |
| **Resolution** | Manually configured DNS servers via Cockpit NetworkManager (Primary 1.1.1.1, Secondary 8.8.8.8). DNS resolution was restored immediately; all dependent services returned to normal. |
| **Key Learning** | Static IP configuration does not automatically inherit DNS settings — DNS must always be explicitly defined. Multiple network config layers (Netplan, NetworkManager, cloud-init) can conflict on Ubuntu. IP connectivity and DNS resolution are independent — always test both separately. |

### TS-03 — NFS Permission Denied (Proxmox → TrueNAS)

| | |
|---|---|
| **Affected System** | Proxmox VE — TrueNAS NFS storage integration |
| **Symptom** | Adding TrueNAS NFS share to Proxmox as a backup storage target failed with: `create storage failed: mkdir /mnt/pve/<storage>/dump: Permission denied` |
| **Root Cause** | NFS uses UID/GID-based permissions. Proxmox operates as root (UID 0), but TrueNAS applies root squashing by default — mapping root to the unprivileged "nobody" account — so Proxmox had no write access to create the backup directory structure. |
| **Resolution** | 1. Set dataset ownership to `root:wheel` in TrueNAS. 2. Configured the NFS share with Maproot User: `root` and Maproot Group: `wheel`. 3. Restricted NFS access to the Proxmox host IP. 4. Restarted the NFS service. 5. Added the NFS storage target in Proxmox (Datacenter → Storage → Add → NFS). |
| **Verification** | Mounted the NFS share manually on the Proxmox CLI and ran a touch test to confirm write access before adding it via the GUI. |
| **Key Learning** | NFS permission failures are almost always a UID/GID mismatch, not a traditional file permission issue. Understanding NFS root squashing is essential when integrating Linux-based storage with hypervisors — always validate with a manual CLI mount test before configuring via a GUI. |

### TS-04 — Nginx Proxy Manager SSL Certificate Issuance Failure

| | |
|---|---|
| **Affected System** | Nginx Proxy Manager LXC Container / Cloudflare DNS |
| **Symptom** | Provisioning a Let's Encrypt SSL certificate via Nginx Proxy Manager failed. The Cloudflare DNS challenge could not be validated, leaving services inaccessible via HTTPS on the custom domain. |
| **Root Cause** | The Cloudflare API token supplied to Nginx Proxy Manager lacked the correct permissions (requires `Zone:Read` and `Zone:DNS:Edit` scoped to the specific zone). An incorrectly scoped or expired token causes the DNS-01 challenge to fail silently. |
| **Resolution** | Generated a new Cloudflare API token with the correct `Zone:Read` and `DNS:Edit` permissions scoped to the target domain zone, updated it in Nginx Proxy Manager, and re-triggered the certificate request. |
| **Key Learning** | DNS-01 challenges require precise API token scoping — over- and under-permissioned tokens both cause failures. Always scope API tokens to the minimum required permissions and verify zone scope when using Cloudflare with automated certificate tooling. |

### TS-05 — Laptop Intermittently Fails to Connect to Personal SSID

| | |
|---|---|
| **Affected System** | Personal laptop / VLAN 30 (My Devices) SSID — Cisco Meraki MR56 |
| **Symptom** | The laptop intermittently fails to associate with the SSID mapped to VLAN 30 (/29 subnet). Non-reproducible on demand; other devices on the same SSID are unaffected. |
| **Status** | 🔍 Under investigation. The small /29 subnet (6 usable host addresses) is a suspected contributing factor — if the DHCP pool is exhausted or a lease is held by a stale device, the laptop cannot obtain an IP address. |
| **Next Steps** | 1. Review Meraki Dashboard event log for association failures. 2. Check active DHCP leases on VLAN 30 — verify available addresses in the /29 pool. 3. If pool exhaustion is confirmed, consider a static IP or expanding the subnet. 4. Check for 802.11 band steering or roaming issues on the MR56. |

---

## 7. Future Roadmap

> 🚧 Coming soon.

---

## 8. Appendix

### 8.1 Technologies & Tools Referenced

| Technology | Category |
|---|---|
| Cisco Meraki MX67W | Network Security / Firewall / VPN |
| Cisco Meraki MS350-24P | Managed Switching / VLAN Trunking |
| Cisco Meraki MR56 | Wireless (Wi-Fi 6) |
| Proxmox VE | Virtualisation / Hypervisor |
| TrueNAS SCALE | Network Attached Storage / ZFS |
| Docker | Containerisation |
| AdGuard Home | DNS Filtering / Network Security |
| Nginx Proxy Manager | Reverse Proxy / SSL/TLS Management |
| Wazuh | SIEM / Security Monitoring |
| Prometheus + Grafana | Observability / Metrics |
| Netdata | Real-time System Monitoring |
| Uptime Kuma | Service Availability Monitoring |
| Immich | Self-Hosted Photo Management |
| Nextcloud | Self-Hosted Cloud Storage |
| Vaultwarden | Self-Hosted Password Management |
| Paperless-ngx | Document Management / OCR |
| Tailscale | Zero-Trust VPN / Mesh Networking |
| Cisco Secure Client | Enterprise Remote Access VPN |
| Cloudflare | DNS / SSL Certificate Automation |
| ZFS (via TrueNAS) | Storage / Data Integrity / RAID |

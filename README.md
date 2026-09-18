# 🛡️ Enterprise-Grade Multi-Tier Homelab & Edge Routing Infrastructure

[![Hypervisor](https://img.shields.io/badge/Hypervisor-Proxmox%20VE%209.2-E57000?logo=proxmox&logoColor=white)](https://proxmox.com)
[![Workstation OS](https://img.shields.io/badge/Workstation-CachyOS%20(Arch)--Linux-blue?logo=archlinux&logoColor=white)](https://cachyos.org)
[![Router](https://img.shields.io/badge/Edge%20Router-OpenWrt%20(Raspberry%20Pi)-00D7D7?logo=openwrt&logoColor=white)](https://openwrt.org)
[![Security](https://img.shields.io/badge/Security-LUKS%20%7C%20UFW%20%7C%20AdGuard-brightgreen)](#)
[![Mesh VPN](https://img.shields.io/badge/Mesh%20VPN-Tailscale-1E293B?logo=tailscale&logoColor=white)](https://tailscale.com)

Welcome to the documentation of my self-hosted homelab, edge routing, and virtualization environment. This repository details the physical architecture, subnets, firewall rules, and zero-trust mesh topologies I designed and maintain for continuous hands-on learning in **IT Systems Integration (Fachinformatiker für Systemintegration - FISI)**.

---

## 🗺️ High-Level Network Topology

```
                                [ WAN / Internet ]
                                         │
                         ┌───────────────┴───────────────┐
                         │   ISP Main Router / Gateway   │
                         │        192.168.100.1          │
                         └───────────────┬───────────────┘
                                         │
        ┌────────────────────────────────┼────────────────────────────────┐
        ▼                                ▼                                ▼
┌──────────────────────┐     ┌──────────────────────┐        ┌──────────────────────┐
│  Edge Router / DNS   │     │ Hypervisor Server    │        │ Personal Workstation │
│  Raspberry Pi 4      │     │ Proxmox VE 9.2       │        │ SafeHouse (CachyOS)  │
│  (OpenWrt)           │     │ Core i7 / 31 GB RAM  │        │ Ryzen 7 / 30 GB RAM  │
│  192.168.100.2 (WAN) │     │ 192.168.100.66       │        │ 192.168.100.95       │
│  10.0.0.1 (LAN)      │     └──────────┬───────────┘        └──────────┬───────────┘
│  • AdGuard Home DNS  │                │                               │
│  • Tailscale Gateway │                ▼                               ▼
└──────────┬───────────┘      [ Guests & Workloads ]         [ Host Security & Disks ]
           │                  • CT 100: Pi-hole / DNS        • LUKS Encrypted NVMe
           ▼                  • CT 101: Tailscale Node       • UFW Strict Firewall
  ┌──────────────────┐        • VM 100: ParrotOS (KVM)       • Btrfs Snapper Backups
  │ Dummy Wi-Fi AP   │        • VM 200: Debian 12 Docker     • Docker Engine
  │ (DHCP Subnet)    │
  └────────┬─────────┘
           │
 ┌─────────┴─────────┐
 ▼                   ▼
[ Mobile Clients ]  [ Laptop / End Users ]
(Phones / Tablets)  (MacBook Air M3: 192.168.100.5)
```

---

## 🏛️ Node Inventory & Architecture Specifications

| Node / Host | IP Address | Operating System | Hardware / Role | Security & Storage Features |
| :--- | :--- | :--- | :--- | :--- |
| **Main Gateway** | `192.168.100.1` | Embedded Linux | ISP Optical/VDSL Router | NAT, WAN Edge, Port Filtering |
| **OpenWrt Router** | `192.168.100.2` (WAN)<br>`10.0.0.1` (LAN) | OpenWrt Linux | **Raspberry Pi** Edge Router & Gateway | **AdGuard Home** DNS sinkhole, WireGuard/Tailscale |
| **Proxmox Node** | `192.168.100.66` | Proxmox VE 9.2 (Debian 13) | Intel Core i7-1165G7 · 31 GB RAM | LVM-thin storage, automated ZSTD snapshot backups |
| **SafeHouse** | `192.168.100.95` | CachyOS (Arch-based) | AMD Ryzen 7 7700X · 30 GB RAM · RX 6800 | **LUKS** full-disk encryption, **UFW**, Btrfs subvols |
| **Mobile Client** | `192.168.100.5` | macOS (Darwin 25) | MacBook Air (13-inch, Apple M3, 16 GB) | APFS encrypted, remote SSH management station |
| **Dummy AP** | Dynamic / Auto | Access Point Firmware | Dedicated Wireless Access Point | Segregated wireless access for IoT and mobile users |

---

## 🛡️ Security Layers & Implementation Details

### 1. Dual-Tier Network Segmentation
* The **192.168.100.0/24** subnet acts as the core administrative and hypervisor backbone connecting `SafeHouse`, the `Proxmox` virtualization node, and the OpenWrt router's WAN uplink.
* The **10.0.0.0/24** isolated subnet is managed directly by the **Raspberry Pi running OpenWrt**, providing dedicated DHCP assignment, traffic isolation, and Wi-Fi client isolation via the secondary access point.

### 2. Network-Wide DNS Sinkholing & Ad-Blocking
* **AdGuard Home** is deployed on the OpenWrt Raspberry Pi to enforce network-wide tracking prevention, malware domain blocking, and custom split-horizon DNS routing (`.home` and `.lan` records).
* Internal services (e.g. `pve.home`, `safehouse.home`) resolve locally without querying external public resolvers.

### 3. Zero-Trust Remote Mesh (Tailscale)
* All primary endpoints (`SafeHouse`, `Proxmox`, `MacBook Air`, and mobile devices) are enrolled in a private **Tailscale** overlay network.
* Allows end-to-end encrypted WireGuard tunneling back into the home LAN without opening public ports on the ISP router or exposing services to WAN attacks.

### 4. Host Defense: UFW & Storage Encryption
* **Host Firewalls:** `SafeHouse` runs an active **UFW (Uncomplicated Firewall)** configuration enforcing the principle of least privilege (inbound default DENY, explicit whitelisting for SSH port 22 and internal Docker reverse proxies).
* **Storage Encryption (LUKS):** Critical workstation NVMe drives (Samsung 990 PRO and Kingston NVMe) utilize **LUKS (Linux Unified Key Setup)** volume encryption to protect sensitive data at rest.

---

## 🛠️ Virtualization & Workloads (Proxmox VE 9.2)

The dedicated hypervisor (`192.168.100.66`) hosts isolated virtual machines and lightweight Linux containers (LXC):

* **VM 100 — ParrotOS (KVM):** Dedicated security testing and penetration-testing environment, isolated on its own virtual NIC.
* **VM 200 — Debian 12 Docker Host:**
  * **Uptime Kuma:** Continuous ping and HTTP health monitoring of all infrastructure nodes and edge gateways.
  * **IT-Tools:** Local suite of developer and network administrator utilities.
  * **Homepage:** Consolidated operational dashboard displaying real-time metrics, node uptime, and direct service shortcuts.

---

## 💡 Practical Skills Demonstrated

This homelab serves as the living workbench for my preparation toward the **German IHK Vocational Examination (*Fachinformatiker für Systemintegration*)**:
* **Lernfeld 3 & 9:** Subnet routing, VLAN isolation, DNS sinkholes (AdGuard), Tailscale mesh VPN, TCP/IP diagnostics (`ss`, `dig`, `traceroute`, `ip`).
* **Lernfeld 4 & 7:** LUKS disk encryption, UFW firewall rule orchestration, LVM and Btrfs snapshot management.
* **Lernfeld 8 & 11:** Type-1 hypervisor operations (Proxmox VE), LXC container lifecycle, Docker Compose microservices orchestration, automated ZSTD snapshot backups.

---

*Authored and maintained by Azeddine Taleb Ahmed · Last updated: September 2026*

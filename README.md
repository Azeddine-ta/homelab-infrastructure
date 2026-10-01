# 🛡️ Homelab mit zwei Zonen und Heimnetz-Routing

[![Hypervisor](https://img.shields.io/badge/Hypervisor-Proxmox%20VE%209.2-E57000?logo=proxmox&logoColor=white)](https://proxmox.com)
[![Workstation](https://img.shields.io/badge/Workstation-CachyOS%20(Arch)--Linux-blue?logo=archlinux&logoColor=white)](https://cachyos.org)
[![Router](https://img.shields.io/badge/Edge%20Router-OpenWrt%20(Raspberry%20Pi%204B)-00D7D7?logo=openwrt&logoColor=white)](https://openwrt.org)
[![Gateway](https://img.shields.io/badge/GPON%20Gateway-Huawei%20HG8145X6--10-red)](#)
[![Access Point](https://img.shields.io/badge/AP-TP--Link%20Archer%20C6%20v2.80-008080)](#)
[![Security](https://img.shields.io/badge/Security-LUKS2%20%7C%20UFW%20%7C%20AdGuard-brightgreen)](#)
[![Mesh VPN](https://img.shields.io/badge/Mesh%20VPN-Tailscale%20(WireGuard)-1E293B?logo=tailscale&logoColor=white)](https://tailscale.com)



<p align="center">
  <img src="homelab-Diagram.jpg" alt="Topologie des Dual-Zone-Homelabs und des Enterprise-Edge-Routings" width="100%">
</p>


Willkommen in der Dokumentation meines produktiv genutzten Homelabs, meines Heimnetzes mit zwei Zonen und meiner Virtualisierungsumgebung. Dieses Repository beschreibt die physische Architektur, die Subnetze, die Firewall-Regeln und die Zero-Trust-Mesh-Topologie. Ich habe alles selbst geplant und pflege es, um in der Praxis weiter zu lernen – in der **IT-Systemadministration**.

---

## 🗺️ Netzwerktopologie

```
                                  [ WAN / Glasfaser ]
                                             │
                        ┌────────────────────┴────────────────────┐
                        │        Haupt-Router + Wi-Fi-6-AP        │
                        │       Huawei HG8145X6-10 Gateway        │
                        │          192.168.100.1 / DHCP           │
                        └────────────────────┬────────────────────┘
                                             │
               ┌─────────────────────────────┼─────────────────────────────┐
               │ 1 Gb/s Kupfer               │ 1 Gb/s Kupfer               │ Wi-Fi 6 (802.11ax)
               ▼                             ▼                             ▼
    ┌──────────────────────┐      ┌──────────────────────┐      ┌──────────────────────┐
    │  Privat-Workstation  │      │  Virtualisierungshost│      │  Mobile Workstation  │
    │  SafeHouse (CachyOS) │      │  homelab (Proxmox VE)│      │  MacBook Air (M3)    │
    │  192.168.100.95/24   │      │  192.168.100.66/24   │      │  192.168.100.5/24    │
    │  • eno1: 1 Gb/s voll │      │  • vmbr0 Linux-Bridge│      │  • en0: Ch 48/80 MHz │
    │  • LUKS2 + Btrfs     │      │  • TCP/22 SSH offen  │      │  • WPA2 / 5 GHz      │
    │  • UFW: 631/53317 auf│      │  • PVE-Firewall / nft│      │  • Tailscale-Client  │
    └──────────┬───────────┘      └──────────────────────┘      └──────────┬───────────┘
               │                                                           │
               │                      1 Gb/s Eth0-Uplink                   │
               └─────────────────────────────┬─────────────────────────────┘
                                             ▼
                            ┌─────────────────────────────────┐
                            │  Raspberry Pi 4B (OpenWrt)      │
                            │  WAN: 192.168.100.2 (eth0)      │
                            │  LAN: 10.0.0.1 (br-lan/eth1)    │
                            │  • Linux 6.12.x aarch64  │
                            │  • AdGuard Home DNS (:53)       │
                            │  • OpenWrt firewall4 (nftables) │
                            │  • Tailscale-Gateway-Knoten     │
                            └────────────────┬────────────────┘
                                             │
                                             │ 1 Gb/s Eth1-Trunk
                                             ▼
                            ┌─────────────────────────────────┐
                            │  Archer C6 v2.xx (Dummy-AP)     │
                            │  10.0.0.3 / Layer-2-Bridge      │
                            │  f0:09:0d:xx:xx:xx              │
                            └────────────────┬────────────────┘
                                             │
                                             │ WLAN-Bridge
                                             ▼
                            ┌─────────────────────────────────┐
                            │  Isolierte Client-Zone (LAN-B)  │
                            │  Pool: 10.0.0.0/24              │
                            │  • Mobilgeräte                  │
                            │  • Benutzer & Gastgeräte        │
                            └─────────────────────────────────┘

═══════════════════════════════════════════════════════════════════════════════════════
   🔐 OVERLAY-NETZWERK: Tailscale-Mesh (100.64.0.0/10 WireGuard / UDP 41641)
   • SafeHouse:        100.***.***.*** (ONLINE, fd7a:115c:a1e0::****)
   • OpenWrt-Gateway:  100.***.***.*** (ONLINE, TCP/xxxx Verwaltungsport)
   • MacBook Air M3:   100.***.***.*** (konfigurierter Client-Knoten)
═══════════════════════════════════════════════════════════════════════════════════════
```

---

## 🏛️ Geräteübersicht und Hardware-Spezifikationen

| Knoten / Hostname | IP / Subnetz | Betriebssystem / Firmware | Hardware / Rolle | Sicherheit & Speicher |
| :--- | :--- | :--- | :--- | :--- |
| **Haupt-Gateway** | `192.168.100.1/24` | Huawei VOS | **Huawei HG8145X6-10** (GPON-ONT + Wi-Fi-6-AP) | Glasfaser-Abschluss (WAN), DHCP-Server, primäres NAT |
| **SafeHouse** | `192.168.100.95/24` | **CachyOS** (Linux 7.x.x, Arch) | **AMD Ryzen 7 7700X** · 30 GB RAM · RX 6800 XT | **LUKS2**-Festplattenverschlüsselung (Full-Disk), **Btrfs**-Subvolumes für root/home, **UFW** aktiv (eingehend standardmäßig gesperrt, Ports 631/53317/52345 offen) |
| **homelab** | `192.168.100.66/24` | **Proxmox VE 9.x.xx** (Debian 13) | **Intel Core i7-1165G7** · 31 GB RAM · 256 GB NVMe | **vmbr0**-Bridge, PVE-Firewall (nftables, ICMP gefiltert), TCP/22 SSH mit Schlüssel-Authentifizierung, LVM-thin |
| **OpenWrt Edge** | `192.168.100.2` (WAN)<br>`10.0.0.1` (LAN) | **OpenWrt** (Linux 6.xx.xx aarch64) | **Raspberry Pi 4B** (4 GB RAM, zwei Netzwerkkarten: eth0 + eth1) | **AdGuard Home** (:53, DNS-Sinkhole), **firewall4** (nftables), `br-lan = bridge{eth1, phy0-ap0}`, Tailscale-Gateway |
| **Archer C6** | `10.0.0.3/24` | TP-Link Hersteller-Firmware | **Archer C6 v2.xx** (im Access-Point-Bridge-Modus) | MAC `f0:09:0d:xx:xx:xx`, sendet das isolierte Client-WLAN für `10.0.0.0/24` |
| **MacBook Air** | `192.168.100.5/24` | **macOS**  | **Apple M(x)** · 16 GB Unified Memory | Verbunden über `en0` (802.11ax, 5 GHz, Kanal 48 / 80 MHz, WPA2), Tailscale-Client |

---

## 🛡️ Netzwerkarchitektur und Sicherheit

### 1. Netzwerksegmentierung in zwei Zonen (LAN-A & LAN-B)
* **LAN-A (`192.168.100.0/24`): Vertrauenswürdiges Infrastruktur-Netz**
  * Hier sind das Huawei-GPON-Gateway, die **CachyOS-Workstation (SafeHouse)**, der **Proxmox-VE-Hypervisor**, das **MacBook Air M3** und der WAN-Anschluss (`eth0`) des Raspberry Pi verbunden.
  * Direkte Kupferverbindungen mit 1 Gb/s im Vollduplex-Betrieb sorgen für geringe Latenz und hohe Bandbreite bei der Hypervisor-Verwaltung und der lokalen Speicherreplikation.
* **LAN-B (`10.0.0.0/24`): Isoliertes Netz für Clients und Benutzer**
  * Wird komplett vom **Raspberry Pi 4B mit OpenWrt** verwaltet.
  * Der physische Port `eth1` führt per Kabel zum **TP-Link Archer C6 v2.xx**. Dieser arbeitet als reiner Layer-2-Access-Point (Bridge).
  * Smartphones, Gast-Tablets und IoT-Geräte sind physisch von den Verwaltungsschnittstellen des Hypervisors getrennt.

### 2. DNS-Sinkhole für das ganze Netz (AdGuard Home)
* AdGuard Home läuft direkt auf dem OpenWrt-Raspberry-Pi (`:53`). Es dient als maßgeblicher Upstream-Resolver für LAN-B und für interne DNS-Überschreibungen.
* Es blockiert Telemetrie und Tracking im ganzen Netz und bietet Split-Horizon-DNS (Namensauflösung für `.home` / `.lan`) – ohne Software auf den Clients.

### 3. Mehrschichtiger Host-Schutz (UFW & PVE-Firewall)
* **SafeHouse-Workstation:** Gesichert mit **UFW**. Eingehender Verkehr wird standardmäßig blockiert (DENY). Nur wenige erlaubte Ports (CUPS 631, interne P2P-Ports) sind erreichbar.
* **Proxmox-VE-Knoten:** Geschützt durch die eingebaute **pve-firewall (nftables)**. ICMP-Ping wird gefiltert. Der Administrationszugriff ist nur mit Ed25519-SSH-Schlüsseln und über die verschlüsselte HTTPS-Weboberfläche (:8006) möglich.

### 4. Zero-Trust-Overlay-Mesh (Tailscale WireGuard)
* Die eingebundenen Knoten nutzen den CGNAT-Bereich `100.64.0.0/10` und verschlüsseln den Verkehr mit WireGuard (UDP 41641).
* So ist eine durchgehend verschlüsselte Fernwartung von mobilen Geräten außerhalb des lokalen Netzes möglich – ohne Port-Weiterleitung und ohne interne IP-Adressen im öffentlichen Internet zu zeigen.

---

## 💡 one more step towards my dream 🇩🇪 FISI



---

*Verfasst und gepflegt von Azeddine TA · Zuletzt aktualisiert: Oktober 2026*

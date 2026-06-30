# Network Foundation — pfSense Build Log (Phase 1)

**Status:** ✅ Complete
**Date:** June 2026
**Phase:** 1 of the Home NOC Monitoring Lab — segmented firewall + base network

---

## Objective

Stand up a pfSense firewall in a virtual lab with three segments (WAN / LAN / DMZ), confirm routing and management access, and establish the foundation that the rest of the NOC monitoring lab is built on.

---

## Environment

| Component | Detail |
|---|---|
| Host CPU | Intel Core i5-10400F @ 2.90 GHz |
| Hypervisor | VMware Workstation Pro 25H2 (Windows host) |
| Storage | WD Black SN7100 NVMe |
| Firewall OS | pfSense CE **2.8.1-RELEASE** (amd64), FreeBSD 15.0 base |
| VM resources | 2 GB RAM, 1 vCPU, 16 GB disk (thin), 3 vNICs |
| Home network | `10.0.0.0/24`, gateway `10.0.0.1` |

---

## Architecture

**IP scheme**

| Segment | Interface | VMware vSwitch | Subnet | pfSense IP | Notes |
|---|---|---|---|---|---|
| WAN | em0 | VMnet0 (Bridged → Realtek 2.5GbE) | `10.0.0.0/24` | `10.0.0.38` (DHCP) | Uplink to home LAN / internet |
| LAN | em2 | VMnet2 (Host-only) | `10.10.10.0/24` | `10.10.10.1` | Monitored devices live here; DHCP pool `.100–.199` |
| DMZ (OPT1) | em1 | VMnet3 (Host-only) | `10.10.20.0/24` (planned) | `10.10.20.1` (planned) | Reserved for Phase 2+ |

**Interface assignment note:** because adapters were removed/re-added during setup, VMware's adapter numbers did not match list order. Final verified mapping is em0→WAN, **em2→LAN**, em1→OPT1. Assignment was confirmed by VMnet binding, not by adapter list position.

---

## Build steps performed

1. Created the VM in VMware (Custom): FreeBSD 14 64-bit guest profile, 2 GB RAM, 1 vCPU, 16 GB thin disk on NVMe.
2. Defined virtual networks in the Virtual Network Editor: VMnet0 bridged (explicitly bound to the physical Realtek NIC), VMnet2 and VMnet3 as host-only segments with VMware's local DHCP **disabled** (pfSense owns DHCP).
3. Attached three vNICs mapped to VMnet0 / VMnet2 / VMnet3.
4. Booted the Netgate online installer, selected **Install CE**, ZFS filesystem. (Note: pfSense now ships a single online installer that fetches packages during install; WAN connectivity is required at install time.)
5. Assigned interfaces at the console: WAN=em0, LAN=em2, OPT1=em1.
6. Set addressing via console option 2: WAN = DHCP (received `10.0.0.38`), LAN = static `10.10.10.1/24` with DHCP server enabled (`.100–.199`).
7. Reached the web GUI at `https://10.10.10.1`, completed the setup wizard, set a non-default admin password, set timezone.
8. Verified on the dashboard: both interfaces up at 1000baseT full-duplex, WAN online with valid DNS, system reporting latest version.

---

## Issues encountered & resolved

**1. Adapter numbering mismatch (LAN/DMZ ambiguity).**
Removing and re-adding network adapters caused VMware to reuse slot numbers out of order, so the device list (Adapter 1/3/2) did not reflect the actual em0/em1/em2 mapping. *Resolution:* mapped interfaces by their VMnet binding rather than list order, and confirmed LAN landed on VMnet2.

**2. Host could not reach the LAN GUI (`ERR_CONNECTION_TIMED_OUT`).**
The host's VMnet2 adapter held an APIPA address (`169.254.x.x`) because VMware had VMnet2 on a `192.168.40.0` subnet while pfSense LAN was `10.10.10.0/24` — a subnet mismatch, and no DHCP lease was being served to the host adapter. *Resolution:* assigned the host's "VMware Network Adapter VMnet2" a static IP of `10.10.10.5/24` (gateway `10.10.10.1`), placing the host directly on the lab subnet. `ping 10.10.10.1` then succeeded and the GUI loaded.

*Takeaway:* host management access to an isolated lab segment requires the host's virtual adapter and the firewall's LAN interface to share the same subnet. Static-assigning the host adapter is the most reliable approach.

---

## Verification (definition of done)

- [x] pfSense GUI reachable at `https://10.10.10.1`
- [x] WAN online via DHCP (`10.0.0.38`), valid DNS resolving
- [x] LAN interface up and serving DHCP on `10.10.10.0/24`
- [x] Admin password changed from default
- [x] VM snapshot `01-base-pfsense-working` captured
- [x] Dashboard screenshot captured for documentation

---

## Next steps

- **Phase 2a — Routing proof:** deploy a Linux VM on VMnet2, confirm it pulls a `10.10.10.x` DHCP lease and reaches the internet through pfSense (NAT working).
- **Phase 2b — Monitoring path:** connect the LibreNMS monitoring server (separate Pop!_OS laptop) to the lab via a static route through the pfSense WAN IP, plus a WAN firewall rule permitting management traffic from the laptop (requires unchecking "Block private networks" on WAN, since the home net is RFC1918).
- **Phase 2c — Instrumentation:** enable SNMP on pfSense and endpoints; add devices to LibreNMS.
- **Phase 3+** — dashboards, alerting, syslog/NetFlow, simulated incidents.

---

## Skills demonstrated

Virtual network design and segmentation · firewall installation and configuration (pfSense) · interface assignment and IP addressing · DHCP server configuration · Layer 3 troubleshooting (subnet mismatch, APIPA, static addressing) · hypervisor networking (VMware vSwitches, bridged vs host-only) · methodical verification and documentation.

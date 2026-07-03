# Home NOC Monitoring Lab

A network operations monitoring environment built from scratch to demonstrate NOC analyst skills: network design, firewall configuration, device monitoring, alerting, troubleshooting, and documentation.

**Author:** Shawn Emmanuel · [LinkedIn](https://linkedin.com/in/shawn-emmanuel-940048340)

---

## Architecture

A segmented virtual network running under VMware Workstation Pro, monitored by a dedicated LibreNMS server on separate physical hardware — mirroring real NOC design, where the monitoring platform survives failures in the network it watches.

| Component | Role | Address |
|---|---|---|
| pfSense CE 2.8.1 | Edge firewall / router (WAN, LAN, DMZ) | WAN `10.0.0.50` · LAN `10.10.10.1/24` |
| lab-server-01 (Ubuntu Server) | Monitored LAN endpoint | `10.10.10.10` (DHCP static mapping) |
| Pop!_OS laptop | Dedicated monitoring server (LibreNMS) | Home LAN, routed into lab via pfSense |
| DMZ segment | Reserved for segmentation phase | `10.10.20.0/24` (planned) |

**Stack:** pfSense · Ubuntu Server · LibreNMS · SNMP · syslog · NetFlow · VMware Workstation Pro

---

## Progress

- [x] **Phase 1 — Network foundation:** pfSense installed and configured; WAN/LAN/DMZ segmentation; DHCP; management access — [build log](docs/01-network-foundation-buildlog.md)
- [x] **Phase 2a — Endpoint deployment:** Ubuntu server on the LAN; DHCP static mapping; routing/NAT verified; remote SSH management
- [ ] **Phase 2b — Monitoring connectivity:** static WAN, firewall policy, and routing to connect the monitoring server *(in progress)*
- [ ] **Phase 2c — SNMP + LibreNMS:** device polling and discovery
- [ ] **Phase 3 — Dashboards & visualization**
- [ ] **Phase 4 — Alerting** (device-down, interface, CPU, bandwidth rules with notifications)
- [ ] **Phase 5 — Centralized syslog & NetFlow**
- [ ] **Phase 6 — Simulated incidents & runbooks** (outage, saturation, flap)
- [ ] **Phase 7 — DMZ policy enforcement & Cisco device monitoring (GNS3)**

---

## Verified so far

**Endpoint routing through the firewall** — gateway, internet (NAT), and DNS from the LAN endpoint:

![Routing verification](screenshots/2a-routing-test-pings.png)

**Address reservation** — DHCP static mapping pinning the endpoint for reliable monitoring:

![DHCP static mapping](screenshots/2a_-dhcp-static-mapping.png)

**Remote management** — SSH from the host, across the lab segment:

![SSH remote access](screenshots/2a-remote-access.png)

---

## Troubleshooting log (selected)

Real problems i encountered and resolved during the build — full detail in the phase build logs:

- **Host couldn't reach the firewall GUI (timeout):** traced to a subnet mismatch — the hypervisor's host-only adapter sat on a different network than the firewall's LAN, leaving the host with an APIPA address. Resolved by statically addressing the host adapter into the lab subnet. 
- **DHCP-to-static WAN conversion blocked:** pfSense's system-managed DHCP gateway could not be deleted or replaced while the interface still referenced it. Resolved by releasing the interface from DHCP first (static, no gateway), then creating and attaching the new gateway object. Likely just misunterstanding of how pfsense manages gateways, but still note worthy
- **SSH sessions hanging after authentication:** kernel logs on the endpoint revealed RCU CPU stalls — the VM was being starved of CPU time, so sshd could not complete session setup. Diagnosed via `ssh -v` (healthy handshake, silent post-auth) correlated with `rcu_preempt detected stalls` in the console. 

---

## Why this project

Built to demonstrate the core loop of NOC work — **monitor, detect, triage, resolve, document** — on real (virtualized) infrastructure, targeting NOC Technician / Analyst roles. Each phase ends in verifiable checkpoints and is documented as a build log with evidence.


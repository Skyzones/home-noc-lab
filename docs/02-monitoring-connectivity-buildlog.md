# Phase 2c — Troubleshooting Log

Running log of problems hit, investigated, and resolved during the SNMP/LibreNMS build.
Environment: pfSense CE 2.8.1, Ubuntu 26.04 (lab-server-01), Pop!_OS (poller), VMware Workstation Pro.

---

## Issue #1 — `dhclient: command not found` after creating DHCP static mapping

**Date:** 2026-07-31

**Symptom:** Created a DHCP static mapping in pfSense binding lab-server-01's MAC (`00:0c:29:43:94:d2`) to 10.10.10.10. To force the client to pick up the reservation rather than wait out its existing lease, ran the conventional lease-release/renew command:

```
sudo dhclient -r ens33 && sudo dhclient ens33
sudo: 'dhclient': command not found
```

**Investigation:** The binary genuinely wasn't present — not a PATH issue. Checked what was actually managing the interface: netplan on this release renders to **systemd-networkd**, which has its own built-in DHCP client. ISC dhclient was deprecated upstream (ISC ended maintenance) and Ubuntu dropped it from the default install starting with 24.04.

**Root cause:** Followed muscle memory from older Ubuntu/Debian releases. The tool no longer exists on this distro version.

**Fix:**

```
sudo systemctl restart systemd-networkd
```

Confirmed with `ip a`: address still 10.10.10.10, with a fresh lease timer (`valid_lft 7188sec` — pfSense's 2-hour default counting down from the top), proving a new DHCP transaction had occurred and the reservation was honored.

**Note:** The address still displays as `dynamic`. That is expected and correct — a static mapping is still DHCP, just with a guaranteed answer. What matters for monitoring is that the address survives lease expiry and reboots.

**Lesson:** Watch for distro version drift. Commands from tutorials written against older releases will fail silently-ish, and the replacement is often a different subsystem entirely rather than a renamed binary.

**Time lost:** ~10 min

---

## Issue #2 — Monitoring server is not on the lab subnet

**Date:** 2026-07-31

**Symptom:** Not an error — a design assumption that turned out to be wrong. Ran `ip a` on the Pop!_OS monitoring server expecting an address in 10.10.10.0/24. Got:

- `wlp1s0`: 10.0.0.82/24 — the home WiFi network
- `enp2s0`: `NO-CARRIER` — ethernet unplugged

The poller sits on the **home LAN**, on the **WAN side** of pfSense. Its polling targets sit behind the firewall on 10.10.10.0/24. Two different subnets with a firewall in between.

**Investigation:** Confirmed pfSense's interface assignments from the dashboard: WAN 10.0.0.50 (bridged adapter, same subnet as the poller — so no VMware NAT layer to contend with), LAN 10.10.10.1. Confirmed lab-server-01's default route points at 10.10.10.1 via `ip route`.

Three options considered:

1. **Static route + explicit firewall rules** — leave the poller where it is, teach it how to reach the lab subnet, and permit the traffic inbound on pfSense.
2. **Cable the laptop into the lab segment** via `enp2s0` — makes the problem disappear, depends on physical reachability of the VMnet.
3. **Move LibreNMS into a VM on the lab subnet** — architecturally simplest, but discards the Phase 2a/2b work done on the Pop!_OS box.

Chose option 1. It preserves prior phases, and a monitoring server that reaches its targets across a routed firewall boundary is a more realistic enterprise topology than everything sitting flat on one segment.

**Root cause:** Assumed a flat lab network without verifying the poller's actual interface configuration.

**Fix — three parts:**

1. Static route on Pop!_OS so it knows 10.10.10.0/24 isn't out on the internet:

```
sudo ip route add 10.10.10.0/24 via 10.0.0.50        # test
sudo nmcli connection modify "<wifi>" +ipv4.routes "10.10.10.0/24 10.0.0.50"   # persistent
```

2. **Interfaces → WAN → unchecked "Block private networks and loopback addresses."** This rule silently drops all RFC1918-sourced traffic arriving on WAN, which is exactly what the poller is. Nothing works until it's off. "Block bogon networks" left enabled.

3. **Firewall → Rules → WAN**, pass rule: source `10.0.0.82` → destination `10.10.10.0/24`. Left deliberately broad during the build phase to avoid debugging my own firewall while debugging SNMP; scheduled for tightening to explicit ports once polling is verified.

Verified: Pop!_OS successfully pings both 10.10.10.1 and 10.10.10.10.

**Two things worth understanding from this:**

- **pfSense filters on the interface where traffic enters.** These packets arrive on WAN, so WAN rules govern them. LAN rules apply only to traffic originating *from* devices on 10.10.10.0/24 — irrelevant to this flow.
- **No port forward or NAT rule was needed.** This is routing to real addresses, not translation. pf is stateful, so once a rule passes the initial packet, return traffic is permitted automatically without a matching reverse rule.

**Lesson:** Verify the actual topology before designing against an assumed one. The correction produced a better lab than the original plan.

**Time lost:** ~45 min (mostly design decision, not troubleshooting)

---

## Issue #3 — Unidentified host at 10.10.10.5

**Date:** 2026-07-31

**Symptom:** An address on the lab subnet responding to ping that wasn't in my documentation. Needed to identify it before writing the IP table — an unknown device on a monitored segment is exactly the thing you don't hand-wave.

**Investigation:** Two passive fingerprinting signals from a single ping:

```
ping -c1 10.10.10.5 && ip neigh | grep 10.10.10.5
64 bytes from 10.10.10.5: icmp_seq=1 ttl=128 time=1.32 ms
10.10.10.5 dev ens33 lladdr 00:50:56:c0:00:02 REACHABLE
```

- **TTL 128** — Windows default initial TTL. (Linux/BSD default to 64, which is what pfSense and lab-server-01 return.)
- **MAC OUI `00:50:56`** — assigned to VMware. Specifically, `00:50:56:c0:xx:xx` is the range VMware uses for host-side virtual adapters.

**Root cause / conclusion:** Not a rogue device. It's the Windows host machine running VMware Workstation, via the VMnet virtual adapter that gives the host a presence on the lab segment.

**Useful side effect:** the host already has a foot on 10.10.10.0/24, so running LibreNMS in Docker Desktop on the host is a viable fallback that bypasses the cross-subnet path entirely. Not needed — the routed path works — but documented as a considered alternative.

**Lesson:** TTL and MAC OUI identify an unknown host in seconds without needing credentials or a port scan.

**Time lost:** ~5 min

---

## Issue #4 — `snmpwalk` fails instantly with "Unknown Object Identifier"

**Date:** 2026-07-31

**Symptom:** First SNMP verification attempt against pfSense returned an error **immediately** — no delay:

```
snmpwalk -v2c -c <COMMUNITY> 10.10.10.1 system
system: Unknown Object Identifier (Sub-id not found: (top) -> system)
```

**Investigation:** The instant failure was the key diagnostic. A firewall block or an unreachable host produces a **timeout of several seconds**. An immediate error means the tool failed locally and never put a packet on the wire — so this was not a network problem, despite looking like one at first glance.

Tested the hypothesis by bypassing name resolution entirely and asking for the same subtree by number:

```
snmpwalk -v2c -c <COMMUNITY> 10.10.10.1 1.3.6.1.2.1.1
iso.3.6.1.2.1.1.1.0 = STRING: "pfSense pfSense.home.arpa 2.8.1-RELEASE FreeBSD 15.0-CURRENT amd64"
```

That worked. Confirmed: SNMP, routing, and firewall rules were all fine. The problem was purely local OID name translation.

**Root cause:** Debian/Ubuntu ship the `snmp` package **without MIB files** for licensing reasons, and `/etc/snmp/snmp.conf` contains a `mibs :` directive that disables MIB loading. Without MIBs, `snmpwalk` can't translate the symbolic name `system` into `1.3.6.1.2.1.1` and gives up before sending anything.

**Fix:**

```
sudo apt install snmp-mibs-downloader
sudo download-mibs
sudo sed -i 's/^mibs :/#mibs :/' /etc/snmp/snmp.conf
```

Symbolic walks then returned fully translated output (`SNMPv2-MIB::sysDescr.0`, etc.).

**Lesson — the most transferable one from this session:** *Failure timing is diagnostic.* Instant failure = local problem. Slow failure = network problem. Asking "did my packet even leave this machine?" before touching firewall rules saves a lot of wasted investigation.

**Time lost:** ~15 min

---

## Issue #5 — Default `rocommunity` ships with a restrictive view (avoided, not hit)

**Date:** 2026-07-31

**Status:** Caught during configuration rather than after the fact. Logging it because the failure mode is genuinely nasty and it would have surfaced much later, at the graphing stage, looking like a LibreNMS problem.

**The trap:** Ubuntu's stock `/etc/snmp/snmpd.conf` ships:

```
rocommunity  public  default    -V systemonly
rocommunity6 public  default    -V systemonly
```

The `-V systemonly` flag restricts readable OIDs to two branches — `.1.3.6.1.2.1.1` (system) and `.1.3.6.1.2.1.25.1` (hrSystem). If you keep that flag and only ever test with `snmpwalk <host> system`, everything looks perfect. Then LibreNMS shows no CPU, memory, or storage graphs, and the natural instinct is to blame LibreNMS, the poller, or the network — none of which are the problem.

**Configuration applied:** Commented out both stock lines, replaced with a source-restricted community and no view flag:

```
rocommunity noclab-ro 10.0.0.82
```

**Verification — tested for the failure mode explicitly** rather than stopping at the first success:

```
snmpwalk -v2c -c <COMMUNITY> 10.10.10.10 system      # baseline
snmpwalk -v2c -c <COMMUNITY> 10.10.10.10 hrStorage   # proves view isn't restricted
HOST-RESOURCES-MIB::hrMemorySize.0 = INTEGER: 3431060 KBytes
```

`hrStorage` returning real memory and disk data confirms the full tree is readable to this community.

**Also noted for later:** the stock `rouser authPrivUser authpriv -V systemonly` line is still present and carries the same restriction. Currently harmless (that user doesn't exist), but it's the line to edit during the SNMPv3 hardening pass in Step 7b — otherwise the same trap reappears there.

**Lesson:** When a default config has a scoping or filtering flag on it, test *outside* the default scope before declaring success. Verifying only what you expect to work isn't verification.

---

## Session summary

| | |
|---|---|
| Issues logged | 5 (4 encountered, 1 avoided by inspection) |
| Total time lost to troubleshooting | ~1h 15m |
| Steps completed | Session 0 pre-flight, Step 1 (pfSense SNMP), Step 2 (lab-server-01 SNMP) |
| Next | Step 3 — Docker + LibreNMS on Pop!_OS |
| State at stop | Both SNMP targets verified responding. Poll pfSense at 10.10.10.1, lab-server-01 at 10.10.10.10. Community string in password manager. VM snapshots: `phase2c-snmp-configured`. |

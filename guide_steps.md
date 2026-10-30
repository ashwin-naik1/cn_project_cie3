# Secure Dual-Stack Hospital Network with IPv4-to-IPv6 Transition

**Cisco Packet Tracer | Computer Networks Mini Project (CIE3, 15 marks)**

> **Full title:** Design and Simulation of a Secure Dual-Stack Hospital Network with IPv4-to-IPv6 Transition (Dual-Stack and Tunneling), DHCPv4/DHCPv6, and Redundant GRE-over-IPsec VPN using Cisco Packet Tracer

Built by Ashwin (solo build). Presented and defended in the viva by a team of 3.
Software only: Cisco Packet Tracer 8.x, no physical hardware.

---

## Table of contents

1. [Project story](#1-project-story)
2. [Status at a glance](#2-status-at-a-glance)
3. [Technologies covered](#3-technologies-covered)
4. [Devices and cabling](#4-devices-and-cabling)
5. [Addressing plan](#5-addressing-plan)
6. [Design decisions and assumptions](#6-design-decisions-and-assumptions)
7. [Part 1: Topology (done)](#7-part-1-topology-done)
8. [Part 2: Foundation, VLANs to HSRP (done)](#8-part-2-foundation-vlans-etherchannel-rstp-router-on-a-stick-hsrp-done)
9. [Part 3: IPv4 services (done, VoIP pending)](#9-part-3-ipv4-services-done-voip-pending)
10. [Part 4A: Dual-stack, WAN, Tunnel0 (done)](#10-part-4a-dual-stack-ipv6-wan-and-tunnel0-done)
11. [Part 4B: IPsec, backup tunnel, failover (to do, next)](#11-part-4b-ipsec-backup-tunnel-floating-routes-hsrp-tracking-failover-to-do-next)
12. [Part 5: Security, testing, report, viva (to do)](#12-part-5-security-extras-testing-and-submission-to-do)
13. [Known issues and deviations from the plan](#13-known-issues-and-deviations-from-the-plan)
14. [Packet Tracer limitations (state these in the viva)](#14-packet-tracer-limitations-state-these-in-the-viva)
15. [Troubleshooting log: mistakes we hit and fixes](#15-troubleshooting-log-mistakes-we-hit-and-fixes)
16. [Viva Q&A](#16-viva-qa)
17. [Presentation split](#17-presentation-split)
18. [Save-file checkpoints](#18-save-file-checkpoints)
19. [Remaining to-do checklist](#19-remaining-to-do-checklist)
20. [Quick verification command sheet](#20-quick-verification-command-sheet)

---

## 1. Project story

A **main hospital (HQ)** runs IPv4 and IPv6 together (**dual-stack**). A **branch clinic** is IPv6-ready. The ISP between them carries **only IPv4**, so IPv6 traffic is **tunneled** across it. The tunnel is protected with **IPsec** (AES-256, SHA, pre-shared key). A **second tunnel over a backup ISP** takes over automatically if the primary path fails.

Addresses are handed out by **DHCPv4** (central server plus relay at HQ, local DHCP at the clinic) and by **SLAAC with stateless DHCPv6** for IPv6. **VLANs, ACLs and port security** isolate sensitive zones, such as the EMR/servers network, from the patient Wi-Fi network.

---

## 2. Status at a glance

| Part | Scope | Status |
|---|---|---|
| **1** | Topology, cabling, hostnames | Done |
| **2** | VLANs, trunks, EtherChannel, RSTP, router-on-a-stick, HSRP | Done |
| **3** | DHCPv4 + relay, clinic DHCP, DNS, web server, port security, VoIP | Done, except VoIP (phones stuck on "Configuring Vlan", revisit later) |
| **4A** | Dual-stack IPv6, SLAAC, stateless DHCPv6, IPv4 WAN, Tunnel0 (IPv6 over IPv4) | Done. PC-Doctor reaches clinic PC over IPv6 |
| **4B** | Security license, IPsec on Tunnel0, Tunnel1 backup, floating routes, HSRP tracking, failover demo | **To do (next)** |
| **5** | IPv4 + IPv6 ACLs, NAT/PAT, optional remote-access VPN, full test table, report, slides, mock viva | **To do** |

**Planned timeline (3 days):** Day 1 = Parts 1 to 3, Day 2 = Part 4, Day 3 = Part 5 plus documents and viva practice.

---

## 3. Technologies covered

VLANs, 802.1Q trunks, EtherChannel (LACP), RSTP (rapid-PVST), router-on-a-stick, HSRP (with interface tracking), DHCPv4 with `ip helper-address`, SLAAC, stateless DHCPv6 (on routers), dual-stack, IPv6-in-IPv4 tunnel (GRE planned, `ipv6ip` fallback in use), IPsec (IKE, AES-256, SHA, pre-shared key) protecting the tunnel, backup tunnel with floating IPv6 static route (AD 200), static routing, IPv4 extended ACL and IPv6 ACL, port security (sticky MAC), NAT/PAT (VPN traffic denied in the NAT ACL), DNS, web server, VoIP (`telephony-service`), optional remote-access VPN. **NAT64 is theory only.**

---

## 4. Devices and cabling

### Devices (17)

| Type | Model | Names |
|---|---|---|
| Routers (5) | Cisco 2911 | R-HQ1, R-HQ2, R-CLINIC, ISP1, ISP2 |
| Switches (5) | Cisco 2960-24TT | DS1, DS2 (distribution), AS1, AS2 (access), SW-CLINIC |
| Servers (2) | Server-PT | SRV-DHCP, SRV-WEB |
| PCs (5) | PC-PT | PC-Doctor, PC-Nurse, PC-Patient, Clinic-1, Clinic-2 |

### Cabling (20 cables; use "Automatically Choose Connection Type")

| # | From | To |
|---|---|---|
| 1 | R-HQ1 g0/0 | DS1 fa0/1 |
| 2 | R-HQ2 g0/0 | DS2 fa0/1 |
| 3 | DS1 fa0/23 | DS2 fa0/23 (EtherChannel member) |
| 4 | DS1 fa0/24 | DS2 fa0/24 (EtherChannel member) |
| 5 | AS1 fa0/1 | DS1 fa0/2 |
| 6 | AS1 fa0/2 | DS2 fa0/2 |
| 7 | AS2 fa0/1 | DS1 fa0/3 |
| 8 | AS2 fa0/2 | DS2 fa0/3 |
| 9 | PC-Doctor | AS1 fa0/5 |
| 10 | PC-Nurse | AS1 fa0/6 |
| 11 | PC-Patient | AS1 fa0/7 |
| 12 | SRV-DHCP | AS2 fa0/5 |
| 13 | SRV-WEB | AS2 fa0/6 |
| 14 | R-HQ1 g0/1 | ISP1 g0/0 |
| 15 | R-HQ2 g0/1 | ISP2 g0/0 |
| 16 | ISP1 g0/1 | R-CLINIC g0/0 |
| 17 | ISP2 g0/1 | R-CLINIC g0/1 |
| 18 | R-CLINIC g0/2 | SW-CLINIC fa0/1 |
| 19 | Clinic-1 | SW-CLINIC fa0/2 |
| 20 | Clinic-2 | SW-CLINIC fa0/3 |

**Port naming reminder:** 2911 routers have `g0/0`, `g0/1`, `g0/2`. 2960 switches have `fa0/1` to `fa0/24` plus `gi0/1`, `gi0/2`. A switch has **no** `g0/0` port.

---

## 5. Addressing plan

### HQ VLANs (one /24 per VLAN, RFC 1918, third octet = VLAN ID)

| VLAN | Name | IPv4 network | IPv4 gateway (HSRP virtual) | R-HQ1 | R-HQ2 | IPv6 network | R-HQ1 (IPv6) | R-HQ2 (IPv6) |
|---|---|---|---|---|---|---|---|---|
| 10 | DOCTORS | 10.0.10.0/24 | 10.0.10.1 | 10.0.10.2 | 10.0.10.3 | 2001:DB8:10::/64 | ::2 | ::3 |
| 20 | NURSES | 10.0.20.0/24 | 10.0.20.1 | 10.0.20.2 | 10.0.20.3 | 2001:DB8:20::/64 | ::2 | ::3 |
| 30 | PATIENT-WIFI | 10.0.30.0/24 | 10.0.30.1 | 10.0.30.2 | 10.0.30.3 | 2001:DB8:30::/64 | ::2 | ::3 |
| 40 | SERVERS | 10.0.40.0/24 | 10.0.40.1 | 10.0.40.2 | 10.0.40.3 | 2001:DB8:40::/64 | ::2 | ::3 |

Each /24 gives 254 usable hosts. The HSRP virtual gateway is always `.1`.

### Clinic LAN

| Item | Value |
|---|---|
| IPv4 | 192.168.50.0/24, gateway 192.168.50.1 (R-CLINIC g0/2) |
| IPv6 | 2001:DB8:50::/64, R-CLINIC g0/2 = 2001:DB8:50::1 |

### Servers

| Server | IPv4 | IPv6 |
|---|---|---|
| SRV-DHCP (also DNS) | 10.0.40.10 | 2001:DB8:40::10 |
| SRV-WEB | 10.0.40.20 | 2001:DB8:40::20 |

Both use gateway 10.0.40.1 (IPv4) and 2001:DB8:40::2 (IPv6).

### WAN links (IPv4 only, /30)

| Link | Network | Side A | Side B |
|---|---|---|---|
| R-HQ1 to ISP1 | 200.1.1.0/30 | R-HQ1 g0/1 = .1 | ISP1 g0/0 = .2 |
| ISP1 to R-CLINIC | 200.2.2.0/30 | ISP1 g0/1 = .1 | R-CLINIC g0/0 = .2 |
| R-HQ2 to ISP2 | 200.3.3.0/30 | R-HQ2 g0/1 = .1 | ISP2 g0/0 = .2 |
| ISP2 to R-CLINIC | 200.4.4.0/30 | ISP2 g0/1 = .1 | R-CLINIC g0/1 = .2 |

### Tunnels (IPv6 over IPv4)

| | Tunnel0 (primary) | Tunnel1 (backup) |
|---|---|---|
| Path | R-HQ1, ISP1, R-CLINIC | R-HQ2, ISP2, R-CLINIC |
| Tunnel network | 2001:DB8:FF1::/64 | 2001:DB8:FF2::/64 |
| HQ end | R-HQ1 = ::1 | R-HQ2 = ::1 |
| Clinic end | ::2 | ::2 |
| HQ source to destination | 200.1.1.1 to 200.2.2.2 | 200.3.3.1 to 200.4.4.2 |
| Clinic source to destination | 200.2.2.2 to 200.1.1.1 | 200.4.4.2 to 200.3.3.1 |

---

## 6. Design decisions and assumptions

These were our own choices where the project brief left a gap. State them confidently in the viva.

| Decision | Reason |
|---|---|
| One /24 per VLAN with the third octet equal to the VLAN ID | Easy to read, 254 hosts per VLAN, private RFC 1918 space |
| HSRP virtual IP is `.1` in every VLAN; R-HQ1 = `.2` (priority 110, preempt), R-HQ2 = `.3` (priority 100) | PCs keep a working gateway if one router fails |
| DHCP pools start at `.11` | Packet Tracer's server has no "excluded addresses" box, so starting at `.11` protects .1 to .10 |
| DNS runs on SRV-DHCP (10.0.40.10) | Keeps the server count at two |
| Clinic gateway `192.168.50.1`, clinic runs its own DHCP | Clinic still works if the WAN or central server is down |
| R-HQ1 is the only IPv6 router advertiser (R-HQ2 suppresses RAs) | PCs always pick R-HQ1 as IPv6 gateway. PT does not support HSRP for IPv6 |
| ISP routers have interface addresses only (no routes, no IPv6) | Each ISP router is directly connected to both of its networks; ISPs are IPv4-only by design |
| HQ and clinic LANs are **not** routed over IPv4 across the WAN | Only IPv6 crosses the tunnel; the branch is "IPv6-ready" |
| Tunnels use `tunnel mode ipv6ip` | `tunnel mode gre ip` did not pass IPv6 traffic in our PT version; `ipv6ip` is the planned fallback |
| IPsec crypto ACL matches IP protocol 41 (`permit 41 host ... host ...`) | `ipv6ip` tunnel packets are protocol 41, not GRE |
| Pre-shared key `HospVPN123` (letters and numbers only) | Avoids typing errors. Case-sensitive. A real deployment would use a long random key |
| IPsec failover test shuts down **all of ISP1** | PT tunnels have no keepalive, so one-sided failures are not detected by the far end |
| Server zone (VLAN 40) hosts the web server | ACLs control who reaches it. A stricter DMZ would be its own isolated VLAN behind a firewall (possible improvement) |

---

## 7. Part 1: Topology (done)

**Goal:** place 17 devices, cable them, and set hostnames.

**Setup steps**
1. Get Packet Tracer 8.x from netacad.com (free account, "Introduction to Packet Tracer" course).
2. Turn on **Options > Preferences > Interface > Always Show Port Labels in Logical Workspace**. It makes cabling checks easy.
3. Place devices: routers under *Network Devices > Routers > 2911*, switches under *Network Devices > Switches > 2960-24TT*, PCs and servers under *End Devices*.
4. Rename each device by clicking the label under its icon.
5. Cable using the *Connections > Automatically Choose Connection Type* tool and the table in section 4.

**Hostnames (CLI of each router/switch)**

```
enable
configure terminal
hostname R-HQ1
no ip domain-lookup
end
write memory
```

Routers show an "initial configuration dialog" on first CLI use: answer `no`.

**Verify**
- 17 devices, 20 cables.
- Router links stay red until `no shutdown`. Switch links turn amber then green.
- `show cdp neighbors` on DS1 shows DS2 twice (Fa 0/23, Fa 0/24), AS1 on Fa 0/2, AS2 on Fa 0/3.

**Concept notes:** DS = distribution switches (aggregate access switches), AS = access switches (PCs plug in). Orange ports on the canvas are normal: spanning tree blocks redundant links so loops cannot form.

**Save:** `v0_topology.pkt`

---

## 8. Part 2: Foundation, VLANs, EtherChannel, RSTP, router-on-a-stick, HSRP (done)

**Concepts**

| Technology | Purpose |
|---|---|
| VLAN | Splits a switch into separate virtual networks (doctors, nurses, patient Wi-Fi, servers) |
| 802.1Q trunk | One cable carrying many VLANs, with each frame tagged |
| EtherChannel (LACP) | Bundles DS1-DS2 cables into one fast, redundant link |
| RSTP | Prevents loops and converges in 1 to 2 seconds. DS1 is root, DS2 is backup root |
| Router-on-a-stick | One router port split into 802.1Q sub-interfaces, one per VLAN, to route between VLANs |
| HSRP | Two routers share a virtual gateway IP; the standby router takes over if the active one fails |

### DS1

```
enable
configure terminal
vlan 10
 name DOCTORS
vlan 20
 name NURSES
vlan 30
 name PATIENT-WIFI
vlan 40
 name SERVERS
exit
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20,30,40 root primary
interface range fa0/1-3
 switchport mode trunk
exit
interface range fa0/23-24
 channel-group 1 mode active
exit
interface port-channel 1
 switchport mode trunk
end
write memory
```

### DS2 (identical, but `root secondary`)

Same block as DS1 with this line instead:
```
spanning-tree vlan 10,20,30,40 root secondary
```

### AS1

```
enable
configure terminal
vlan 10
 name DOCTORS
vlan 20
 name NURSES
vlan 30
 name PATIENT-WIFI
vlan 40
 name SERVERS
exit
spanning-tree mode rapid-pvst
interface range fa0/1-2
 switchport mode trunk
exit
interface fa0/5
 switchport mode access
 switchport access vlan 10
exit
interface fa0/6
 switchport mode access
 switchport access vlan 20
exit
interface fa0/7
 switchport mode access
 switchport access vlan 30
end
write memory
```

### AS2

```
enable
configure terminal
vlan 10
 name DOCTORS
vlan 20
 name NURSES
vlan 30
 name PATIENT-WIFI
vlan 40
 name SERVERS
exit
spanning-tree mode rapid-pvst
interface range fa0/1-2
 switchport mode trunk
exit
interface range fa0/5-6
 switchport mode access
 switchport access vlan 40
end
write memory
```

SW-CLINIC needs no VLAN config (single LAN on default VLAN 1).

### R-HQ1: router-on-a-stick + HSRP

```
enable
configure terminal
interface g0/0
 no shutdown
exit
interface g0/0.10
 encapsulation dot1Q 10
 ip address 10.0.10.2 255.255.255.0
 standby 10 ip 10.0.10.1
 standby 10 priority 110
 standby 10 preempt
exit
interface g0/0.20
 encapsulation dot1Q 20
 ip address 10.0.20.2 255.255.255.0
 standby 20 ip 10.0.20.1
 standby 20 priority 110
 standby 20 preempt
exit
interface g0/0.30
 encapsulation dot1Q 30
 ip address 10.0.30.2 255.255.255.0
 standby 30 ip 10.0.30.1
 standby 30 priority 110
 standby 30 preempt
exit
interface g0/0.40
 encapsulation dot1Q 40
 ip address 10.0.40.2 255.255.255.0
 standby 40 ip 10.0.40.1
 standby 40 priority 110
 standby 40 preempt
end
write memory
```

Rules: the parent `g0/0` gets no IP address, and `encapsulation` must come before `ip address` on each sub-interface. Sub-interface numbers (`.10`, `.20`, ...) should match the VLAN IDs.

### R-HQ2: same sub-interfaces, host `.3`, default priority

```
enable
configure terminal
interface g0/0
 no shutdown
exit
interface g0/0.10
 encapsulation dot1Q 10
 ip address 10.0.10.3 255.255.255.0
 standby 10 ip 10.0.10.1
exit
interface g0/0.20
 encapsulation dot1Q 20
 ip address 10.0.20.3 255.255.255.0
 standby 20 ip 10.0.20.1
exit
interface g0/0.30
 encapsulation dot1Q 30
 ip address 10.0.30.3 255.255.255.0
 standby 30 ip 10.0.30.1
exit
interface g0/0.40
 encapsulation dot1Q 40
 ip address 10.0.40.3 255.255.255.0
 standby 40 ip 10.0.40.1
end
write memory
```

(`standby X preempt` is added on R-HQ2 in Part 4B for HSRP tracking.)

### Temporary static test addresses (before DHCP)

| Device | IP | Mask | Gateway |
|---|---|---|---|
| PC-Doctor | 10.0.10.50 | 255.255.255.0 | 10.0.10.1 |
| PC-Nurse | 10.0.20.50 | 255.255.255.0 | 10.0.20.1 |
| PC-Patient | 10.0.30.50 | 255.255.255.0 | 10.0.30.1 |
| SRV-DHCP | 10.0.40.10 | 255.255.255.0 | 10.0.40.1 |

### Verify

| Command | Expected |
|---|---|
| `show etherchannel summary` (DS1) | `Po1(SU)` with `Fa0/23(P)` and `Fa0/24(P)` |
| `show spanning-tree vlan 10` (DS1) | "This bridge is the root" |
| `show vlan brief` (AS1) | Fa0/5 in VLAN 10, Fa0/6 in VLAN 20, Fa0/7 in VLAN 30. Uplinks Fa0/1-2 **not** listed (they are trunks) |
| `show interfaces trunk` (DS1, AS1, AS2) | Uplinks listed as trunking, native VLAN 1 |
| `show ip interface brief` (R-HQ1, R-HQ2) | `g0/0` up/up with no IP; `.10/.20/.30/.40` up/up with the right addresses |
| `show standby brief` | R-HQ1 **Active**, R-HQ2 **Standby** for every group |
| Ping PC-Doctor to 10.0.20.50, 10.0.30.50, 10.0.40.10 | Success (first ping may time out once for ARP) |

**Failover test:** `ping -t 10.0.40.10` from PC-Doctor, then `shutdown` on R-HQ1 `g0/0`. Pings recover within about 10 seconds and R-HQ2 becomes Active. `no shutdown` brings R-HQ1 back as Active because of preempt.

**Save:** `v1_base.pkt`

---

## 9. Part 3: IPv4 services (done, VoIP pending)

**Goal:** automatic addressing, names, a web service, port security and VoIP.

### 9.1 Central DHCP server (SRV-DHCP)

**Why:** one central server holds a pool per VLAN. A DHCP request is a broadcast and cannot cross a router, so each router sub-interface relays it (`ip helper-address`). The router stamps its sub-interface address in the request, and the server uses that to choose the right pool (DORA: Discover, Offer, Request, Acknowledge).

SRV-DHCP static config: `10.0.40.10`, mask `255.255.255.0`, gateway `10.0.40.1`, DNS `10.0.40.10`.

*Services > DHCP*: set Service **On**, select and **Remove** the default `serverPool`, then **Add** each pool (the Pool Name must be new each time; **Save** only updates a selected row):

| Pool name | Default gateway | DNS | Start IP | Mask | Max users | TFTP |
|---|---|---|---|---|---|---|
| DOCTORS | 10.0.10.1 | 10.0.40.10 | 10.0.10.11 | 255.255.255.0 | 200 | 0.0.0.0 |
| NURSES | 10.0.20.1 | 10.0.40.10 | 10.0.20.11 | 255.255.255.0 | 200 | 10.0.20.2 (for the IP phones) |
| PATIENT-WIFI | 10.0.30.1 | 10.0.40.10 | 10.0.30.11 | 255.255.255.0 | 200 | 0.0.0.0 |
| SERVERS | 10.0.40.1 | 10.0.40.10 | 10.0.40.100 | 255.255.255.0 | 100 | 0.0.0.0 |

The default gateway in every pool is the **HSRP virtual IP (.1)**, so the gateway survives a router failure.

### 9.2 DHCP relay on R-HQ1 and R-HQ2

```
enable
configure terminal
interface g0/0.10
 ip helper-address 10.0.40.10
interface g0/0.20
 ip helper-address 10.0.40.10
interface g0/0.30
 ip helper-address 10.0.40.10
interface g0/0.40
 ip helper-address 10.0.40.10
end
write memory
```

Set PC-Doctor, PC-Nurse and PC-Patient to **DHCP**. Expected: `10.0.10.11`, `10.0.20.11`, `10.0.30.11` with gateway `.1` and DNS `10.0.40.10`.

### 9.3 Clinic: local DHCP on R-CLINIC

```
enable
configure terminal
interface g0/2
 ip address 192.168.50.1 255.255.255.0
 no shutdown
exit
ip dhcp excluded-address 192.168.50.1 192.168.50.10
ip dhcp pool CLINIC-LAN
 network 192.168.50.0 255.255.255.0
 default-router 192.168.50.1
 dns-server 10.0.40.10
end
write memory
```

Expected: Clinic-1 and Clinic-2 get `192.168.50.11` and `.12`. Verify with `show ip dhcp binding`.

### 9.4 DNS (SRV-DHCP > Services > DNS)

DNS Service **On**, then add A records:

| Name | Address |
|---|---|
| web.hospital.local | 10.0.40.20 |
| dhcp.hospital.local | 10.0.40.10 |

### 9.5 Web server (SRV-WEB)

Static: `10.0.40.20`, mask `255.255.255.0`, gateway `10.0.40.1`, DNS `10.0.40.10`. *Services > HTTP* **On**, edit `index.html` (for example "City Hospital Patient Portal").

Verify: from PC-Doctor's Web Browser, `http://10.0.40.20` and `http://web.hospital.local` both load. `nslookup web.hospital.local` returns `10.0.40.20`.

### 9.6 Port security

**What:** limits each access port to the one device it first sees (sticky MAC). A different MAC triggers a violation and the port shuts down (err-disabled). Works only on access ports.

AS1 (user ports):
```
enable
configure terminal
interface range fa0/5-7
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
end
write memory
```

AS2 (server ports): same block on `interface range fa0/5-6`.

SW-CLINIC: same block on `interface range fa0/2-3` after `switchport mode access`.

After applying, ping from every PC and server once so the MAC is learned, then `write memory`.

Verify: `show port-security` shows for each port: Max 1, Current 1, Violations 0, Action Shutdown (confirmed on AS1 for Fa0/5, Fa0/6, Fa0/7).

**Break-it demo for the viva:** change PC-Doctor's MAC (Config > FastEthernet0 > MAC Address, note the original first), ping, and watch Fa0/5 go err-disabled. Restore with the original MAC and `shutdown` then `no shutdown` on the port.

### 9.7 VoIP (not finished, revisit later)

**Design attempted:** two IP phones (7960) on AS1 `fa0/8` and `fa0/9` in VLAN 20, with R-HQ1 as the phone system (extensions 1001 and 1002). Phones power via the Physical tab power adapter (the 2960 has no PoE).

AS1 phone ports:
```
interface range fa0/8-9
 switchport mode access
 switchport access vlan 20
 switchport voice vlan 20
```

R-HQ1 needs the **UC license** (the 2911 module name is `c2900`, not `c1900`):
```
configure terminal
license boot module c2900 technology-package uck9
end
write memory
reload
```

Then on R-HQ1:
```
configure terminal
telephony-service
 max-ephones 5
 max-dn 5
 ip source-address 10.0.20.2 port 2000
 auto assign 1 to 5
exit
ephone-dn 1
 number 1001
exit
ephone-dn 2
 number 1002
end
write memory
```

**Current status:** the phone screens stay on "Configuring Vlan". The `telephony-service` command was initially rejected (license not active). Next things to try: confirm `uck9` is active with `show version`, confirm `switchport voice vlan 20` was accepted, bounce the phone ports, use Fast Forward Time, then check `show ephone registered`. The phone should show IP `10.0.20.x` and TFTP `10.0.20.2`. This is the least important part of the story, so it is time-boxed. If it stays unresolved, explain it in the viva as a call-control demo on one router.

**Save:** `v2_dhcp.pkt` after DHCP, `v2_services.pkt` after the rest.

---

## 10. Part 4A: Dual-stack IPv6, WAN and Tunnel0 (done)

### 10.1 Concepts

- IPv6 addresses are 128 bits, written as 8 hex groups. `::` replaces a run of zeros. `/64` is the standard LAN prefix. `2001:DB8::/32` is the documentation range.
- Every IPv6 interface has a link-local address `FE80::...`.
- **SLAAC:** the router advertises the prefix (Router Advertisement), and the host builds its own address. No server tracks it.
- **Stateless DHCPv6:** SLAAC gives the address, and DHCPv6 only supplies extras (DNS, domain). The "O" flag in the RA tells hosts to ask for them.
- **Dual-stack:** IPv4 and IPv6 run side by side on the same devices.
- **Tunnel:** the ISP carries only IPv4, so each IPv6 packet is wrapped in an IPv4 packet and unwrapped at the far end.

### 10.2 IPv6 on R-HQ1

```
enable
configure terminal
ipv6 unicast-routing
interface g0/0.10
 ipv6 address 2001:DB8:10::2/64
interface g0/0.20
 ipv6 address 2001:DB8:20::2/64
interface g0/0.30
 ipv6 address 2001:DB8:30::2/64
interface g0/0.40
 ipv6 address 2001:DB8:40::2/64
end
write memory
```

### 10.3 IPv6 on R-HQ2 (addresses, but silent on RAs)

```
enable
configure terminal
ipv6 unicast-routing
interface g0/0.10
 ipv6 address 2001:DB8:10::3/64
 ipv6 nd ra suppress
interface g0/0.20
 ipv6 address 2001:DB8:20::3/64
 ipv6 nd ra suppress
interface g0/0.30
 ipv6 address 2001:DB8:30::3/64
 ipv6 nd ra suppress
interface g0/0.40
 ipv6 address 2001:DB8:40::3/64
 ipv6 nd ra suppress
end
write memory
```

Note: the older spelling `ipv6 nd suppress-ra` was rejected on our router. If `ipv6 nd ra suppress` is also rejected, try `ipv6 nd ra suppress all`, or `ipv6 nd ra-lifetime 0` (or `ipv6 nd ra lifetime 0`) on all four sub-interfaces.

### 10.4 IPv6 and stateless DHCPv6 on R-CLINIC

```
enable
configure terminal
ipv6 unicast-routing
ipv6 dhcp pool CLINIC-DNS
 dns-server 2001:DB8:40::10
 domain-name hospital.local
exit
interface g0/2
 ipv6 address 2001:DB8:50::1/64
 ipv6 nd other-config-flag
 ipv6 dhcp server CLINIC-DNS
end
write memory
```

### 10.5 Hosts

- Servers (static IPv6): SRV-DHCP `2001:DB8:40::10/64`, SRV-WEB `2001:DB8:40::20/64`, gateway `2001:DB8:40::2` on both.
- PCs: *Desktop > IP Configuration > IPv6 Configuration = Automatic* (SLAAC). IPv4 stays on DHCP, so each PC is dual-stack.

Verify: `show ipv6 interface brief` on R-HQ1 (all sub-interfaces up/up with link-local and global addresses). `ipconfig` on PC-Doctor shows an IPv4 address and a `2001:DB8:10:0:...` IPv6 address with an `FE80::` gateway. `ping 2001:DB8:40::10` from PC-Doctor succeeds. **Save:** `v3_dualstack.pkt`.

### 10.6 The IPv4-only WAN

```
! R-HQ1
interface g0/1
 ip address 200.1.1.1 255.255.255.252
 no shutdown
exit
ip route 200.2.2.0 255.255.255.252 200.1.1.2

! R-HQ2
interface g0/1
 ip address 200.3.3.1 255.255.255.252
 no shutdown
exit
ip route 200.4.4.0 255.255.255.252 200.3.3.2

! ISP1
interface g0/0
 ip address 200.1.1.2 255.255.255.252
 no shutdown
exit
interface g0/1
 ip address 200.2.2.1 255.255.255.252
 no shutdown

! ISP2
interface g0/0
 ip address 200.3.3.2 255.255.255.252
 no shutdown
exit
interface g0/1
 ip address 200.4.4.1 255.255.255.252
 no shutdown

! R-CLINIC
interface g0/0
 ip address 200.2.2.2 255.255.255.252
 no shutdown
exit
interface g0/1
 ip address 200.4.4.2 255.255.255.252
 no shutdown
exit
ip route 200.1.1.0 255.255.255.252 200.2.2.1
ip route 200.3.3.0 255.255.255.252 200.4.4.1
```

(Enter `enable` and `configure terminal` first on each router, and `write memory` at the end.) **Do not enable `ipv6 unicast-routing` on the ISP routers.**

Verify: `ping 200.2.2.2` from R-HQ1, `ping 200.4.4.2` from R-HQ2, and from R-CLINIC `ping 200.1.1.1` and `ping 200.3.3.1`. All must succeed before building tunnels. **Save:** `v3_wan.pkt`.

### 10.7 Tunnel0 (IPv6 over IPv4) between R-HQ1 and R-CLINIC

**Final working mode is `ipv6ip`.** The planned `tunnel mode gre ip` showed the tunnel as up/up but sent 0 packets, so both ends were switched to `ipv6ip`.

R-HQ1:
```
interface Tunnel0
 ipv6 address 2001:DB8:FF1::1/64
 tunnel source GigabitEthernet0/1
 tunnel destination 200.2.2.2
 tunnel mode ipv6ip
exit
ipv6 route 2001:DB8:50::/64 2001:DB8:FF1::2
```

R-CLINIC:
```
interface Tunnel0
 ipv6 address 2001:DB8:FF1::2/64
 tunnel source GigabitEthernet0/0
 tunnel destination 200.1.1.1
 tunnel mode ipv6ip
exit
ipv6 route 2001:DB8:10::/64 2001:DB8:FF1::1
ipv6 route 2001:DB8:20::/64 2001:DB8:FF1::1
ipv6 route 2001:DB8:30::/64 2001:DB8:FF1::1
ipv6 route 2001:DB8:40::/64 2001:DB8:FF1::1
```

Verify (the first ping over a tunnel often times out, so run it twice):
- `ping 2001:DB8:FF1::2` from R-HQ1 works.
- `show interfaces tunnel 0 | include transport` shows `IPv6/IP`.
- **PC-Doctor pings a clinic PC's `2001:DB8:50:0:...` address over IPv6. This works (confirmed).**
- In Simulation mode, the packet crossing ISP1 appears as IPv4, not IPv6.

**Important lesson:** a tunnel can show up/up even when it passes nothing (no keepalive in PT). Always prove it with a real ping and with `show interfaces tunnel 0` packet counters. **Save:** `v3_tunnel0.pkt`.

---

## 11. Part 4B: IPsec, backup tunnel, floating routes, HSRP tracking, failover (to do, next)

**Time:** about 3.5 to 4 hours. **Start from `v3_tunnel0.pkt`** (save as `v4_start.pkt`).

| Step | What | Where |
|---|---|---|
| 1 | Pre-flight checks | R-HQ1 |
| 2 | Security license and reload | R-HQ1, R-HQ2, R-CLINIC |
| 3 | IPsec on Tunnel0 | R-HQ1, R-CLINIC |
| 4 | Tunnel1 (backup) over ISP2 | R-HQ2, R-CLINIC |
| 5 | IPsec on Tunnel1 | R-HQ2, R-CLINIC |
| 6 | Floating routes | R-HQ1, R-CLINIC |
| 7 | HSRP tracking | R-HQ1, R-HQ2 |
| 8 | Failover demo and failback | ISP1 and all |

### Step 1: Pre-flight

`ping 2001:DB8:FF1::2` and `show interfaces tunnel 0 | include transport` on R-HQ1 (expect `IPv6/IP`). Run `write memory` on all three routers before the reload step.

### Step 2: Security license (R-HQ1, R-HQ2, R-CLINIC, one at a time)

```
enable
configure terminal
license boot module c2900 technology-package securityk9
```
Type `yes` at the agreement, then:
```
end
write memory
reload
```
After reload: `show version` (license table shows `securityk9`) and `crypto isakmp policy 10` should open `(config-isakmp)#`. ISP routers do not need this license. **Save:** `v4_license.pkt`.

### Step 3: IPsec on Tunnel0

**Concepts**

| Phase | What happens |
|---|---|
| Phase 1 (IKE/ISAKMP) | Routers authenticate with the pre-shared key and build a secure channel |
| Phase 2 (IPsec) | They agree how to encrypt data (AES-256, SHA integrity) and start encrypting |

A **crypto ACL** picks the traffic to protect, and a **crypto map** ties ACL, peer and settings to the WAN interface. Because PT cannot do native IPsec for IPv6, IPsec encrypts the **outer IPv4 tunnel packets** (protocol 41) between the WAN addresses.

R-HQ1:
```
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
exit
crypto isakmp key HospVPN123 address 200.2.2.2
crypto ipsec transform-set TS-HOSP esp-aes 256 esp-sha-hmac
exit
ip access-list extended VPN-T0
 permit 41 host 200.1.1.1 host 200.2.2.2
exit
crypto map CMAP-HQ1 10 ipsec-isakmp
 set peer 200.2.2.2
 set transform-set TS-HOSP
 match address VPN-T0
exit
interface g0/1
 crypto map CMAP-HQ1
end
write memory
```

R-CLINIC (mirror image):
```
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
exit
crypto isakmp key HospVPN123 address 200.1.1.1
crypto ipsec transform-set TS-HOSP esp-aes 256 esp-sha-hmac
exit
ip access-list extended VPN-T0
 permit 41 host 200.2.2.2 host 200.1.1.1
exit
crypto map CMAP-CLINIC 10 ipsec-isakmp
 set peer 200.1.1.1
 set transform-set TS-HOSP
 match address VPN-T0
exit
interface g0/0
 crypto map CMAP-CLINIC
end
write memory
```

**Fallback** if `permit 41` is rejected or counters stay at 0: replace the ACL line with `permit ip host 200.1.1.1 host 200.2.2.2` (and the mirror on the clinic).

**Verify:** ping `2001:DB8:FF1::2` from R-HQ1 three or four times (first ones may time out while IKE negotiates). Then:
- `show crypto isakmp sa` shows `200.1.1.1` to `200.2.2.2`, state **QM_IDLE**, status **ACTIVE**.
- `show crypto ipsec sa` shows `#pkts encaps/encrypt` and `#pkts decaps/decrypt` above 0 and rising after each ping.
- PC-Doctor can still ping a clinic PC over IPv6.

**Checkpoint:** do not continue until this works. **Save:** `v4_ipsec.pkt`.

### Step 4: Tunnel1 (backup) over ISP2, without IPsec first

R-HQ2:
```
interface Tunnel1
 ipv6 address 2001:DB8:FF2::1/64
 tunnel source GigabitEthernet0/1
 tunnel destination 200.4.4.2
 tunnel mode ipv6ip
exit
ipv6 route 2001:DB8:50::/64 2001:DB8:FF2::2
```

R-CLINIC:
```
interface Tunnel1
 ipv6 address 2001:DB8:FF2::2/64
 tunnel source GigabitEthernet0/1
 tunnel destination 200.3.3.1
 tunnel mode ipv6ip
```

Verify from R-HQ2: `ping 200.4.4.2`, then `ping 2001:DB8:FF2::2` twice. Both tunnels must use the same mode (`ipv6ip`).

### Step 5: IPsec on Tunnel1

R-HQ2:
```
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
exit
crypto isakmp key HospVPN123 address 200.4.4.2
crypto ipsec transform-set TS-HOSP esp-aes 256 esp-sha-hmac
exit
ip access-list extended VPN-T1
 permit 41 host 200.3.3.1 host 200.4.4.2
exit
crypto map CMAP-HQ2 10 ipsec-isakmp
 set peer 200.4.4.2
 set transform-set TS-HOSP
 match address VPN-T1
exit
interface g0/1
 crypto map CMAP-HQ2
end
write memory
```

R-CLINIC (add to the existing config; a crypto map can have several numbered entries, one per peer):
```
crypto isakmp key HospVPN123 address 200.3.3.1
ip access-list extended VPN-T1
 permit 41 host 200.4.4.2 host 200.3.3.1
exit
crypto map CMAP-CLINIC 20 ipsec-isakmp
 set peer 200.3.3.1
 set transform-set TS-HOSP
 match address VPN-T1
exit
interface g0/1
 crypto map CMAP-CLINIC
end
write memory
```

Verify: ping `2001:DB8:FF2::2` from R-HQ2 several times. `show crypto isakmp sa` and `show crypto ipsec sa` show an ACTIVE SA with rising counters. On R-CLINIC, `show crypto isakmp sa` shows **two** SAs (200.1.1.1 and 200.3.3.1). This also pre-warms the backup tunnel for faster failover. **Save:** `v4_tunnel1.pkt`.

### Step 6: Floating static routes (AD 200)

A floating route is a static route with a deliberately high administrative distance. It stays hidden until the preferred route (AD 1) disappears.

R-CLINIC (normal routes via Tunnel0 already exist; add the floating backups via Tunnel1):
```
ipv6 route 2001:DB8:10::/64 2001:DB8:FF2::1 200
ipv6 route 2001:DB8:20::/64 2001:DB8:FF2::1 200
ipv6 route 2001:DB8:30::/64 2001:DB8:FF2::1 200
ipv6 route 2001:DB8:40::/64 2001:DB8:FF2::1 200
```

R-HQ1 (it has no Tunnel1, so its backup goes through R-HQ2 on the servers VLAN):
```
ipv6 route 2001:DB8:50::/64 2001:DB8:40::3 200
```

Verify with `show running-config | include ipv6 route`. In normal operation `show ipv6 route` shows only the AD 1 routes via Tunnel0.

### Step 7: HSRP tracking

HSRP only watches the LAN, so if R-HQ1's WAN dies it would stay Active and black-hole traffic. Tracking drops R-HQ1's priority from 110 to 90 (below R-HQ2's 100) when its WAN line protocol goes down.

R-HQ1:
```
track 1 interface GigabitEthernet0/1 line-protocol
interface g0/0.10
 standby 10 track 1 decrement 20
interface g0/0.20
 standby 20 track 1 decrement 20
interface g0/0.30
 standby 30 track 1 decrement 20
interface g0/0.40
 standby 40 track 1 decrement 20
```

R-HQ2 (needs `preempt` to take over):
```
interface g0/0.10
 standby 10 preempt
interface g0/0.20
 standby 20 preempt
interface g0/0.30
 standby 30 preempt
interface g0/0.40
 standby 40 preempt
```

Verify: `show track` on R-HQ1 (Track 1 Up) and `show standby brief` (R-HQ1 still Active, priority 110). **Save:** `v4_tracking.pkt`.

### Step 8: Failover demo (the showpiece)

**Why we shut down all of ISP1:** PT tunnels have no keepalive, so a one-sided failure would not be noticed by the far end. Simulating a full ISP1 outage makes both tunnel ends see the failure. Real networks solve this with GRE keepalives or IP SLA.

Before: PC-Doctor pings a clinic PC over IPv6, `show standby brief` shows R-HQ1 Active, and `show ipv6 route` on R-HQ1 shows `S 2001:DB8:50::/64 [1/0]` via `2001:DB8:FF1::2`.

1. PC-Doctor: `ping -t <clinic PC IPv6>`
2. On ISP1:
```
configure terminal
interface range g0/0 - 1
 shutdown
end
```
3. Watch timeouts, then replies resume over the backup path (allow up to about 30 seconds).

| Where | Command | Expected after failure |
|---|---|---|
| R-HQ1 | `show interfaces tunnel 0` | Tunnel0 down |
| R-HQ1 | `show ipv6 route` | `S 2001:DB8:50::/64 [200/0] via 2001:DB8:40::3` |
| R-CLINIC | `show ipv6 route` | HQ prefixes via `2001:DB8:FF2::1` with `[200/0]` |
| R-HQ2 | `show crypto ipsec sa` | Tunnel1 counters rising |
| R-HQ1 | `show track` | Track 1 Down |
| R-HQ1/R-HQ2 | `show standby brief` | R-HQ2 Active, R-HQ1 Standby |

**Failback:** on ISP1, `interface range g0/0 - 1` then `no shutdown`. After about a minute Tunnel0 returns, routes revert to AD 1, and R-HQ1 regains Active (preempt).

**Save:** `v4_final.pkt`. Screenshot the ping before, during and after, the route tables before and after, `show standby brief` before and after, and `show crypto isakmp sa` on R-CLINIC with both peers.

### 4B checklist

- [ ] `securityk9` active on all 3 routers
- [ ] Tunnel0 IPsec: ACTIVE SA, counters rising
- [ ] Tunnel1 up with IPsec
- [ ] Floating routes on R-HQ1 and R-CLINIC
- [ ] HSRP tracking plus preempt on R-HQ2
- [ ] ISP1 outage test passes, then failback works
- [ ] Screenshots saved

---

## 12. Part 5: Security extras, testing and submission (to do)

Detailed commands for Part 5 will be worked out in the next session. This is the plan and design intent.

### Block 1: ACLs and NAT/PAT (about 1.5 hrs)

- **IPv4 extended ACL:** block **VLAN 30 (patient Wi-Fi) from VLAN 40 (servers/EMR)**, permit everything else. Apply it inbound on the VLAN 30 sub-interface (`g0/0.30`) on **both** R-HQ1 and R-HQ2, because either may be the active gateway.
- **IPv6 ACL:** the same policy for `2001:DB8:30::/64` to `2001:DB8:40::/64`, applied with `ipv6 traffic-filter` inbound on `g0/0.30`. IPv6 ACLs implicitly allow neighbor discovery, but check that RA and ND still work after applying.
- **NAT/PAT:** inside = HQ LAN sub-interfaces, outside = WAN `g0/1`, overload on the WAN interface. The NAT ACL **denies VPN/tunnel traffic** so the tunnel is never translated, and permits other inside traffic.
- Test that patient Wi-Fi cannot reach servers, but doctors still can.

### Block 2: Optional remote-access VPN (about 1 hr)

A doctor's laptop connecting from home (Easy VPN style), only if the PT version and router model support it. It is the lowest-priority feature. If time is short, hand it to a teammate.

### Block 3: Full test table (about 1 hr)

Run every test and record: test, expected result, actual result, screenshot.

| # | Test | Expected |
|---|---|---|
| 1 | `show etherchannel summary` (DS1) | Po1(SU), both ports (P) |
| 2 | `show spanning-tree vlan 10` (DS1) | DS1 is root |
| 3 | `show vlan brief` | VLANs 10/20/30/40 with ports |
| 4 | Inter-VLAN ping (Doctor to Nurse, Server) | Success |
| 5 | `show standby brief` | HQ1 Active, HQ2 Standby |
| 6 | HSRP failover (shut R-HQ1 g0/0) | HQ2 Active, pings recover; preempt returns HQ1 |
| 7 | DHCP on each VLAN | Correct address, gateway .1, DNS |
| 8 | Clinic DHCP | 192.168.50.x from R-CLINIC |
| 9 | DNS and web by name | `http://web.hospital.local` loads |
| 10 | Port security violation | Port err-disabled, then recovered |
| 11 | Dual-stack PCs | IPv4 and IPv6 addresses |
| 12 | IPv6 ping Doctor to Clinic over tunnel | Success |
| 13 | Packet at ISP1 in Simulation mode | Appears as IPv4 / ESP, not plain IPv6 |
| 14 | `show crypto isakmp sa` / `ipsec sa` | ACTIVE, counters rising |
| 15 | ISP1 outage failover | Traffic recovers via Tunnel1 |
| 16 | Failback | Traffic returns to Tunnel0 |
| 17 | Patient VLAN to server VLAN | Blocked by ACL (IPv4 and IPv6) |
| 18 | Doctor VLAN to server VLAN | Allowed |
| 19 | NAT/PAT translations | `show ip nat translations` shows PAT; tunnel traffic not translated |
| 20 | VoIP call 1001 to 1002 (if fixed) | Call connects |

### Block 4: Report and slides (about 2 hrs)

Report sections: introduction and story, topology diagram, addressing table, VLAN table, key configs per technology, test results table with screenshots, Packet Tracer limitations and workarounds, conclusion and future work.

Future work ideas: HSRP/VRRPv3 for IPv6, GRE keepalives or IP SLA, a real DMZ behind a firewall, NAT64 for IPv6-to-IPv4 access, certificate-based IKE instead of a pre-shared key.

### Block 5: Mock viva (about 1.5 hrs)

Rehearse with both teammates. Four live demos to practice:
1. Inter-VLAN routing and DHCP
2. IPv6 over the tunnel
3. The blocked ACL (patient to server)
4. Failover with `show crypto ipsec sa`

---

## 13. Known issues and deviations from the plan

| Item | Status | Viva wording |
|---|---|---|
| VoIP phones stuck on "Configuring Vlan" | Open, revisit later | "IP phones register to a router running `telephony-service`; we demonstrated the configuration on R-HQ1" |
| Tunnel mode is `ipv6ip`, not `gre ip` | Closed (deliberate fallback) | "Packet Tracer did not forward IPv6 over `gre ip` in our version, so we used the allowed 6in4 (`ipv6ip`) mode. It is still IPv6 wrapped in IPv4 with IPsec on top." (Optional: retry `gre ip` at the end if time permits.) |
| No HSRP for IPv6 | By design | "PT does not support HSRP for IPv6. R-HQ1 is the IPv6 gateway, and R-HQ2 is reached via a floating static route." |
| Failover test uses a full ISP1 shutdown | By design | "No tunnel keepalive in PT; in real networks we would use GRE keepalives or IP SLA." |
| Web server sits in the server VLAN, not an isolated DMZ | By design | "ACLs control access; a stricter DMZ would sit behind a firewall." |
| Some screenshots still needed | Open | Collect while running the Part 5 test table |

---

## 14. Packet Tracer limitations (state these in the viva)

- **NAT64 is not supported**, so it is covered in theory only.
- **Native IPsec for IPv6 is not supported.** IPv6 rides inside an IPv4 tunnel, and IPsec protects the outer IPv4 packets.
- **The PT server's DHCP is IPv4 only.** DHCPv6 runs on the routers.
- **The first ping over a tunnel often times out** because of neighbor/ARP resolution and IKE negotiation.
- **Tunnels have no keepalive**, so a tunnel can show up/up while passing nothing, and one-sided failures are not detected by the far end.
- **HSRP for IPv6 is not supported.**
- **IP phones need the power adapter** (the 2960 has no PoE), and the router needs the UC license.

---

## 15. Troubleshooting log: mistakes we hit and fixes

| Problem | Cause | Fix |
|---|---|---|
| `% Invalid interface type and number` on `interface g0/0` | Router commands pasted into a **switch** (DS2). Switches have no `g0/0` | Paste router commands on routers only. Always read the prompt (`DS2#` vs `R-HQ1#`) before pasting |
| `%CDP-4-NATIVE_VLAN_MISMATCH` between AS1 and DS2 | AS1 `Fa0/1` and `Fa0/2` were access ports stuck in VLAN 10 instead of trunks | On AS1: `interface range fa0/1-2`, `no switchport access vlan`, `switchport mode trunk`, `switchport trunk native vlan 1` |
| PC could not ping its gateway, `show interfaces trunk` empty on AS1 | Same: uplinks not trunks | Same fix. Check AS2 too |
| Sub-interfaces named `g0/0.1` to `.4` | Typo | `no interface g0/0.1` (etc.), then recreate as `.10/.20/.30/.40` with the right `encapsulation dot1Q` |
| Ping to the server VLAN failed while other VLANs worked | Server's gateway/VLAN or AS2 uplinks | Check server IP config (gateway `10.0.40.1`), `show standby brief`, AS2 trunk and VLAN 40 ports |
| "Pool Name already exists" in the DHCP tab | Add needs a new name; Remove needs a selected row | Select the row, Remove, then change the Pool Name for every Add |
| `telephony-service` invalid | UC license not active | `license boot module c2900 technology-package uck9`, `write memory`, `reload` |
| `ipv6 nd suppress-ra` invalid | Newer IOS spelling | Use `ipv6 nd ra suppress` |
| R-HQ1 could not ping R-CLINIC WAN while the ISP2 path worked | ISP1 side not finished/configured yet | Check `show ip interface brief` and `show cdp neighbors` hop by hop. After fixing, `ping 200.2.2.2` worked |
| Tunnel0 up/up but 0 packets sent, ping failed | `gre ip` mode not forwarding IPv6 in this PT version | Switch both ends to `tunnel mode ipv6ip` (shutdown, change, no shutdown) |

**General rules that saved time:** save a new `.pkt` after every block; follow the 20-minute rule (if stuck that long, reload the last save and redo only that block); verify each layer before building the next (switches, then routers, then WAN, then tunnel, then IPsec).

---

## 16. Viva Q&A

**VLANs and switching**
- *What is a VLAN and why use it?* It splits one switch into separate broadcast domains. Here it isolates sensitive zones, like keeping patient Wi-Fi away from the EMR servers.
- *Trunk vs access port?* Access carries one VLAN untagged to an end device. A trunk carries many VLANs with 802.1Q tags between network devices.
- *Why EtherChannel?* More bandwidth and redundancy, and spanning tree treats the bundle as one link so neither cable is blocked.
- *How is RSTP better than STP?* Convergence in about 1 to 2 seconds instead of 30 to 50. DS1 is root and DS2 is backup root.
- *What is router-on-a-stick?* One physical router port split into 802.1Q sub-interfaces, one per VLAN, to route between VLANs.
- *Why /24 per VLAN, not whole /8 blocks?* 254 hosts per VLAN is enough, and private RFC 1918 space is required. Public ranges like 20.0.0.0 belong to real organizations.

**HSRP**
- *How does HSRP work?* Two routers share a virtual IP. The highest priority is Active (R-HQ1 = 110) and the other is Standby. If Active stops sending hellos, Standby takes over.
- *What do priority, preempt and tracking do?* Priority picks the Active router. Preempt lets a better router reclaim the role. Tracking lowers priority when the WAN fails so the gateway follows the working WAN.

**DHCP, DNS, port security**
- *How does a client on another network get a DHCP address?* The router's `ip helper-address` turns the broadcast into a unicast to the server. The server picks the pool matching the router's interface address (DORA).
- *Why is the pool gateway the HSRP virtual IP?* So clients keep a working gateway when one router fails.
- *Why does the clinic have local DHCP?* So it still works if the WAN or central server is down.
- *What does port security protect against, and what are the violation modes?* Unauthorized devices on a port. Sticky learns the allowed MAC. Modes: protect (silent drop), restrict (drop and log), shutdown (err-disabled). We used shutdown.
- *How do IP phones find the call controller?* DHCP gives them a TFTP server address (option 150) pointing to the router running `telephony-service`.

**IPv6 and tunneling**
- *What is dual-stack?* IPv4 and IPv6 running together on the same devices.
- *What is SLAAC?* The router advertises the /64 prefix and the host builds its own address. No server keeps track.
- *What does stateless DHCPv6 add?* DNS and other settings, signalled by the O flag, while no addresses are tracked.
- *Why tunnel instead of native IPv6?* The ISP carries only IPv4, so IPv6 is encapsulated in IPv4.
- *Why did the first ping time out?* Neighbor/ARP resolution (and IKE negotiation once IPsec is on) happens on the first packet.
- *Why is NAT64 not used?* It is unsupported in Packet Tracer. NAT64 translates IPv6 to IPv4 for IPv4-only hosts, which is different from tunneling.
- *Your title says GRE but you used `ipv6ip`. Why?* PT did not forward IPv6 over `gre ip` in our version, so we used the permitted 6in4 fallback. The concept is the same.

**IPsec and failover**
- *Why add IPsec to a tunnel?* A tunnel gives reachability but no security. IPsec adds encryption, integrity and authentication.
- *Phase 1 vs Phase 2?* Phase 1 (IKE) authenticates peers and builds a secure channel. Phase 2 sets up the IPsec SAs that encrypt data.
- *What does the crypto ACL do, and why must it mirror?* It selects traffic to encrypt. Each side must be the exact reverse of the other or the SAs will not match.
- *What does IPsec protect here, given PT cannot do IPv6 IPsec?* IPv6 rides inside IPv4 tunnel packets, and ESP encrypts those outer IPv4 packets.
- *What is a floating static route?* A static route with a high AD (200) that stays unused until the preferred AD 1 route disappears.
- *How does failover work end to end?* When ISP1 fails, Tunnel0 goes down and its route is withdrawn. The AD 200 routes take over via R-HQ2 and Tunnel1 over ISP2. HSRP tracking moves the IPv4 gateway to R-HQ2.
- *Why is R-HQ1's backup route via R-HQ2?* R-HQ1 has no tunnel over ISP2, so it hands traffic to R-HQ2 over the servers VLAN.

**Security and extras (Part 5)**
- *How do your ACLs isolate the patient network?* An extended ACL (and an IPv6 ACL) on the VLAN 30 gateway interfaces denies VLAN 30 to VLAN 40 and permits other traffic.
- *Why does the NAT ACL deny VPN traffic?* So tunnel and VPN traffic is not translated, which would break IPsec.

---

## 17. Presentation split

| Person | Topics |
|---|---|
| **Ashwin** | Topology, VLANs, EtherChannel/RSTP, router-on-a-stick, HSRP, DHCP |
| **Person 2** | IPv6, dual-stack, SLAAC/DHCPv6, tunneling (GRE/`ipv6ip`) |
| **Person 3** | IPsec, ACLs, NAT, floating routes and failover |

All three should be able to answer basic questions about every section, since the viva is taken together.

---

## 18. Save-file checkpoints

| File | Meaning |
|---|---|
| `v0_topology.pkt` | Devices, cables, hostnames |
| `v1_base.pkt` | VLANs, trunks, EtherChannel, RSTP, router-on-a-stick, HSRP working |
| `v2_dhcp.pkt` | After DHCP and relay |
| `v2_services.pkt` | After DNS, web, port security (VoIP pending) |
| `v3_dualstack.pkt` | IPv6 on LANs, SLAAC, DHCPv6 |
| `v3_wan.pkt` | IPv4 WAN working end to end |
| `v3_tunnel0.pkt` | Tunnel0 passing IPv6 (`ipv6ip`) |
| `v4_license.pkt` | `securityk9` on all three routers |
| `v4_ipsec.pkt` | IPsec working on Tunnel0 |
| `v4_tunnel1.pkt` | Backup tunnel with IPsec |
| `v4_tracking.pkt` | Floating routes and HSRP tracking in place |
| `v4_final.pkt` | Failover demo passes |
| `v5_final.pkt` | ACLs, NAT, final submission file (planned) |

---

## 19. Remaining to-do checklist

**Admin**
- [ ] Message the faculty with the project title and the team of 3, and confirm all three names are on the sheet (the title deadline was October 11 and has passed, so do this first)

**Build**
- [ ] Part 4B (IPsec, Tunnel1, floating routes, HSRP tracking, failover)
- [ ] Part 5 (ACLs, NAT/PAT, optional remote-access VPN)
- [ ] VoIP fix (low priority)

**Evidence and documents**
- [ ] Re-check DNS, web, DHCP and port-security screenshots
- [ ] Full test table with screenshots
- [ ] Report (topology, tables, key configs, results, limitations)
- [ ] Slides
- [ ] Mock viva with both teammates, 4 live demos rehearsed

---

## 20. Quick verification command sheet

| Goal | Command |
|---|---|
| EtherChannel | `show etherchannel summary` |
| Root bridge | `show spanning-tree vlan 10` |
| VLANs | `show vlan brief` |
| Trunks | `show interfaces trunk` |
| HSRP | `show standby brief` |
| HSRP tracking | `show track` |
| Router interfaces | `show ip interface brief` / `show ipv6 interface brief` |
| DHCP leases | `show ip dhcp binding`, `show ip dhcp pool` |
| Port security | `show port-security`, `show port-security interface fa0/5` |
| IPv4 routes | `show ip route` |
| IPv6 routes | `show ipv6 route` |
| Tunnel state | `show interfaces tunnel 0` |
| Neighbors | `show cdp neighbors` |
| IPsec Phase 1 | `show crypto isakmp sa` |
| IPsec Phase 2 | `show crypto ipsec sa` |
| Licenses | `show version` |
| Saved config check | `show running-config` |

**Golden rules:** read the prompt before pasting; `write memory` after every block; verify each layer before the next; 20-minute rule; save a new `.pkt` at every checkpoint.
# Secure Dual-Stack Hospital Network (Cisco Packet Tracer)

**Computer Networks Mini Project (CIE3, 15 marks)**

**Title:** Design and Simulation of a Secure Dual-Stack Hospital Network with IPv4-to-IPv6 Transition (Dual-Stack and Tunneling), DHCPv4/DHCPv6, and Redundant GRE-over-IPsec VPN using Cisco Packet Tracer

Built by Ashwin in Packet Tracer 8.x. Presented and defended in the viva by a team of 3.

---

## Project in 4 lines

A main hospital (HQ) runs IPv4 and IPv6 together (dual-stack). A branch clinic is IPv6-ready. The ISP carries only IPv4, so IPv6 is tunneled over it and the tunnel is protected with IPsec. A second tunnel over a backup ISP takes over automatically if the primary path fails. VLANs, ACLs and port security isolate sensitive zones (EMR servers) from patient Wi-Fi.

---

## Status

| Part | What | Status |
|---|---|---|
| 1 | Topology, cabling, hostnames | Done |
| 2 | VLANs, trunks, EtherChannel, RSTP, router-on-a-stick, HSRP | Done |
| 3 | DHCPv4 + relay, clinic DHCP, DNS, web server, port security | Done |
| 3 | VoIP (IP phones + telephony-service) | Not finished (phones stuck on "Configuring Vlan"), revisit later |
| 4A | Dual-stack IPv6, SLAAC, stateless DHCPv6, IPv4 WAN, Tunnel0 | Done (PC-Doctor pings clinic PC over IPv6) |
| 4B | Security license, IPsec, backup Tunnel1, floating routes, HSRP tracking, failover | **To do (next)** |
| 5 | IPv4 + IPv6 ACLs, NAT/PAT, optional remote-access VPN, test table, report, slides, mock viva | **To do** |

---

## Topology

17 devices, 20 cables.

- **Routers (2911):** R-HQ1, R-HQ2, R-CLINIC, ISP1, ISP2
- **Switches (2960-24TT):** DS1, DS2 (distribution), AS1, AS2 (access), SW-CLINIC
- **Servers:** SRV-DHCP (also DNS), SRV-WEB
- **PCs:** PC-Doctor, PC-Nurse, PC-Patient, Clinic-1, Clinic-2

```
R-HQ1 --g0/1-- ISP1 --g0/1-- R-CLINIC -- SW-CLINIC -- Clinic-1, Clinic-2
  |                              |
 DS1 ==EtherChannel== DS2        |
  |  \                /  |       |
 AS1  AS2 (each to both DS)      |
 PCs  Servers                    |
  |                              |
R-HQ2 --g0/1-- ISP2 ------g0/1---+
 (R-HQ2 g0/0 connects to DS2)
```

## Addressing

| VLAN | Name | IPv4 | HSRP gateway | IPv6 |
|---|---|---|---|---|
| 10 | DOCTORS | 10.0.10.0/24 | 10.0.10.1 | 2001:DB8:10::/64 |
| 20 | NURSES | 10.0.20.0/24 | 10.0.20.1 | 2001:DB8:20::/64 |
| 30 | PATIENT-WIFI | 10.0.30.0/24 | 10.0.30.1 | 2001:DB8:30::/64 |
| 40 | SERVERS | 10.0.40.0/24 | 10.0.40.1 | 2001:DB8:40::/64 |
| Clinic | | 192.168.50.0/24 | 192.168.50.1 | 2001:DB8:50::/64 |

R-HQ1 = `.2` (HSRP priority 110, preempt), R-HQ2 = `.3` (priority 100). SRV-DHCP = 10.0.40.10, SRV-WEB = 10.0.40.20.

| WAN link | Network |
|---|---|
| R-HQ1 to ISP1 | 200.1.1.0/30 |
| ISP1 to R-CLINIC | 200.2.2.0/30 |
| R-HQ2 to ISP2 | 200.3.3.0/30 |
| ISP2 to R-CLINIC | 200.4.4.0/30 |

Tunnel0 (primary) = 2001:DB8:FF1::/64. Tunnel1 (backup) = 2001:DB8:FF2::/64.

---

## What is done

**Part 1: Topology**
- All devices placed and cabled, hostnames set on routers and switches.

**Part 2: Layer 2 and gateway redundancy**
- VLANs 10/20/30/40 on all switches, 802.1Q trunks, EtherChannel (LACP) between DS1 and DS2.
- Rapid-PVST with DS1 as root and DS2 as backup root.
- Router-on-a-stick sub-interfaces on R-HQ1 and R-HQ2.
- HSRP on every VLAN (R-HQ1 active, R-HQ2 standby, preempt), failover tested.

**Part 3: IPv4 services**
- Central DHCP server with one pool per VLAN (gateway = HSRP virtual IP), `ip helper-address` on both HQ routers.
- Local DHCP on R-CLINIC for the clinic LAN.
- DNS (`web.hospital.local`) and a web server in the server VLAN.
- Port security (sticky MAC, violation shutdown) on access ports of AS1, AS2 and SW-CLINIC.

**Part 4A: IPv6 and the first tunnel**
- IPv6 on all HQ sub-interfaces (dual-stack), SLAAC for PCs, stateless DHCPv6 on R-CLINIC. R-HQ1 is the only IPv6 router advertiser.
- IPv4-only WAN with ISP1 and ISP2 (no IPv6 on the ISPs, by design).
- Tunnel0 between R-HQ1 and R-CLINIC carrying IPv6 over IPv4, with static IPv6 routes. Tunnel mode is `ipv6ip` (`gre ip` did not pass IPv6 in this Packet Tracer version).
- Verified: PC-Doctor pings a clinic PC over IPv6 through the tunnel.

---

## What is left

**Part 4B (next, about 3.5 to 4 hours)**
1. Enable the `securityk9` license on R-HQ1, R-HQ2, R-CLINIC (`license boot module c2900 technology-package securityk9`, write, reload).
2. IPsec on Tunnel0: IKE policy (AES-256, SHA, pre-shared key, DH group 2), transform set, crypto ACL for IP protocol 41, crypto map on the WAN interface. Verify with `show crypto isakmp sa` and `show crypto ipsec sa`.
3. Build Tunnel1 (R-HQ2 to R-CLINIC over ISP2) with its own IPsec.
4. Floating IPv6 static routes (AD 200): clinic returns via Tunnel1, R-HQ1 backs up via R-HQ2.
5. HSRP tracking of R-HQ1's WAN interface (decrement 20) and `preempt` on R-HQ2.
6. Failover demo: shut down ISP1, confirm traffic moves to Tunnel1, then fail back.

**Part 5 (about 7 hours)**
1. IPv4 extended ACL and IPv6 ACL blocking VLAN 30 (patient Wi-Fi) from VLAN 40 (servers), on both HQ routers.
2. NAT/PAT on the WAN interface with VPN/tunnel traffic denied in the NAT ACL.
3. Optional remote-access VPN (doctor's laptop from home), if Packet Tracer supports it.
4. Full test table: test, expected result, actual result, screenshot.
5. Report and slides.
6. Mock viva with the team, rehearsing 4 demos: inter-VLAN and DHCP, IPv6 over the tunnel, blocked ACL, failover with `show crypto ipsec sa`.

**Other**
- Fix VoIP (low priority): confirm `uck9` license, `switchport voice vlan 20` on phone ports, then `show ephone registered`.
- Message the faculty with the title and the team of 3 and confirm all three are on the sheet (the title deadline of October 11 has passed).

---

## Known issues and deviations

| Item | Note for the viva |
|---|---|
| Tunnel mode is `ipv6ip`, not `gre ip` | "PT did not forward IPv6 over `gre ip`, so we used the allowed 6in4 mode. Same idea: IPv6 wrapped in IPv4, with IPsec on top." |
| VoIP unfinished | "IP phones register to a router running telephony-service" |
| No HSRP for IPv6 | PT does not support it. R-HQ1 is the IPv6 gateway, and R-HQ2 is the backup path via a floating route |
| Failover test shuts down all of ISP1 | PT tunnels have no keepalive. Real networks use GRE keepalives or IP SLA |

## Packet Tracer limitations

- NAT64 is not supported (theory only).
- Native IPsec for IPv6 is not supported, so IPsec protects the outer IPv4 tunnel packets.
- The PT server's DHCP is IPv4 only, so DHCPv6 runs on the routers.
- The first ping over a tunnel often times out.

---


## Useful commands

`show etherchannel summary`, `show spanning-tree vlan 10`, `show vlan brief`, `show interfaces trunk`, `show standby brief`, `show ip interface brief`, `show ipv6 interface brief`, `show ip dhcp binding`, `show port-security`, `show ipv6 route`, `show interfaces tunnel 0`, `show crypto isakmp sa`, `show crypto ipsec sa`.

The full step-by-step version with every command is in `README_full.md`.
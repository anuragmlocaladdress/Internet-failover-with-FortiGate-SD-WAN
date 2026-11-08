# 🌐 FortiGate SD-WAN — Dual ISP Internet Failover (UC1)

### FortiOS 7.0 | SD-WAN Zones | Performance SLA | Lowest Cost (SLA) Rule | Dual ISP with BGP-Simulated Internet

**HQ Internet Breakout over Two ISPs with Automatic Failover — Full Lab Walkthrough**

![FortiGate](https://img.shields.io/badge/FortiGate-FortiOS%207.0.3-EE3124?style=for-the-badge&logo=fortinet&logoColor=white)
![SD-WAN](https://img.shields.io/badge/SD--WAN-Dual%20ISP%20Failover-0A66C2?style=for-the-badge)
![ISP](https://img.shields.io/badge/ISP%20Simulation-Cisco%20vIOS%20%2B%20eBGP-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![Lab](https://img.shields.io/badge/Lab-PNETLab%20%2F%20EVE--NG-6E40C9?style=for-the-badge)

---

**Anurag Mishra**
Solution Architect - IT & Security | Hyderabad, India

[![LinkedIn](https://img.shields.io/badge/LinkedIn-anuragmishra6-0A66C2?style=flat&logo=linkedin)](https://www.linkedin.com/in/anuragmishra6/)
[![GitHub](https://img.shields.io/badge/GitHub-anuragmlocaladdress-181717?style=flat&logo=github)](https://github.com/anuragmlocaladdress)

---

## 🗺️ Lab Topology

<img width="701" height="298" alt="image" src="https://github.com/user-attachments/assets/317a1b9f-9d35-4b68-8b7d-ff124afd07cd" />

*Two ISP routers peer with eBGP to simulate the internet. HQ FortiGate connects to both ISPs and uses SD-WAN to choose the healthy link.*

> 💡 **Overview:** HQ users reach the internet through **ISP1 (primary)**. When ISP1 fails — or stays up but stops reaching the internet — FortiGate SD-WAN moves traffic to **ISP2 (backup)** automatically, without any manual change.

---

## 📋 Table of Contents

1. [Introduction](#1-introduction)
2. [Components Used](#2-components-used)
3. [Device Role Assignment](#3-device-role-assignment)
4. [Network & IP Addressing](#4-network--ip-addressing)
5. [Prerequisites](#5-prerequisites)
6. [Phase 1 — Simulated Internet (ISP Routers)](#6-phase-1--simulated-internet-isp-routers)
7. [Phase 2 — HQ Core Switch](#7-phase-2--hq-core-switch)
8. [Phase 3 — FortiGate Interfaces & LAN Routes](#8-phase-3--fortigate-interfaces--lan-routes)
9. [Phase 4 — SD-WAN Zone & Members](#9-phase-4--sd-wan-zone--members)
10. [Phase 5 — Default Route, Performance SLA & SD-WAN Rule](#10-phase-5--default-route-performance-sla--sd-wan-rule)
11. [Phase 6 — Firewall Policy with NAT](#11-phase-6--firewall-policy-with-nat)
12. [Phase 7 — Verification & Failover Test](#12-phase-7--verification--failover-test)
13. [Lessons Learned](#13-lessons-learned)
14. [Limitations & Next Use Cases](#14-limitations--next-use-cases)
15. [References](#15-references)

---

## 1. Introduction

| Feature | Detail |
|---|---|
| Firewall | FortiGate VM, FortiOS 7.0.3 (evaluation license) |
| Use case | Internet breakout with two ISPs and automatic failover |
| SD-WAN zone | `UNDERLAY` (ISP1 + ISP2) |
| Health check | Ping to 8.8.8.8 and 1.1.1.1 through each ISP |
| SLA target | Latency 150 ms, jitter 30 ms, packet loss 2% |
| Steering strategy | Lowest Cost (SLA) — ISP1 cost 0 (primary), ISP2 cost 10 (backup) |
| NAT | Source NAT to the outgoing interface IP of whichever ISP is selected |
| Internet | Simulated with two Cisco vIOS routers peering over eBGP |

### Why SD-WAN instead of legacy routing?

| Legacy (static route + link-monitor / IP SLA) | FortiGate SD-WAN |
|---|---|
| Only knows **UP** or **DOWN** | Measures **latency, jitter and packet loss** continuously |
| Cannot detect a link that is up but slow | Moves traffic when a link is **out of SLA** |
| Misses "cable up, internet dead" if it only pings the gateway | Probes targets **beyond** the ISP |
| One route + one monitor per ISP | **One** default route to the SD-WAN zone |
| Duplicate firewall policies per ISP | **One** policy to the zone covers both ISPs |

---

## 2. Components Used

| Component | Version / Image | Detail |
|---|---|---|
| FortiGate (HQ_FW) | FortiGate-VM 7.0.3 | 1 vCPU, 2 GB RAM, evaluation license |
| ISP1 / ISP2 routers | Cisco vIOS (L3) | eBGP peering, anycast loopbacks, uRPF |
| HQ core switch (HQ-SW) | Cisco vIOS-L2 | Inter-VLAN routing, transit VLAN to firewall |
| Windows Server | Windows Server | AD / DNS / DHCP (10.10.100.10), test client |
| Lab platform | PNETLab (EVE-NG based) on VMware Workstation | Host: 64 GB RAM, 8 vCPU |

---

## 3. Device Role Assignment

| Device | Role |
|---|---|
| ISP1 | Internet service provider 1 (AS 65101) — HQ primary link |
| ISP2 | Internet service provider 2 (AS 65102) — HQ backup link |
| HQ_FW | Internet edge firewall with SD-WAN |
| HQ-SW | Layer 3 core switch — gateway for server and user VLANs |
| WinServer | AD / DNS / DHCP server and test host |

### How the ISP routers simulate the internet

There is no real internet in this lab, so each ISP router plays the role of the internet itself. Both routers act as **Google DNS (8.8.8.8)** and **Cloudflare DNS (1.1.1.1)** using loopback interfaces. This is **anycast** — the same IP lives in many places at once, just as Google runs 8.8.8.8 from many locations worldwide.

| Loopback | IP | Acts as | Used for |
|---|---|---|---|
| Loopback0 | 8.8.8.8/32 | Google DNS | SD-WAN health-check target |
| Loopback1 | 1.1.1.1/32 | Cloudflare DNS | Second health-check target |
| Loopback10 | 192.0.2.80/32 (ISP1), 192.0.2.90/32 (ISP2) | Public website | Browser and traceroute tests |

> 💡 8.8.8.8 and 1.1.1.1 are **not** advertised in BGP, so each ISP answers them locally. Shutting these loopbacks on ISP1 keeps the cable to the firewall **up** while the "internet" behind ISP1 is **dead** — exactly the failure SD-WAN detects and legacy routing cannot.

**Other real-ISP behaviour simulated:**
- **eBGP peering** between ISP1 and ISP2 — customers of one ISP can reach websites hosted on the other.
- **uRPF anti-spoofing** on customer ports — if the firewall forgets NAT, or sends traffic with the wrong ISP's source IP, the ISP drops it.
- **/29 public block per customer**, like a real leased-line handoff.

---

## 4. Network & IP Addressing

> ⚠️ All public IPs use the RFC 5737 documentation ranges (192.0.2.0/24, 198.51.100.0/24, 203.0.113.0/24) — safe for labs and public documentation.

### Physical interfaces

| Device | Interface | IP / Mask | Connects to |
|---|---|---|---|
| ISP1 | Gi0/0 | 198.51.100.1/29 | HQ_FW port1 |
| ISP1 | Gi0/1 | 198.51.100.9/29 | Branch_FW port1 *(planned)* |
| ISP1 | Gi0/3 | 192.0.2.1/30 | ISP2 Gi0/3 |
| ISP2 | Gi0/0 | 203.0.113.1/29 | HQ_FW port2 |
| ISP2 | Gi0/1 | 203.0.113.9/29 | Branch_FW port2 *(planned)* |
| ISP2 | Gi0/3 | 192.0.2.2/30 | ISP1 Gi0/3 |
| HQ_FW | port1 (WAN1-ISP1) | 198.51.100.2/29 | ISP1 Gi0/0 |
| HQ_FW | port2 (WAN2-ISP2) | 203.0.113.2/29 | ISP2 Gi0/0 |
| HQ_FW | port3 (TO-HQ_SW) | 172.16.100.1/30 | HQ-SW Gi0/0 |
| HQ_FW | port4 (MGMT) | 172.16.16.100/24 | Management network |
| WinServer | Ethernet1 | 10.10.100.10/24 | HQ-SW Gi0/1 |

### HQ VLANs

| VLAN | Name | Subnet | Gateway (HQ-SW SVI) |
|---|---|---|---|
| 100 | HQ-SERVERS | 10.10.100.0/24 | 10.10.100.2 |
| 99 | HQ-USERS | 10.10.101.0/24 | 10.10.101.2 |
| 500 | FW-TRANSIT | 172.16.100.0/30 | 172.16.100.2 (firewall .1) |

---

## 5. Prerequisites

- [x] PNETLab / EVE-NG with FortiGate-VM 7.0, Cisco vIOS and vIOS-L2 images
- [x] Windows Server VM reachable from the lab (bridged through a cloud network)
- [x] Dedicated management port on the FortiGate (port4) — never part of SD-WAN
- [x] **No** static default route on port1/port2 before SD-WAN is configured

> ⚠️ FortiOS refuses to add an interface as an SD-WAN member while a static route or firewall policy references it. Keep port1 and port2 clean until Phase 4.

---

## 6. Phase 1 — Simulated Internet (ISP Routers)

### Step 1.1 — ISP1

```
hostname ISP1
no ip domain-lookup
!
interface Loopback0
 description ANYCAST-PUBLIC-DNS-GOOGLE
 ip address 8.8.8.8 255.255.255.255
interface Loopback1
 description ANYCAST-PUBLIC-DNS-CLOUDFLARE
 ip address 1.1.1.1 255.255.255.255
interface Loopback10
 description PUBLIC-WEB-SERVER-ON-ISP1
 ip address 192.0.2.80 255.255.255.255
!
interface GigabitEthernet0/0
 description CUST-HQ
 ip address 198.51.100.1 255.255.255.248
 ip verify unicast source reachable-via rx
 no shutdown
interface GigabitEthernet0/1
 description CUST-BRANCH1
 ip address 198.51.100.9 255.255.255.248
 ip verify unicast source reachable-via rx
 no shutdown
interface GigabitEthernet0/2
 description CUST-BRANCH2-RESERVED
 ip address 198.51.100.17 255.255.255.248
 ip verify unicast source reachable-via rx
 no shutdown
interface GigabitEthernet0/3
 description PEERING-TO-ISP2
 ip address 192.0.2.1 255.255.255.252
 no shutdown
!
ip route 198.51.100.0 255.255.255.0 Null0
!
router bgp 65101
 bgp log-neighbor-changes
 neighbor 192.0.2.2 remote-as 65102
 network 198.51.100.0 mask 255.255.255.0
 network 192.0.2.80 mask 255.255.255.255
!
ip http server
end
write memory
```

### Step 1.2 — ISP2

```
hostname ISP2
no ip domain-lookup
!
interface Loopback0
 description ANYCAST-PUBLIC-DNS-GOOGLE
 ip address 8.8.8.8 255.255.255.255
interface Loopback1
 description ANYCAST-PUBLIC-DNS-CLOUDFLARE
 ip address 1.1.1.1 255.255.255.255
interface Loopback10
 description PUBLIC-WEB-SERVER-ON-ISP2
 ip address 192.0.2.90 255.255.255.255
!
interface GigabitEthernet0/0
 description CUST-HQ
 ip address 203.0.113.1 255.255.255.248
 ip verify unicast source reachable-via rx
 no shutdown
interface GigabitEthernet0/1
 description CUST-BRANCH1
 ip address 203.0.113.9 255.255.255.248
 ip verify unicast source reachable-via rx
 no shutdown
interface GigabitEthernet0/2
 description CUST-BRANCH2-RESERVED
 ip address 203.0.113.17 255.255.255.248
 ip verify unicast source reachable-via rx
 no shutdown
interface GigabitEthernet0/3
 description PEERING-TO-ISP1
 ip address 192.0.2.2 255.255.255.252
 no shutdown
!
ip route 203.0.113.0 255.255.255.0 Null0
!
router bgp 65102
 bgp log-neighbor-changes
 neighbor 192.0.2.1 remote-as 65101
 network 203.0.113.0 mask 255.255.255.0
 network 192.0.2.90 mask 255.255.255.255
!
ip http server
end
write memory
```

> 💡 **Why the Null0 route?** BGP only advertises a `network` that exists exactly in the routing table. The Null0 route creates the /24 so it can be announced. More specific connected /29s still win for real customer traffic.

### ✅ Checkpoint — BGP

```
ISP1# show ip bgp
     Network          Next Hop            Metric LocPrf Weight Path
 *>   192.0.2.80/32    0.0.0.0                  0         32768 i
 *>   192.0.2.90/32    192.0.2.2                0             0 65102 i
 *>   198.51.100.0     0.0.0.0                  0         32768 i
 *>   203.0.113.0      192.0.2.2                0             0 65102 i
```

> ⚠️ Right after the session comes up, `PfxRcd` may show **0** for up to ~60 seconds. Wait before troubleshooting.

---

## 7. Phase 2 — HQ Core Switch

```
hostname HQ-SW
ip routing
!
vlan 99
 name HQ-USERS
vlan 100
 name HQ-SERVERS
vlan 500
 name FW-TRANSIT
!
interface GigabitEthernet0/0
 description TO-FW_HQ-port3
 switchport mode access
 switchport access vlan 500
 spanning-tree portfast
interface GigabitEthernet0/1
 description TO-VMNET2-SERVERS
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
interface GigabitEthernet0/3
 description HQ-PC
 switchport mode access
 switchport access vlan 99
 spanning-tree portfast
!
interface Vlan500
 description TRANSIT-TO-FW_HQ
 ip address 172.16.100.2 255.255.255.252
interface Vlan100
 ip address 10.10.100.2 255.255.255.0
interface Vlan99
 ip address 10.10.101.2 255.255.255.0
 ip helper-address 10.10.100.10
!
ip route 0.0.0.0 0.0.0.0 172.16.100.1
end
write memory
```

> 💡 A dedicated **transit VLAN (500)** between the core and the firewall keeps server and user VLANs off the firewall uplink — standard production design.

---

## 8. Phase 3 — FortiGate Interfaces & LAN Routes

### Step 3.1 — Interfaces (Network → Interfaces)

| Port | Alias | Role | IP / Mask | Administrative Access |
|---|---|---|---|---|
| port1 | WAN1-ISP1 | WAN | 198.51.100.2/29 | PING only |
| port2 | WAN2-ISP2 | WAN | 203.0.113.2/29 | PING only |
| port3 | TO-HQ_SW | LAN | 172.16.100.1/30 | PING only |
| port4 | MGMT | LAN | 172.16.16.100/24 | HTTPS, SSH, PING |

<img width="1695" height="688" alt="image" src="https://github.com/user-attachments/assets/a1c92898-c40b-4a97-a786-1145ed7c921c" />

*FortiGate interfaces with aliases and roles.*

> ⚠️ Never enable HTTPS/SSH/FMG-Access on WAN ports. Management stays on the dedicated port4.

### Step 3.2 — Static routes to the HQ VLANs (Network → Static Routes)
<img width="1880" height="490" alt="image" src="https://github.com/user-attachments/assets/e7df5baa-2812-47ed-951f-f36d9697bcce" />


| Destination | Gateway | Interface |
|---|---|---|
| 10.10.100.0/24 | 172.16.100.2 | port3 |
| 10.10.101.0/24 | 172.16.100.2 | port3 |

> ⚠️ Always set the **gateway** on routes to networks behind another router. Without it, the route shows "directly connected" and only works by accident through proxy ARP.

<img width="1880" height="490" alt="image" src="https://github.com/user-attachments/assets/2b7bbe8e-6cfc-4851-894f-a544dc4729e3" />

*Routes to the HQ VLANs and the SD-WAN default route.*

---

## 9. Phase 4 — SD-WAN Zone & Members

### Step 4.1 — Create the zone (Network → SD-WAN → SD-WAN Zones)

Create a zone named **`UNDERLAY`** (leave members empty), then create the members and assign them to it.

### Step 4.2 — Create the members

| Interface | Zone | Gateway | Cost |
|---|---|---|---|
| WAN1-ISP1 (port1) | UNDERLAY | 198.51.100.1 | 0 (primary) |
| WAN2-ISP2 (port2) | UNDERLAY | 203.0.113.1 | 10 (backup) |

<img width="1694" height="688" alt="image" src="https://github.com/user-attachments/assets/a6c1a94a-fee4-4662-b2c7-f0b3acb7391d" />

*UNDERLAY zone with both ISP members.*

> 💡 **Why a custom zone?** Policies use zones, not members. A separate `UNDERLAY` zone (internet) and, later, an `OVERLAY` zone (VPN tunnels) allow NAT on internet traffic and no NAT on VPN traffic.

### ✅ Checkpoint

```
HQ_FW # diagnose sys sdwan member
Member(1): interface: port1, flags=0x0 , gateway: 198.51.100.1, priority: 0 1024, weight: 0
Member(2): interface: port2, flags=0x0 , gateway: 203.0.113.1, priority: 0 1024, weight: 0
```

---

## 10. Phase 5 — Default Route, Performance SLA & SD-WAN Rule

### Step 5.1 — Default route

Network → Static Routes → Create New: Destination `0.0.0.0/0.0.0.0`, Interface **UNDERLAY**.

<img width="1153" height="591" alt="image" src="https://github.com/user-attachments/assets/6bdc5701-c38c-47a4-b567-82fb853821b6" />


```
HQ_FW # get router info routing-table all
S*      0.0.0.0/0 [1/0] via 198.51.100.1, port1
                  [1/0] via 203.0.113.1, port2
S       10.10.100.0/24 [10/0] via 172.16.100.2, port3
S       10.10.101.0/24 [10/0] via 172.16.100.2, port3
```

> 💡 One route for both ISPs. Both paths appear in the routing table; the **SD-WAN rule** decides which one each session uses.

### Step 5.2 — Performance SLA (Network → SD-WAN → Performance SLAs)

| Field | Value |
|---|---|
| Probe mode | Active |
| Protocol | Ping |
| Servers | 8.8.8.8, 1.1.1.1 |
| Participants | Specify → port1, port2 |
| SLA target | Latency 150 ms, jitter 30 ms, packet loss 2% |
| Check interval | 500 ms |
| Failures before inactive | 5 (≈2.5 s detection) |
| Restore link after | 5 (anti-flapping) |
| Update static route | Enabled |

<img width="1011" height="785" alt="image" src="https://github.com/user-attachments/assets/fc221413-a864-4a9a-9d2c-7d6e289396ad" />

*Health check probing both ISPs.*

```
HQ_FW # diagnose sys sdwan health-check
Health Check(SLA):
Seq(1 port1): state(alive), packet-loss(0.000%) latency(1.354), jitter(0.273) ... sla_map=0x1
Seq(2 port2): state(alive), packet-loss(0.000%) latency(1.356), jitter(0.223) ... sla_map=0x1
```

### Step 5.3 — SD-WAN rule (Network → SD-WAN → SD-WAN Rules)

| Field | Value |
|---|---|
| Name | INTERNET-ISP1-PRIMARY |
| Source | HQ-SERVERS (10.10.100.0/24), HQ-USERS (10.10.101.0/24) |
| Destination | all |
| Strategy | **Lowest Cost (SLA)** |
| Interface preference | WAN1-ISP1, WAN2-ISP2 |
| Required SLA target | SLA#1 |

<img width="1207" height="917" alt="image" src="https://github.com/user-attachments/assets/12d39ef8-5e65-4cf3-8f74-967865733572" />

*Lowest Cost (SLA): use the cheapest link that meets the SLA.*

```
HQ_FW # diagnose sys sdwan service
Service(1): Address Mode(IPV4) flags=0x200 use-shortcut-sla
  Mode(sla), sla-compare-order
  Members(2):
    1: Seq_num(1 port1), alive, sla(0x1), gid(0), cfg_order(0), cost(0), selected
    2: Seq_num(2 port2), alive, sla(0x1), gid(0), cfg_order(1), cost(10), selected
  Src address(2):
        10.10.101.0-10.10.101.255
        10.10.100.0-10.10.100.255
  Dst address(1):
        0.0.0.0-255.255.255.255
```

> 💡 **The order of the members list is the decision.** port1 is first, so new sessions use ISP1.

---

## 11. Phase 6 — Firewall Policy with NAT

| Field | Value |
|---|---|
| Name | LAN-to-Internet |
| Incoming interface | TO-HQ_SW (port3) |
| Outgoing interface | **UNDERLAY** |
| Source | HQ-SERVERS, HQ-USERS |
| Destination | all |
| Service | ALL |
| NAT | Enabled — Use Outgoing Interface Address |
| Log allowed traffic | All Sessions |

<img width="1200" height="948" alt="image" src="https://github.com/user-attachments/assets/839093c2-16c4-4b83-8e26-a9393fc51d1e" />

*One policy to the zone covers both ISPs.*

> 💡 With **Use Outgoing Interface Address**, the NAT IP follows the ISP that SD-WAN selects — 198.51.100.2 on ISP1, 203.0.113.2 on ISP2 — so the source IP always belongs to the ISP carrying the packet and passes the ISP's uRPF check.

---

## 12. Phase 7 — Verification & Failover Test

### ✅ T1 — Baseline (traffic on ISP1)

```
C:\> tracert -d 192.0.2.90
  1    1 ms    1 ms    1 ms  10.10.100.2
  2    2 ms    1 ms    1 ms  172.16.100.1
  3    3 ms    2 ms    2 ms  198.51.100.1
  4    3 ms    4 ms    3 ms  192.0.2.90
```

Session table on HQ_FW:

```
hook=post dir=org act=snat 10.10.100.10:1->8.8.8.8:8(198.51.100.2:60417)
hook=pre dir=reply act=dnat 8.8.8.8:60417->198.51.100.2:0(10.10.100.10:1)
misc=0 policy_id=1 ...
sdwan_mbr_seq=1 sdwan_service_id=1
```

| Field | Meaning |
|---|---|
| `act=snat ... (198.51.100.2)` | Server NATed to ISP1's public IP |
| `policy_id=1` | LAN-to-Internet policy matched |
| `sdwan_service_id=1` | SD-WAN rule INTERNET-ISP1-PRIMARY applied |
| `sdwan_mbr_seq=1` | Member 1 (port1 / ISP1) selected |

### ✅ T2 — ISP1 failure

1. Start `ping -t 8.8.8.8` on the Windows Server.
2. On ISP1: `interface GigabitEthernet0/0` → `shutdown`.

![Failover Ping](images/08-failover-ping.png)
*A short drop while SD-WAN detects the failure, then the ping recovers through ISP2 automatically.*

**Result:** Traffic shifted to ISP2 automatically. ✅

> ⚠️ An existing NAT session may stay tied to the original ISP. If a continuous ping does not recover, restart it — the new session uses ISP2. Applications reconnect the same way in production.

### 🔜 Planned tests

| Test | Method | Expected |
|---|---|---|
| T3 — Link up, internet dead | `shutdown` Loopback0 + Loopback1 on ISP1 | port1 stays up, SLA fails, traffic moves to ISP2 |
| T4 — Recovery | `no shutdown` on ISP1 | After 5 good probes, traffic returns to ISP1 |
| T5 — NAT mistake | Disable NAT in the policy | Internet breaks; uRPF drop counter on ISP1 increases |

---

## 13. Lessons Learned

| # | Issue | How it was found | Fix |
|---|---|---|---|
| 1 | IOL L2 switch image showed correct routes/ARP/CEF but did **not** forward traffic between VLANs | Packet capture on the firewall: requests arrived, replies never reached the host | Replaced with **vIOS-L2** |
| 2 | Gateway typo on SD-WAN member (192.51.100.1 instead of 198.51.100.1) | `diagnose sys sdwan member` | Corrected the member gateway |
| 3 | Cost 10 missing on ISP2 member (both members cost 0) | `show system sdwan` — no `set cost 10` line | Set cost 10 on port2 |
| 4 | Static route saved without a gateway ("directly connected") | `get router info routing-table all` | Added next hop 172.16.x.2 |
| 5 | HTTP admin access left on the management port | Session list showed GUI over port 80 | Disabled HTTP, use HTTPS only |
| 6 | BGP showed `PfxRcd 0` right after the session came up | `show ip bgp summary` | Normal — routes arrived after ~60 seconds |

> 💡 **Method over guessing:** one packet capture at the next device shows exactly where traffic stops.

---

## 14. Limitations & Next Use Cases

**Limitations of this lab:**
- Evaluation license — no FortiGuard updates, so application/ISDB-based steering is not tested.
- No WAN latency/loss emulation on EVE-NG Community — brownout (slow link) testing is planned with a strict SLA target instead.
- Failover drops existing internet sessions (NAT IP changes) — expected behaviour.

**Next use cases in this series:**

| UC | Topic |
|---|---|
| UC2 | HQ ↔ Branch IPsec VPN over both ISPs (OVERLAY zone) |
| UC3 | SLA-based steering when a link is slow but up |
| UC4 | Path selection per application / service |
| UC5 | Load balancing across both ISPs |
| UC6 | Hub-and-spoke VPN with multiple branches |
| UC7 | ADVPN — on-demand branch-to-branch tunnels |
| UC8 | BGP over IPsec |
| UC9 | Branch internet through HQ (central breakout) |

---

## 15. References

- FortiOS 7.0 Administration Guide — SD-WAN (zones, members, performance SLA, SD-WAN rules)
- FortiOS 7.0 Administration Guide — Firewall policies and NAT
- RFC 5737 — IPv4 address blocks reserved for documentation
- BCP 38 — Network ingress filtering (uRPF anti-spoofing)

---

⭐ *If this lab helped you, star the repository and follow the series on LinkedIn.*

# 17 - Dynamic Routing Protocols

Module Number: 17
Status: Completed
Topic: RIP, OSPF, EIGRP, AD, Metrics, Floating Static Routes, Loopback, Passive Interfaces

> **Lab Type:** Exercise + Answer Key | **Devices:** 5 Routers (R1–R5), 2 Switches, 3 PCs in Packet Tracer
> 

---

## 🧭 Overview

This module explores features **common to all Interior Gateway Protocols (IGPs)**. You will run RIP, OSPF, and EIGRP side-by-side to understand:

- How routing protocol updates work (broadcast vs multicast)
- Administrative Distance (AD) — which protocol wins when multiple run simultaneously
- Metrics — how each protocol chooses the best path
- Floating static routes — backup routes that activate on failure
- Loopback interfaces — virtual, always-up interfaces
- Passive interfaces — suppress routing updates on selected interfaces

---

## 📂 Lab Setup

- Download `17 Dynamic Routing Protocols.zip`, extract and open in Packet Tracer.
- Topology: 5 routers (R1–R5) with multiple interconnected links.

---

## 🔑 Part 1 – Routing Protocol Updates (RIPv1 vs RIPv2)

### Configure RIPv1 on All Routers

```
R1(config)#router rip
R1(config-router)#network 10.0.0.0
R1(config-router)#no auto-summary
```

Repeat on R2–R5.

### Debug RIP Updates

```
R1#debug ip rip         ! Watch live RIP update traffic
R1#undebug all          ! Turn off all debugging
```

### RIPv1 vs RIPv2 Update Traffic

| Version | Update Traffic | Address | Notes |
| --- | --- | --- | --- |
| RIPv1 | Broadcast | 255.255.255.255 | All hosts must process |
| RIPv2 | Multicast | 224.0.0.9 | Only RIPv2 routers process beyond L3 |

### Upgrade to RIPv2

```
R1(config)#router rip
R1(config-router)#version 2
```

### Verify RIP Routes on R1

```
R1#show ip route
R   10.1.0.0/24 [120/1] via 10.0.0.2    ! AD=120, metric=1 hop
R   10.1.1.0/24 [120/2] via 10.0.0.2
               [120/2] via 10.0.3.2     ! Two equal-cost paths = ECMP load balancing
```

> 💡 Two routes to 10.1.1.0/24 appear because both paths have equal hop count of 2 → ECMP load balancing.
> 

### View RIP Database

```
R1#show ip rip database
```

> RIP is a **Distance Vector** protocol — it only knows its directly connected neighbors and the routes they advertised. It does NOT know the full network topology.
> 

---

## 🔑 Part 2 – Administrative Distance (AD)

### What is Administrative Distance?

AD is a **trustworthiness rating** assigned to route sources. Lower AD = more trusted = preferred in the routing table. When multiple protocols advertise the same destination, the **lowest AD wins**.

| Route Source | AD |
| --- | --- |
| Connected | 0 |
| Static | 1 |
| EIGRP | 90 |
| OSPF | 110 |
| RIP | 120 |

> 📖 The AD is shown in the routing table as the first number in brackets: `[110/3]` = AD 110, metric 3.
> 

---

### Enable OSPF on All Routers

```
R1(config)#router ospf 1
R1(config-router)#network 10.0.0.0 0.255.255.255 area 0
```

> After OSPF converges, **RIP routes disappear** from the routing table — OSPF (AD 110) beats RIP (AD 120).
> 

### Enable EIGRP on All Routers

```
R1(config)#router eigrp 100
R1(config-router)#no auto-summary
R1(config-router)#network 10.0.0.0 0.255.255.255
```

> After EIGRP converges, **RIP and OSPF routes are replaced** — EIGRP (AD 90) beats both.
> 

### Remove a Protocol

```
R1(config)#no router ospf 1      ! Remove OSPF
R1(config)#no router rip         ! Remove RIP
R1(config)#no router eigrp 100   ! Remove EIGRP
```

---

## 🔑 Part 3 – Routing Protocol Metrics

### How Each Protocol Chooses the Best Path

| Protocol | Metric | Description |
| --- | --- | --- |
| RIP | Hop count | Number of routers between source and destination (max 15) |
| OSPF | Cost | Based on interface bandwidth — higher bandwidth = lower cost |
| EIGRP | Composite | Bandwidth + delay (primarily) — more accurate than RIP |

### Why OSPF Prefers R2 Path Over R5

R5's interfaces have `bandwidth 10000` (10 Mbps) configured vs default FastEthernet at 100 Mbps. OSPF calculates lower cost for higher-bandwidth paths → top path via R2 wins.

```
O   10.1.1.0/24 [110/3]  via 10.0.0.2   ! Low cost via R2 (100Mbps)
               [110/21] via 10.0.3.2    ! NOT shown — higher cost via R5
```

### Why EIGRP Prefers R2 Path Over R5

Same reason — EIGRP's composite metric also factors in bandwidth. 10Mbps R5 links give a much higher (worse) metric.

```
D   10.1.1.0/24 [90/33280]  via 10.0.0.2   ! Lower metric via R2
D   10.1.3.0/24 [90/261120] via 10.0.3.2   ! Higher metric forces use of R5 for this network
```

### OSPF vs RIP Database: Key Difference

| Protocol | Type | Database Contents |
| --- | --- | --- |
| RIP | Distance Vector | Only neighbor advertisements — partial view |
| OSPF | Link State | Full topology of every link in the area — complete view |

```
R1#show ip rip database       ! Shows learned routes per neighbor
R1#show ip ospf database      ! Shows full link-state topology (LSAs)
```

---

## 🔑 Part 4 – Floating Static Routes

### What is a Floating Static Route?

A floating static route is a **backup static route** with a manually set AD **higher than the primary routing protocol**. It stays hidden in the routing table while the primary protocol route is available, and "floats" to the surface only when the primary route disappears.

```
ip route <network> <mask> <next-hop> <AD>
```

### Configure Floating Static Routes (Backup to EIGRP AD 90)

Set AD to 95 (higher than EIGRP's 90, so EIGRP is preferred when available):

```
R1(config)#ip route 10.1.0.0 255.255.0.0 10.0.3.2 95
R2(config)#ip route 10.0.0.0 255.255.0.0 10.1.0.1 95
R3(config)#ip route 10.0.0.0 255.255.0.0 10.1.1.1 95
R4(config)#ip route 10.0.0.0 255.255.0.0 10.1.3.2 95
R5(config)#ip route 10.0.0.0 255.255.0.0 10.0.3.1 95
R5(config)#ip route 10.1.0.0 255.255.0.0 10.1.3.1 95
```

> ⚠️ Summary routes (`/16`) are used here so the task is accomplished with just 6 commands.
> 

> ⚠️ Set AD to 95 on R5 even though EIGRP isn't running there yet — prevents issues when EIGRP is enabled in future.
> 

### Routing Table with Floating Static Route

```
S   10.1.0.0/16 [95/0] via 10.0.3.2      ! Floating static — visible but NOT used
D   10.1.0.0/24 [90/30720] via 10.0.0.2  ! EIGRP /24 wins — longer prefix + lower AD
D   10.1.1.0/24 [90/33280] via 10.0.0.2
D   10.1.2.0/24 [90/35840] via 10.0.0.2
```

> 💡 The floating static /16 appears in the routing table but is NOT used because the EIGRP /24 routes have both **longer prefix** AND **lower AD**.
> 

### Failover Test — Shut Down R2's Interface

```
R2(config)#interface f0/0
R2(config-if)#shutdown
```

**After failover — R1 routing table:**

```
S   10.1.0.0/16 [95/0] via 10.0.3.2   ! Floating static now ACTIVE — only route left
```

Connectivity restored via R5 path:

```
PC1> tracert 10.1.2.10
  1  10.0.1.1   (R1)
  2  10.0.3.2   (R5)
  3  10.1.3.1   (R4)
  4  10.1.2.10  (PC3)
```

---

## 🔑 Part 5 – Loopback Interfaces

### What is a Loopback Interface?

A **loopback interface** is a virtual, software-only interface that is **always up** as long as the router is running. It is never physically connected to anything.

**Uses:**

- Stable management IP that never goes down
- Router ID for OSPF/EIGRP
- Testing and diagnostics

### Configure Loopback Interfaces

```
R1(config)#interface loopback0
R1(config-if)#ip address 192.168.0.1 255.255.255.255    ! /32 host route

R2(config)#interface loopback0
R2(config-if)#ip address 192.168.0.2 255.255.255.255
! Repeat for R3 (192.168.0.3), R4 (192.168.0.4), R5 (192.168.0.5)
```

> 💡 Loopbacks use /32 — a single host route. No need for a network prefix.
> 

### Advertise Loopbacks via EIGRP

Loopbacks in the 192.168.0.0/24 range are NOT automatically included:

```
R1(config)#router eigrp 100
R1(config-router)#network 192.168.0.0 0.0.0.255    ! Include loopbacks in EIGRP
```

Repeat on all routers. Routing table on R1 then shows:

```
C   192.168.0.1/32 is directly connected, Loopback0
D   192.168.0.2/32 [90/156160] via 10.0.0.2
D   192.168.0.3/32 [90/158720] via 10.0.0.2
D   192.168.0.4/32 [90/161280] via 10.0.0.2
D   192.168.0.5/32 [90/386560] via 10.0.3.2
```

---

## 🔑 Part 6 – Adjacencies and Passive Interfaces

### View EIGRP Neighbors

```
R1#show ip eigrp neighbors
H    Address      Interface    Hold   Uptime    SRTT    RTO    Q   Seq
1    10.0.3.2     Fa1/1        14     00:17:21  33      198    0   16
0    10.0.0.2     Fa0/0        11     00:19:21  36      216    0   32
```

> R1 has EIGRP adjacencies (neighbors) with R2 (via Fa0/0) and R5 (via Fa1/1).
> 

---

### What is a Passive Interface?

A **passive interface** tells the routing protocol: *"include this interface's network in routing updates, but do NOT send or receive routing updates on this interface."*

**Use cases:**

- Loopback interfaces (no neighbors — no point sending updates)
- LAN-facing interfaces (suppress updates to end-user devices)
- Security — don't advertise routing info to untrusted segments

### Configure Passive Interfaces on R1

```
R1(config)#router eigrp 100
R1(config-router)#passive-interface loopback0         ! Suppress updates on loopback
R1(config-router)#passive-interface fastethernet1/1   ! Suppress updates toward R5
```

### Effect on R5

- EIGRP adjacency between R1 and R5 **goes down** (hold timer expires)
- R5 removes all EIGRP routes learned via R1
- R5 reroutes all traffic via R4 instead

```
%DUAL-5-NBRCHANGE: IP-EIGRP 100: Neighbor 10.0.3.2 (FastEthernet1/1) is down: holding time expired
```

---

## 📋 Key Commands Summary

| Command | Purpose |
| --- | --- |
| `router rip` | Enable RIP routing process |
| `network <network>` | Advertise a network in RIP |
| `version 2` | Use RIPv2 (multicast updates) |
| `router ospf <pid>` | Enable OSPF routing process |
| `network <net> <wildcard> area <id>` | Include interface in OSPF area |
| `router eigrp <asn>` | Enable EIGRP routing process |
| `no auto-summary` | Disable classful summarization |
| `debug ip rip` | Watch live RIP update messages |
| `undebug all` | Stop all debugging |
| `show ip rip database` | View RIP topology database |
| `show ip ospf database` | View OSPF link-state database |
| `show ip eigrp neighbors` | View EIGRP adjacency table |
| `ip route <net> <mask> <hop> <AD>` | Add floating static route with custom AD |
| `interface loopback0` | Create/enter loopback interface |
| `passive-interface <int>` | Suppress routing updates on interface |
| `no router ospf 1` | Remove OSPF process |

---

## 🗒️ Key Takeaways

- **RIPv1** uses broadcast (255.255.255.255); **RIPv2** uses multicast (224.0.0.9) — more efficient.
- **Administrative Distance** decides which protocol wins when multiple protocols know the same route — lower AD wins.
- **Metrics** decide which path is best within a single protocol — lower metric wins.
- **OSPF and EIGRP** both consider **bandwidth** in their metrics — unlike RIP which only counts hops.
- **Floating static routes** use a custom high AD to stay dormant while the primary protocol works, then activate on failure.
- **Loopback interfaces** are always up — ideal for router IDs, management IPs, and testing.
- **Passive interfaces** include a network in routing updates but stop sending/receiving updates on that interface — use on loopbacks and LAN segments.
- OSPF is **Link State** (knows full topology); RIP is **Distance Vector** (knows only neighbor advertisements).
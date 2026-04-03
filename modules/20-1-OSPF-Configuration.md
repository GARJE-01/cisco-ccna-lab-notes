# 20-1 - OSPF Configuration

Module Number: 20-1
Status: Completed
Topic: OSPF Basic Config, Cost, Reference Bandwidth, Default Route, Multi-Area, DR/BDR

> **Lab Type:** Exercise + Answer Key | **Devices:** R1–R9, Switches, PCs in Packet Tracer
> 

---

## 🧭 Overview

This is a comprehensive OSPF lab covering four major areas:

- **Basic OSPF configuration** — Router ID, adjacencies, network statements
- **OSPF Cost** — reference bandwidth, manual cost override, load balancing
- **Default route injection** — redistributing a static default route into OSPF
- **Multi-Area OSPF** — ABRs, inter-area routes, manual summarisation
- **DR/BDR Election** — Designated Router election and priority manipulation

---

## 📂 Lab Setup

- Download `20-1 OSPF Configuration.zip`, extract and open in Packet Tracer.
- IP addresses are **already configured** on all router interfaces.
- Topology: R1–R5 multi-router network + R4 has ISP link at `203.0.113.0/24`; R6–R9 on a shared Ethernet segment for DR/BDR exercise.

---

## 🔑 Part 1 – OSPF Basic Configuration

### Step 1 – Configure Loopback Interfaces (Router IDs)

```
R1(config)#interface loopback0
R1(config-if)#ip address 192.168.0.1 255.255.255.255
! Repeat: R2=192.168.0.2, R3=192.168.0.3, R4=192.168.0.4, R5=192.168.0.5
```

> 💡 OSPF Router ID selection priority:
> 

> 1. Manually configured Router ID (`router-id` command)
> 

> 2. Highest loopback IP address ← used here
> 

> 3. Highest active physical interface IP
> 

---

### Step 2 – Enable Single-Area OSPF (Area 0)

On all routers R1–R5:

```
R1(config)#router ospf 1
R1(config-router)#network 10.0.0.0 0.255.255.255 area 0
R1(config-router)#network 192.168.0.0 0.0.0.255 area 0
```

> `network` uses **wildcard masks** (inverse of subnet mask): `0.255.255.255` = match any address starting with 10.
> 

> Do NOT include `203.0.113.0/24` — this is the ISP network, added separately.
> 

---

### Step 3 – Verify Router ID

```
R1#show ip protocols
Router ID 192.168.0.1     ! Loopback address used as Router ID
```

---

### Step 4 – Verify Adjacencies

```
R1#show ip ospf neighbor
Neighbor ID     Pri   State       Dead Time   Address     Interface
192.168.0.5     1     FULL/BDR    00:00:31    10.0.3.2    FastEthernet1/1
192.168.0.2     1     FULL/DR     00:00:39    10.0.0.2    FastEthernet0/0
```

> **FULL** = adjacency fully established, routes exchanged.
> 

> **DR/BDR** = Designated Router / Backup Designated Router roles on the segment.
> 

---

### Step 5 – Verify Routing Table

```
R1#show ip route
O   10.1.0.0/24 [110/2] via 10.0.0.2    ! AD=110, cost=2
O   10.1.1.0/24 [110/3] via 10.0.0.2
               [110/3] via 10.0.3.2    ! ECMP — equal cost both paths
O   192.168.0.2/32 [110/2] via 10.0.0.2
O   192.168.0.5/32 [110/2] via 10.0.3.2
```

> OSPF uses **cost** (based on bandwidth), not hop count. Lower cost = preferred path.
> 

---

## 🔑 Part 2 – OSPF Cost and Reference Bandwidth

### OSPF Cost Formula

```
Cost = Reference Bandwidth / Interface Bandwidth
```

**Default reference bandwidth = 100 Mbps**, so:

- FastEthernet (100 Mbps) → cost = 100/100 = **1**
- Serial (1.5 Mbps) → cost = 100/1.5 = **64**

> ⚠️ Problem: Gigabit (1000 Mbps) and 10 Gigabit both get cost 1 with default settings — OSPF can't differentiate high-speed links. Fix by increasing reference bandwidth.
> 

---

### Step 6 – Set Reference Bandwidth (All Routers)

To make 100 Gbps = cost of 1:

```
R1(config)#router ospf 1
R1(config-router)#auto-cost reference-bandwidth 100000    ! 100,000 Mbps = 100 Gbps
```

Repeat on **all routers R1–R5**.

**New FastEthernet cost:**

```
Cost = 100000 / 100 = 1000
```

**Verify:**

```
R1#show ip ospf interface FastEthernet 0/0
Cost: 1000
```

> After reference bandwidth change, cost to 10.1.2.0/24 changes from **3** to **3000**.
> 

---

### Step 7 – Configure Manual OSPF Cost for Load Balancing

Two paths to 10.1.2.0/24:

- R1 → R5 → R4: cost = 1000 + 1000 + 1000 = **3000** (via R5, 2 hops)
- R1 → R2 → R3 → R4: cost = 1000 + 1000 + 1000 + 1000 = **4000** (via R2, 3 hops)

To equalize both paths to **4000**, increase R1→R5 and R5→R4 links to cost 1500 each:

```
R1(config)#int f1/1
R1(config-if)#ip ospf cost 1500

R5(config)#int f0/0
R5(config-if)#ip ospf cost 1500
R5(config)#int f0/1
R5(config-if)#ip ospf cost 1500

R4(config)#int f1/0
R4(config-if)#ip ospf cost 1500
```

**Math check:** R1(1500) + R5(1500) + R4-dest(1000) = **4000** ✓

**Verify load balancing:**

```
R1#show ip route
O   10.1.2.0/24 [110/4000] via 10.0.3.2, FastEthernet1/1
               [110/4000] via 10.0.0.2, FastEthernet0/0   ! ECMP!
```

---

## 🔑 Part 3 – Default Route Injection into OSPF

### Step 8 – Advertise ISP Network via OSPF with Passive Interface

```
R4(config)#router ospf 1
R4(config-router)#passive-interface f1/1          ! Don’t send OSPF updates to ISP
R4(config-router)#network 203.0.113.0 0.0.0.255 area 0   ! Include ISP network in OSPF
```

**Verify all routers see 203.0.113.0/24:**

```
R1#show ip route
O   203.0.113.0/24 [110/3001] via 10.0.3.2 ...
                  [110/3001] via 10.0.0.2 ...
```

---

### Step 9 – Default Route to Internet + Inject into OSPF

```
R4(config)#ip route 0.0.0.0 0.0.0.0 203.0.113.2        ! Static default to ISP
R4(config)#router ospf 1
R4(config-router)#default-information originate          ! Share default via OSPF
```

**Verify all routers have a default route:**

```
R1#show ip route
Gateway of last resort is 10.0.3.2 to network 0.0.0.0
O*E2 0.0.0.0/0 [110/1] via 10.0.3.2 ...
              [110/1] via 10.0.0.2 ...
```

> `O*E2` = OSPF External Type 2 default route. The `*` means gateway of last resort. **E2** means the metric does not increase as it crosses routers (stays at 1).
> 

---

## 🔑 Part 4 – Multi-Area OSPF

### OSPF Router Roles

| Role | Description |
| --- | --- |
| **Internal Router** | All interfaces in one area |
| **Backbone Router** | Has at least one interface in Area 0 |
| **ABR (Area Border Router)** | Interfaces in multiple areas — connects area to backbone |
| **ASBR** | Redistributes external routes (e.g., static, RIP) into OSPF |

### Multi-Area Design for This Lab

| Router | Role | Areas |
| --- | --- | --- |
| R1 | Internal Router | Area 1 only |
| R2 | ABR | Area 0 + Area 1 |
| R3 | Backbone Router | Area 0 only |
| R4 | Backbone Router + ASBR | Area 0 (redistributes default route) |
| R5 | ABR | Area 0 + Area 1 |

---

### Step 10 – Reconfigure Routers for Multi-Area

**R1** – move all to Area 1:

```
R1(config)#router ospf 1
R1(config-router)#network 10.0.0.0 0.255.255.255 area 1
R1(config-router)#network 192.168.0.0 0.0.0.255 area 1
R1#copy run start
R1#reload
```

**R2** – split: Fa0/1 stays Area 0, Fa0/0 moves to Area 1:

```
R2(config)#router ospf 1
R2(config-router)#no network 10.0.0.0 0.255.255.255 area 0
R2(config-router)#network 10.1.0.0 0.0.0.255 area 0    ! Fa0/1 side stays Area 0
R2(config-router)#network 10.0.0.0 0.0.0.255 area 1    ! Fa0/0 side moves to Area 1
R2#copy run start
R2#reload
```

**R5** – split: Fa0/0 stays Area 0, Fa0/1 moves to Area 1:

```
R5(config)#router ospf 1
R5(config-router)#no network 10.0.0.0 0.255.255.255 area 0
R5(config-router)#network 10.1.3.0 0.0.0.255 area 0    ! Fa0/0 side stays Area 0
R5(config-router)#network 10.0.3.0 0.0.0.255 area 1    ! Fa0/1 side moves to Area 1
R5#copy run start
R5#reload
```

> R3 and R4 require **no changes** — all their interfaces are already in Area 0.
> 

---

### Step 11 – Verify Interface Areas

```
R2#show ip ospf interface
FastEthernet0/1    Area 0    ! R2 backbone side
FastEthernet0/0    Area 1    ! R2 toward R1
```

---

### Step 12 – Inter-Area Routes in Routing Table

After multi-area conversion, R1's routing table shows `O IA` (inter-area) routes:

```
R1#show ip route
O IA 10.1.0.0/24 [110/2000] via 10.0.0.2    ! Inter-area route
O IA 10.1.1.0/24 [110/3000] via 10.0.0.2
O IA 10.1.2.0/24 [110/4000] via 10.0.0.2
O*E2 0.0.0.0/0   [110/1]    via 10.0.0.2    ! External route stays E2
```

> ⚠️ Route count is the **same** as single-area — OSPF does NOT auto-summarise between areas. Manual summarisation is required.
> 

---

### Step 13 – Configure Manual Summarisation on ABRs

ABRs summarise routes when advertising between areas:

```
R2(config)#router ospf 1
R2(config-router)#area 0 range 10.1.0.0 255.255.0.0   ! Summarise 10.1.x.x into Area 0
R2(config-router)#area 1 range 10.0.0.0 255.255.0.0   ! Summarise 10.0.x.x into Area 1

R5(config)#router ospf 1
R5(config-router)#area 0 range 10.1.0.0 255.255.0.0
R5(config-router)#area 1 range 10.0.0.0 255.255.0.0
```

**Result on R1 — 4 routes replaced by 1 summary:**

```
O IA 10.1.0.0/16 [110/2000] via 10.0.0.2    ! One summary instead of four /24s
```

> R1 is routing via R2 only (not load balancing via R5) because `ip ospf cost 1500` was set earlier on the R1–R5 link, making that path more expensive.
> 

---

## 🔑 Part 5 – DR and BDR Election

### What are DR and BDR?

On **multi-access networks** (Ethernet), OSPF elects:

- **DR (Designated Router)** — all other routers form full adjacencies with the DR only
- **BDR (Backup Designated Router)** — takes over if DR fails
- **DROther** — all other routers; only form 2-WAY adjacency with each other

This reduces the number of OSPF adjacencies from n(n-1)/2 to just n.

### DR/BDR Election Rules

1. Highest **OSPF Priority** wins — default is 1 for all routers
2. If priority is tied, highest **Router ID** wins
3. Priority 0 = never elected as DR/BDR
4. Election is **non-preemptive** — changing priority doesn't remove an existing DR

---

### Step 14 – Configure Loopbacks on R6–R9

```
R6(config)#interface loopback0
R6(config-if)#ip address 192.168.0.6 255.255.255.255
! R7=192.168.0.7, R8=192.168.0.8, R9=192.168.0.9
```

### Step 15 – Enable OSPF on R6–R9

```
R6(config)#router ospf 1
R6(config-router)#network 172.16.0.0 0.0.0.255 area 0
R6(config-router)#network 192.168.0.0 0.0.0.255 area 0
R6(config-router)#auto-cost reference-bandwidth 100000
```

### Step 16 – Verify DR/BDR Election (Default)

```
R6#show ip ospf interface FastEthernet 0/0
State DROTHER, Priority 1
Designated Router (ID) 192.168.0.9, Interface address 172.16.0.9    ! Highest Router ID wins
Backup Designated Router (ID) 192.168.0.8, Interface address 172.16.0.8
```

**Neighbor states:**

```
R6#show ip ospf neighbor
Neighbor ID    State
192.168.0.9    FULL/DR        ! Full adjacency with DR
192.168.0.8    FULL/BDR       ! Full adjacency with BDR
192.168.0.7    2WAY/DROTHER   ! Only 2-way with other DROthers
```

---

### Step 17 – Force R6 as DR Using Priority

Since election is non-preemptive, must clear the OSPF process after setting priority:

```
R6(config)#interface FastEthernet0/0
R6(config-if)#ip ospf priority 100    ! Higher than default of 1 — R6 wins next election
R6(config-if)#end
R6#clear ip ospf process              ! Force re-election
Reset ALL OSPF processes? [no]: yes
```

**Verify R6 is now DR:**

```
R6#show ip ospf interface FastEthernet 0/0
State DR, Priority 100
Designated Router (ID) 192.168.0.6, Interface address 172.16.0.6
Backup Designated Router (ID) 192.168.0.8, Interface address 172.16.0.8
```

---

## 📋 Key Commands Summary

| Command | Purpose |
| --- | --- |
| `router ospf <pid>` | Start OSPF process |
| `network <net> <wildcard> area <id>` | Include interfaces matching network in OSPF area |
| `auto-cost reference-bandwidth <Mbps>` | Set reference bandwidth for cost calculation |
| `ip ospf cost <value>` | Manually set OSPF cost on an interface |
| `passive-interface <int>` | Stop sending OSPF hellos on interface |
| `default-information originate` | Inject default static route into OSPF |
| `area <id> range <net> <mask>` | Configure inter-area route summarisation on ABR |
| `ip ospf priority <0-255>` | Set OSPF DR election priority (0 = never DR) |
| `clear ip ospf process` | Force OSPF re-election and re-convergence |
| `show ip ospf neighbor` | View OSPF adjacency table and DR/BDR states |
| `show ip ospf interface <int>` | View OSPF details per interface (cost, DR, area) |
| `show ip protocols` | View OSPF Router ID and network statements |
| `show ip ospf database` | View OSPF link-state database (LSAs) |

---

## 🗒️ Key Takeaways

- OSPF **Router ID** = highest loopback IP, or highest active interface IP if no loopback.
- OSPF uses **cost** (bandwidth-based), not hop count — always set `auto-cost reference-bandwidth` on all routers to the same value.
- `ip ospf cost` lets you **manually override** the cost on a per-interface basis for traffic engineering.
- `default-information originate` redistributes a static default route into OSPF as an **E2 external route**.
- Multi-area OSPF reduces LSA flooding but does NOT auto-summarise — use `area range` on ABRs for manual summarisation.
- `O IA` = inter-area route (learned from another area via an ABR).
- `O*E2` = external default route redistributed into OSPF (metric stays flat as it crosses routers).
- DR/BDR election is won by **highest priority** then **highest Router ID** — it is **non-preemptive**, so use `clear ip ospf process` to force re-election.
# 25-1 - Spanning Tree Troubleshooting

Module Number: 25-1
Status: Completed
Topic: STP Root Bridge, Port Roles, Forwarding Path Analysis, MAC Address Table

> **Lab Type:** Troubleshooting Exercise + Answer Key | **Devices:** 4 Switches (Acc3, Acc4, CD1, CD2), 2 Routers (R1, R2 via HSRP), PCs in Packet Tracer
> 

---

## 🧭 Overview

Spanning Tree Protocol (STP) is enabled by default on all Cisco switches. This module teaches you how to **read and understand the STP topology**, identify when traffic is not taking the optimal path, and trace the exact forwarding path through the switched network.

**Scenario:** NOC reports traffic is not taking the most direct path from PCs to the Internet. Investigate and report — do NOT change any configuration.

---

## 🧠 Spanning Tree Fundamentals

### Why STP Exists

Switched networks can have **redundant links** for resilience, but redundant links create **Layer 2 loops** — broadcast frames loop endlessly, crashing the network. STP prevents loops by **blocking redundant ports**, creating a loop-free logical tree.

### STP Port Roles

| Port Role | Description | State |
| --- | --- | --- |
| **Root Port (RP)** | Best path toward the Root Bridge — one per non-root switch | Forwarding |
| **Designated Port (DP)** | Best port on each network segment for forwarding | Forwarding |
| **Alternate Port (AP)** | Backup path — blocked to prevent loops | Blocking |

### Root Bridge Election

1. Switch with **lowest Bridge Priority** wins — default priority is **32768** for all
2. If priorities are equal, switch with **lowest MAC address** wins
3. Root Bridge has **all ports as Designated Ports** (all forwarding)

---

## 🔬 Troubleshooting Walkthrough

### Step 1 – Verify HSRP Active Router (Layer 3)

```
R1#show standby
  State is Active
  Priority 110 (configured 110)
  Preemption enabled
  Active router is local
  Standby router is 10.10.10.3
```

> R1 is the HSRP Active gateway — all PC traffic should route through R1. R1 connects directly to CD1 (not CD2), so ideally traffic should flow through CD1.
> 

---

### Step 2 – Verify Layer 3 Connectivity and Path

```
PC1> ping 203.0.113.9      ✅ Success
PC2> ping 203.0.113.9      ✅ Success

PC1> tracert 203.0.113.9
  1  10.10.10.1 (HSRP VIP via R1)   ! Direct path ✓
PC2> tracert 203.0.113.9
  1  10.10.10.1 (HSRP VIP via R1)   ! Also direct path at L3
```

> Layer 3 looks fine — both PCs reach the internet. The problem must be at **Layer 2** (switched path).
> 

---

### Step 3 – Verify VLAN and Trunk Configuration

```
CD1#show run
interface GigabitEthernet0/1
  switchport access vlan 10
  switchport mode access      ! Router port in VLAN 10

interface FastEthernet0/21
  switchport trunk native vlan 199
  switchport mode trunk       ! Trunk to Acc4

interface FastEthernet0/24
  switchport trunk native vlan 199
  switchport mode trunk       ! Trunk between core switches
```

> VLAN and trunk config looks correct. Time to check Spanning Tree.
> 

---

### Step 4 – Check the STP Root Bridge

```
Acc3#show spanning-tree vlan 10
  Root ID  Priority  32768
           Address   0001.C962.D43D
           This bridge is the root    ← ACC3 IS THE ROOT BRIDGE!
```

> 🚨 **Problem Found:** Acc3 (an access-layer switch) is the Root Bridge. It should be a **core/distribution switch** (CD1 or CD2) to ensure optimal traffic paths.
> 

---

### Step 5 – Why is Acc3 the Root Bridge?

```
Acc3#show run | include priority
Acc3#                           ! No output — Bridge Priority not configured

CD1#show run | include priority
CD1#                            ! Not configured either

CD2#show run | include priority
CD2#                            ! Not configured

Acc4#show run | include priority
Acc4#                           ! Not configured
```

> No priority configured anywhere — all switches have default priority **32768**. With equal priorities, the switch with the **lowest MAC address** wins. Acc3 happened to have the lowest MAC (`0001.C962.D43D`).
> 

**MAC addresses in this lab:**

| Switch | MAC Address | Result |
| --- | --- | --- |
| Acc3 | 0001.C962.D43D | **Root Bridge ← lowest MAC** |
| Acc4 | 0060.708A.D564 | DROther |
| CD1 | 0090.0CA0.3902 | DROther |
| CD2 | 0090.0C16.7A9B | DROther |

---

### Step 6 – Map the STP Topology

Using `show spanning-tree vlan 10` on each switch:

**Root Bridge:** Acc3 → All ports are **Designated Ports (forwarding)**

**Non-root switches — each has one Root Port (toward Acc3):**

| Switch | Root Port | Path to Root |
| --- | --- | --- |
| CD1 | F0/24 | CD1 → Acc3 |
| CD2 | F0/24 | CD2 → Acc3 |
| Acc4 | F0/24 | Acc4 → CD2 |

**Blocking ports (Alternate):**

- CD1 G0/2 → **BLOCKING** (Alternate Port)
- Acc4 F0/21 → **BLOCKING** (Alternate Port)

---

### Step 7 – Determine Actual Traffic Paths

**PC1 path (connected to Acc3):**

```
PC1 → Acc3 → CD1 → R1 → SP1
```

> ✅ This IS the most direct path — PC1 traffic is fine.
> 

**PC2 path (connected to Acc4):**

```
PC2 → Acc4 → CD2 → Acc3 → CD1 → R1 → SP1
```

> ❌ This is NOT optimal — traffic bounces through CD2 and Acc3 before reaching CD1 and R1. The direct link from Acc4 to CD1 (F0/21) is blocked by STP!
> 

---

### Step 8 – Verify Path with MAC Address Table

```
R1#show standby
  Active virtual MAC address is 0000.0C07.AC01   ! HSRP virtual MAC
```

Clear ARP on PC2, ping the HSRP VIP, then trace through MAC tables:

```
Acc4#show mac address-table
  0000.0c07.ac01    DYNAMIC    F0/24    ! Reached via CD2, NOT via CD1 F0/21
```

> Confirms PC2 traffic goes through F0/24 (to CD2) — not F0/21 (direct to CD1) — because F0/21 on Acc4 is blocked by STP.
> 

---

## 📋 Key STP Troubleshooting Commands

| Command | Purpose |
| --- | --- |
| `show spanning-tree vlan <id>` | Full STP topology for a VLAN (root bridge, port roles, states) |
| `show spanning-tree vlan <id> brief` | Compact view of port roles and states |
| `show spanning-tree summary` | Overview of STP status across all VLANs |
| `show run \ | include priority` |
| `show mac address-table` | Trace which port a MAC is reachable through |
| `show standby` | Verify HSRP active router and virtual MAC |

---

## 🗒️ Key Takeaways

- STP **Root Bridge election**: lowest Bridge Priority wins; tiebreaker = lowest MAC address.
- The default Bridge Priority is **32768** — if no priority is configured, the switch with the lowest MAC becomes Root Bridge, which may not be optimal.
- The Root Bridge should always be a **central, high-capacity switch** (core/distribution layer) — NOT an access layer switch.
- All Root Bridge ports are **Designated (forwarding)**. Non-root switches have one **Root Port** toward the root, and all remaining links are either Designated or **Alternate (blocking)**.
- When the Root Bridge is in the wrong place, traffic takes **suboptimal paths** — verify with `show spanning-tree` and `show mac address-table`.
- Use `show spanning-tree vlan <id>` on every switch to build a complete picture of the Layer 2 forwarding topology.
- STP problems are fixed in the **next module (25-2)** using `spanning-tree vlan root primary/secondary`.
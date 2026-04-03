# 16 - Routing Fundamentals

Module Number: 16
Status: Completed
Topic: Connected/Local/Static/Summary/Default Routes, Longest Prefix Match, Load Balancing

> **Lab Type:** Exercise + Answer Key | **Devices:** 5 Routers (R1–R5), 2 Switches, 3 PCs in Packet Tracer
> 

---

## 🧭 Overview

This module covers all the foundational route types and routing concepts you need to understand before moving to dynamic routing protocols:

- **Connected & Local routes** — auto-created when interfaces come up
- **Static routes** — manually configured per-network routes
- **Summary routes** — one route covering multiple subnets
- **Default route** — the gateway of last resort for unknown destinations
- **Longest Prefix Match** — how routers pick the best route when multiple match
- **Load Balancing** — splitting traffic over multiple equal-cost paths

---

## 📂 Lab Setup

- Download `16 Routing Fundamentals.zip`, extract and open in Packet Tracer.
- All routers are unconfigured at start. PCs are pre-configured with IPs/gateways.
- Say `no` when prompted with the initial configuration dialog on each router.

---

## 🔑 Part 1 – Connected and Local Routes

### Concepts

| Route Type | Code | Created By | Description |
| --- | --- | --- | --- |
| Connected | `C` | Automatically | Network address of a configured, up interface |
| Local | `L` | Automatically (IOS 15+) | Host /32 route for the router's own IP on that interface |

> 💡 Both C and L routes are **automatically added** when you configure an IP on an interface and bring it up. No manual route config needed.
> 

---

### Configure R1 Interfaces

```
R1(config)#int f0/0
R1(config-if)#ip address 10.0.0.1 255.255.255.0
R1(config-if)#no shut
R1(config-if)#int f0/1
R1(config-if)#ip address 10.0.1.1 255.255.255.0
R1(config-if)#no shut
R1(config-if)#int f1/0
R1(config-if)#ip address 10.0.2.1 255.255.255.0
R1(config-if)#no shut
R1(config-if)#int f1/1
R1(config-if)#ip address 10.0.3.1 255.255.255.0
R1(config-if)#no shut
```

---

### Verify Routing Table

```
R1#show ip route

Gateway of last resort is not set

10.0.0.0/8 is variably subnetted, 4 subnets, 2 masks
C   10.0.1.0/24 is directly connected, FastEthernet0/1
L   10.0.1.1/32 is directly connected, FastEthernet0/1
C   10.0.2.0/24 is directly connected, FastEthernet1/0
L   10.0.2.1/32 is directly connected, FastEthernet1/0
```

> ⚠️ Routes for 10.0.0.0/24 and 10.0.3.0/24 do **NOT** appear — the other ends (R2, R5) are still shutdown. **Both sides of a link must be up** for routes to appear.
> 

> 💡 Switch ports are **not shutdown by default** — so links to SW1/SW2 come up immediately. Router ports **are** shutdown by default.
> 

---

### Ping Tests

```
PC1> ping 10.0.2.10     ✅ Success — R1 is directly connected to both subnets
PC1> ping 10.1.2.10     ❌ Fails — R1 has no route to 10.1.2.0/24
PC1> tracert 10.0.2.10
  1  10.0.1.1   (R1)
  2  10.0.2.10  (PC2)
```

---

## 🔑 Part 2 – Static Routes

### Configure R2, R3, R4 Interfaces

```
R2: f0/0 = 10.0.0.2/24,  f0/1 = 10.1.0.2/24
R3: f0/1 = 10.1.0.1/24,  f0/0 = 10.1.1.2/24
R4: f0/0 = 10.1.1.1/24,  f0/1 = 10.1.2.1/24,  f1/0 = 10.1.3.1/24
```

---

### Static Route Syntax

```
ip route <destination-network> <subnet-mask> <next-hop-ip>
```

---

### Configure Static Routes on All Routers

**R1** (needs routes to all 10.1.x.x networks via R2):

```
R1(config)#ip route 10.1.0.0 255.255.255.0 10.0.0.2
R1(config)#ip route 10.1.1.0 255.255.255.0 10.0.0.2
R1(config)#ip route 10.1.2.0 255.255.255.0 10.0.0.2
R1(config)#ip route 10.1.3.0 255.255.255.0 10.0.0.2
```

**R2** (needs routes to 10.0.x.x via R1, and 10.1.x.x via R3):

```
R2(config)#ip route 10.0.1.0 255.255.255.0 10.0.0.1
R2(config)#ip route 10.0.2.0 255.255.255.0 10.0.0.1
R2(config)#ip route 10.0.3.0 255.255.255.0 10.0.0.1
R2(config)#ip route 10.1.1.0 255.255.255.0 10.1.0.1
R2(config)#ip route 10.1.2.0 255.255.255.0 10.1.0.1
R2(config)#ip route 10.1.3.0 255.255.255.0 10.1.0.1
```

**R3** (needs routes to 10.0.x.x via R2, and 10.1.2/3.x via R4):

```
R3(config)#ip route 10.0.0.0 255.255.255.0 10.1.0.2
R3(config)#ip route 10.0.1.0 255.255.255.0 10.1.0.2
R3(config)#ip route 10.0.2.0 255.255.255.0 10.1.0.2
R3(config)#ip route 10.0.3.0 255.255.255.0 10.1.0.2
R3(config)#ip route 10.1.2.0 255.255.255.0 10.1.1.1
R3(config)#ip route 10.1.3.0 255.255.255.0 10.1.1.1
```

**R4** (needs routes to all 10.0.x.x networks via R3):

```
R4(config)#ip route 10.0.0.0 255.255.255.0 10.1.1.2
R4(config)#ip route 10.0.1.0 255.255.255.0 10.1.1.2
R4(config)#ip route 10.0.2.0 255.255.255.0 10.1.1.2
R4(config)#ip route 10.0.3.0 255.255.255.0 10.1.1.2
R4(config)#ip route 10.1.0.0 255.255.255.0 10.1.1.2
```

**Verify path PC1 → PC3:**

```
PC1> tracert 10.1.2.10
  1  10.0.1.1  (R1)
  2  10.0.0.2  (R2)
  3  10.1.0.1  (R3)
  4  10.1.1.1  (R4)
  5  10.1.2.10 (PC3)
```

> 💡 IP return traffic does not have to follow the same path — each router makes independent forwarding decisions based on its own routing table.
> 

---

## 🔑 Part 3 – Summary Routes

### What is a Summary Route?

A summary route (also called an **aggregate route**) uses a **shorter prefix** to represent multiple more-specific subnets with a single routing entry. This reduces the number of routes in the table.

---

### Replace 4 Static Routes with 1 Summary

**Remove individual /24 routes on R1:**

```
R1(config)#no ip route 10.1.0.0 255.255.255.0 10.0.0.2
R1(config)#no ip route 10.1.1.0 255.255.255.0 10.0.0.2
R1(config)#no ip route 10.1.2.0 255.255.255.0 10.0.0.2
R1(config)#no ip route 10.1.3.0 255.255.255.0 10.0.0.2
```

**Add one summary route covering all 10.1.x.x networks:**

```
R1(config)#ip route 10.1.0.0 255.255.0.0 10.0.0.2     ! /16 covers 10.1.0.0 – 10.1.255.255
```

**Routing table now shows:**

```
S   10.1.0.0/16 [1/0] via 10.0.0.2     ! One route replaces four
```

> ✅ Connectivity to PC3 is fully restored with just this one route.
> 

---

## 🔑 Part 4 – Longest Prefix Match

### The Rule

When a router has **multiple routes that match** a destination IP, it always uses the route with the **longest (most specific) prefix**.

| Route | Prefix Length | Matches |
| --- | --- | --- |
| 10.1.0.0/16 | /16 | 10.1.0.0 – 10.1.255.255 |
| 10.1.3.0/24 | /24 | 10.1.3.0 – 10.1.3.255 only |

> If a packet is destined for 10.1.3.2, the **/24 route wins** — it is more specific.
> 

---

### Lab Demonstration

**Configure R5 interfaces:**

```
R5(config)#int f0/0
R5(config-if)#ip add 10.1.3.2 255.255.255.0
R5(config-if)#no shut
R5(config-if)#int f0/1
R5(config-if)#ip add 10.0.3.2 255.255.255.0
R5(config-if)#no shut
```

**Add summary route on R5 for return path:**

```
R5(config)#ip route 10.0.0.0 255.255.0.0 10.0.3.1     ! One route back to all R1 networks
```

> Without a /24 route to 10.1.3.0/24 on R1, traffic takes the long path R1→R2→R3→R4→R5.
> 

**Add more-specific /24 route on R1 to force direct path:**

```
R1(config)#ip route 10.1.3.0 255.255.255.0 10.0.3.2
```

**R1 routing table now shows both:**

```
S   10.1.0.0/16 [1/0] via 10.0.0.2     ! Summary — used for all 10.1.x.x except...
S   10.1.3.0/24 [1/0] via 10.0.3.2     ! More specific — wins for 10.1.3.x traffic
```

**Verify direct path PC1 → R5:**

```
PC1> tracert 10.1.3.2
  1  10.0.1.1   (R1)
  2  10.1.3.2   (R5)  ← Direct! Only 2 hops now
```

---

## 🔑 Part 5 – Default Route

### What is a Default Route?

A default route matches **any destination** not found in the routing table. It acts as the **gateway of last resort** — the "catch-all" for unknown destinations (e.g., Internet traffic).

```
ip route 0.0.0.0 0.0.0.0 <next-hop>
```

> 0.0.0.0/0 has the **shortest possible prefix** (length 0), so it only wins when no other route matches.
> 

---

### Configure Default Routes on All Routers

```
R1(config)#ip route 0.0.0.0 0.0.0.0 10.0.0.2          ! Send Internet traffic to R2
R2(config)#ip route 0.0.0.0 0.0.0.0 10.1.0.1          ! Forward to R3
R3(config)#ip route 0.0.0.0 0.0.0.0 10.1.1.1          ! Forward to R4
R4(config)#ip route 0.0.0.0 0.0.0.0 203.0.113.2       ! R4's ISP link
R5(config)#ip route 0.0.0.0 0.0.0.0 10.1.3.1          ! Forward to R4 via R1 link
```

**Verify on R4:**

```
R4#show ip route
Gateway of last resort is 203.0.113.2 to network 0.0.0.0
S*  0.0.0.0/0 [1/0] via 203.0.113.2
```

> `S*` means static default route — the `*` indicates it is the **candidate default** (gateway of last resort).
> 

---

## 🔑 Part 6 – Load Balancing

### What is Load Balancing?

When a router has **two routes to the same destination with equal metrics**, it installs both in the routing table and splits traffic across them (Equal-Cost Multi-Path, **ECMP**).

---

### Configure Load Balancing on R1 for Internet Traffic

```
R1(config)#ip route 0.0.0.0 0.0.0.0 10.0.0.2     ! Path via R2
R1(config)#ip route 0.0.0.0 0.0.0.0 10.0.3.2     ! Path via R5
```

**R1 routing table shows both paths:**

```
S*  0.0.0.0/0 [1/0] via 10.0.0.2
              [1/0] via 10.0.3.2
```

**Configure R4 to load balance return traffic via both paths:**

```
R4(config)#ip route 10.0.1.0 255.255.255.0 10.1.3.2    ! PC1 subnet via R5
R4(config)#ip route 10.0.2.0 255.255.255.0 10.1.3.2    ! PC2 subnet via R5
```

---

## 📋 Key Commands Summary

| Command | Purpose |
| --- | --- |
| `show ip route` | View the full routing table |
| `ip route <net> <mask> <next-hop>` | Add a static route |
| `no ip route <net> <mask> <next-hop>` | Remove a static route |
| `ip route 0.0.0.0 0.0.0.0 <next-hop>` | Configure a default route |
| `tracert <ip>` (PC) | Trace path from a Windows PC |
| `traceroute <ip>` (Router) | Trace path from a Cisco router |

---

## 🗒️ Key Takeaways

- **Connected (C) and Local (L)** routes are auto-created when an interface is configured and up — both ends of a link must be up.
- **Static routes** require manual configuration for every remote network on every router.
- **Summary routes** use a shorter prefix to represent multiple subnets — reduces routing table size.
- **Default route** `0.0.0.0/0` is the gateway of last resort — matches everything not specifically routed.
- **Longest prefix match** always wins — a /24 beats a /16 beats a /8 beats the default /0.
- **Load balancing (ECMP)** occurs automatically when two routes have the same destination and equal administrative distance/metric.
- Return traffic does **not** have to follow the same path as forward traffic — each router decides independently.
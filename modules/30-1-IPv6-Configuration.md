# 30-1 - IPv6 Configuration

Module Number: 30-1
Status: Completed
Topic: IPv6 Addressing, EUI-64, Link Local, Global Unicast, ipv6 unicast-routing, Static Routes

> **Lab Type:** Exercise + Answer Key | **Devices:** 3 Routers (R1–R3), 2 PCs (IOS routers as hosts) in Packet Tracer
> 

---

## 🧭 Overview

This module adds **dual-stack IPv6** to an existing IPv4 network. The company runs EIGRP for IPv4 and now needs IPv6 support for a new application. You will:

- Configure global unicast and link-local IPv6 addresses
- Use EUI-64 for auto-generating host addresses
- Discover the critical `ipv6 unicast-routing` command
- Configure IPv6 static routes end-to-end

---

## 🧠 IPv6 Address Types

| Type | Prefix | Scope | Description |
| --- | --- | --- | --- |
| **Global Unicast** | `2000::/3` | Internet-routable | Public IPv6 addresses (equivalent to public IPv4) |
| **Link-Local** | `FE80::/10` | Link-only | Auto-generated, not routable beyond local link |
| **Loopback** | `::1/128` | Local host only | Equivalent to IPv4 127.0.0.1 |
| **Unspecified** | `::/128` | — | Used as source when no address assigned |
| **Multicast** | `FF00::/8` | Varies | Replaces IPv4 broadcast |

---

## 📂 Lab Setup

- Download `30-1 IPv6 Configuration.zip`, extract and open in Packet Tracer.
- IPv4 already configured and working (EIGRP routing between all networks).
- No IPv6 configuration exists at the start.
- PC1 and PC2 are IOS routers acting as end hosts (using `ipv6 route ::/0` for default gateway).

**IPv4 topology:**

| Device | Interface | IPv4 Address |
| --- | --- | --- |
| R1 | Fa0/1 | 10.10.0.1/24 (toward PC1) |
| R1 | Fa0/0 | 10.10.1.1/24 (toward R2) |
| R2 | Fa0/0 | 10.10.1.2/24 (toward R1) |
| R2 | Fa0/1 | 10.10.2.2/24 (toward R3) |
| R3 | Fa0/0 | 10.10.2.1/24 (toward R2) |
| R3 | Fa0/1 | 10.10.3.1/24 (toward PC2) |
| PC1 | Fa0/0 | 10.10.0.10 |
| PC2 | Fa0/0 | 10.10.3.10 |

---

## 🔑 Part 1 – IPv6 Global Unicast Address Configuration

### IPv6 Address Notation

- IPv6 addresses are **128 bits**, written as 8 groups of 4 hex digits: `2001:0db8:0000:0001:0000:0000:0000:0001`
- Leading zeros in each group can be omitted: `2001:db8:0:1::1`
- Consecutive all-zero groups can be replaced with `::` (once only)

### Configure Global Unicast Addresses on Routers

**R1:**

```
R1(config)#int f0/1
R1(config-if)#ipv6 address 2001:db8::1/64        ! PC1-facing interface
R1(config-if)#no shutdown

R1(config)#int f0/0
R1(config-if)#ipv6 address 2001:db8:0:1::1/64   ! R2-facing interface
R1(config-if)#no shutdown
```

**R2:**

```
R2(config)#int f0/0
R2(config-if)#ipv6 address 2001:db8:0:1::2/64   ! R1-facing
R2(config)#int f0/1
R2(config-if)#ipv6 address 2001:db8:0:2::2/64   ! R3-facing
```

**R3:**

```
R3(config)#int f0/0
R3(config-if)#ipv6 address 2001:db8:0:2::1/64   ! R2-facing
R3(config)#int f0/1
R3(config-if)#ipv6 address 2001:db8:0:3::1/64   ! PC2-facing
```

**IPv6 Network Map:**

| Network | Subnet |
| --- | --- |
| PC1 subnet | 2001:db8::/64 |
| R1 ↔ R2 link | 2001:db8:0:1::/64 |
| R2 ↔ R3 link | 2001:db8:0:2::/64 |
| PC2 subnet | 2001:db8:0:3::/64 |

---

## 🔑 Part 2 – EUI-64 Addressing

### What is EUI-64?

**EUI-64** automatically generates a 64-bit interface ID from the device's **MAC address** by:

1. Splitting the 48-bit MAC in half
2. Inserting `FFFE` in the middle
3. Flipping the 7th bit (Universal/Local bit)

**Example:** MAC `0000.0C47.14C0` → EUI-64 host ID `0200:0CFF:FE47:14C0` → Full address: `2001:db8::200:CFF:FE47:14C0`

### Configure EUI-64 on PC1 and PC2

```
PC1(config)#int f0/0
PC1(config-if)#ipv6 address 2001:db8::/64 eui-64    ! Auto-generates host ID from MAC
PC1(config-if)#no shutdown

PC2(config)#int f0/0
PC2(config-if)#ipv6 address 2001:db8:0:3::/64 eui-64
PC2(config-if)#no shutdown
```

**Verify EUI-64 addresses:**

```
PC1#show ipv6 interface brief
FastEthernet0/0    [up/up]
  FE80::200:CFF:FE47:14C0
  2001:DB8::200:CFF:FE47:14C0     ! EUI-64 generated global address

PC2#show ipv6 interface brief
FastEthernet0/0    [up/up]
  FE80::201:C7FF:FE50:8E8A
  2001:DB8:0:3:201:C7FF:FE50:8E8A
```

> ⚠️ EUI-64 addresses **vary by MAC address** — note down PC2's actual address from `show ipv6 interface brief` before pinging it.
> 

---

## 🔑 Part 3 – Link-Local Addresses

### What are Link-Local Addresses?

- **Automatically created** on every IPv6-enabled interface (EUI-64 generated from MAC)
- Prefix: `FE80::/10`
- **Not routable** — only valid on the local link (same subnet)
- Required for IPv6 to work — used for neighbor discovery, routing protocol hellos

### Auto-Generated vs Manual Link-Local

When you configure a global unicast address, the router **automatically creates** an EUI-64 link-local address on that interface. Interfaces without any IPv6 config have **no link-local** address.

```
R1#show ipv6 interface brief
FastEthernet0/0    [up/up]
  FE80::20D:BDFF:FE2D:27D4     ! Auto EUI-64 link-local
  2001:DB8:0:1::1
FastEthernet0/1    [up/up]
  FE80::2D0:97FF:FE64:3118     ! Auto EUI-64 link-local
  2001:DB8::1
FastEthernet1/0    [administratively down/down]
  unassigned                    ! No link-local — no IPv6 configured here
```

### Configure Manual Link-Local Addresses (Simpler for routing)

```
R1(config)#int f0/0
R1(config-if)#ipv6 address fe80::1 link-local    ! Override auto EUI-64 with simple address
R1(config)#int f0/1
R1(config-if)#ipv6 address fe80::1 link-local    ! Same link-local on all R1 interfaces

R2(config)#int f0/0
R2(config-if)#ipv6 address fe80::2 link-local
R2(config)#int f0/1
R2(config-if)#ipv6 address fe80::2 link-local

R3(config)#int f0/0
R3(config-if)#ipv6 address fe80::3 link-local
R3(config)#int f0/1
R3(config-if)#ipv6 address fe80::3 link-local
```

> 💡 The same link-local address can be used on multiple interfaces of the same router — link-locals are only significant on the local link.
> 

### Ping Link-Local Address (Must Specify Interface)

```
R2#ping fe80::1
Output Interface: FastEthernet0/0    ! Must specify — link-local is not globally unique
!!!!!
Success rate is 100 percent (5/5)
```

### View IPv6 Neighbor Table (IPv6 ARP equivalent)

```
R2#show ipv6 neighbors
IPv6 Address           Age  Link-layer Addr  State   Interface
2001:DB8:0:1::1        0    000D.BD2D.27D4   REACH   Fa0/0
2001:DB8:0:2::1        0    0030.F2BA.30E7   REACH   Fa0/1
FE80::1                0    000D.BD2D.27D4   REACH   Fa0/0
FE80::3                0    0030.F2BA.30E7   REACH   Fa0/1
```

> IPv6 uses **NDP (Neighbor Discovery Protocol)** instead of ARP. `show ipv6 neighbors` is the equivalent of `show arp`.
> 

---

## 🔑 Part 4 – The Critical ipv6 unicast-routing Command

### Why Static Routes Alone Aren't Enough

Even with static routes configured and PC1 having a default gateway, pings can still fail:

```
PC1#ping 2001:db8:0:1::2
.....
Success rate is 0 percent (0/5)
```

### Root Cause

By default, Cisco routers **do not forward IPv6 packets between interfaces** — they act as IPv6 hosts, not IPv6 routers. You must explicitly enable IPv6 routing:

```
R1(config)#ipv6 unicast-routing
R2(config)#ipv6 unicast-routing
R3(config)#ipv6 unicast-routing
```

> ⚠️ This is one of the most common IPv6 gotchas — forgetting `ipv6 unicast-routing` means the router drops all forwarded IPv6 packets even if routes exist.
> 

**Verify IPv6 protocols running:**

```
R1#show ipv6 protocols
IPv6 Routing Protocol is "connected"
IPv6 Routing Protocol is "ND"
! No dynamic routing protocol — only connected and ND
```

---

## 🔑 Part 5 – IPv6 Static Routes

### IPv6 Static Route Syntax

```
ipv6 route <prefix/length> <next-hop-IPv6-address>
```

### Configure Default Routes on PCs

```
PC1(config)#ipv6 route ::/0 2001:db8::1       ! ::/0 = IPv6 default route (all destinations)
PC2(config)#ipv6 route ::/0 2001:db8:0:3::1
```

> `::/0` is the IPv6 equivalent of `0.0.0.0/0` — matches all destinations.
> 

### Configure Static Routes on Routers

**R1** (needs routes to R2-R3 side networks):

```
R1(config)#ipv6 route 2001:db8:0:2::/64 2001:db8:0:1::2    ! Via R2
R1(config)#ipv6 route 2001:db8:0:3::/64 2001:db8:0:1::2    ! Via R2
```

**R2** (needs routes to both end networks):

```
R2(config)#ipv6 route 2001:db8::/64     2001:db8:0:1::1    ! PC1 subnet via R1
R2(config)#ipv6 route 2001:db8:0:3::/64 2001:db8:0:2::1    ! PC2 subnet via R3
```

**R3** (needs routes to R1-R2 side networks):

```
R3(config)#ipv6 route 2001:db8::/64     2001:db8:0:2::2    ! PC1 subnet via R2
R3(config)#ipv6 route 2001:db8:0:1::/64 2001:db8:0:2::2   ! R1-R2 link via R2
```

### Verify IPv6 Routing Table

```
R1#show ipv6 route
IPv6 Routing Table - 7 entries
C   2001:DB8::/64     [0/0] via Fa0/1, directly connected
L   2001:DB8::1/128   [0/0] via Fa0/1, receive
C   2001:DB8:0:1::/64 [0/0] via Fa0/0, directly connected
L   2001:DB8:0:1::1/128 [0/0] via Fa0/0, receive
S   2001:DB8:0:2::/64 [1/0] via 2001:DB8:0:1::2    ! Static
S   2001:DB8:0:3::/64 [1/0] via 2001:DB8:0:1::2    ! Static
L   FF00::/8          [0/0] via Null0, receive
```

**IPv6 Route Codes:**

| Code | Meaning |
| --- | --- |
| `C` | Connected |
| `L` | Local (/128 host route for router's own address) |
| `S` | Static |
| `D` | EIGRP |
| `O` | OSPF |
| `FF00::/8` | Multicast range — always present |

### Final Verification

```
PC1#ping 2001:DB8:0:3:201:C7FF:FE50:8E8A    ! PC2's EUI-64 address
!!!!!
Success rate is 100 percent (5/5)
```

---

## 📋 Key Commands Summary

| Command | Purpose |
| --- | --- |
| `ipv6 address <addr>/<prefix>` | Assign global unicast IPv6 address |
| `ipv6 address <prefix>/<len> eui-64` | Auto-generate host ID from MAC (EUI-64) |
| `ipv6 address <addr> link-local` | Manually set link-local address |
| `ipv6 unicast-routing` | Enable IPv6 packet forwarding (router mode) |
| `ipv6 route <prefix>/<len> <next-hop>` | Add an IPv6 static route |
| `ipv6 route ::/0 <next-hop>` | IPv6 default route |
| `show ipv6 interface brief` | View all IPv6 addresses per interface |
| `show ipv6 route` | View IPv6 routing table |
| `show ipv6 protocols` | View which IPv6 routing protocols are active |
| `show ipv6 neighbors` | View NDP neighbor table (IPv6 ARP equivalent) |
| `show run \ | include ipv6 route` |
| `ping <ipv6-addr>` | Test IPv6 reachability |

---

## 🗒️ Key Takeaways

- **Global Unicast** addresses (`2001:db8::/32` is documentation range) are routable; **Link-Local** (`FE80::/10`) are link-only.
- When you configure any IPv6 address on an interface, an **EUI-64 link-local** is automatically created too.
- **EUI-64** generates a host ID from the MAC address — convenient for hosts, avoids manual address assignment.
- **`ipv6 unicast-routing`** is mandatory on all routers — without it, routers silently drop forwarded IPv6 packets even if static routes exist.
- Link-local pings require specifying the **output interface** (`ping fe80::1 output-if Fa0/0`) because link-locals are not globally unique.
- **NDP** (Neighbor Discovery Protocol) replaces ARP in IPv6 — `show ipv6 neighbors` shows the equivalent of the ARP table.
- IPv6 static route syntax: `ipv6 route <prefix>/<len> <next-hop>` — use `::/0` for the default route.
# Module 11 – Cisco Device Functions

> **Lab Type:** Guided Walkthrough | **Devices Required:** 4 Routers (R1–R4), 2 Switches (SW1, SW2) in Packet Tracer
> **Topics:** MAC Address Table, Routing Table, Switch & Router Functions

---

## Overview

This module explores two core functions of Cisco network devices:

- **Switches** → maintain a **MAC Address Table** to forward frames at Layer 2
- **Routers** → maintain a **Routing Table** to forward packets at Layer 3

---

## Lab Setup

Download `11 Cisco Device Functions.zip`, extract and open the `.pkt` file in Cisco Packet Tracer. All routers are pre-configured with IP addresses in the `10.10.10.0/24` network.

| Device | Interface | IP Address |
|--------|-----------|------------|
| R1 | GigabitEthernet0/0 | 10.10.10.1 |
| R2 | GigabitEthernet0/0 | 10.10.10.2 |
| R3 | GigabitEthernet0/1 | 10.10.10.3 |
| R4 | GigabitEthernet0/0 | 10.10.10.4 |

---

## Part 1 – Switch MAC Address Table

### What is the MAC Address Table?

A switch learns which devices are reachable via which ports by inspecting the **source MAC address** of incoming frames. This mapping is stored in the **MAC Address Table** (also called the CAM table).

- Entries are learned **dynamically** as traffic passes through — no manual config needed
- Entries have an **aging timer** — the switch flushes old entries periodically
- Entries can also be cleared manually

---

### Step 1 – Verify Interface IP Addresses

```
R1#show ip interface brief
R2#show ip interface brief
R3#show ip interface brief
R4#show ip interface brief
```

Sample output:
```
Interface          IP-Address    OK? Method Status   Protocol
GigabitEthernet0/0 10.10.10.1   YES manual up       up
GigabitEthernet0/1 unassigned   YES unset  admin down down
```

> Confirms which interface on each router is active on the `10.10.10.0/24` network.

---

### Step 2 – Note Router MAC Addresses

```
R1#show interface gig0/0
R2#show interface gig0/0
R3#show interface gig0/1
R4#show interface gig0/0
```

Sample output:
```
GigabitEthernet0/0 is up, line protocol is up (connected)
Hardware is CN Gigabit Ethernet, address is 0090.2b82.ab01
(bia 0090.2b82.ab01)
```

> **bia** = Burned-In Address — the factory-assigned permanent MAC address.

| Router | Interface | MAC Address |
|--------|-----------|-------------|
| R1 | Gig0/0 | 0090.2b82.ab01 |
| R2 | Gig0/0 | 0060.2fb3.9152 |
| R3 | Gig0/1 | 0001.9626.8970 |
| R4 | Gig0/0 | 00d0.9701.02a9 |

> ⚠️ MAC addresses in your own lab instance will differ from the above.

---

### Step 3 – Test Connectivity with Ping

```
R1#ping 10.10.10.2
R1#ping 10.10.10.3
R1#ping 10.10.10.4

R2#ping 10.10.10.3
R2#ping 10.10.10.4
```

Sample output:
```
Sending 5, 100-byte ICMP Echos to 10.10.10.2, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5)
```

> 💡 The first ping often fails (`.`) due to **ARP resolution** happening for the first time. Subsequent pings succeed (`!`). A success rate of 80% (4/5) is expected and normal.

---

### Step 4 – View Dynamic MAC Address Table on Switches

```
SW1#show mac address-table dynamic
SW2#show mac address-table dynamic
```

Sample SW1 output:

| VLAN | MAC Address | Type | Port |
|------|-------------|------|------|
| 1 | 0090.2b82.ab01 (R1) | DYNAMIC | Fa0/1 |
| 1 | 0060.2fb3.9152 (R2) | DYNAMIC | Fa0/2 |
| 1 | 0001.9626.8970 (R3) | DYNAMIC | Fa0/24 |
| 1 | 00d0.9701.02a9 (R4) | DYNAMIC | Fa0/24 |

> 💡 Multiple MAC addresses on the same port (e.g., Fa0/24) means another switch is connected on that port — traffic passes through an **uplink/trunk** port. SW2 is uplinked to SW1, so the same MACs appear on different ports on SW2.

---

### Step 5 – Clear the MAC Address Table

```
SW1#clear mac address-table dynamic     ! Wipe all dynamically learned entries
SW1#show mac address-table dynamic      ! Re-check — entries reappear quickly
```

> 💡 Entries reappear almost immediately in a real network because devices constantly send traffic. In Packet Tracer, you may see fewer entries temporarily.

---

## Part 2 – Router Routing Table

### What is the Routing Table?

A router uses its **routing table** to decide where to forward each IP packet. Routes are added:

- **Automatically** when an interface is configured with an IP address (Connected & Local routes)
- **Manually** via static routes
- **Dynamically** via routing protocols (OSPF, EIGRP, RIP, etc.)

### Routing Table Code Reference

| Code | Route Type | Description |
|------|------------|-------------|
| `C` | Connected | Directly attached network — auto-created when interface is configured |
| `L` | Local | The router's own IP address on that interface (host /32 route) |
| `S` | Static | Manually configured route |
| `R` | RIP | Learned via RIP dynamic routing protocol |
| `O` | OSPF | Learned via OSPF dynamic routing protocol |
| `D` | EIGRP | Learned via EIGRP dynamic routing protocol |
| `B` | BGP | Learned via BGP |
| `*` | Default | Candidate default route |

---

### Step 1 – View the Initial Routing Table

```
R1#show ip route
```

Initial output (before adding anything):
```
Gateway of last resort is not set

      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.10.10.0/24 is directly connected, GigabitEthernet0/0
L        10.10.10.1/32 is directly connected, GigabitEthernet0/0
```

> Both `C` and `L` routes are **automatically created** when IP address `10.10.10.1/24` was assigned to Gig0/0 — no extra configuration needed.

---

### Step 2 – Add a Second Interface IP Address

```
R1(config)#interface GigabitEthernet 0/1
R1(config-if)#ip address 10.10.20.1 255.255.255.0
R1(config-if)#no shutdown
```

Routing table now shows:
```
C     10.10.10.0/24 is directly connected, GigabitEthernet0/0
L     10.10.10.1/32 is directly connected, GigabitEthernet0/0
C     10.10.20.0/24 is directly connected, GigabitEthernet0/1
L     10.10.20.1/32 is directly connected, GigabitEthernet0/1
```

> R1 can now **route between** `10.10.10.0/24` and `10.10.20.0/24` — hosts on both networks can communicate through R1 with no additional configuration.

---

### Step 3 – Add a Static Route

```
R1(config)#ip route 10.10.30.0 255.255.255.0 10.10.10.2
```

Syntax breakdown:

| Part | Meaning |
|------|---------|
| `ip route` | Command to add a static route |
| `10.10.30.0` | Destination network |
| `255.255.255.0` | Subnet mask (/24) |
| `10.10.10.2` | Next-hop IP address (send packets to R2) |

Routing table after adding the static route:
```
C     10.10.10.0/24 is directly connected, GigabitEthernet0/0
L     10.10.10.1/32 is directly connected, GigabitEthernet0/0
C     10.10.20.0/24 is directly connected, GigabitEthernet0/1
L     10.10.20.1/32 is directly connected, GigabitEthernet0/1
S     10.10.30.0/24 [1/0] via 10.10.10.2
```

> `[1/0]` = **Administrative Distance / Metric**. Static routes have AD of **1** — very trustworthy; only Connected routes (AD 0) are more trusted.

---

## Switch vs Router – Core Difference

| Device | Operates at | Uses Table | Forwards |
|--------|-------------|------------|---------|
| Switch | Layer 2 | MAC Address Table | Ethernet Frames |
| Router | Layer 3 | Routing Table | IP Packets |

---

## Key Commands Summary

| Command | Device | Purpose |
|---------|--------|---------|
| `show ip interface brief` | Router | Show all interfaces and IP addresses |
| `show interface gig0/0` | Router/Switch | Show detailed interface info including MAC |
| `ping <ip>` | Router | Test Layer 3 connectivity |
| `show mac address-table dynamic` | Switch | View dynamically learned MAC-to-port mappings |
| `clear mac address-table dynamic` | Switch | Flush all dynamic MAC entries |
| `show ip route` | Router | View the full routing table |
| `ip address <ip> <mask>` | Router | Assign IP to an interface |
| `no shutdown` | Router | Bring a shutdown interface up |
| `ip route <net> <mask> <next-hop>` | Router | Add a static route |

---

## Key Takeaways

- Switches use the **MAC Address Table** to forward frames at Layer 2 — learned dynamically, flushed periodically.
- Routers use the **Routing Table** to forward packets at Layer 3.
- **Connected (C)** and **Local (L)** routes are auto-created when you configure an IP on an interface.
- **Static routes** must be manually added and require a destination network, subnet mask, and next-hop IP.
- The first ping in a sequence often fails due to **ARP resolution** — this is expected behavior.
- Multiple MACs on one switch port = another switch or hub is connected on that port.
- Use `show mac address-table dynamic` and `show ip route` regularly — these are essential CCNA exam commands.

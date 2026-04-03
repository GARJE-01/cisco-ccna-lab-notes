# 22-1 - VLAN and Inter-VLAN Routing Configuration

Module Number: 22-1
Status: Completed
Topic: VTP, Access/Trunk Ports, Inter-VLAN Routing (3 methods), Layer 3 Switch

> **Lab Type:** Exercise + Answer Key | **Devices:** 1 Router (R1), 3 Switches (SW1–SW3), multiple PCs in Packet Tracer

---

## 🧭 Overview

This module configures a full VLAN environment for a campus network, including:

- **VTP (VLAN Trunking Protocol)** — centralised VLAN database management
- **Access and Trunk ports** — segmenting and carrying VLAN traffic
- **Native VLAN** — security best practice
- **Inter-VLAN Routing** via 3 different methods:
  1. Separate physical router interfaces
  2. Router on a Stick (sub-interfaces)
  3. Layer 3 Switch (SVIs)

---

## 📂 Lab Setup

- Download `22-1 VLAN and Inter-VLAN Routing Configuration.zip`, extract and open in Packet Tracer.
- All devices start in factory default state.
- VLANs used: **VLAN 10 (Eng)**, **VLAN 20 (Sales)**, **VLAN 199 (Native)**

---

## 🔑 Part 1 – VTP, Access and Trunk Ports

### What is VTP?

VTP (VLAN Trunking Protocol) allows a VTP **Server** to propagate VLAN database changes to VTP **Clients** automatically over trunk links — no need to add VLANs manually on every switch.

| VTP Mode        | Behaviour                                                        |
| --------------- | ---------------------------------------------------------------- |
| **Server**      | Creates/edits/deletes VLANs; propagates to clients               |
| **Client**      | Learns VLANs from server; cannot edit VLAN database              |
| **Transparent** | Passes VTP messages but does not sync; manages own local VLAN DB |

---

### Step 1 – Configure Trunk Links Between Switches

```
SW1(config)#int g0/1
SW1(config-if)#switchport mode trunk

SW2(config)#int g0/1
SW2(config-if)#switchport trunk encapsulation dot1q    ! Required on some switches before trunk mode
SW2(config-if)#switchport mode trunk
SW2(config-if)#int g0/2
SW2(config-if)#switchport trunk encapsulation dot1q
SW2(config-if)#switchport mode trunk

SW3(config)#int g0/2
SW3(config-if)#switchport mode trunk
```

> 💡 Some switches require `switchport trunk encapsulation dot1q` before `switchport mode trunk` (older Catalyst switches).

---

### Step 2 – Configure VTP Roles

**SW1 – VTP Server (manages and propagates VLANs):**

```
SW1(config)#vtp domain Flackbox
SW1(config)#vtp mode server
```

**SW2 – VTP Transparent (does NOT sync with SW1; manages its own VLANs):**

```
SW2(config)#vtp mode transparent
```

**SW3 – VTP Client (learns VLANs from SW1; cannot edit VLANs):**

```
SW3(config)#vtp mode client
SW3(config)#vtp domain Flackbox
```

> ⚠️ SW2 in Transparent mode must have VLANs configured **locally** — it will not receive them from SW1.

---

### Step 3 – Add VLANs

VLANs must be added on **SW1** (VTP Server) and **SW2** (VTP Transparent). SW3 (VTP Client) learns them automatically from SW1.

```
SW1(config)#vlan 10
SW1(config-vlan)#name Eng
SW1(config-vlan)#vlan 20
SW1(config-vlan)#name Sales
SW1(config-vlan)#vlan 199
SW1(config-vlan)#name Native

SW2(config)#vlan 10
SW2(config-vlan)#name Eng
SW2(config-vlan)#vlan 20
SW2(config-vlan)#name Sales
SW2(config-vlan)#vlan 199
SW2(config-vlan)#name Native
```

---

### Step 4 – Set Native VLAN on All Trunk Links

The native VLAN carries untagged traffic on a trunk. Changing it from the default (VLAN 1) improves security.

```
SW1(config)#interface gig0/1
SW1(config-if)#switchport trunk native vlan 199

SW2(config)#int gig0/1
SW2(config-if)#switchport trunk native vlan 199
SW2(config-if)#int gig0/2
SW2(config-if)#switchport trunk native vlan 199

SW3(config)#int gig0/2
SW3(config-if)#switchport trunk native vlan 199
```

> ⚠️ Native VLAN **must match on both ends** of a trunk — mismatch causes a CDP warning and can cause security issues.

---

### Step 5 – Configure Access Ports for PCs

```
SW1(config)#int range f0/1 - 2
SW1(config-if-range)#switchport mode access
SW1(config-if-range)#switchport access vlan 10    ! Eng PCs

SW1(config)#int f0/3
SW1(config-if)#switchport mode access
SW1(config-if)#switchport access vlan 20           ! Sales PC

SW3(config)#int range f0/1 - 2
SW3(config-if-range)#switchport mode access
SW3(config-if-range)#switchport access vlan 20    ! Sales PCs

SW3(config)#int f0/3
SW3(config-if)#switchport mode access
SW3(config-if)#switchport access vlan 10           ! Eng PC
```

**Verify intra-VLAN connectivity:**

- Eng1 ↔ Eng3 ✅ (both VLAN 10, connected via trunk)
- Sales1 ↔ Sales3 ✅ (both VLAN 20, connected via trunk)
- Eng1 ↔ Sales1 ❌ (different VLANs — need inter-VLAN routing)

---

## 🔑 Part 2 – Inter-VLAN Routing: Option 1 – Separate Router Interfaces

### Concept

Connect a separate physical router interface to each VLAN. Simple but **wastes physical ports** and doesn't scale.

```
R1(config)#interface FastEthernet 0/0
R1(config-if)#ip address 10.10.10.1 255.255.255.0    ! Gateway for VLAN 10 (Eng)
R1(config-if)#no shutdown

R1(config)#interface FastEthernet 0/1
R1(config-if)#ip address 10.10.20.1 255.255.255.0    ! Gateway for VLAN 20 (Sales)
R1(config-if)#no shutdown
```

**Configure SW2 access ports toward R1:**

```
SW2(config)#interface FastEthernet 0/1
SW2(config-if)#switchport mode access
SW2(config-if)#switchport access vlan 10

SW2(config)#interface FastEthernet 0/2
SW2(config-if)#switchport mode access
SW2(config-if)#switchport access vlan 20
```

**Cleanup:**

```
R1(config)#int f0/1
R1(config-if)#shutdown
```

---

## 🔑 Part 3 – Inter-VLAN Routing: Option 2 – Router on a Stick

### Concept

Use **one physical interface** with multiple **sub-interfaces** — each tagged with a VLAN using 802.1Q encapsulation. One cable to the switch carries all VLANs. Scalable, cost-effective.

```
R1(config)#interface FastEthernet 0/0
R1(config-if)#no ip address
R1(config-if)#no shutdown

R1(config-if)#interface FastEthernet 0/0.10
R1(config-subif)#encapsulation dot1q 10              ! Tag sub-interface with VLAN 10
R1(config-subif)#ip address 10.10.10.1 255.255.255.0

R1(config-subif)#interface FastEthernet 0/0.20
R1(config-subif)#encapsulation dot1q 20              ! Tag sub-interface with VLAN 20
R1(config-subif)#ip address 10.10.20.1 255.255.255.0
```

**Configure SW2 port toward R1 as a trunk:**

```
SW2(config)#interface FastEthernet 0/1
SW2(config-if)#switchport trunk encapsulation dot1q
SW2(config-if)#switchport mode trunk
```

> 💡 The physical interface Fa0/0 has **no IP address** — only the sub-interfaces have IPs. The physical interface just needs to be `no shutdown`.

**Cleanup:**

```
R1(config)#int f0/0
R1(config-if)#shutdown
```

---

## 🔑 Part 4 – Inter-VLAN Routing: Option 3 – Layer 3 Switch

### Concept

A **Layer 3 (multilayer) switch** can route between VLANs internally using **SVIs (Switched Virtual Interfaces)** — no external router needed. Best performance for campus networks.

**Step 1 – Enable IP routing on SW2:**

```
SW2(config)#ip routing
```

**Step 2 – Configure SVIs for each VLAN:**

```
SW2(config)#interface vlan 10
SW2(config-if)#ip address 10.10.10.1 255.255.255.0    ! Gateway for Eng VLAN

SW2(config)#interface vlan 20
SW2(config-if)#ip address 10.10.20.1 255.255.255.0    ! Gateway for Sales VLAN
```

> 💡 No external router or trunk to router needed — routing happens inside the switch at wire speed.

---

## 📊 Inter-VLAN Routing Methods Comparison

| Method                     | Interfaces Used      | Scalability | Cost   | Notes                                    |
| -------------------------- | -------------------- | ----------- | ------ | ---------------------------------------- |
| Separate Router Interfaces | 1 per VLAN           | Poor        | High   | Wastes ports, simple config              |
| Router on a Stick          | 1 physical + sub-ifs | Medium      | Low    | Single uplink, popular in small networks |
| Layer 3 Switch (SVI)       | Internal SVIs        | Excellent   | Medium | Best for campus LANs, wire-speed routing |

---

## 📋 Key Commands Summary

| Command                                | Purpose                                                |
| -------------------------------------- | ------------------------------------------------------ |
| `switchport mode trunk`                | Set port as 802.1Q trunk                               |
| `switchport trunk encapsulation dot1q` | Set trunk encapsulation (needed on some switches)      |
| `switchport trunk native vlan <id>`    | Change native VLAN on trunk                            |
| `switchport mode access`               | Set port as access port                                |
| `switchport access vlan <id>`          | Assign access port to VLAN                             |
| `vtp domain <name>`                    | Set VTP domain name                                    |
| `vtp mode server/client/transparent`   | Set VTP operating mode                                 |
| `vlan <id>` / `name <name>`            | Create and name a VLAN                                 |
| `show vlan brief`                      | Verify VLAN database and port assignments              |
| `show interfaces trunk`                | Verify trunk links and allowed VLANs                   |
| `show vtp status`                      | Verify VTP mode and domain                             |
| `encapsulation dot1q <vlan>`           | Tag router sub-interface with VLAN (Router on a Stick) |
| `ip routing`                           | Enable Layer 3 routing on a multilayer switch          |
| `interface vlan <id>`                  | Create SVI for inter-VLAN routing on L3 switch         |

---

## 🗒️ Key Takeaways

- VLANs segment Layer 2 broadcast domains — devices in different VLANs cannot communicate without a router or Layer 3 switch.
- **VTP Server** propagates VLANs to **VTP Clients** automatically. **VTP Transparent** passes VTP messages but manages its own local VLAN DB.
- **Native VLAN** should be changed from the default VLAN 1 to an unused VLAN (e.g., 199) for security — must match on both ends of a trunk.
- **Router on a Stick** uses `encapsulation dot1q <vlan>` on sub-interfaces and requires the uplink SW port to be a trunk.
- **Layer 3 Switch SVIs** are the most efficient inter-VLAN routing method — enable with `ip routing` then create `interface vlan` SVIs.
- Intra-VLAN traffic is switched at Layer 2; inter-VLAN traffic must be routed at Layer 3.

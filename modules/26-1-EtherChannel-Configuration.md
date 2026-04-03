# 26-1 - EtherChannel Configuration

Module Number: 26-1
Status: Completed
Topic: LACP, PAgP, Static EtherChannel, Layer 3 EtherChannel, OSPF over Port-Channel

> **Lab Type:** Exercise + Answer Key | **Devices:** Acc3, Acc4, CD1, CD2 (L2 EtherChannel) + Switch1, Switch2, Switch3 (L3 EtherChannel) in Packet Tracer
> 

---

## 🧭 Overview

EtherChannel bundles multiple physical links into a single **logical port-channel interface**, providing:

- **Increased bandwidth** — aggregate the bandwidth of all member links
- **Redundancy** — if one physical link fails, traffic continues on remaining links
- **No STP blocking** — STP sees the bundle as a single link, not redundant links

This module covers all three EtherChannel negotiation protocols and a Layer 3 EtherChannel with OSPF.

---

## 🧠 EtherChannel Protocols

| Protocol | Type | Mode Keywords | Notes |
| --- | --- | --- | --- |
| **LACP** | IEEE 802.3ad (open standard) | `active` / `passive` | Preferred — works between different vendors |
| **PAgP** | Cisco proprietary | `desirable` / `auto` | Cisco-only |
| **Static** | No negotiation | `on` | Both sides must be `on`; no dynamic negotiation |

### Mode Compatibility Table

| Side A | Side B | Result |
| --- | --- | --- |
| `active` | `active` | ✅ LACP forms |
| `active` | `passive` | ✅ LACP forms |
| `passive` | `passive` | ❌ Neither initiates |
| `desirable` | `desirable` | ✅ PAgP forms |
| `desirable` | `auto` | ✅ PAgP forms |
| `auto` | `auto` | ❌ Neither initiates |
| `on` | `on` | ✅ Static forms |
| `on` | `active/passive` | ❌ Mismatch — will not form |

> 💡 **Best practice:** Always use LACP (`active/active`) in production environments — it's an open standard and compatible with non-Cisco gear.
> 

---

## 📂 Lab Setup

- Download `26-1 EtherChannel Configuration.zip`, extract and open in Packet Tracer.
- **Part 1 topology:** Acc3, Acc4 (access), CD1, CD2 (core/distribution) — Layer 2 switches with trunk uplinks.
- **Part 2 topology:** Switch1, Switch2, Switch3 — separate Layer 3 switches.
- Before EtherChannel: STP blocks all but one uplink per switch → only **100 Mbps** available between Acc3 and Acc4 PCs.

---

## 🔑 Part 1 – Layer 2 EtherChannel (LACP)

### Configure LACP EtherChannel — Acc3 to CD1 (Port-Channel 1)

**On Acc3:**

```
Acc3(config)#interface range f0/23 - 24
Acc3(config-if-range)#channel-group 1 mode active    ! LACP active
Acc3(config-if-range)#exit
Acc3(config)#interface port-channel 1
Acc3(config-if)#description Link to CD1
Acc3(config-if)#switchport mode trunk
Acc3(config-if)#switchport trunk native vlan 199
```

**On CD1 (matching config):**

```
CD1(config)#interface range f0/23 - 24
CD1(config-if-range)#channel-group 1 mode active
CD1(config-if-range)#exit
CD1(config)#interface port-channel 1
CD1(config-if)#description Link to Acc3
CD1(config-if)#switchport mode trunk
CD1(config-if)#switchport trunk native vlan 199
```

### Configure LACP EtherChannel — Acc3 to CD2 (Port-Channel 2)

**On Acc3:**

```
Acc3(config)#interface range f0/21 - 22
Acc3(config-if-range)#channel-group 2 mode active
Acc3(config-if-range)#exit
Acc3(config)#interface port-channel 2
Acc3(config-if)#description Link to CD2
Acc3(config-if)#switchport mode trunk
Acc3(config-if)#switchport trunk native vlan 199
```

**On CD2:**

```
CD2(config)#interface range f0/21 - 22
CD2(config-if-range)#channel-group 2 mode active
CD2(config-if-range)#exit
CD2(config)#interface port-channel 2
CD2(config-if)#description Link to Acc3
CD2(config-if)#switchport mode trunk
CD2(config-if)#switchport trunk native vlan 199
```

### Verify LACP EtherChannel

```
Acc3#show etherchannel summary
Flags: D - down, P - in port-channel, I - stand-alone
       U - in use, S - Layer 2, R - Layer 3
       f - failed, M - not in use (min-links not met)

Group  Port-channel  Protocol    Ports
1      Po1(SU)       LACP        Fa0/23(P)  Fa0/24(P)    ! SU = L2, in use
2      Po2(SU)       LACP        Fa0/21(P)  Fa0/22(P)
```

> ✅ `(SU)` = Layer **2** (**S**) and in **U**se. Member ports show `(P)` = in **P**ort-channel.
> 

---

## 🔑 Part 2 – Layer 2 EtherChannel (PAgP)

### Configure PAgP EtherChannel — Acc4 to CD2 (Port-Channel 1)

**On Acc4:**

```
Acc4(config)#interface range f0/23 - 24
Acc4(config-if-range)#channel-group 1 mode desirable    ! PAgP desirable
Acc4(config-if-range)#exit
Acc4(config)#interface port-channel 1
Acc4(config-if)#description Link to CD2
Acc4(config-if)#switchport mode trunk
Acc4(config-if)#switchport trunk native vlan 199
```

**On CD2:**

```
CD2(config)#interface range f0/23 - 24
CD2(config-if-range)#channel-group 1 mode desirable
CD2(config-if-range)#exit
CD2(config)#interface port-channel 1
CD2(config-if)#description Link to Acc4
CD2(config-if)#switchport mode trunk
CD2(config-if)#switchport trunk native vlan 199
```

### Configure PAgP EtherChannel — Acc4 to CD1 (Port-Channel 2)

```
Acc4(config)#interface range f0/21 - 22
Acc4(config-if-range)#channel-group 2 mode desirable
Acc4(config-if-range)#exit
Acc4(config)#interface port-channel 2
Acc4(config-if)#description Link to CD1
Acc4(config-if)#switchport mode trunk
Acc4(config-if)#switchport trunk native vlan 199

CD1(config)#interface range f0/21 - 22
CD1(config-if-range)#channel-group 2 mode desirable
CD1(config-if-range)#exit
CD1(config)#interface port-channel 2
CD1(config-if)#description Link to Acc4
CD1(config-if)#switchport mode trunk
CD1(config-if)#switchport trunk native vlan 199
```

---

## 🔑 Part 3 – Static EtherChannel

### Configure Static EtherChannel — CD1 to CD2 (Port-Channel 3)

```
CD1(config)#interface range g0/1 - 2
CD1(config-if-range)#channel-group 3 mode on    ! Static — no negotiation
CD1(config-if-range)#exit
CD1(config)#interface port-channel 3
CD1(config-if)#description Link to CD2
CD1(config-if)#switchport mode trunk
CD1(config-if)#switchport trunk native vlan 199

CD2(config)#interface range g0/1 - 2
CD2(config-if-range)#channel-group 3 mode on
CD2(config-if-range)#exit
CD2(config)#interface port-channel 3
CD2(config-if)#description Link to CD1
CD2(config-if)#switchport mode trunk
CD2(config-if)#switchport trunk native vlan 199
```

> ⚠️ Both sides MUST use `mode on` for static EtherChannel. Never mix `on` with `active`, `passive`, `desirable`, or `auto`.
> 

### Bandwidth After EtherChannel

- Before EtherChannel: STP blocked all but 1 link → **100 Mbps**
- After EtherChannel: Port-channels to CD1 use 2 x FastEthernet → **200 Mbps** total between Acc3 and Acc4 PCs
- STP still blocks port-channels toward CD2 (redundant path) — but now each blocked port-channel is 200 Mbps ready for failover

---

## 🔑 Part 4 – Layer 3 EtherChannel

### Concept

A **Layer 3 EtherChannel** converts the port-channel interface into a routed interface (like a router port) by using `no switchport`. IP addresses are assigned directly to the port-channel interface. Physical ports must also have `no switchport` applied.

---

### Configure L3 EtherChannel — Switch1 to Switch2 (Port-Channel 1)

```
Switch1(config)#interface range GigabitEthernet 1/0/1 - 2
Switch1(config-if-range)#no switchport              ! Convert to L3 routed port
Switch1(config-if-range)#channel-group 1 mode active
Switch1(config-if-range)#exit
Switch1(config)#interface port-channel 1
Switch1(config-if)#ip address 192.168.0.1 255.255.255.252
Switch1(config-if)#no shutdown

Switch2(config)#interface range GigabitEthernet 1/0/1 - 2
Switch2(config-if-range)#no switchport
Switch2(config-if-range)#channel-group 1 mode active
Switch2(config-if-range)#exit
Switch2(config)#interface port-channel 1
Switch2(config-if)#ip address 192.168.0.2 255.255.255.252
Switch2(config-if)#no shutdown
```

### Configure L3 EtherChannel — Switch1 to Switch3 (Port-Channel 2)

```
Switch1(config)#interface range GigabitEthernet 1/0/3 - 4
Switch1(config-if-range)#no switchport
Switch1(config-if-range)#channel-group 2 mode active
Switch1(config-if-range)#exit
Switch1(config)#interface port-channel 2
Switch1(config-if)#ip address 192.168.0.5 255.255.255.252
Switch1(config-if)#no shutdown

Switch3(config)#interface range GigabitEthernet 1/0/3 - 4
Switch3(config-if-range)#no switchport
Switch3(config-if-range)#channel-group 2 mode active
Switch3(config-if-range)#exit
Switch3(config)#interface port-channel 2
Switch3(config-if)#ip address 192.168.0.6 255.255.255.252
Switch3(config-if)#no shutdown
```

### Configure L3 EtherChannel — Switch2 to Switch3 (Port-Channel 3)

```
Switch2(config)#interface range GigabitEthernet 1/0/5 - 6
Switch2(config-if-range)#no switchport
Switch2(config-if-range)#channel-group 3 mode active
Switch2(config-if-range)#exit
Switch2(config)#interface port-channel 3
Switch2(config-if)#ip address 192.168.0.9 255.255.255.252
Switch2(config-if)#no shutdown

Switch3(config)#interface range GigabitEthernet 1/0/5 - 6
Switch3(config-if-range)#no switchport
Switch3(config-if-range)#channel-group 3 mode active
Switch3(config-if-range)#exit
Switch3(config)#interface port-channel 3
Switch3(config-if)#ip address 192.168.0.10 255.255.255.252
Switch3(config-if)#no shutdown
```

### Configure OSPF on Layer 3 Switches

```
Switch1(config)#ip routing
Switch1(config)#router ospf 1
Switch1(config-router)#network 192.168.0.0 0.0.0.255 area 0
! Repeat on Switch2 and Switch3
```

### STP on Layer 3 EtherChannel?

> 💡 **STP does NOT run on Layer 3 interfaces.** Since all ports use `no switchport`, they are routed ports — STP only applies to Layer 2 switchports. Path selection, redundancy and load balancing are handled by the **routing table** (OSPF in this case), not STP.
> 

---

## 📋 Key Commands Summary

| Command | Purpose |
| --- | --- |
| `interface range <ports>` | Select multiple interfaces simultaneously |
| `channel-group <n> mode active` | Add ports to EtherChannel with LACP active |
| `channel-group <n> mode passive` | Add ports to EtherChannel with LACP passive |
| `channel-group <n> mode desirable` | Add ports to EtherChannel with PAgP desirable |
| `channel-group <n> mode auto` | Add ports to EtherChannel with PAgP auto |
| `channel-group <n> mode on` | Add ports to static EtherChannel (no negotiation) |
| `interface port-channel <n>` | Enter/configure the logical port-channel interface |
| `no switchport` | Convert port to Layer 3 routed port (L3 EtherChannel) |
| `show etherchannel summary` | View all EtherChannel groups and port member status |
| `show etherchannel <n> detail` | Detailed view of a specific EtherChannel group |
| `show interfaces port-channel <n>` | View port-channel interface status and stats |

---

## 🗒️ Key Takeaways

- EtherChannel bundles physical links into one logical interface — STP sees it as a single link and won't block individual member ports.
- **LACP** (`active/active` or `active/passive`) is the industry standard — use it whenever possible.
- **PAgP** (`desirable/desirable` or `desirable/auto`) is Cisco-only — avoid in mixed-vendor environments.
- **Static** (`on/on`) requires both ends configured identically — no negotiation, so misconfigurations are harder to detect.
- Physical port settings (speed, duplex, VLAN) must be **identical** on all member ports in a channel group.
- Configure trunk settings (mode, native VLAN) on the **port-channel interface**, not on individual member ports.
- Layer 3 EtherChannel uses `no switchport` on physical ports and assigns the IP directly to the port-channel interface — STP does not apply.
- After EtherChannel, bandwidth between access switches doubled from 100 Mbps to **200 Mbps** per bundle.
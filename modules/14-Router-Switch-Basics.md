# 14 - Cisco Router and Switch Basics

Module Number: 14
Status: Completed
Topic: Initial Config, CDP, Speed & Duplex, Interface Troubleshooting

> **Lab Type:** Exercise + Answer Key | **Devices:** 2 Routers (R1, R2), 1 Switch (SW1) in Packet Tracer
> 

---

## 🧭 Overview

This module covers the foundational configuration tasks you will perform on nearly every Cisco router and switch:

- Basic initial configuration (hostname, IPs, descriptions)
- Switch management IP and default gateway
- Speed and duplex settings
- Cisco Discovery Protocol (CDP)
- Interface troubleshooting

---

## 📂 Lab Setup

- Download `14 Cisco Router and Switch Basics.zip`, extract and open in Packet Tracer.
- Topology: R1 and R2 connected to SW1.
- R1 and R2 are on the `10.10.10.0/24` network.

---

## 🔑 Part 1 – Initial Device Configuration

### Hostnames

```
Router(config)#hostname R1
Router(config)#hostname R2
Switch(config)#hostname SW1
```

---

### Router Interface IP Addresses

```
R1(config)#interface FastEthernet0/0
R1(config-if)#ip address 10.10.10.1 255.255.255.0
R1(config-if)#no shutdown

R2(config)#interface FastEthernet0/0
R2(config-if)#ip address 10.10.10.2 255.255.255.0
R2(config-if)#no shutdown
```

---

### Switch Management IP Address

Switches are Layer 2 devices and do not have routed interfaces. To give a switch an IP address for management (SSH, Telnet, ping), you configure it on the **SVI (Switched Virtual Interface)** — `Vlan1` by default.

```
SW1(config)#interface vlan1
SW1(config-if)#ip address 10.10.10.10 255.255.255.0
SW1(config-if)#no shutdown
```

---

### Switch Default Gateway

A switch needs a default gateway to communicate with devices on **other subnets** (e.g., to reach a management PC on a different network).

```
SW1(config)#ip default-gateway 10.10.10.2
```

> 💡 This is different from a router. A router uses routing tables; a switch uses `ip default-gateway` for management traffic only.
> 

**Verify:**

```
SW1#ping 10.10.10.2
!!!!!
Success rate is 100 percent (5/5)
```

---

### Interface Descriptions

Descriptions help document what each interface is connected to — critical in production environments.

```
R1(config)#interface FastEthernet 0/0
R1(config-if)#description Link to SW1

R2(config)#interface FastEthernet 0/0
R2(config-if)#description Link to SW1

SW1(config)#interface FastEthernet 0/1
SW1(config-if)#description Link to R1

SW1(config)#interface FastEthernet 0/2
SW1(config-if)#description Link to R2
```

---

### Check IOS Version

```
SW1#show version
Cisco IOS Software, C2960 Software (C2960-LANBASE-M), Version 12.2(25)FX
```

> `show version` displays IOS version, hardware model, uptime, and memory.
> 

---

## 🔑 Part 2 – Speed and Duplex Configuration

### Concepts

| Term | Meaning |
| --- | --- |
| **Auto-negotiation** | Devices negotiate best speed/duplex automatically |
| **Full duplex** | Send and receive simultaneously — used for switched links |
| **Half duplex** | Only send OR receive at a time — legacy (hubs) |
| **Speed mismatch** | Interface goes down if speeds don't match |
| **Duplex mismatch** | Interface stays up but performance degrades with errors |

---

### Verify Auto-Negotiated Settings

```
SW1#show interface f0/1
Full-duplex, 100Mb/s     ! Auto-negotiated to 100 Mbps full duplex
```

---

### Manually Configure Speed and Duplex

> ⚠️ **Always configure matching settings on BOTH ends** of a link.
> 

```
SW1(config)#interface FastEthernet 0/2
SW1(config-if)#speed 100
SW1(config-if)#duplex full

R2(config)#interface FastEthernet 0/0
R2(config-if)#speed 100
R2(config-if)#duplex full
```

---

### Effect of Duplex Mismatch

```
SW1(config-if)#duplex half     ! Set SW1 to half — R2 stays at full
```

Result: Interface goes **down/down** — duplex mismatch causes the link to fail.

```
SW1#show ip interface brief
FastEthernet0/2    unassigned    YES manual down    down
```

**Fix:** Set both sides back to matching duplex:

```
SW1(config-if)#duplex full
```

---

### Effect of Speed Mismatch

```
SW1(config-if)#speed 10       ! SW1 set to 10 Mbps — R2 stays at 100 Mbps
```

Result on SW1:

```
FastEthernet0/2    unassigned    YES manual down    down
```

Result on R2:

```
FastEthernet0/0    10.10.10.2    YES manual up    down
```

> 💡 **Key difference:** Speed mismatch = SW1 side goes `down/down`, but R2 side shows `up/down` (physical layer detects signal, but data link layer fails). This is a useful troubleshooting clue!
> 

---

## 🔑 Part 3 – Cisco Discovery Protocol (CDP)

### What is CDP?

CDP (Cisco Discovery Protocol) is a **Cisco-proprietary Layer 2 protocol** that allows Cisco devices to discover directly connected Cisco neighbors. It is enabled by default on all Cisco interfaces.

CDP advertises:

- Device ID (hostname)
- Local and remote interface
- Platform (device model)
- Capabilities (Router, Switch, etc.)
- IOS version
- IP address

> ⚠️ CDP can be a **security risk** — it leaks device info to anyone connected. Disable it on external/untrusted interfaces.
> 

---

### View CDP Neighbors

```
SW1#show cdp neighbors
Device ID    Local Intrfce    Holdtme    Capability    Platform    Port ID
R1           Fas 0/1          170        R             C2800       Fas 0/0
R2           Fas 0/2          134        R             C2800       Fas 0/0
```

**Capability codes:**

| Code | Meaning |
| --- | --- |
| R | Router |
| S | Switch |
| T | Trans Bridge |
| H | Host |
| I | IGMP |

---

### Disable CDP on a Specific Interface

To stop R1 from seeing SW1 via CDP, disable CDP on the SW1 interface facing R1:

```
SW1(config)#interface FastEthernet 0/1
SW1(config-if)#no cdp enable          ! Disable CDP on this interface only
```

---

### Flush CDP Cache and Verify

```
R1(config)#no cdp run                  ! Disable CDP globally (clears cache)
R1(config)#cdp run                     ! Re-enable CDP

R1#show cdp neighbors                  ! SW1 no longer appears
Device ID    Local Intrfce    Holdtme    Capability    Platform    Port ID
R1#
```

> CDP entries persist in the cache until the holdtime expires (~180 sec by default). Toggling `cdp run` flushes it immediately.
> 

---

## 📋 Key Commands Summary

| Command | Purpose |
| --- | --- |
| `hostname <name>` | Set device hostname |
| `interface vlan1` | Enter SVI for switch management IP |
| `ip default-gateway <ip>` | Set default gateway on a switch |
| `description <text>` | Add description to an interface |
| `show version` | Display IOS version and hardware info |
| `show interface <id>` | Detailed interface stats including speed/duplex |
| `show ip interface brief` | Quick status of all interfaces |
| `speed <10/100/1000/auto>` | Set interface speed |
| `duplex <full/half/auto>` | Set interface duplex |
| `show cdp neighbors` | List directly connected Cisco devices |
| `show cdp neighbors detail` | Full info including IOS version and IP |
| `no cdp enable` | Disable CDP on a specific interface |
| `no cdp run` | Disable CDP globally on device |

---

## 🗒️ Key Takeaways

- Switch management IPs are configured on **Vlan1 SVI**, not physical interfaces.
- Switches need `ip default-gateway` to communicate outside their subnet — routers do not.
- Always configure **matching speed and duplex on both ends** of a link.
- **Duplex mismatch** → link goes down. **Speed mismatch** → one side shows `up/down`, other shows `down/down`.
- CDP is useful for network discovery but should be **disabled on untrusted/external interfaces** for security.
- `no cdp enable` disables CDP per-interface; `no cdp run` disables it globally.
# 25-2 - Spanning Tree Configuration

Module Number: 25-2
Status: Completed
Topic: Root Primary/Secondary, PortFast, BPDU Guard, STP Alignment with HSRP

> **Lab Type:** Exercise + Answer Key | **Devices:** 4 Switches (Acc3, Acc4, CD1, CD2), 2 Routers (R1, R2 via HSRP), PCs in Packet Tracer
> 

---

## 🧭 Overview

Building directly on the 25-1 troubleshooting lab, this module **fixes** the broken STP topology. The goal is to align the Spanning Tree Root Bridge with the HSRP active router so that Layer 2 and Layer 3 traffic both follow the same optimal path.

Two key tasks:

1. **Root Bridge Configuration** — force CD1 as primary and CD2 as secondary Root Bridge
2. **PortFast + BPDU Guard** — speed up host port convergence and protect against accidental loops

---

## 📂 Lab Setup

- Download `25-2 EtherChannel Configuration.zip` (note: the file name is shared with the next lab), extract and open in Packet Tracer.
- Same topology as 25-1: Acc3, Acc4, CD1, CD2 switches; R1 (HSRP active), R2 (HSRP standby); PCs.
- **Do NOT change any Layer 3 or HSRP configuration.**

---

## 🔑 Part 1 – Root Bridge Configuration

### The Problem (from 25-1)

- Acc3 is currently the Root Bridge (lowest MAC address with default priority 32768)
- This forces PC2 traffic via the suboptimal path: PC2 → Acc4 → CD2 → Acc3 → CD1 → R1
- The correct Root Bridge should be **CD1** — it connects directly to R1 (HSRP active)

### STP Must Align With HSRP

> 🔑 A golden rule: **STP Root Bridge should be on the same switch as (or closest to) the HSRP Active router.** This ensures Layer 2 and Layer 3 both use the same optimal upstream path.
> 

---

### Step 1 – Set CD1 as Primary Root Bridge

```
CD1(config)#spanning-tree vlan 10 root primary
```

> `root primary` is a macro that sets the Bridge Priority to **24576** (or lower if needed to beat the current root), guaranteeing CD1 wins the election.
> 

**What happens behind the scenes:**

```
CD1#show spanning-tree vlan 10
  Root ID   Priority  24576
            Address   0090.0CA0.3902
            This bridge is the root
```

---

### Step 2 – Set CD2 as Secondary Root Bridge (Failover)

If CD1 fails, STP should elect CD2 as the new root — NOT Acc3 or Acc4.

```
CD2(config)#spanning-tree vlan 10 root secondary
```

> `root secondary` sets Bridge Priority to **28672** — lower than the default 32768 (access switches) but higher than CD1's 24576. This guarantees CD2 wins if CD1 fails.
> 

---

### Bridge Priority Values Summary

| Command | Priority Set | Used For |
| --- | --- | --- |
| `root primary` | **24576** (or lower) | Force this switch as Root Bridge |
| `root secondary` | **28672** | Make this the failover Root Bridge |
| Default | **32768** | All switches by default |
| Manual | `spanning-tree vlan <id> priority <value>` | Any custom value (must be multiple of 4096) |

> ⚠️ Priority values must be **multiples of 4096**: 0, 4096, 8192, 12288, 16384, 20480, 24576, 28672, 32768, 36864...
> 

---

### Step 3 – Verify the New STP Topology

**On CD1:**

```
CD1#show spanning-tree vlan 10
  Root ID   Priority  24576
            This bridge is the root    ✅ CD1 is now Root Bridge
  Bridge ID Priority  24576
```

**On CD2:**

```
CD2#show spanning-tree vlan 10
  Root ID   Priority  24576
            Address   0090.0CA0.3902  (CD1)
  Bridge ID Priority  28672           ! CD2 has next-best priority
```

**On Acc3:**

```
Acc3#show spanning-tree vlan 10
  Root ID   Priority  24576
            Address   0090.0CA0.3902  (CD1 - no longer Acc3!)
  Bridge ID Priority  32768
```

---

### Resulting Optimal Traffic Paths

With CD1 as Root Bridge:

**PC1 path:** PC1 → Acc3 → CD1 → R1 → Internet ✅

**PC2 path:** PC2 → Acc4 → CD1 → R1 → Internet ✅ (direct link to CD1 now forwarding!)

> Both PCs now use the most direct path. The suboptimal CD2 transit path is eliminated.
> 

---

## 🔑 Part 2 – PortFast and BPDU Guard

### What is PortFast?

Normally, when a switch port comes up it goes through STP states before forwarding:

```
Blocking (20s) → Listening (15s) → Learning (15s) → Forwarding
```

Total delay = **~50 seconds** before a host can communicate.

**PortFast** bypasses these states on **access ports connected to end hosts**, transitioning immediately to Forwarding.

> ⚠️ PortFast should **only** be configured on ports connected to **single end hosts** (PCs, printers, servers) — NEVER on ports connecting to other switches, as this would bypass STP loop protection.
> 

---

### What is BPDU Guard?

BPDU (Bridge Protocol Data Unit) messages are sent between switches as part of STP. End hosts never send BPDUs.

**BPDU Guard** automatically **shuts down (err-disables)** a PortFast-enabled port if it receives a BPDU — meaning someone accidentally connected a switch to that port, which could create a loop.

> 💡 PortFast + BPDU Guard is the **industry best practice** for all host-facing access ports.
> 

---

### Step 4 – Configure PortFast and BPDU Guard on Host Ports

**On Acc3 (port facing PC1):**

```
Acc3(config)#int f0/1
Acc3(config-if)#spanning-tree portfast
Acc3(config-if)#spanning-tree bpduguard enable
```

**On Acc4 (port facing PC2):**

```
Acc4(config)#int f0/1
Acc4(config-if)#spanning-tree portfast
Acc4(config-if)#spanning-tree bpduguard enable
```

**On CD1 (port facing R1):**

```
CD1(config)#int g0/1
CD1(config-if)#spanning-tree portfast
CD1(config-if)#spanning-tree bpduguard enable
```

**On CD2 (port facing R2):**

```
CD2(config)#int g0/1
CD2(config-if)#spanning-tree portfast
CD2(config-if)#spanning-tree bpduguard enable
```

---

### Alternative – Global PortFast + BPDU Guard

You can also enable PortFast and BPDU Guard globally for all access ports:

```
SW1(config)#spanning-tree portfast default          ! PortFast on all access ports
SW1(config)#spanning-tree portfast bpduguard default ! BPDU Guard on all PortFast ports
```

> Using the global command is more scalable — any port set to `switchport mode access` automatically gets PortFast and BPDU Guard.
> 

---

### What Happens if BPDU Guard Triggers?

```
%SPANTREE-2-BLOCK_BPDUGUARD: Received BPDU on port Fa0/1 with BPDU Guard enabled.
%PM-4-ERR_DISABLE: bpduguard error detected on Fa0/1, putting Fa0/1 in err-disable state.
```

Port goes into **err-disabled** state — must be manually recovered:

```
SW1(config)#int f0/1
SW1(config-if)#shutdown
SW1(config-if)#no shutdown
```

Or enable automatic recovery:

```
SW1(config)#errdisable recovery cause bpduguard
SW1(config)#errdisable recovery interval 300    ! Recover after 300 seconds
```

---

## 📋 Key Commands Summary

| Command | Purpose |
| --- | --- |
| `spanning-tree vlan <id> root primary` | Set switch as primary Root Bridge (priority 24576) |
| `spanning-tree vlan <id> root secondary` | Set switch as secondary Root Bridge (priority 28672) |
| `spanning-tree vlan <id> priority <value>` | Manually set Bridge Priority (multiples of 4096) |
| `show spanning-tree vlan <id>` | View full STP topology, root info, port roles |
| `show spanning-tree summary` | Overview of all VLANs and port counts |
| `spanning-tree portfast` | Enable PortFast on a single port |
| `spanning-tree bpduguard enable` | Enable BPDU Guard on a single port |
| `spanning-tree portfast default` | Enable PortFast globally on all access ports |
| `spanning-tree portfast bpduguard default` | Enable BPDU Guard globally on all PortFast ports |
| `show spanning-tree interface <int> detail` | Check PortFast/BPDU Guard status on a port |

---

## 🗒️ Key Takeaways

- Always align the STP **Root Bridge** with the **HSRP Active router** — they should be on the same or directly connected switch to ensure Layer 2 and Layer 3 paths match.
- `root primary` sets priority to **24576**; `root secondary` sets it to **28672** — both beat the default 32768 on access switches.
- Bridge Priority must be a **multiple of 4096** when set manually.
- **PortFast** eliminates the ~50 second STP convergence delay on host-facing ports — transition straight to Forwarding.
- **BPDU Guard** protects PortFast ports by err-disabling them if a switch is accidentally connected — preventing loops.
- PortFast + BPDU Guard should be configured on **every host-facing port** as a standard baseline security practice.
- A port in **err-disabled** state must be manually recovered with `shutdown` then `no shutdown`, or with `errdisable recovery`.
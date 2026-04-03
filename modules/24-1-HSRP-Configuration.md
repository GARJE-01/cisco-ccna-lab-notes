# 24-1 - HSRP Configuration

Module Number: 24-1
Status: Completed
Topic: HSRP, Virtual IP, Priority, Pre-emption, Failover Testing

> **Lab Type:** Exercise + Answer Key | **Devices:** 2 Routers (R1, R2), Switches, PCs, SP1 in Packet Tracer
> 

---

## 🧭 Overview

HSRP (Hot Standby Router Protocol) provides **first-hop redundancy** for hosts on a LAN. Instead of configuring a single router as the default gateway (single point of failure), two routers share a **virtual IP and virtual MAC address**. If the active router fails, the standby router takes over automatically — with minimal disruption to hosts.

This lab covers:

- Basic HSRP configuration — virtual IP, active/standby election
- HSRP virtual MAC address
- Priority and pre-emption — controlling which router is active
- Live failover testing — observing the switchover during a reboot

---

## 📂 Lab Setup

- Download `24-1 HSRP Configuration.zip`, extract and open in Packet Tracer.
- Topology: R1 and R2 both connected to the same LAN (10.10.10.0/24). PCs use HSRP virtual IP as their default gateway.
- R1 physical IP: `10.10.10.2` | R2 physical IP: `10.10.10.3` | HSRP virtual IP: `10.10.10.1`

---

## 🧠 How HSRP Works

| Concept | Description |
| --- | --- |
| **Virtual IP** | A shared IP address used as the hosts' default gateway |
| **Virtual MAC** | A shared MAC address (`0000.0C07.ACxx`) used with the virtual IP |
| **Active Router** | Forwards traffic for the virtual IP — elected by priority |
| **Standby Router** | Monitors the active router; takes over if active fails |
| **Hello Messages** | Sent every 3 seconds; if active misses 10 sec (hold time), standby takes over |
| **HSRP Group** | Both routers must be in the same HSRP group number |

> 💡 Hosts only know the virtual IP/MAC — they never know which physical router is active. This makes failover **transparent** to end devices.
> 

---

## 🔑 Part 1 – Basic HSRP Configuration

### Step 1 – Configure Physical IPs on R1 and R2

```
R1(config)#interface g0/1
R1(config-if)#ip address 10.10.10.2 255.255.255.0
R1(config-if)#no shutdown

R2(config)#interface g0/1
R2(config-if)#ip address 10.10.10.3 255.255.255.0
R2(config-if)#no shutdown
```

---

### Step 2 – Configure HSRP Virtual IP

Both routers must use the **same group number (1)** and the **same virtual IP**:

```
R1(config-if)#standby 1 ip 10.10.10.1    ! Group 1, virtual IP 10.10.10.1

R2(config-if)#standby 1 ip 10.10.10.1    ! Same group, same virtual IP
```

> The virtual IP (`10.10.10.1`) must be **different** from both routers' physical IPs. This is the address hosts use as their default gateway.
> 

---

### Step 3 – Verify HSRP Status

```
R1#show standby
GigabitEthernet0/1 - Group 1
  State is Standby
  Virtual IP address is 10.10.10.1
  Active virtual MAC address is 0000.0C07.AC01
  Hello time 3 sec, hold time 10 sec
  Preemption disabled
  Active router is 10.10.10.3    ! R2 is active (higher IP wins by default)
  Standby router is local
  Priority 100 (default 100)
```

```
R2#show standby
  State is Active
  Active router is local
  Standby router is 10.10.10.2
```

> With equal default priority (100), the router with the **highest IP address** becomes active — R2 (10.10.10.3) wins over R1 (10.10.10.2).
> 

---

### Step 4 – Understand the HSRP Virtual MAC Address

|  | Address |
| --- | --- |
| R2 Physical MAC | `0001.6470.2502` (unique to R2) |
| HSRP Virtual MAC | `0000.0C07.AC01` (shared virtual) |

**Virtual MAC format:** `0000.0C07.ACxx` where `xx` = HSRP group number in hex

- Group 1 = `0x01` → MAC = `0000.0C07.AC01`
- Group 10 = `0x0A` → MAC = `0000.0C07.AC0A`

> PCs use the **virtual MAC** as their default gateway MAC — verify with `arp -a` on a PC. When failover occurs, the virtual MAC moves to the new active router.
> 

---

## 🔑 Part 2 – Priority and Pre-emption

### HSRP Election Rules

| Factor | Details |
| --- | --- |
| **Priority** | Configurable 0–255, default 100. Higher priority = preferred active |
| **Tiebreaker** | If priorities are equal, highest IP address wins |
| **Pre-emption** | If disabled (default), a higher-priority router that comes online will NOT take over from the current active router |
| **Pre-emption enabled** | A higher-priority router **will** take over when it comes online |

---

### Step 5 – Set R1 as Preferred Active Router

```
R1(config)#interface g0/1
R1(config-if)#standby 1 priority 110    ! Higher than R2's default of 100
```

> After setting priority 110, R1 is preferred — but R2 **remains active** because pre-emption is disabled by default. HSRP election is **non-preemptive** — the current active stays until it fails.
> 

---

### Step 6 – Enable Pre-emption to Force R1 Active

```
R1(config-if)#standby 1 preempt
```

> `preempt` allows R1 to immediately **take over** as active because it now has the higher priority (110 > 100).
> 

**Verify R1 is now active:**

```
R1#show standby
  State is Active
  Preemption enabled
  Active router is local
  Standby router is 10.10.10.3, priority 100
  Priority 110 (configured 110)
```

---

## 🔑 Part 3 – HSRP Failover Test

### Step 7 – Run Continuous Ping from PC1

```
PC1> ping 10.10.10.1 -n 1000    ! Sends 1000 pings to the HSRP virtual IP
```

### Step 8 – Reboot R1 (Active Router)

```
R1#copy run start
R1#reload
Proceed with reload? [confirm]
```

### Step 9 – Observe Failover

- PC1 ping output shows **a few dropped packets** as R2 detects R1 is gone (after 10-second hold timer) and transitions to Active
- R2 becomes Active, takes ownership of virtual IP `10.10.10.1` and virtual MAC `0000.0C07.AC01`
- Connectivity resumes automatically

```
R2#show standby
  State is Active
  Active router is local
```

### Step 10 – R1 Returns and Pre-empts

- R1 boots up, HSRP comes up, R1 sees it has higher priority (110) and pre-emption enabled
- R1 takes back the Active role from R2
- PC1 sees another brief drop, then connectivity resumes

```
R1#show standby
  State is Active
  Preemption enabled
  Priority 110 (configured 110)
```

---

## 📊 HSRP States

| State | Description |
| --- | --- |
| **Initial** | HSRP just started, not yet running |
| **Learn** | Waiting to hear the virtual IP from active router |
| **Listen** | Knows the virtual IP, monitoring hellos |
| **Speak** | Sending hellos, participating in election |
| **Standby** | Backup router, monitoring active |
| **Active** | Forwarding traffic for the virtual IP |

---

## 📋 Key Commands Summary

| Command | Purpose |
| --- | --- |
| `standby <grp> ip <vip>` | Configure HSRP group and virtual IP |
| `standby <grp> priority <0-255>` | Set HSRP priority (default 100, higher wins) |
| `standby <grp> preempt` | Allow higher-priority router to take over |
| `show standby` | View HSRP group status, active/standby, priority |
| `show standby brief` | One-line summary of all HSRP groups |
| `show interface <int>` | View physical interface MAC address |
| `ping <ip> -n <count>` | Send extended ping from PC |

---

## 🗒️ Key Takeaways

- HSRP provides **gateway redundancy** — hosts point to the virtual IP, not a physical router IP.
- The **virtual MAC** (`0000.0C07.ACxx`) is what hosts use in their ARP cache — it moves seamlessly to the new active router on failover.
- Default election: **highest priority wins**; tiebreaker = **highest IP**.
- **Pre-emption is disabled by default** — without it, a higher-priority router coming online will NOT displace the current active. Enable with `standby preempt`.
- Failover causes only **a few dropped packets** (determined by the hold timer, default 10 seconds).
- Always configure **matching HSRP group numbers** on both routers or they won't form a group.
- `show standby` is the go-to verification command — shows state, virtual IP, virtual MAC, priority, and pre-emption status.
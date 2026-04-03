# 27-1 - Port Security Configuration

Module Number: 27-1
Status: Completed
Topic: Disable Unused Ports, Port Security, MAC Address Limits, Violation Modes, Sticky MAC

> **Lab Type:** Exercise + Answer Key | **Devices:** 1 Switch (SW1), 2 PCs in Packet Tracer
> 

---

## 🧭 Overview

Port Security is a Layer 2 security feature that controls which devices can connect to a switch port based on **MAC address**. It protects against:

- Unauthorised hosts plugging into unused ports
- MAC address flooding attacks (filling the CAM table)
- Rogue devices connecting to secure ports

This module covers:

1. **Disabling unused ports** — deny access at the physical level
2. **Port Security with static MAC** — manually specify allowed MACs
3. **Port Security with dynamic learning** — let the switch learn MACs automatically
4. **Verifying Port Security** — `show port-security` commands

---

## 📂 Lab Setup

- Download `27-1 Port Security Configuration.zip`, extract and open in Packet Tracer.
- Topology: SW1 with PC1 on Fa0/1 and PC2 on Fa0/2. All other ports are unused.
- PC1 MAC: `0000.1111.1111` | PC2 MAC: `0000.2222.2222`

> ⚠️ **Important Packet Tracer Note:** Enable Port Security and add PC1's static MAC address **before** generating any traffic. If PC1's MAC is dynamically learned first, Packet Tracer cannot remove it and you'll get a "Found duplicate mac-address" error. If this happens: shutdown Fa0/1, save config, reload, then add the static MAC before re-enabling the interface.
> 

---

## 🔑 Part 1 – Disable Unused Ports

### Why Disable Unused Ports?

Unused open switch ports are a security risk — anyone can plug in a device and gain network access. Shutting them down prevents unauthorised access at the physical layer.

### Step 1 – Identify Active and Unused Ports

```
SW1#show ip interface brief
FastEthernet0/1    unassigned    YES manual up   up     ! PC1 → in use
FastEthernet0/2    unassigned    YES manual up   up     ! PC2 → in use
FastEthernet0/3    unassigned    YES manual down down   ! Unused
...
FastEthernet0/24   unassigned    YES manual down down   ! Unused
GigabitEthernet0/1 unassigned    YES manual down down   ! Unused
GigabitEthernet0/2 unassigned    YES manual down down   ! Unused
```

### Step 2 – Shutdown All Unused Ports

```
SW1(config)#interface range f0/3 - 24
SW1(config-if-range)#shutdown

SW1(config-if-range)#interface range g0/1 - 2
SW1(config-if-range)#shutdown
```

> 💡 Use `interface range` to shutdown multiple ports with a single command. This is much faster than configuring each port individually.
> 

---

## 🔑 Part 2 – Port Security with Static MAC Address

### Port Security Defaults

| Setting | Default Value |
| --- | --- |
| Maximum MAC addresses | 1 |
| Violation mode | Shutdown |
| Sticky learning | Disabled |
| Aging | Disabled |

### Step 3 – Find PC1's MAC Address

From the PC:

```
PC1> ipconfig /all
Physical Address: 0000.1111.1111
```

Or from the switch (generate traffic first with a ping):

```
SW1#show mac address-table
VLAN  MAC Address       Type    Ports
1     0000.1111.1111    DYNAMIC Fa0/1
```

---

### Step 4 – Configure Port Security on Fa0/1

Port Security requires the port to be an **access port** first:

```
SW1(config)#interface f0/1
SW1(config-if)#switchport mode access           ! Required before port-security
SW1(config-if)#switchport port-security         ! Enable port security (default: max 1 MAC, shutdown on violation)
SW1(config-if)#switchport port-security maximum 2         ! Allow up to 2 MAC addresses
SW1(config-if)#switchport port-security mac-address 0000.1111.1111  ! Statically add PC1's MAC
```

> 💡 The static MAC entry counts toward the maximum. With `maximum 2` and one static MAC, the port can dynamically learn one additional MAC.
> 

---

## 🔑 Part 3 – Port Security with Default Settings (Dynamic Learning)

### Step 5 – Configure Port Security on Fa0/2 (Defaults Only)

```
SW1(config)#interface f0/2
SW1(config-if)#switchport mode access
SW1(config-if)#switchport port-security          ! Enable with defaults: max=1, mode=shutdown
```

> With default settings, the switch dynamically learns the **first** MAC address that sends traffic on this port and locks it in. Any other MAC triggers a violation.
> 

### Step 6 – Generate Traffic to Trigger Dynamic Learning

```
PC2> ping 10.0.0.1     ! Any traffic causes SW1 to learn PC2's MAC
```

```
SW1#show port-security
Secure Port  MaxSecureAddr  CurrentAddr  SecurityViolation  Security Action
    Fa0/1              2            1                  0         Shutdown
    Fa0/2              1            1                  0         Shutdown
```

> PC2's MAC (`0000.2222.2222`) is now dynamically learned and locked to Fa0/2.
> 

---

## 🔑 Part 4 – Port Security Violation Modes

| Mode | What Happens on Violation | Port State | Counter Increments | Syslog/SNMP |
| --- | --- | --- | --- | --- |
| **Shutdown** (default) | Port goes to **err-disabled** state | err-disabled | Yes | Yes |
| **Restrict** | Drops violating frames, port stays up | Up | Yes | Yes |
| **Protect** | Drops violating frames silently, port stays up | Up | No | No |

**Configure violation mode:**

```
SW1(config-if)#switchport port-security violation shutdown    ! Default
SW1(config-if)#switchport port-security violation restrict
SW1(config-if)#switchport port-security violation protect
```

**Recover an err-disabled port:**

```
SW1(config)#interface f0/1
SW1(config-if)#shutdown
SW1(config-if)#no shutdown
```

---

## 🔑 Part 5 – Sticky MAC Addresses

**Sticky learning** dynamically learns MAC addresses and **saves them to the running configuration** as static entries — they survive a port flap but persist until manually removed (unlike pure dynamic entries which are flushed when the port goes down).

```
SW1(config-if)#switchport port-security mac-address sticky
```

After enabling sticky and generating traffic:

```
SW1#show run | include sticky
switchport port-security mac-address sticky 0000.2222.2222
```

> 💡 Sticky is ideal for environments where you want dynamic learning convenience but persistent security — no need to manually enter MAC addresses, but they are saved across reboots (after `copy run start`).
> 

---

## 🔑 Part 6 – Verify Port Security

### show port-security (Summary)

```
SW1#show port-security
Secure Port  MaxSecureAddr  CurrentAddr  SecurityViolation  Security Action
    Fa0/1              2            1                  0         Shutdown
    Fa0/2              1            1                  0         Shutdown
```

### show port-security interface (Detailed)

```
SW1#show port-security interface f0/1
Port Security              : Enabled
Port Status                : Secure-up
Violation Mode             : Shutdown
Aging Time                 : 0 mins
Aging Type                 : Absolute
SecureStatic Address Aging : Disabled
Maximum MAC Addresses      : 2
Total MAC Addresses        : 1
Configured MAC Addresses   : 1          ! Static entry
Sticky MAC Addresses       : 0
Last Source Address:Vlan   : 0000.1111.1111:1
Security Violation Count   : 0
```

```
SW1#show port-security interface f0/2
Port Security              : Enabled
Port Status                : Secure-up
Violation Mode             : Shutdown
Maximum MAC Addresses      : 1
Total MAC Addresses        : 1
Configured MAC Addresses   : 0          ! Dynamically learned
Last Source Address:Vlan   : 0000.2222.2222:1
Security Violation Count   : 0
```

### show port-security address

```
SW1#show port-security address
               Secure Mac Address Table
Vlan  Mac Address       Type            Ports   Remaining Age
1     0000.1111.1111    SecureConfigured Fa0/1   -
1     0000.2222.2222    SecureDynamic    Fa0/2   -
```

---

## 📋 Key Commands Summary

| Command | Purpose |
| --- | --- |
| `interface range <ports>` | Select multiple interfaces |
| `shutdown` | Disable a port |
| `switchport mode access` | Required before enabling port security |
| `switchport port-security` | Enable port security with defaults |
| `switchport port-security maximum <n>` | Set max allowed MAC addresses |
| `switchport port-security mac-address <mac>` | Statically add an allowed MAC |
| `switchport port-security mac-address sticky` | Enable sticky MAC learning |
| `switchport port-security violation <mode>` | Set violation action (shutdown/restrict/protect) |
| `show port-security` | Summary of port security on all interfaces |
| `show port-security interface <int>` | Detailed port security info per interface |
| `show port-security address` | View all secured MAC addresses |
| `ipconfig /all` (PC) | Find PC's MAC address on Windows |
| `show mac address-table` | View switch MAC address table |

---

## 🗒️ Key Takeaways

- **Always shutdown unused ports** — it's the simplest and most effective Layer 2 access control.
- Port Security requires `switchport mode access` before it can be enabled — it does not work on trunk ports.
- Default Port Security: **max 1 MAC, violation mode = shutdown**.
- **Static MACs** are manually configured and saved to config; **dynamic MACs** are learned but lost on port down; **sticky MACs** are learned and saved to running config.
- **Shutdown** violation mode err-disables the port and requires manual recovery; **Restrict** keeps the port up but drops violating frames and logs; **Protect** silently drops with no logging.
- `show port-security interface` is the best command for detailed verification — shows status, mode, MAC counts, and violation count.
- In Packet Tracer, always configure static MAC entries **before** generating traffic to avoid the duplicate MAC issue.
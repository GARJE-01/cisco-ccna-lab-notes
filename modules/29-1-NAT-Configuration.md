# 29-1 - NAT Configuration

Module Number: 29-1
Status: Completed
Topic: Static NAT, Dynamic NAT, PAT, NAT Pool, ip nat inside/outside, overload

> **Lab Type:** Exercise + Answer Key | **Devices:** R1 (WAN edge), SP1, Int-S1, Ext-S1, PC1, PC2 in Packet Tracer
> 

---

## 🧭 Overview

NAT (Network Address Translation) translates private IP addresses to public IP addresses, allowing internal hosts to communicate with the Internet. This module covers all three NAT types:

- **Static NAT** — one-to-one permanent mapping (for servers)
- **Dynamic NAT** — pool of public IPs assigned on demand (first-come, first-served)
- **PAT (Port Address Translation)** — many-to-one using port numbers (most common in real networks)

---

## 🧠 NAT Key Concepts

| Term | Description |
| --- | --- |
| **Inside Local** | Private IP of internal host |
| **Inside Global** | Public IP representing the internal host |
| **Outside Local** | IP of external host as seen from inside |
| **Outside Global** | Actual IP of external host |
| **ip nat inside** | Applied to interface facing internal network |
| **ip nat outside** | Applied to interface facing Internet/ISP |

---

## 📂 Lab Setup

- Download `29-1 NAT Configuration.zip`, extract and open in Packet Tracer.
- R1 is the WAN edge router with a default route to SP1.
- Public IP range purchased: `203.0.113.0/28`
    - `203.0.113.1` — SP1 (ISP gateway)
    - `203.0.113.2` — R1 Fa0/0 (Internet-facing interface)
    - `203.0.113.3–14` — Available for NAT
- Int-S1 (internal web server): `10.0.1.10`
- PCs (internal clients): `10.0.2.0/24` subnet
- Ext-S1 (external server): `203.0.113.20`

> ⚠️ NAT translation table entries **age out quickly** in Packet Tracer — regenerate traffic if entries disappear.
> 

---

## 🔑 Part 1 – Static NAT

### When to Use Static NAT

Use **Static NAT** when an internal server needs a **permanent, fixed public IP** so external clients can always reach it (e.g., web servers, mail servers).

### Step 1 – Mark Inside and Outside Interfaces

```
R1(config)#interface f0/0
R1(config-if)#ip nat outside       ! Internet-facing interface

R1(config)#interface f0/1
R1(config-if)#ip nat inside        ! Interface facing Int-S1 (web server)
```

### Step 2 – Create Static NAT Mapping

```
R1(config)#ip nat inside source static 10.0.1.10 203.0.113.3
!                                       ^inside    ^outside
!                                       local      global
```

This permanently maps:

- `10.0.1.10` (Int-S1 private IP) → `203.0.113.3` (public IP)

### Verify in NAT Translation Table

```
R1#show ip nat translations
Pro  Inside global     Inside local    Outside local        Outside global
---  203.0.113.3       10.0.1.10       ---                  ---
tcp  203.0.113.3:443   10.0.1.10:443   203.0.113.20:1027    203.0.113.20:1027
```

> 💡 The static mapping (`---` protocol) always exists. Active sessions show the port-specific entries.
> 

---

## 🔑 Part 2 – Dynamic NAT

### When to Use Dynamic NAT

Use **Dynamic NAT** when you have a **pool of public IPs** to assign to internal hosts on demand. Each host gets a unique public IP. When the pool is exhausted, new connections fail.

### Step 1 – Mark Inside Interface (F0/0 is already outside)

```
R1(config)#interface f1/0
R1(config-if)#ip nat inside        ! Interface facing PC subnet 10.0.2.0/24
```

### Step 2 – Create NAT Pool

```
R1(config)#ip nat pool Flackbox 203.0.113.4 203.0.113.12 netmask 255.255.255.240
!                     ^name     ^first IP    ^last IP
```

### Step 3 – Create ACL to Match Internal Hosts

```
R1(config)#access-list 1 permit 10.0.2.0 0.0.0.255
```

### Step 4 – Link ACL to NAT Pool

```
R1(config)#ip nat inside source list 1 pool Flackbox
```

### Debug NAT Translations Live

```
R1#debug ip nat
IP NAT debugging is on

! After PC1 pings Ext-S1:
*NAT: s=10.0.2.10->203.0.113.4, d=203.0.113.20   ! PC1 translated to first pool IP
*NAT*: s=203.0.113.20, d=203.0.113.4->10.0.2.10  ! Return traffic translated back
```

```
R1#undebug all      ! Turn off debugging when done
```

### What Happens When Pool is Exhausted?

> ⚠️ When all 9 addresses (203.0.113.4–12) are in use, the next PC attempting to connect will **fail** — no public IP available. Traffic drops until a translation times out and releases an IP back to the pool.
> 

### Step 5 – Add PAT Overload to Dynamic NAT Pool

```
R1#clear ip nat translation *                              ! Clear existing translations first
R1(config)#ip nat inside source list 1 pool Flackbox overload
!                                                          ^overload = enable PAT
```

> `overload` enables Port Address Translation on top of the pool — when all IPs are in use, the last IP in the pool is shared using different port numbers.
> 

### Cleanup – Remove All NAT Config

```
R1(config)#int f0/0
R1(config-if)#no ip nat outside
R1(config-if)#int f0/1
R1(config-if)#no ip nat inside
R1(config-if)#int f1/0
R1(config-if)#no ip nat inside
R1(config)#no ip nat inside source static 10.0.1.10 203.0.113.3
R1#clear ip nat translation *
R1(config)#no ip nat inside source list 1 pool Flackbox overload
R1(config)#no ip nat pool Flackbox 203.0.113.4 203.0.113.12 netmask 255.255.255.240
R1(config)#no access-list 1

R1#show run | section nat     ! Should be empty
R1#show access-list           ! Should be empty
```

---

## 🔑 Part 3 – Port Address Translation (PAT)

### When to Use PAT

PAT (**NAT Overload**) maps **many internal hosts to a single public IP** using unique **source port numbers** to track each session. This is how virtually all home and small business routers work — one public IP shared by all internal devices.

### Step 1 – Configure WAN Interface as DHCP Client

```
R1(config)#int f0/0
R1(config-if)#shutdown
R1(config-if)#no ip address
R1(config-if)#ip address dhcp       ! Get public IP dynamically from ISP
R1(config-if)#no shutdown

R1#show ip interface brief
FastEthernet0/0    203.0.113.13    YES DHCP    up    up   ! Assigned by ISP
```

### Step 2 – Configure Inside and Outside Interfaces

```
R1(config)#int f0/0
R1(config-if)#ip nat outside

R1(config)#int f1/0
R1(config-if)#ip nat inside
```

### Step 3 – Create ACL for Internal Hosts

```
R1(config)#access-list 1 permit 10.0.2.0 0.0.0.255
```

### Step 4 – Configure PAT Using Interface IP

```
R1(config)#ip nat inside source list 1 interface f0/0 overload
!                                       ^ACL      ^use interface IP, not pool
!                                                 ^overload = PAT
```

> 💡 Using `interface f0/0` instead of a named pool means NAT **automatically tracks the DHCP-assigned IP** — if the ISP assigns a different IP, PAT continues working without reconfiguration.
> 

### Verify PAT — Both PCs Use Same Public IP with Different Ports

```
R1#show ip nat translations
Pro  Inside global           Inside local      Outside local        Outside global
tcp  203.0.113.13:1025       10.0.2.10:1025    203.0.113.20:80      203.0.113.20:80
tcp  203.0.113.13:1024       10.0.2.11:1025    203.0.113.20:80      203.0.113.20:80
```

> PC1 (10.0.2.10) and PC2 (10.0.2.11) **both** translate to `203.0.113.13` but with **different source ports** (1025 vs 1024). This is how PAT distinguishes between sessions.
> 

### NAT Statistics

```
R1#show ip nat statistics
Total translations: 2 (0 static, 2 dynamic, 2 extended)
Outside Interfaces: FastEthernet0/0
Inside Interfaces: FastEthernet1/0
Hits: 41   Misses: 32
Expired translations: 16
```

---

## 📊 NAT Types Comparison

| Type | Config Command | Public IPs Used | Use Case |
| --- | --- | --- | --- |
| Static | `ip nat inside source static <local> <global>` | 1 per server | Servers needing fixed public IP |
| Dynamic | `ip nat inside source list <acl> pool <pool>` | 1 per active client | Pool of IPs for multiple clients |
| PAT | `ip nat inside source list <acl> interface <int> overload` | 1 for all clients | Home/office internet sharing |
| Dynamic+PAT | `ip nat inside source list <acl> pool <pool> overload` | Pool + port multiplexing | Pool with overload fallback |

---

## 📋 Key Commands Summary

| Command | Purpose |
| --- | --- |
| `ip nat inside` | Mark interface as NAT inside (private side) |
| `ip nat outside` | Mark interface as NAT outside (public side) |
| `ip nat inside source static <local> <global>` | Configure static NAT one-to-one mapping |
| `ip nat pool <n> <start> <end> netmask <mask>` | Create a pool of public IP addresses |
| `ip nat inside source list <acl> pool <pool>` | Dynamic NAT — ACL to pool |
| `ip nat inside source list <acl> interface <int> overload` | PAT using interface IP |
| `ip nat inside source list <acl> pool <pool> overload` | Dynamic NAT + PAT overload |
| `show ip nat translations` | View active NAT translation table |
| `show ip nat statistics` | View NAT hit/miss counters and interfaces |
| `debug ip nat` | Live NAT translation debug output |
| `clear ip nat translation *` | Clear all dynamic NAT entries |
| `undebug all` | Stop all debug output |

---

## 🗒️ Key Takeaways

- Always mark interfaces with `ip nat inside` (private) and `ip nat outside` (public) — NAT won't work without this.
- **Static NAT** gives servers a permanent public IP — initiated from either direction.
- **Dynamic NAT** assigns pool IPs first-come-first-served — connections fail when pool is exhausted.
- **PAT** (`overload`) solves the exhaustion problem by sharing one IP with port numbers — used in almost all real networks.
- Use `interface <int> overload` (not a pool) when the public IP is DHCP-assigned — it automatically follows IP changes.
- `debug ip nat` shows real-time translation: `s=<src>-><translated>` and `d=<dst>-><original>` — invaluable for troubleshooting.
- `clear ip nat translation *` is required before changing dynamic NAT config (e.g., adding `overload`).
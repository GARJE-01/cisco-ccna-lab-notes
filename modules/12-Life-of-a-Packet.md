# 12 - The Life of a Packet

Module Number: 12
Status: Completed
Topic: DNS Client Configuration, ARP Cache

> **Lab Type:** Exercise + Answer Key | **Devices:** 3 Routers (R1–R3), 1 DNS Server (10.10.10.10) in Packet Tracer
> 

---

## 🧭 Overview

This module covers two key networking concepts that describe how a packet travels across a network:

- **DNS (Domain Name System)** — how routers resolve hostnames to IP addresses
- **ARP (Address Resolution Protocol)** — how devices map IP addresses to MAC addresses on local networks

---

## 📂 Lab Setup

- Download `12 The Life of a Packet.zip`, extract and open in Packet Tracer.
- Pre-configured topology with static routes between R1 and R3.
- A **Packet Tracer server device** at `10.10.10.10` acts as the DNS server (Packet Tracer does not support `ip dns server` on routers).
- DNS server is pre-configured to resolve: `R1`, `R2`, and `R3`.

---

## 🔑 Part 1 – DNS Client Configuration

### What is DNS?

DNS (Domain Name System) translates **hostnames** (like `R2`) into **IP addresses** (like `10.10.10.2`), so devices can reach each other by name instead of having to remember IP addresses.

---

### Configure Routers as DNS Clients

```
R1(config)#ip domain-lookup            ! Enable DNS lookups (on by default, use if disabled)
R1(config)#ip name-server 10.10.10.10  ! Point to the DNS server

R2(config)#ip domain-lookup
R2(config)#ip name-server 10.10.10.10

R3(config)#ip domain-lookup
R3(config)#ip name-server 10.10.10.10
```

> 💡 No domain name or domain-list is required in this lab since hostnames are used without a suffix.
> 

---

### Verify DNS Resolution with Ping by Hostname

```
R1#ping R2      ! Router resolves 'R2' to 10.10.10.2 via DNS, then pings
R1#ping R3      ! Router resolves 'R3' to 10.10.20.1 via DNS, then pings

R3#ping R1
R3#ping R2
```

**Expected output:**

```
R1#ping R2
Translating "R2"...domain server (10.10.10.10)
Sending 5, 100-byte ICMP Echos to 10.10.10.2 ...
!!!!!
Success rate is 100 percent (5/5)
```

> ⚠️ DNS resolution may take a moment on first attempt — be patient before declaring a failure.
> 

---

## 🔑 Part 2 – ARP Cache

### What is ARP?

ARP (Address Resolution Protocol) maps a known **IP address** to an unknown **MAC address** on a local network segment. Devices cache ARP results to avoid sending broadcast requests every time.

**Key rules about ARP:**

- ARP uses **broadcast** messages — they are NOT forwarded by routers
- A device only has ARP entries for hosts on its **directly connected** networks
- Entries have an aging timer and are flushed periodically

---

### ARP Cache Logic — Will R1 have R3 in its ARP table?

> ❓ **Answer: NO.**
> 
- R1 is connected to `10.10.10.0/24` — so ARP entries exist only for hosts on that subnet
- R3 is at `10.10.20.1` (different subnet) — R1 has no direct Layer 2 connection to R3
- R1 reaches R3 **via R2** — so only R2's MAC (`10.10.10.2`) appears in R1's ARP cache as next-hop

---

### Verify the ARP Cache

```
R1#show arp
R2#show arp
R3#show arp
```

**R1 ARP Cache:**

| Protocol | Address | Age | Hardware Addr | Interface |
| --- | --- | --- | --- | --- |
| Internet | 10.10.10.1 | - | 0090.0CD7.0D01 | Fa0/0 (self) |
| Internet | 10.10.10.2 | 4 | 0004.9A96.A9A5 | Fa0/0 (R2) |
| Internet | 10.10.10.10 | 2 | 0090.21C6.D284 | Fa0/0 (DNS server) |

> 💡 `-` in the Age column means this is the **device's own** IP address (no aging).
> 

**R2 ARP Cache** (connected to both subnets):

| Address | Interface | Note |
| --- | --- | --- |
| 10.10.10.1 | Fa0/0 | R1 |
| 10.10.10.2 | Fa0/0 | Self |
| 10.10.10.10 | Fa0/0 | DNS server |
| 10.10.20.1 | Fa1/0 | R3's gateway side |
| 10.10.20.2 | Fa1/0 | Self |

**R3 ARP Cache** (only connected to 10.10.20.0/24):

| Address | Interface | Note |
| --- | --- | --- |
| 10.10.20.1 | Fa0/0 | Self |
| 10.10.20.2 | Fa0/0 | R2's side |

> R3 has NO entries for the `10.10.10.0/24` network — it is not directly connected.
> 

---

## 📋 Key Commands Summary

| Command | Purpose |
| --- | --- |
| `ip domain-lookup` | Enable DNS resolution on the router |
| `ip name-server <ip>` | Specify the DNS server IP address |
| `ping <hostname>` | Ping using a hostname (triggers DNS lookup) |
| `show arp` | View the ARP cache (IP-to-MAC mappings) |

---

## 🗒️ Key Takeaways

- Routers act as **DNS clients** using `ip name-server` — they do NOT run DNS server software in Packet Tracer.
- ARP is **Layer 2 and broadcast-based** — it only works within a directly connected subnet.
- Routers do NOT have ARP entries for remote hosts — only for **next-hop** routers and local devices.
- R2, being connected to both subnets, has ARP entries for both `10.10.10.0/24` and `10.10.20.0/24`.
- The `-` in the ARP Age field means the entry belongs to the device itself.
# 23-1 - DHCP Configuration

Module Number: 23-1
Status: Completed
Topic: DHCP Client, DHCP Server, Excluded Addresses, DHCP Pool, ip helper-address

> **Lab Type:** Exercise + Answer Key | **Devices:** 1 Router (R1), PCs, External DHCP Server (10.10.20.10) in Packet Tracer
> 

---

## 🧭 Overview

This module covers DHCP from all three angles:

- **DHCP Client** — configuring a router interface to receive its IP from a DHCP server (like a home router getting an IP from the ISP)
- **Cisco DHCP Server** — configuring a router to hand out IP addresses to hosts
- **External DHCP Server** — forwarding DHCP requests across routers using `ip helper-address`

---

## 📂 Lab Setup

- Download `23-1 DHCP Configuration.zip`, extract and open in Packet Tracer.
- Topology: R1 with outside interface facing ISP, inside interface facing LAN (10.10.10.0/24).
- External DHCP server at `10.10.20.10` (used in Part 3 only).
- DNS server at `10.10.20.10`.

---

## 🔑 Part 1 – Cisco DHCP Client

### What is a DHCP Client?

Instead of manually assigning an IP to a router interface, you can configure it to **automatically receive** an IP from a DHCP server — common on ISP-facing (WAN) interfaces.

### Configure Interface as DHCP Client

```
R1(config)#interface f0/0
R1(config-if)#ip address dhcp        ! Request IP via DHCP instead of static config
R1(config-if)#no shutdown
```

### Verify IP Was Received

```
R1#show ip interface brief
FastEthernet0/0    203.0.113.2    YES DHCP    up    up
```

> The `Method` column shows `DHCP` — confirming the IP was dynamically assigned.
> 

### Find the DHCP Server Address

```
R1#show dhcp lease
Temp IP addr:      203.0.113.2
DHCP Lease server: 203.0.113.1        ! ISP's DHCP server
Temp default-gateway addr: 203.0.113.1
Lease: 86400 secs                      ! 24-hour lease
```

> `show dhcp lease` shows the full lease info including the server IP, lease duration, renewal and rebind timers.
> 

---

## 🔑 Part 2 – Cisco DHCP Server

### How DHCP Works

The DORA process:

| Step | Message | Direction | Description |
| --- | --- | --- | --- |
| 1 | **Discover** | Client → Broadcast | Client looks for any DHCP server |
| 2 | **Offer** | Server → Client | Server offers an IP address |
| 3 | **Request** | Client → Broadcast | Client formally requests the offered IP |
| 4 | **Acknowledge** | Server → Client | Server confirms the lease |

---

### Step 1 – Exclude Static IP Range

Prevent DHCP from handing out addresses reserved for servers/printers:

```
R1(config)#ip dhcp excluded-address 10.10.10.1 10.10.10.10
```

> Always exclude **before** creating the pool. Addresses 10.10.10.1–10 are reserved; DHCP will only assign 10.10.10.11 and above.
> 

---

### Step 2 – Create a DHCP Pool

```
R1(config)#ip dhcp pool Flackbox
R1(dhcp-config)#network 10.10.10.0 255.255.255.0    ! Subnet to assign from
R1(dhcp-config)#default-router 10.10.10.1           ! Gateway given to clients
R1(dhcp-config)#dns-server 10.10.20.10              ! DNS server given to clients
```

**DHCP Pool Configuration Options:**

| Command | Purpose |
| --- | --- |
| `network <subnet> <mask>` | Define the address pool subnet |
| `default-router <ip>` | Assign default gateway to clients |
| `dns-server <ip>` | Assign DNS server to clients |
| `lease <days> <hours> <mins>` | Set lease duration (default 1 day) |
| `domain-name <name>` | Assign domain name to clients |

---

### Step 3 – Verify Clients Received IPs

On each PC:

```
PC> ipconfig
IPv4 Address: 10.10.10.11      ! First available after excluded range
Subnet Mask:  255.255.255.0
Default Gateway: 10.10.10.1
DNS Server: 10.10.20.10
```

**On R1 — verify DHCP bindings:**

```
R1#show ip dhcp binding
IP address      Client-ID               Lease expiration        Type
10.10.10.11     0100.5079.6668.01       --                      Automatic
10.10.10.12     0100.5079.6668.02       --                      Automatic
```

**Verify DNS resolution works:**

```
PC> ping DNSserver
```

---

### Step 4 – Remove DHCP Server Config (Cleanup)

```
R1(config)#no ip dhcp excluded-address 10.10.10.1 10.10.10.10
R1(config)#no ip dhcp pool Flackbox
```

**Release and renew on PCs:**

```
PC> ipconfig /release    ! Release current DHCP lease
PC> ipconfig /renew      ! Request new IP — will fail with no DHCP server
```

---

## 🔑 Part 3 – External DHCP Server (ip helper-address)

### The Problem

The external DHCP server at `10.10.20.10` is on a **different subnet** than the PCs (10.10.10.0/24). DHCP uses **broadcast** messages — and routers **do not forward broadcasts** by default.

```
PC broadcast: "Is there a DHCP server?"
R1: Receives the broadcast, but does NOT forward it across subnets → PCs never get a response
```

---

### The Solution – ip helper-address

`ip helper-address` converts the DHCP **broadcast** into a **unicast** and forwards it to the specified DHCP server. Configured on the router interface **facing the clients**.

```
R1(config)#interface f0/1
R1(config-if)#ip helper-address 10.10.20.10    ! Forward DHCP requests to external server
```

> 💡 `ip helper-address` actually relays several UDP broadcast services by default (DHCP, TFTP, DNS, etc.). DHCP relay is the most common use case.
> 

**Flow with ip helper-address:**

```
PC → Broadcast DHCP Discover
R1 (Fa0/1) → Converts to unicast, forwards to 10.10.20.10
DHCP Server → Unicast DHCP Offer back to R1
R1 → Relays offer back to PC as broadcast
PC → Gets IP address ✅
```

### Verify

```
PC> ipconfig /renew
IPv4 Address: 10.10.10.11     ! Received from external DHCP server
```

---

## 📋 Key Commands Summary

| Command | Purpose |
| --- | --- |
| `ip address dhcp` | Configure interface as DHCP client |
| `show dhcp lease` | View DHCP lease info including server IP |
| `ip dhcp excluded-address <start> <end>` | Reserve a range of IPs from DHCP pool |
| `ip dhcp pool <name>` | Create a DHCP pool |
| `network <subnet> <mask>` | Define subnet for the pool |
| `default-router <ip>` | Set gateway to give to clients |
| `dns-server <ip>` | Set DNS server to give to clients |
| `show ip dhcp binding` | View active DHCP leases |
| `show ip dhcp pool` | View pool statistics |
| `no ip dhcp pool <name>` | Remove a DHCP pool |
| `ip helper-address <server-ip>` | Relay DHCP broadcasts to external server |
| `ipconfig /release` (PC) | Release DHCP lease on Windows PC |
| `ipconfig /renew` (PC) | Request new DHCP lease on Windows PC |

---

## 🗒️ Key Takeaways

- `ip address dhcp` on a router interface makes it a DHCP **client** — common for ISP-facing WAN ports.
- Always configure `ip dhcp excluded-address` **before** creating the pool to reserve static IPs for infrastructure devices.
- The DHCP pool `default-router` gives clients their gateway; `dns-server` gives them DNS — these are pushed automatically with every IP assignment.
- DHCP uses **broadcasts** (DORA process) — routers block broadcasts between subnets by default.
- `ip helper-address` converts DHCP broadcasts to **unicast** and relays them to an external DHCP server — configured on the interface **closest to the clients**.
- `show ip dhcp binding` on the router confirms which IPs have been assigned and to which clients.
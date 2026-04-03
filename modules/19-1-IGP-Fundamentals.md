# 19-1 - IGP Fundamentals Configuration

Module Number: 19-1
Status: Completed
Topic: RIPv2 Config, EIGRP Config, Default Route Injection, AD Comparison

> **Lab Type:** Exercise + Answer Key | **Devices:** 5 Routers (R1–R5), 2 Switches, 3 PCs, 1 ISP link in Packet Tracer
> 

---

## 🧭 Overview

This lab focuses on **hands-on configuration** of RIPv2 and EIGRP routing protocols in a real multi-router topology. IP addresses are pre-configured — your job is to enable the routing protocols, advertise the correct networks, inject a default route to the Internet, and observe how AD determines which protocol wins when both run simultaneously.

---

## 📂 Lab Setup

- Download `19-1 IGP Fundamentals.zip`, extract and open in Packet Tracer.
- IP addresses are **already configured** on all router interfaces.
- Topology: R1–R5 interconnected; R4 has an ISP link at `203.0.113.0/24`.

---

## 🔑 Part 1 – RIPv2 Configuration

### Step 1 – Enable RIPv2 on All Routers

Run on **every router (R1–R5)**:

```
R1(config)#router rip
R1(config-router)#version 2
R1(config-router)#no auto-summary
R1(config-router)#network 10.0.0.0
```

> `no auto-summary` — disables classful summarisation so /24 routes are advertised correctly.
> 

> `network 10.0.0.0` — enables RIP on all interfaces with addresses in the 10.x.x.x range.
> 

> The 203.0.113.0/24 ISP network is **not** advertised at this stage.
> 

---

### Step 2 – Verify RIP Routes

```
R1#show ip route
R   10.1.0.0/24 [120/1] via 10.0.0.2, Fa0/0
R   10.1.1.0/24 [120/2] via 10.0.0.2, Fa0/0
               [120/2] via 10.0.3.2, Fa1/1   ! ECMP — equal hop count
R   10.1.2.0/24 [120/2] via 10.0.3.2, Fa1/1
R   10.1.3.0/24 [120/1] via 10.0.3.2, Fa1/1
```

> Two routes to 10.1.1.0/24 because both paths have equal hop count of 2 → ECMP load balancing.
> 

---

### Step 3 – Verify End-to-End Connectivity

```
PC1> ping 10.1.2.10
Reply from 10.1.2.10 ...  Success!
```

---

### Step 4 – Advertise the ISP Network (203.0.113.0/24) via RIP

We want internal routers to know about 203.0.113.0/24, but we must **not** send internal routes to the ISP.

```
R4(config)#router rip
R4(config-router)#passive-interface f1/1       ! Block RIP updates toward ISP
R4(config-router)#network 203.0.113.0          ! Include ISP network in RIP
```

> `passive-interface f1/1` — R4 will no longer send RIP updates out its ISP-facing interface, protecting internal route information.
> 

**Verify all routers see 203.0.113.0/24:**

```
R1#show ip route
R   203.0.113.0/24 [120/2] via 10.0.3.2, FastEthernet1/1
```

---

### Step 5 – Configure Default Route and Inject into RIP

**On R4 — create a static default route to the ISP:**

```
R4(config)#ip route 0.0.0.0 0.0.0.0 203.0.113.2
```

**Inject the default route into RIP so other routers learn it:**

```
R4(config)#router rip
R4(config-router)#default-information originate
```

> `default-information originate` — redistributes R4's default static route into the RIP process so all other routers receive it as `R* 0.0.0.0/0`.
> 

**Verify all routers have a default route:**

```
R1#show ip route
Gateway of last resort is 10.0.3.2 to network 0.0.0.0
R*  0.0.0.0/0 [120/2] via 10.0.3.2, FastEthernet1/1
```

> `R*` = RIP-learned default route. The `*` means it is the **candidate default** (gateway of last resort).
> 

---

## 🔑 Part 2 – EIGRP Configuration

### Step 1 – Enable EIGRP AS 100 on All Routers

Run on **every router (R1–R5)**:

```
R1(config)#router eigrp 100
R1(config-router)#network 10.0.0.0
```

> The 203.0.113.0/24 ISP network is **not** included — no `network 203.0.113.0` here.
> 

---

### Step 2 – Verify EIGRP Adjacencies

```
R1#show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H    Address     Interface    Hold   Uptime
0    10.0.0.2    Fa0/0        11     00:00:20
1    10.0.3.2    Fa1/1        11     00:00:10
```

> R1 has formed EIGRP adjacencies with R2 (10.0.0.2) and R5 (10.0.3.2).
> 

---

### Step 3 – Which Protocol Wins? (RIP vs EIGRP)

Both RIP and EIGRP are advertising the 10.x.x.x networks. The winner is decided by **Administrative Distance**:

| Protocol | AD |
| --- | --- |
| EIGRP | 90 ← wins |
| RIP | 120 |

> ✅ **EIGRP routes replace RIP routes** for all 10.x.x.x networks.
> 

However, RIP is still the **only** protocol advertising:

- `203.0.113.0/24` (ISP network)
- `0.0.0.0/0` (default route via `default-information originate`)

Those remain as RIP routes since EIGRP is not advertising them.

---

### Step 4 – Verify Final Routing Table (RIP + EIGRP)

```
R1#show ip route
Gateway of last resort is 10.0.3.2 to network 0.0.0.0

C   10.0.0.0/24  directly connected, Fa0/0
C   10.0.1.0/24  directly connected, Fa0/1
C   10.0.2.0/24  directly connected, Fa1/0
C   10.0.3.0/24  directly connected, Fa1/1
D   10.1.0.0/24 [90/30720]  via 10.0.0.2    ! EIGRP wins
D   10.1.1.0/24 [90/33280]  via 10.0.0.2    ! EIGRP wins
D   10.1.2.0/24 [90/35840]  via 10.0.0.2    ! EIGRP wins
D   10.1.3.0/24 [90/261120] via 10.0.3.2    ! EIGRP wins
R   203.0.113.0/24 [120/2]  via 10.0.3.2    ! RIP only — EIGRP not advertising this
R*  0.0.0.0/0    [120/2]    via 10.0.3.2    ! RIP default route
```

> 💡 This demonstrates how **multiple protocols can coexist** in the same routing table — each winning on the networks it uniquely advertises or has the best AD for.
> 

---

## 📋 Key Commands Summary

| Command | Purpose |
| --- | --- |
| `router rip` | Enter RIP routing process |
| `version 2` | Enable RIPv2 (multicast updates) |
| `no auto-summary` | Disable classful summarisation |
| `network <classful-network>` | Enable routing protocol on matching interfaces |
| `passive-interface <int>` | Stop sending updates out an interface |
| `default-information originate` | Inject default static route into the routing protocol |
| `router eigrp <asn>` | Enter EIGRP routing process with AS number |
| `show ip route` | View routing table |
| `show ip eigrp neighbors` | View EIGRP adjacency table |
| `show ip rip database` | View RIP topology database |

---

## 🗒️ Key Takeaways

- RIPv2 requires `version 2` and `no auto-summary` for correct operation in modern networks.
- `network` statements in RIP/EIGRP use **classful** addresses (e.g., `network 10.0.0.0` covers all 10.x.x.x interfaces).
- `passive-interface` on ISP-facing ports prevents leaking internal routing info to the provider — critical for security.
- `default-information originate` is how a border router shares its default route with the rest of the network via a routing protocol.
- When RIP and EIGRP both run, **EIGRP wins (AD 90)** for shared networks. RIP-only routes remain if EIGRP doesn’t advertise them.
- The `R*` prefix in the routing table means: RIP-learned default route (gateway of last resort).
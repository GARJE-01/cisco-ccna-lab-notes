# 18 - Connectivity Troubleshooting

Module Number: 18
Status: Completed
Topic: Ping, Traceroute, Missing Static Route, Routing Table Analysis

> **Lab Type:** Troubleshooting Exercise + Answer Key | **Devices:** 5 Routers (R1–R5), 2 Switches, 3 PCs in Packet Tracer
> 

---

## 🧭 Overview

This module is a focused troubleshooting lab. You are given a broken network and must use a structured methodology — ping, traceroute, routing table analysis — to identify and fix the fault.

**Scenario:** PC1 cannot reach PC3. Find the cause and fix it.

---

## 📂 Lab Setup

- Download `18 Connectivity Troubleshooting.zip`, extract and open in Packet Tracer.
- Pre-configured topology with static routes on all routers — except one is missing.
- **Goal:** Restore full connectivity between PC1 and PC3.

---

## 🔬 Troubleshooting Walkthrough

### Step 1 – Confirm the Problem with Ping

```
PC1> ping 10.1.2.10
Request timed out.
Request timed out.
Reply from 10.1.0.1: Destination host unreachable.
Reply from 10.1.0.1: Destination host unreachable.
Success rate is 0 percent (0/5)
```

> `Destination host unreachable` from `10.1.0.1` (R3) tells us R3 is actively responding that it can't forward the packet — the fault is at or beyond R3.
> 

---

### Step 2 – Narrow Down the Fault with Traceroute

```
PC1> tracert 10.1.2.10
  1   10.0.1.1    (R1)  ✅
  2   10.0.0.2    (R2)  ✅
  3   10.1.0.1    (R3)  ✅
  4   10.1.0.1    (R3)  ← looping/unreachable here
  5   *  *  *     Ctrl-C
```

> Traceroute successfully reaches R3 (10.1.0.1) but **goes no further** — traffic is being bounced back or dropped at R3. This points to a **routing problem on R3** for traffic destined to 10.1.2.0/24.
> 

---

### Step 3 – Inspect R3's Routing Table

```
R3#show ip route

Gateway of last resort is not set

S   10.0.0.0/24 [1/0] via 10.1.0.2
S   10.0.1.0/24 [1/0] via 10.1.0.2
S   10.0.2.0/24 [1/0] via 10.1.0.2
S   10.0.3.0/24 [1/0] via 10.1.0.2
C   10.1.0.0/24 is directly connected, FastEthernet0/1
L   10.1.0.1/32 is directly connected, FastEthernet0/1
C   10.1.1.0/24 is directly connected, FastEthernet0/0
L   10.1.1.2/32 is directly connected, FastEthernet0/0
S   10.1.3.0/24 [1/0] via 10.1.1.1
```

> 🚨 **Root Cause Found:** There is **no route to 10.1.2.0/24** in R3's routing table!
> 

R3 has static routes for 10.0.x.x (toward R1/R2) and for 10.1.3.0/24 (toward R4), but **10.1.2.0/24 is completely missing**.

---

### Step 4 – Fix: Add the Missing Static Route on R3

```
R3(config)#ip route 10.1.2.0 255.255.255.0 10.1.1.1    ! R4 (10.1.1.1) is the next hop to PC3's network
```

---

### Step 5 – Verify Connectivity is Restored

```
PC1> ping 10.1.2.10
Request timed out.          ← first ping may still fail (ARP)
Request timed out.
Reply from 10.1.2.10: bytes=32 TTL=124
Reply from 10.1.2.10: bytes=32 TTL=124
Success rate is 50% → improves to 100% on next ping
```

> ✅ Connectivity restored!
> 

---

## 🧠 Troubleshooting Logic Summary

```
1. ping → confirms problem exists at Layer 3
2. traceroute → identifies WHICH hop the traffic fails at
3. show ip route on that router → finds the missing/incorrect route
4. ip route → adds the missing route
5. ping again → verifies fix
```

---

## ⚠️ Reading Ping Output

| Symbol / Message | Meaning |
| --- | --- |
| `!` | Success — reply received |
| `.` / `Request timed out` | No reply — possible routing or firewall issue |
| `Destination host unreachable` | A router along the path has no route to the destination |
| `U` | Unreachable — active ICMP unreachable message received |

> 💡 `Destination host unreachable` with a **source IP in the reply** (e.g., `10.1.0.1`) tells you **exactly which router** is dropping the traffic — that's your starting point.
> 

---

## 📋 Key Commands Summary

| Command | Purpose |
| --- | --- |
| `ping <ip>` | Test Layer 3 reachability |
| `tracert <ip>` (PC) | Trace path hop-by-hop from a Windows PC |
| `traceroute <ip>` (Router) | Trace path hop-by-hop from a Cisco router |
| `show ip route` | View routing table — check for missing routes |
| `ip route <net> <mask> <next-hop>` | Add a missing static route |

---

## 🗒️ Key Takeaways

- When `Destination host unreachable` appears with a **source IP**, that IP identifies the router with the routing problem — go there first.
- `show ip route` on the suspect router will almost always reveal the issue — a missing route, wrong next-hop, or interface that's down.
- Always **verify the fix end-to-end** from the original source (PC1), not just from the router you fixed.
- The first 1–2 pings after fixing a route may still fail due to **ARP cache updates** — this is normal, run the ping again.
- In a static routing environment, every router needs a route for **every destination network** — a single missing entry breaks connectivity.
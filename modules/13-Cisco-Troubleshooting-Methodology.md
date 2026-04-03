# 13 - The Cisco Troubleshooting Methodology

Module Number: 13
Status: Completed
Topic: Troubleshooting Methodology, ping, traceroute, interface status

> **Lab Type:** Troubleshooting Exercise + Answer Key | **Devices:** 3 Routers (R1–R3), 1 DNS Server (10.10.10.10) in Packet Tracer

---

## 🧭 Overview

This module teaches a **structured approach to network troubleshooting**. Rather than randomly guessing at fixes, Cisco recommends a disciplined, layered methodology to isolate and resolve problems efficiently.

The lab scenario: DNS is broken across the network. You must find and fix **three separate issues** causing the failure.

---

## 📂 Lab Setup

- Download `13 The Cisco Troubleshooting Methodology.zip`, extract and open in Packet Tracer.
- Pre-configured with static routes between R1 and R3.
- A Packet Tracer server device at `10.10.10.10` acts as the DNS server.
- **Goal:** Get R3 to successfully `ping R1` using the hostname.

---

## 🧠 The Cisco Troubleshooting Methodology

### The Two First Questions to Always Ask

> **1. Was it working before? If so, what changed?**

> This is the most powerful question — changes are almost always the cause of new problems.

> **2. Is the problem affecting everyone or just one user?**

> - **One user affected** → problem is likely at the user's end

> - **All users affected** → problem is on the server side or network infrastructure

---

### General Troubleshooting Workflow

```
1. Define the problem clearly (what is broken, who is affected)
2. Gather information (error messages, symptoms, scope)
3. Analyse the information (form a hypothesis)
4. Eliminate variables (work from bottom-up: Layer 1 → Layer 7)
5. Propose and implement a fix
6. Verify the fix works end-to-end
7. Document the solution
```

---

## 🔬 Lab Walkthrough – Fixing DNS Step by Step

### Step 1 – Confirm the Problem

```
R3#telnet 10.10.10.10
Trying 10.10.10.10 ...
% Connection timed out; remote host not responding
```

> The Telnet timeout confirms the problem is at the **network layer** (Layer 3) — no point checking the DNS application (Layer 7) until connectivity is restored.

---

### Step 2 – Test Layer 3 Connectivity with Ping

```
R3#ping 10.10.10.10
U.U.U
Success rate is 0 percent (0/5)
```

**Ping output codes:**

| Symbol | Meaning                                                           |
| ------ | ----------------------------------------------------------------- |
| `!`    | Success — reply received                                          |
| `.`    | Timeout — no reply                                                |
| `U`    | Unreachable — destination or network unreachable message received |
| `?`    | Unknown packet type                                               |

> `U` (Unreachable) indicates a router along the path is actively rejecting or cannot forward the packet — not just a timeout.

---

### Step 3 – Use Traceroute to Locate the Fault

```
R3#traceroute 10.10.10.10
1  10.10.20.2    0 msec 0 msec 0 msec
2  10.10.20.2   !H * !H
3  * *
```

**Traceroute output codes:**

| Symbol       | Meaning                             |
| ------------ | ----------------------------------- |
| `msec` value | Hop reachable — RTT in milliseconds |
| `*`          | No response (timeout)               |
| `!H`         | Host unreachable                    |
| `!N`         | Network unreachable                 |
| `!P`         | Protocol unreachable                |

> Traceroute reached **R2** (10.10.20.2) at hop 1, then got `!H` at hop 2 — meaning R2 cannot reach the DNS server. The problem is **between R2 and the DNS server**.

---

### Step 4 – Check Interfaces on R2

```
R2#show ip interface brief
Interface         IP-Address    OK? Method Status                Protocol
FastEthernet0/0   10.10.10.2    YES NVRAM  administratively down down
FastEthernet1/0   10.10.20.2    YES NVRAM  up                   up
```

> **Found Issue #1:** `FastEthernet0/0` (facing the DNS server) is **administratively shutdown**.

**Fix:**

```
R2(config)#interface f0/0
R2(config-if)#no shutdown
```

**Verify fix:**

```
R3#ping 10.10.10.10
..!!!
Success rate is 60 percent (3/5)    ! First pings may drop due to ARP — this is normal
```

---

### Step 5 – Test DNS Resolution by Hostname

```
R3#ping R1
Translating "R1"...domain server (10.10.10.1)
% Unrecognized host or address or protocol not running.
```

> **Read the error carefully!** R3 is querying `10.10.10.1` as its DNS server — but the correct DNS server is `10.10.10.10`.

> **Found Issue #2:** R3 has the **wrong DNS server IP** configured.

**Fix:**

```
R3(config)#no ip name-server 10.10.10.1     ! Remove incorrect entry first
R3(config)#ip name-server 10.10.10.10       ! Add correct DNS server
```

---

### Step 6 – Test Again

```
R3#ping R1
Translating "R1"...domain server (10.10.10.1)
% Unrecognized host or address or protocol not running.
```

> Still failing. Connectivity is confirmed working (ping to 10.10.10.10 succeeds), DNS is pointed correctly — so the problem must be **on the DNS server itself**.

> **Found Issue #3:** The DNS **service is turned off** on the server device.

**Fix (in Packet Tracer GUI):**

- Click the DNS server → **Services tab** → Turn **DNS service ON**
- Verify that A records for `R1`, `R2`, and `R3` are present

---

### Step 7 – Final Verification

```
R3#ping R1
Translating "R1"...domain server (10.10.10.10)
Sending 5, 100-byte ICMP Echos to 10.10.10.1 ...
!!!!!
Success rate is 100 percent (5/5)
```

✅ **Problem solved!**

---

## 📋 Summary of Issues Found

| #   | Issue                               | Location                 | Fix                                            |
| --- | ----------------------------------- | ------------------------ | ---------------------------------------------- |
| 1   | Interface administratively shutdown | R2 Fa0/0                 | `no shutdown`                                  |
| 2   | Wrong DNS server IP configured      | R3                       | `no ip name-server 10.10.10.1` • correct entry |
| 3   | DNS service disabled                | DNS Server (10.10.10.10) | Enable DNS service in GUI                      |

> 💡 In real-world scenarios, problems are usually caused by **just one error**. Multiple simultaneous faults (like this lab) are common when deploying a new service for the first time.

---

## 📋 Key Commands Summary

| Command                   | Purpose                                         |
| ------------------------- | ----------------------------------------------- |
| `ping <ip>`               | Test Layer 3 connectivity                       |
| `traceroute <ip>`         | Trace path hop-by-hop to isolate fault location |
| `telnet <ip>`             | Test TCP application-layer connectivity         |
| `show ip interface brief` | Check interface status (up/down/admin down)     |
| `no shutdown`             | Bring an administratively shutdown interface up |
| `no ip name-server <ip>`  | Remove a DNS server entry                       |
| `ip name-server <ip>`     | Add a DNS server entry                          |

---

## 🗒️ Key Takeaways

- Always **work from the bottom up** (Layer 1 → 7) — fix connectivity before checking applications.
- **Read error messages carefully** — they often tell you exactly what is wrong (`domain server (10.10.10.1)` revealed the wrong DNS IP).
- `ping` with `U` responses = a router is actively rejecting traffic — use `traceroute` to pinpoint where.
- `traceroute` narrows down the fault to a specific **segment or device** — saves time vs. checking every hop.
- An interface showing `administratively down` means it was manually shut down with the `shutdown` command — fix with `no shutdown`.
- Interfaces show **two status fields**: Line Status (physical) and Protocol Status (data link). Both must be `up` for the interface to work.

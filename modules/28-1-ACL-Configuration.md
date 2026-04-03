# 28-1 - ACL Configuration

Module Number: 28-1
Status: Completed
Topic: Standard ACL, Extended ACL, Named ACL, Wildcard Masks, ACL Placement

> **Lab Type:** Exercise + Answer Key | **Devices:** 2 Routers (R1, R2), 3 PCs in Packet Tracer
> 

---

## 🧭 Overview

ACLs (Access Control Lists) are ordered lists of permit/deny rules applied to router interfaces to filter traffic. This module covers all three ACL types used in CCNA:

- **Numbered Standard ACL** — filter by source IP only
- **Numbered Extended ACL** — filter by source, destination, protocol, and port
- **Named Extended ACL** — same as extended but with a descriptive name and editable entries

---

## 🧠 ACL Fundamentals

### ACL Processing Rules

1. Rules are processed **top to bottom** — first match wins
2. There is an **implicit `deny any`** at the end of every ACL — if no rule matches, traffic is dropped
3. ACLs must be **applied to an interface** in a direction (in or out) to take effect
4. Only **one ACL per interface per direction** is allowed

### ACL Types and Numbers

| Type | Number Range | Matches On |
| --- | --- | --- |
| Standard | 1– 99, 1300–1999 | Source IP only |
| Extended | 100–199, 2000–2699 | Source IP, Dest IP, Protocol, Port |
| Named | Any name | Same as numbered equivalents |

### Wildcard Masks

Wildcard masks are the **inverse of subnet masks** — `0` means match, `1` means ignore.

| Subnet Mask | Wildcard Mask | Meaning |
| --- | --- | --- |
| 255.255.255.255 | 0.0.0.0 | Match exact host (`host` keyword) |
| 255.255.255.0 | 0.0.0.255 | Match any host in /24 subnet |
| 0.0.0.0 | 255.255.255.255 | Match any IP (`any` keyword) |

### ACL Placement Best Practice

| ACL Type | Apply | Why |
| --- | --- | --- |
| **Standard** | **Close to destination** | Only checks source — placing near source blocks too broadly |
| **Extended** | **Close to source** | Checks source + dest — drops traffic early before it wastes bandwidth |

---

## 📂 Lab Setup

- Download `28-1 ACL Configuration.zip`, extract and open in Packet Tracer.
- Network: PC1 (10.0.1.10), PC2 (10.0.1.11) on 10.0.1.0/24; PC3 (10.0.2.10) on 10.0.2.0/24.
- R1 connects internal networks to R2 (10.0.0.2) via F0/0.
- R2 has static routes for 10.0.1.0/24 and 10.0.2.0/24.

---

## 🔑 Part 1 – Numbered Standard ACL

### Goal

Deny all traffic from 10.0.2.0/24 (PC3) to R2, while allowing 10.0.1.0/24 (PC1, PC2) to reach R2. PCs must maintain connectivity to each other.

### Why Outbound on F0/0?

A **standard ACL** only checks the **source IP**. If applied inbound on F0/1 (facing 10.0.2.0), it would block PC3 from reaching everything — including PC1/PC2 on the other subnet. The only valid placement is **outbound on F0/0** (facing R2) where we can permit 10.0.1.0 and deny 10.0.2.0 without blocking inter-subnet traffic.

### Configure the ACL

```
R1(config)#access-list 1 deny   10.0.2.0 0.0.0.255    ! Deny PC3's subnet
R1(config)#access-list 1 permit 10.0.1.0 0.0.0.255    ! Permit PC1/PC2's subnet
! Implicit: deny any (blocks everything else)
```

### Apply to Interface

```
R1(config)#interface f0/0
R1(config-if)#ip access-group 1 out    ! Applied outbound toward R2
```

### Verify Results

| Test | Expected | Result |
| --- | --- | --- |
| PC1 ping R2 | ✅ Allowed | Permitted by ACL line 2 |
| PC2 ping R2 | ✅ Allowed | Permitted by ACL line 2 |
| PC3 ping R2 | ❌ Blocked | Denied by ACL line 1 |
| PC3 ping PC1 | ✅ Allowed | Traffic doesn't exit F0/0 |
| PC3 ping PC2 | ✅ Allowed | Traffic doesn't exit F0/0 |

---

## 🔑 Part 2 – Numbered Extended ACL

### Goal

Permit Telnet from **PC1 only** to R2. Deny Telnet from all other PCs. Maintain all other connectivity. Do not modify the existing standard ACL.

### Extended ACL Syntax

```
access-list <number> <permit|deny> <protocol> <source> <src-wildcard> <dest> <dst-wildcard> [eq <port>]
```

**Common protocol/port keywords:**

| Keyword | Protocol | Port |
| --- | --- | --- |
| `ip` | Any IP protocol | Any |
| `tcp` | TCP | — |
| `udp` | UDP | — |
| `icmp` | ICMP (ping) | — |
| `eq telnet` | TCP | 23 |
| `eq 80` | TCP | HTTP 80 |
| `eq 443` | TCP | HTTPS 443 |

### Configure the Extended ACL

```
R1(config)#access-list 100 permit tcp host 10.0.1.10 host 10.0.0.2 eq telnet
! Permit Telnet from PC1 (10.0.1.10) to R2 (10.0.0.2)

R1(config)#access-list 100 deny tcp 10.0.1.0 0.0.0.255 host 10.0.0.2 eq telnet
! Deny Telnet from all other 10.0.1.0/24 hosts to R2

R1(config)#access-list 100 permit ip any any
! Permit all other traffic (without this, implicit deny drops everything)
```

> ⚠️ The `permit ip any any` at the end is critical — without it, the implicit deny would block ALL traffic from 10.0.1.0/24, not just Telnet.
> 

### Apply to Interface (Inbound, Close to Source)

```
R1(config)#interface f1/0
R1(config-if)#ip access-group 100 in    ! Inbound from 10.0.1.0/24 side
```

> F0/0 already has an outbound ACL (ACL 1). Cannot apply a second ACL in the same direction on the same interface.
> 

### Verify Hit Counts

```
R1#show access-lists 100
Extended IP access list 100
  permit tcp host 10.0.1.10 host 10.0.0.2 eq telnet  (23 match(es))
  deny tcp 10.0.1.0 0.0.0.255 host 10.0.0.2 eq telnet (12 match(es))
  permit ip any any                                    (12 match(es))
```

### Verify Results

| PC | Ping R2 | Telnet R2 |
| --- | --- | --- |
| PC1 (10.0.1.10) | ✅ | ✅ |
| PC2 (10.0.1.11) | ✅ | ❌ |
| PC3 (10.0.2.10) | ❌ (blocked by ACL 1) | ❌ |

---

## 🔑 Part 3 – Named Extended ACL

### Advantages of Named ACLs

- Descriptive names make ACLs self-documenting
- Individual entries can be **deleted or inserted** by sequence number (unlike numbered ACLs)
- Easier to manage in large configurations

### Step 1 – Remove Numbered ACL from Interface (Don't Delete the ACL)

```
R1(config)#interface f1/0
R1(config-if)#no ip access-group 100 in    ! Remove ACL 100 from interface only
```

### Step 2 – Configure Named Extended ACL

Goal: PC1 → Telnet to R2 only (no ping). PC2 → Ping R2 only (no Telnet). PC3 → blocked from both.

```
R1(config)#ip access-list extended F1/0_in
R1(config-ext-nacl)#permit tcp host 10.0.1.10 host 10.0.0.2 eq telnet
! Permit Telnet from PC1 to R2

R1(config-ext-nacl)#deny tcp 10.0.1.0 0.0.0.255 host 10.0.0.2 eq telnet
! Deny Telnet from all other 10.0.1.0/24 hosts

R1(config-ext-nacl)#permit icmp host 10.0.1.11 host 10.0.0.2 echo
! Permit ping from PC2 to R2

R1(config-ext-nacl)#deny icmp 10.0.1.0 0.0.0.255 host 10.0.0.2 echo
! Deny ping from all other 10.0.1.0/24 hosts

R1(config-ext-nacl)#permit ip any any
! Allow all other traffic
```

### Step 3 – Apply to Interface

```
R1(config)#interface f1/0
R1(config-if)#ip access-group F1/0_in in
```

### Verify Results

| PC | Ping R2 | Telnet R2 |
| --- | --- | --- |
| PC1 (10.0.1.10) | ❌ (ICMP denied) | ✅ |
| PC2 (10.0.1.11) | ✅ | ❌ |
| PC3 (10.0.2.10) | ❌ (blocked by ACL 1) | ❌ |
| PC1 ↔ PC2 ↔ PC3 | ✅ | — |

---

## 📋 Key Commands Summary

| Command | Purpose |
| --- | --- |
| `access-list <1-99> permit/deny <src> <wildcard>` | Create standard numbered ACL entry |
| `access-list <100-199> permit/deny <proto> <src> <wild> <dst> <wild> [eq <port>]` | Create extended numbered ACL entry |
| `ip access-list extended <name>` | Create/enter named extended ACL |
| `ip access-list standard <name>` | Create/enter named standard ACL |
| `ip access-group <acl> in/out` | Apply ACL to interface |
| `no ip access-group <acl> in/out` | Remove ACL from interface |
| `show access-lists` | View all ACLs and hit counts |
| `show access-lists <n>` | View specific ACL and hit counts |
| `show ip interface <int>` | View ACLs applied to an interface |
| `host <ip>` | Shorthand for `<ip> 0.0.0.0` (exact match) |
| `any` | Shorthand for `0.0.0.0 255.255.255.255` (any IP) |

---

## 🗒️ Key Takeaways

- Every ACL ends with an **implicit deny any** — always add `permit ip any any` if you only want to block specific traffic.
- **Standard ACLs** (source only) go **close to destination**; **Extended ACLs** (source + dest + protocol) go **close to source**.
- Only **one ACL per interface per direction** — you cannot have two inbound ACLs on the same interface.
- ACL rules are matched **top to bottom, first match wins** — always put more specific rules before general ones.
- `show access-lists` shows **hit counts** per rule — invaluable for verifying which rules are being matched.
- Named ACLs support **per-entry deletion** by sequence number — makes them much easier to modify than numbered ACLs.
- `no ip access-group` removes an ACL from an interface **without deleting** the ACL itself.
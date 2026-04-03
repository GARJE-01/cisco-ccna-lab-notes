# 34 - Network Device Management

Module Number: 34
Status: Completed
Topic: SNMP Communities, Syslog, Logging Severity Levels, Trap Logging

> **Lab Type:** Exercise + Answer Key | **Devices:** R1, NMS/Syslog Server (10.0.0.100) in Packet Tracer
> 

---

## 🧭 Overview

This module configures two key network management protocols:

- **SNMP (Simple Network Management Protocol)** — allows a Network Management System (NMS) to monitor and manage devices
- **Syslog** — sends event log messages from devices to a centralised log server for monitoring and troubleshooting

---

## 🧠 SNMP Fundamentals

### What is SNMP?

SNMP allows a central **NMS (Network Management Station)** to:

- **Monitor** device status (interfaces, CPU, memory, errors)
- **Receive alerts** (SNMP Traps) when events occur
- **Configure** devices remotely (if Read-Write access is granted)

### SNMP Components

| Component | Description |
| --- | --- |
| **NMS** | Network Management Station — centrally polls and receives data |
| **SNMP Agent** | Software on the managed device (router/switch) |
| **MIB** | Management Information Base — database of variables the agent exposes |
| **Community String** | Password-like string that authenticates SNMP access |
| **SNMP Trap** | Unsolicited alert sent from agent to NMS when an event occurs |

### SNMP Community String Types

| Type | Access | Use Case |
| --- | --- | --- |
| **RO (Read Only)** | NMS can read device info only | Monitoring |
| **RW (Read Write)** | NMS can read AND modify device config | Management |

> ⚠️ Community strings in SNMPv1/v2c are sent in **plain text** — a significant security weakness. Use SNMPv3 in production for encrypted, authenticated access.
> 

---

## 📂 Lab Setup

- Download `34 Network Device Management.zip`, extract and open in Packet Tracer.
- R1 is the managed device.
- NMS/Syslog server is at `10.0.0.100` — acts as both SNMP manager and Syslog server.

---

## 🔑 Part 1 – SNMP Configuration

### Configure SNMP Community Strings on R1

```
R1(config)#snmp-server community Flackbox1 ro    ! Read Only community
R1(config)#snmp-server community Flackbox2 rw    ! Read Write community
```

> The NMS uses `Flackbox1` to poll/read device info, and `Flackbox2` to make configuration changes via SNMP.
> 

---

## 🔑 Part 2 – Syslog Configuration

### What is Syslog?

Syslog is a standard protocol for devices to send **event messages** (logs) to a central server. All Cisco devices generate syslog messages for events like:

- Interface state changes (up/down)
- Authentication failures
- Routing protocol adjacency changes
- Hardware errors

### Syslog Severity Levels

| Level | Keyword | Description | Example |
| --- | --- | --- | --- |
| 0 | emergencies | System unusable | System crash |
| 1 | alerts | Immediate action needed | Temperature critical |
| 2 | critical | Critical conditions | Memory exhausted |
| 3 | errors | Error conditions | Interface error |
| 4 | warnings | Warning conditions | Config change warning |
| 5 | notifications | Normal but significant | Interface state change |
| 6 | informational | Informational messages | Neighbor adjacency |
| 7 | **debugging** | Debug messages | All messages |

> Setting severity to `debugging` (level 7) captures **all** messages (levels 0–7). Setting to `warnings` (level 4) captures only levels 0–4.
> 

---

### Step 1 – Configure Syslog Server

```
R1(config)#logging 10.0.0.100         ! Send logs to external Syslog server
R1(config)#logging trap debugging     ! Send all severity levels (0-7)
```

> `logging trap` sets the **trap level** — the maximum severity level that will be forwarded to the Syslog server. `debugging` = all messages.
> 

---

### Step 2 – Verify Syslog Configuration

```
R1#show logging
Syslog logging: enabled
Console logging: level debugging
Monitor logging:  level debugging
Buffer logging:   disabled

Trap logging: level debugging, 3 message lines logged
Logging to 10.0.0.100 (udp port 514, audit disabled,
  authentication disabled, encryption disabled, link up),
  2 message lines logged
```

**Key fields explained:**

| Field | Meaning |
| --- | --- |
| `Trap logging: level debugging` | All severity levels sent to Syslog server |
| `Logging to 10.0.0.100` | Confirmed syslog destination |
| `udp port 514` | Standard Syslog UDP port |
| `2 message lines logged` | Number of messages sent so far |

---

### Step 3 – Generate Syslog Events

```
R1(config)#interface f0/1
R1(config-if)#no shutdown
%LINK-5-CHANGED: Interface FastEthernet0/1, changed state to up

R1(config-if)#shutdown
%LINK-5-CHANGED: Interface FastEthernet0/1, changed state to administratively down
```

> The `%LINK-5-CHANGED` message format is: `%FACILITY-SEVERITY-MNEMONIC: Description`
> 

> - **LINK** = facility (which subsystem generated the message)
> 

> - **5** = severity level (notifications)
> 

> - **CHANGED** = mnemonic (short code for the event)
> 

### Step 4 – Verify on Syslog Server

On the Syslog server at 10.0.0.100: Click **Services → SYSLOG** — you should see the interface up/down events with timestamps.

---

## 🧠 Syslog Message Format

```
%LINK-5-CHANGED: Interface FastEthernet0/1, changed state to up
  ^      ^  ^       ^
  |      |  |       Message text
  |      |  Mnemonic (event code)
  |      Severity level (0-7)
  Facility (subsystem)
```

**Common facility codes:**

| Facility | Description |
| --- | --- |
| `LINK` | Interface link state changes |
| `LINEPROTO` | Line protocol state changes |
| `OSPF` | OSPF protocol events |
| `SYS` | System events |
| `SEC` | Security events |
| `DUAL` | EIGRP events |

---

## 🧠 Syslog Logging Destinations

Cisco devices can send syslog messages to multiple destinations simultaneously:

| Destination | Command | Notes |
| --- | --- | --- |
| **Console** | `logging console <level>` | Shown on console terminal |
| **VTY (Monitor)** | `logging monitor <level>` | Shown on Telnet/SSH sessions (`terminal monitor` to enable) |
| **Buffer** | `logging buffered <level>` | Stored in router RAM |
| **External Server** | `logging <ip>`  • `logging trap <level>` | Sent to Syslog server |

---

## 📋 Key Commands Summary

| Command | Purpose |
| --- | --- |
| `snmp-server community <string> ro` | Configure SNMP read-only community |
| `snmp-server community <string> rw` | Configure SNMP read-write community |
| `logging <ip>` | Set external Syslog server IP |
| `logging trap <level>` | Set severity level for Syslog server |
| `logging console <level>` | Set severity level for console logging |
| `logging buffered <level>` | Enable/set buffer logging |
| `show logging` | View logging configuration and status |
| `show snmp` | View SNMP statistics |
| `show snmp community` | View configured SNMP communities |
| `terminal monitor` | Enable syslog on VTY (Telnet/SSH) session |

---

## 🗒️ Key Takeaways

- **SNMP RO** = monitoring only; **SNMP RW** = monitoring + configuration changes. Protect RW strings carefully.
- SNMPv1/v2c community strings travel in **plain text** — use SNMPv3 in production for security.
- Syslog **severity levels 0–7**: lower number = more severe. `debugging` (7) captures everything.
- `logging trap debugging` forwards ALL events to the external server — use a more restrictive level in production (e.g., `warnings`) to reduce noise.
- Syslog messages follow the format: `%FACILITY-SEVERITY-MNEMONIC: description`.
- Syslog uses **UDP port 514** — no delivery guarantee (fire and forget).
- `terminal monitor` is required to see syslog messages during a Telnet/SSH session — they are not shown by default on VTY lines.
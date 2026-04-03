# 37 - Wireless Fundamentals Configuration

Module Number: 37
Status: Completed
Topic: WLC, CAPWAP, Lightweight APs, RADIUS, WPA2, Corporate & Guest WLANs, DHCP Scopes

> **Lab Type:** Exercise + Answer Key | **Devices:** Multilayer Switch, WLC (192.168.10.11), 2 Lightweight APs, RADIUS/DNS Server, Corporate & Guest laptops in Packet Tracer
> 

---

## 🧭 Overview

This final module configures a complete **enterprise wireless infrastructure** using a Cisco Wireless LAN Controller (WLC), Lightweight Access Points (LAPs), and RADIUS authentication:

- **Switch VLAN configuration** for wireless management and client VLANs
- **CAPWAP** — how Lightweight APs communicate with the WLC
- **WLC setup** — RADIUS integration, DHCP scopes, logical interfaces
- **Two WLANs**: Corporate (802.1X/RADIUS) and Guest (PSK)
- **Client association** and end-to-end connectivity verification

---

## 🧠 Wireless Architecture Concepts

### Autonomous vs Lightweight APs

| Type | Management | Config Stored On | Use Case |
| --- | --- | --- | --- |
| **Autonomous AP** | Managed individually | On the AP itself | Small deployments |
| **Lightweight AP (LAP)** | Managed centrally via WLC | On the WLC | Enterprise — hundreds of APs |

### CAPWAP (Control and Provisioning of Wireless Access Points)

Lightweight APs use **CAPWAP tunnels** to communicate with the WLC:

- **Control channel** (UDP 5246) — management traffic between AP and WLC
- **Data channel** (UDP 5247) — client data tunnelled to WLC
- APs use **DNS** to find the WLC: they look up `cisco-capwap-controller` to get the WLC IP
- This is **Zero Touch Provisioning (ZTP)** — APs auto-discover the WLC via DNS

### WLC Interfaces

| Interface | Purpose |
| --- | --- |
| **Management Interface** | WLC's own management IP; AP communication; untagged on trunk |
| **Logical Interface** | Maps a WLAN to a specific VLAN; one per WLAN/VLAN |

---

## 📂 Lab Setup

- Download `37 Wireless Fundamentals Configuration.zip`, extract and open in Packet Tracer.
- WLC is pre-configured with IP `192.168.10.11/24`.
- Two Lightweight APs are plugged into the multilayer switch.
- RADIUS/DNS server at `192.168.11.10`.

**Pre-existing VLANs:**

| VLAN | Name | Subnet | Gateway |
| --- | --- | --- | --- |
| 11 | Server | 192.168.11.0/24 | 192.168.11.1 |
| 21 | Admin | 192.168.21.0/24 | 192.168.21.1 |

---

## 🔑 Part 1 – Switch VLAN Configuration

### Step 1 – Create Required VLANs and SVIs

```
Switch(config)#vlan 10
Switch(config-vlan)#name Management         ! AP management VLAN
Switch(config)#interface vlan 10
Switch(config-if)#ip address 192.168.10.1 255.255.255.0

Switch(config)#vlan 22
Switch(config-vlan)#name Corporate          ! Corporate wireless clients
Switch(config)#interface vlan 22
Switch(config-if)#ip address 192.168.22.1 255.255.255.0

Switch(config)#vlan 23
Switch(config-vlan)#name Guest              ! Guest wireless clients
Switch(config)#interface vlan 23
Switch(config-if)#ip address 192.168.23.1 255.255.255.0
```

**Final VLAN table:**

| VLAN | Name | Subnet | Gateway |
| --- | --- | --- | --- |
| 10 | Management | 192.168.10.0/24 | 192.168.10.1 |
| 11 | Server | 192.168.11.0/24 | 192.168.11.1 |
| 21 | Admin | 192.168.21.0/24 | 192.168.21.1 |
| 22 | Corporate | 192.168.22.0/24 | 192.168.22.1 |
| 23 | Guest | 192.168.23.0/24 | 192.168.23.1 |

---

### Step 2 – Configure DNS A Record for CAPWAP Discovery

On the RADIUS/DNS/Web server GUI (Services → DNS), create:

- **Hostname:** `cisco-capwap-controller`
- **IP Address:** `192.168.10.11`

This allows Lightweight APs to discover the WLC via DNS (Zero Touch Provisioning).

**Verify from Admin laptop:**

```
C:\>nslookup cisco-capwap-controller
Name:    cisco-capwap-controller
Address: 192.168.10.11
```

---

### Step 3 – Configure Trunk Port to WLC (Gi1/0/5)

```
Switch(config)#interface GigabitEthernet1/0/5
Switch(config-if)#switchport trunk encapsulation dot1q
Switch(config-if)#switchport mode trunk
Switch(config-if)#switchport trunk allowed vlan 10,22,23    ! Management + WLAN VLANs
Switch(config-if)#spanning-tree portfast trunk              ! Skip STP delay
```

> 💡 The WLC management VLAN (10) is set as the **native VLAN** on this trunk, so management traffic is untagged.
> 

---

### Step 4 – Configure Access Ports to Lightweight APs (Gi1/0/3–4)

```
Switch(config)#interface range GigabitEthernet1/0/3 - 4
Switch(config-if)#switchport mode access
Switch(config-if)#switchport access vlan 10        ! APs join management VLAN
Switch(config-if)#spanning-tree portfast           ! Fast transition to forwarding
```

> APs only need the management VLAN on their access port — all WLAN traffic travels through the **CAPWAP tunnel** back to the WLC, where it is then trunked appropriately.
> 

---

## 🔑 Part 2 – WLC Configuration (GUI)

Access WLC GUI: `https://cisco-capwap-controller` → Login: `admin / Flackbox1`

### Step 5 – Verify APs Registered

WLC Dashboard → **Wireless** page — both Lightweight APs should show as registered.

---

### Step 6 – Add RADIUS Server

**Security → AAA → RADIUS → Authentication → New**

| Setting | Value |
| --- | --- |
| Server IP | 192.168.11.10 |
| Shared Secret | Flackbox1 |
| Port | 1812 (default) |

---

### Step 7 – Configure DHCP Scopes on WLC

**Controller → Internal DHCP Server → DHCP Scope → New**

**Corporate DHCP Scope:**

| Setting | Value |
| --- | --- |
| Name | Corporate |
| Network | 192.168.22.0 |
| Mask | 255.255.255.0 |
| Start IP | 192.168.22.101 |
| End IP | 192.168.22.254 |
| Default Gateway | 192.168.22.1 |
| DNS Server | 192.168.11.10 |

**Guest DHCP Scope:**

| Setting | Value |
| --- | --- |
| Name | Guest |
| Network | 192.168.23.0 |
| Start IP | 192.168.23.101 |
| End IP | 192.168.23.254 |
| Default Gateway | 192.168.23.1 |
| DNS Server | 192.168.11.10 |

---

### Step 8 – Create WLC Logical Interfaces

**Controller → Interfaces → New**

**Corporate Interface:**

| Setting | Value |
| --- | --- |
| Interface Name | Corporate |
| VLAN ID | 22 |
| IP Address | 192.168.22.11 |
| Gateway | 192.168.22.1 |
| Physical Port | 1 |
| DHCP Server | 192.168.10.11 (WLC management IP) |

**Guest Interface:**

| Setting | Value |
| --- | --- |
| Interface Name | Guest |
| VLAN ID | 23 |
| IP Address | 192.168.23.11 |
| Gateway | 192.168.23.1 |
| Physical Port | 1 |
| DHCP Server | 192.168.10.11 |

---

## 🔑 Part 3 – Create WLANs

### WLAN 1 – Corporate (802.1X / RADIUS Authentication)

**WLANs → Create New**

| Setting | Value |
| --- | --- |
| SSID / Profile Name | Corporate |
| Interface | Corporate |
| Status | Enabled (after security config) |

**Security Tab:**

| Setting | Value |
| --- | --- |
| Layer 2 Security | WPA + WPA2 |
| WPA2 Policy | Enabled |
| Encryption | AES |
| Auth Key Management | 802.1X |

**Security → AAA Servers Tab:**

- Server 1: `192.168.10.11, Port 1812` (RADIUS server)

> Corporate users authenticate with **username/password** via RADIUS. Credentials: `Flackbox / Flackbox2`
> 

---

### WLAN 2 – Guest (WPA2 Pre-Shared Key)

**WLANs → Create New**

| Setting | Value |
| --- | --- |
| SSID / Profile Name | Guest |
| Interface | Guest |
| Status | Enabled (after security config) |

**Security Tab:**

| Setting | Value |
| --- | --- |
| Layer 2 Security | WPA + WPA2 |
| WPA2 Policy | Enabled |
| Encryption | AES |
| Auth Key Management | PSK |
| Pre-Shared Key | Flackbox3 |

> Guest users authenticate with a **pre-shared key** (password) — no individual accounts needed.
> 

---

### ⚠️ Important Packet Tracer Step

After creating both WLANs: **Save the PT file, close Packet Tracer, and reopen it.** Otherwise DHCP may not work correctly for wireless clients.

---

## 🔑 Part 4 – Connect Clients to WLANs

### Connect Corporate1 Laptop

On Corporate1 laptop: **Config → Wireless0**

| Setting | Value |
| --- | --- |
| SSID | Corporate |
| Auth | WPA2 (802.1X) |
| Username | Flackbox |
| Password | Flackbox2 |

### Connect Guest1 Laptop

On Guest1 laptop: **Config → Wireless0**

| Setting | Value |
| --- | --- |
| SSID | Guest |
| Auth | WPA2-PSK |
| Pre-Shared Key | Flackbox3 |

### Verify Connectivity

```
Corporate1> ipconfig
IPv4 Address: 192.168.22.101    ! Got IP from Corporate DHCP scope
Default Gateway: 192.168.22.1

Guest1> ping 192.168.22.101
Reply from 192.168.22.101 ... TTL=127   ! Cross-VLAN ping works via L3 switch
```

---

## 📊 Architecture Summary

```
[Corporate1 Laptop]              [Guest1 Laptop]
  SSID: Corporate                 SSID: Guest
  WPA2 802.1X                     WPA2 PSK
       |                               |
  [Lightweight AP]            [Lightweight AP]
       |   (CAPWAP tunnel)            |
       +----------[WLC 192.168.10.11]-+
                       |
              [Multilayer Switch]
              VLAN 10 (Mgmt)
              VLAN 22 (Corporate)
              VLAN 23 (Guest)
                       |
            [RADIUS/DNS Server]
            192.168.11.10
```

---

## 📋 Key Concepts Summary

| Concept | Details |
| --- | --- |
| **CAPWAP** | Tunnel protocol between Lightweight APs and WLC (UDP 5246/5247) |
| **Zero Touch Provisioning** | APs discover WLC via DNS `cisco-capwap-controller` A record |
| **WLC Management Interface** | WLC's own IP; untagged; APs join this VLAN |
| **WLC Logical Interface** | Per-VLAN interface on WLC; maps WLAN to VLAN |
| **DHCP Scope** | IP pool on WLC for wireless clients per VLAN |
| **WPA2 802.1X** | Enterprise auth — username/password via RADIUS server |
| **WPA2 PSK** | Personal auth — shared key for all users |
| **AP Access Port** | AP switchport = access in management VLAN (CAPWAP carries WLAN data) |
| **WLC Trunk Port** | WLC switchport = trunk carrying management + WLAN VLANs |
| **PortFast trunk** | Enables PortFast on trunk port — used on WLC and AP ports |

---

## 🗒️ Key Takeaways

- **Lightweight APs** are Zero Touch — they auto-discover the WLC via DNS and form CAPWAP tunnels. No per-AP configuration needed.
- APs connect to the switch as **access ports in the management VLAN** — all WLAN traffic is carried inside CAPWAP tunnels back to the WLC.
- The **WLC trunk port** carries management VLAN (native/untagged) plus all WLAN VLANs (tagged).
- Each WLAN needs its own **logical interface** on the WLC mapping to its VLAN.
- **Corporate WLAN** uses **WPA2 + 802.1X** (RADIUS) — users authenticate with individual credentials.
- **Guest WLAN** uses **WPA2 + PSK** — one shared password for all guests.
- DHCP for wireless clients can be provided by the WLC internally or by an external DHCP server.
- `spanning-tree portfast trunk` is used on the WLC port; `spanning-tree portfast` on AP access ports — speeds up port transitions without risking L2 loops.
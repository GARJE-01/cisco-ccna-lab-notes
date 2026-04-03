# 🖥️ Cisco CCNA Lab Notes (200-301)

> **Personal lab notes** based on the **Cisco CCNA Lab Guide v200-301** by **Neil Anderson** (Flackbox).  
> These are structured, hands-on notes covering every lab module — written for active revision, exam prep, and real-world reference.

---

## 📖 About the Source Material

| Detail | Info |
|---|---|
| **Book Title** | Cisco CCNA Lab Guide v200-301 |
| **Author** | Neil Anderson |
| **Website** | [flackbox.com](https://www.flackbox.com) |
| **Availability** | Free to use — available at [flackbox.com/cisco-ccna-lab-guide](https://www.flackbox.com/cisco-ccna-lab-guide) |
| **Simulator Used** | Cisco Packet Tracer (free via [Cisco NetAcad](https://www.netacad.com)) |
| **Exam Target** | Cisco CCNA 200-301 |

The lab guide is provided **free of charge** by Neil Anderson as a companion to his CCNA Gold Bootcamp course. It covers practical, hands-on labs for every major CCNA topic. I highly recommend visiting his site and supporting his work if you find it useful.

> 💡 **Neil Anderson's CCNA Gold Bootcamp** — if you want full video lectures alongside these labs, check out his paid course at [flackbox.com/cisco-ccna-training-course](https://www.flackbox.com/cisco-ccna-training-course). It is one of the highest rated CCNA courses available.

---

## 🗒️ About These Notes

These are my **personal structured notes** taken from each lab module in the guide. They are **not a copy of the original PDF** — they are my own interpretation, written for active study and revision.

Each module note includes:
- 📋 **Overview** — what the lab covers and why it matters
- ⚙️ **Lab Setup** — topology and prerequisites
- 🔑 **Step-by-step configuration** — all commands with explanations
- 📊 **Tables and comparisons** — for quick revision
- ⚠️ **Common mistakes and gotchas**
- 🗒️ **Key Takeaways** — summary of the most important concepts

---

## 🚀 How to Use These Notes

### 1. Clone the Repository
```bash
git clone https://github.com/YOUR_USERNAME/cisco-ccna-lab-notes.git
cd cisco-ccna-lab-notes
```

### 2. Read Online (GitHub)
GitHub renders Markdown natively — just click any module in the index below and it opens in your browser with full formatting, tables, and code blocks.

### 3. Convert to PDF
For offline reading or printing, use one of these tools:
- 🌐 **[Dillinger.io](https://dillinger.io)** — paste any `.md` file for instant browser-based PDF export
- 🖥️ **[Pandoc](https://pandoc.org)** — command line tool for bulk conversion to PDF, EPUB, DOCX
- 📱 **[Calibre](https://calibre-ebook.com)** — bulk convert to mobile/iPad formats (EPUB, MOBI)
```bash
# Convert a single module to PDF using Pandoc
pandoc modules/04-IOS-Operating-System.md -o Module-04.pdf

# Bulk convert all modules to PDF
for f in modules/*.md; do pandoc "$f" -o "${f%.md}.pdf"; done
```

### 4. Use with Packet Tracer
Download the **free lab files** from Neil Anderson's site alongside these notes. Open the `.pkt` file for each module in **Cisco Packet Tracer**, then follow the commands in the corresponding note.

> 🎓 Packet Tracer is free — download it from [Cisco NetAcad](https://www.netacad.com/resources/lab-downloads)

### 5. Study Tips

- **Don't just read — type the commands.** Open Packet Tracer alongside each note and configure everything yourself.
- **Use the Key Takeaways** sections for last-minute revision before the exam.
- **Revisit troubleshooting modules** (13, 18, 25-1) regularly — they build the diagnostic mindset CCNA tests.
- **Cross-reference topics** — e.g. STP (25-1, 25-2) and HSRP (24-1) are deeply linked; study them together.
- **The lab guide is progressive** — earlier modules build foundational skills used in later ones. Follow the order.

---

## 📜 License & Credits

- **Original Lab Guide:** © Neil Anderson / Flackbox — used for personal study purposes only. All credit for the lab content goes to the author.
- **These Notes:** Personal study notes — free to use for non-commercial educational purposes.
- **Not affiliated** with Cisco Systems or Neil Anderson in any official capacity.

If you find this repo useful, please ⭐ star it and consider supporting Neil Anderson by checking out his course at [flackbox.com](https://www.flackbox.com).

---

## 📑 Module Index

> Click any module title to open the full notes.

| # | Module | Topics Covered |
|---|---|---|
| 01 | [🖥️ The IOS Operating System](modules/04-IOS-Operating-System.md) | CLI modes, navigation, configuration management |
| 02 | [🔀 Cisco Device Functions](modules/11-Cisco-Device-Functions.md) | MAC address tables, routing tables, static routes |
| 03 | [📦 The Life of a Packet](modules/12-Life-of-a-Packet.md) | DNS client config, ARP cache |
| 04 | [🔧 The Cisco Troubleshooting Methodology](modules/13-Cisco-Troubleshooting-Methodology.md) | ping, traceroute, bottom-up troubleshooting |
| 05 | [🔩 Cisco Router and Switch Basics](modules/14-Router-Switch-Basics.md) | CDP, speed/duplex, initial config |
| 06 | [🛠️ Cisco Device Management](modules/15-Device-Management.md) | Factory reset, password recovery, IOS upgrade, TFTP |
| 07 | [🗺️ Routing Fundamentals](modules/16-Routing-Fundamentals.md) | Static, summary, default routes, longest prefix match, ECMP |
| 08 | [📡 Dynamic Routing Protocols](modules/17-Dynamic-Routing-Protocols.md) | RIP, OSPF, EIGRP, AD, metrics, floating static, loopback, passive interfaces |
| 09 | [🔍 Connectivity Troubleshooting](modules/18-Connectivity-Troubleshooting.md) | Missing routes, ping/traceroute analysis |
| 10 | [📶 IGP Fundamentals Configuration](modules/19-1-IGP-Fundamentals.md) | RIPv2 & EIGRP config, default route injection, AD comparison |
| 11 | [🟧 OSPF Configuration](modules/20-1-OSPF-Configuration.md) | OSPF cost, reference bandwidth, multi-area, DR/BDR |
| 12 | [🔁 VLAN and Inter-VLAN Routing](modules/22-1-VLAN-InterVLAN-Routing.md) | VTP, trunks, Router-on-a-Stick, L3 switch SVIs |
| 13 | [📡 DHCP Configuration](modules/23-1-DHCP-Configuration.md) | DHCP client/server, ip helper-address |
| 14 | [🛡️ HSRP Configuration](modules/24-1-HSRP-Configuration.md) | Virtual IP/MAC, priority, pre-emption, failover |
| 15 | [🌳 Spanning Tree Troubleshooting](modules/25-1-Spanning-Tree-Troubleshooting.md) | Root bridge election, port roles, suboptimal paths |
| 16 | [🌲 Spanning Tree Configuration](modules/25-2-Spanning-Tree-Configuration.md) | Root primary/secondary, PortFast, BPDU Guard |
| 17 | [🔗 EtherChannel Configuration](modules/26-1-EtherChannel-Configuration.md) | LACP, PAgP, static, Layer 3 EtherChannel |
| 18 | [🔒 Port Security Configuration](modules/27-1-Port-Security.md) | Disable unused ports, static/dynamic/sticky MAC, violation modes |
| 19 | [🛡️ ACL Configuration](modules/28-1-ACL-Configuration.md) | Standard, extended, named ACLs, wildcard masks, placement |
| 20 | [🌐 NAT Configuration](modules/29-1-NAT-Configuration.md) | Static NAT, dynamic NAT, PAT, overload |
| 21 | [🌏 IPv6 Configuration](modules/30-1-IPv6-Configuration.md) | Global unicast, EUI-64, link-local, ipv6 unicast-routing, static routes |
| 22 | [🔐 Cisco Device Security](modules/33-1-Device-Security.md) | Enable secret, SSH, VTY ACL, console, NTP, banners |
| 23 | [📊 Network Device Management](modules/34-Network-Device-Management.md) | SNMP communities, Syslog, severity levels |
| 24 | [📶 Wireless Fundamentals Configuration](modules/37-Wireless-Fundamentals.md) | WLC, CAPWAP, Lightweight APs, RADIUS, WPA2, DHCP scopes |

---

---

## ⭐ Acknowledgements

A huge thank you to **Neil Anderson** for making his CCNA Lab Guide freely available to the networking community. His clear, practical approach to teaching Cisco networking made these notes possible.

- 🌐 Website: [flackbox.com](https://www.flackbox.com)
- 🎓 Course: [Cisco CCNA Gold Bootcamp](https://www.flackbox.com/cisco-ccna-training-course)
- 📄 Free Lab Guide: [flackbox.com/cisco-ccna-lab-guide](https://www.flackbox.com/cisco-ccna-lab-guide)

---

*Good luck with your CCNA! 🚀*

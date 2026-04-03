# 04 - The IOS Operating System

Module Number: 04
Status: Completed
Topic: IOS CLI Navigation, Modes, Configuration Management

> **Lab Type:** Guided Walkthrough | **Device Required:** Single Router (Packet Tracer)
> 

---

## 🧭 Overview

This module introduces the **Cisco IOS (Internetwork Operating System) Command Line Interface (CLI)**. You will learn how to navigate between modes, use context-sensitive help, enter and manage configurations, and save/restore device configurations.

---

## 📂 Lab Setup

- Download `04 The IOS Operating System.zip`, extract and open the `.pkt` file in **Cisco Packet Tracer**.
- Click **Router0 → CLI tab** to access the console.
- Press **Return** to get started.

---

## 🔑 IOS CLI Modes

| Mode | Prompt | Access Command |
| --- | --- | --- |
| User Exec | `Router>` | Default on login |
| Privileged Exec (Enable) | `Router#` | `enable` |
| Global Configuration | `Router(config)#` | `configure terminal` |
| Interface Configuration | `Router(config-if)#` | `interface <type> <id>` |

---

## 📋 Key Commands & Explanations

### Entering & Exiting Modes

```
Router>enable                    ! Enter Privileged Exec mode
Router#disable                   ! Return to User Exec mode
Router#configure terminal        ! Enter Global Config mode (abbrev: conf t)
Router(config)#interface gigabitEthernet 0/0   ! Enter Interface Config mode
Router(config-if)#exit           ! Go back one level
Router(config-if)#end            ! Drop back to Privileged Exec from any level
```

> 💡 `Ctrl-C` also drops back to Privileged Exec from any configuration mode.
> 

---

### Context-Sensitive Help

```
Router>?                         ! List all commands available at current mode
Router#sh?                       ! Show all commands starting with 'sh'
Router#sh ?                      ! Show all sub-commands of 'show' (space before ?)
Router#sh aaa ?                  ! Show sub-commands of 'show aaa'
```

> ⚠️ Context-sensitive help (`?`) may be **disabled in CCNA exam simulator** — you must know commands by heart.
> 

---

### Command Abbreviation & Tab Completion

```
Router>en                        ! Abbreviation for 'enable'
Router#sh aaa us<TAB>            ! Tab completes to 'sh aaa user'
```

- Abbreviation works only when letters **uniquely match one command**.
- Example: `di` is ambiguous (`dir`, `disable`, `disconnect`) → use `disa` for disable.

---

### Rebooting the Device

```
Router#reload                    ! Reboot the device
Proceed with reload? [confirm]   ! Press Enter to confirm
```

> 💡 Console connection lets you see bootup messages — not possible via IP/Telnet/SSH.
> 

> If prompted: `Would you like to enter the initial configuration dialog? [yes/no]: no`
> 

---

### Using 'do' to Run Show Commands in Config Mode

```
R1(config)#do show ip interface brief    ! Run show commands from any config level
```

> Without `do`, show commands fail inside configuration modes.
> 

---

### Keyboard Shortcuts

| Shortcut | Action |
| --- | --- |
| `↑` / `↓` Arrow | Cycle through command history |
| `Ctrl-A` | Move cursor to beginning of line |
| `Ctrl-E` | Move cursor to end of line |
| `Tab` | Auto-complete command |
| `Ctrl-C` | Exit to Privileged Exec mode |
| `Space` | Scroll output one page at a time |
| `Enter` | Scroll output one line at a time |

---

### Output Filtering with Pipe (`|`)

```
R1#show running-config                   ! View full device config
R1#sh run | begin hostname               ! Show config starting from 'hostname'
R1#show run | include interface          ! Show only lines containing 'interface'
R1#show run | exclude interface          ! Show all lines NOT containing 'interface'
```

> ⚠️ Pipe filtering IS **case-sensitive**: `| begin Hostname` ≠ `| begin hostname`
> 

---

## 💾 IOS Configuration Management

### Running vs Startup Configuration

| Config Type | Description | Location |
| --- | --- | --- |
| Running Config | Active config in use right now | RAM (lost on reboot) |
| Startup Config | Config loaded on boot | NVRAM (persistent) |

```
R1#copy run start                        ! Save running config to startup config
R1#show running-config                   ! View current active config
R1#show startup-config                   ! View config that loads on boot
```

> ⚠️ Commands take effect **immediately** but are NOT persistent until saved with `copy run start`.
> 

---

### Backup & Restore

```
RouterX#copy run flash:                  ! Backup config to flash memory (on device)
Destination filename [running-config]? config-backup

RouterX#copy run tftp                    ! Backup config to external TFTP server
Address or name of remote host []? 10.10.10.10
Destination filename [RouterX-confg]?
```

> 💡 Backing up to the device itself is **not recommended** — use an external TFTP server in production.
> 

---

### Hostname Configuration

```
Router(config)#hostname R1              ! Set device hostname
R1(config)#                             ! Prompt updates immediately
```

---

## ⚠️ Common Errors & Fixes

| Error Message | Cause | Fix |
| --- | --- | --- |
| `% Invalid input detected at '^' marker` | Wrong mode or typo | Check mode & spelling |
| `% Ambiguous command` | Too few letters entered | Enter more characters |
| `% Incomplete command` | Missing required argument | Add missing argument or use `?` |
| No output after a valid command | Feature not configured | Not an error — nothing to show |

---

## 🗒️ Key Takeaways

- The IOS CLI has **hierarchical modes** — commands are mode-specific.
- Use `?` extensively to explore available commands.
- **Abbreviations** work as long as they are unambiguous.
- Changes are **immediate but not saved** until you run `copy run start`.
- Use `do` to run `show` commands from inside configuration modes.
- Pipe operators (`|`) help filter long output — but are **case-sensitive**.
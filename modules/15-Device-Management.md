# 15 - Cisco Device Management

Module Number: 15
Status: Completed
Topic: Factory Reset, Password Recovery, Config Backup, IOS Upgrade

> **Lab Type:** Exercise + Answer Key | **Devices:** 1 Router (R1), 1 Switch (SW1), 1 TFTP Server (10.10.10.10) in Packet Tracer
> 

---

## 🧭 Overview

This module covers essential device management tasks every network engineer must know:

- **Factory Reset** — wipe a device back to defaults
- **Password Recovery** — regain access to a locked device
- **Configuration Backup & Restore** — to Flash and TFTP server
- **IOS System Image Backup & Recovery** — including rommon recovery
- **IOS Image Upgrade** — upgrading a switch's IOS via TFTP

---

## 📂 Lab Setup

- Download `15 Cisco Device Management.zip`, extract and open in Packet Tracer.
- Topology: R1 and SW1 connected to a generic server at `10.10.10.10` (TFTP server built-in).

---

## 🔑 Part 1 – Factory Reset

### What is a Factory Reset?

A factory reset erases the **startup configuration (NVRAM)**, so the device boots with no configuration. The running config in RAM is not touched until reload.

### Steps

**Step 1 – Back up your config first!**

```
R1#sh run           ! Copy and paste output to Notepad before resetting
```

**Step 2 – Erase NVRAM and reload**

```
R1#write erase
Erasing the nvram filesystem will remove all configuration files! Continue? [confirm]
[OK]
R1#reload
Proceed with reload? [confirm]
```

**Step 3 – After reboot, exit Setup Wizard**

```
Continue with configuration dialog? [yes/no]: no
```

**Step 4 – Confirm device is blank**

```
Router#show run         ! Shows default empty config
Router#show start       ! 'startup-config is not present'
```

**Step 5 – Restore config by pasting it back in**

```
Router#configure terminal
! Paste the previously copied config here line by line
R1#copy run start       ! Save restored config
```

> ⚠️ Filenames in IOS are **case sensitive** — `Backup-1` ≠ `backup-1`.
> 

---

## 🔑 Part 2 – Password Recovery

### Concept

If you forget the enable secret, you can recover access by booting the router with a modified **configuration register** that tells it to **ignore the startup-config**. This bypasses the password, letting you reconfigure it.

### Configuration Register Values

| Value | Behaviour |
| --- | --- |
| `0x2102` | Normal boot (loads startup-config) — **default** |
| `0x2120` | Boot into **rommon** mode |
| `0x2142` | Boot **ignoring startup-config** (password recovery) |

---

### Password Recovery Steps

**Step 1 – Set enable secret and save**

```
R1(config)#enable secret Flackbox1
R1(config)#do copy run start
```

**Step 2 – Configure router to boot into rommon, then reload**

```
R1(config)#config-register 0x2120
R1(config)#end
R1#reload
```

**Step 3 – In rommon, set register to ignore startup-config**

```
rommon 1 > confreg 0x2142
rommon 2 > reset
```

**Step 4 – Router boots without startup-config — exit the wizard**

```
Continue with configuration dialog? [yes/no]: no
```

**Step 5 – Verify running config is empty, startup-config still intact**

```
Router#sh run     ! Empty — startup-config was bypassed
Router#sh start   ! Still has original config including hashed enable secret
```

**Step 6 – ⚠️ CRITICAL: Copy startup-config to running-config (do NOT skip!)**

```
Router#copy start run   ! Loads full config without the password applying
```

> Skipping this step and saving would **factory reset** the router!
> 

**Step 7 – Bring up the interface (it will be shutdown by default)**

```
R1#show ip interface brief    ! Gig0/0 shows 'administratively down'
R1(config)#interface g0/0
R1(config-if)#no shutdown
```

> 💡 Router interfaces are **shutdown by default**. `no shutdown` is not saved in startup-config explicitly, so after copy start run, interfaces revert to shutdown state.
> 

**Step 8 – Remove the enable secret**

```
R1(config)#no enable secret
```

**Step 9 – Restore normal boot register and save**

```
R1(config)#config-register 0x2102
R1(config)#end
R1#copy run start
```

**Step 10 – Reload and verify**

```
R1#reload
R1#show run       ! Should have full expected config, no enable secret
```

---

## 🔑 Part 3 – Configuration Backup

### Backup Running Config to Flash (on-device)

```
R1#copy run flash
Destination filename [running-config]? Backup-1
[OK]

R1#show flash     ! Verify Backup-1 is listed
```

> ⚠️ Backing up to the same device is **not ideal** — if flash is corrupted, the backup is lost too.
> 

---

### Backup Startup Config to TFTP Server (recommended)

```
R1#copy start tftp
Address or name of remote host []? 10.10.10.10
Destination filename [R1-confg]? Backup-2
[OK - 698 bytes]
```

---

### show flash – Verify Backups

```
R1#show flash
File  Length    Name/status
  5   728       Backup-1
  3   33591768  c2900-universalk9-mz.SPA.151-4.M4.bin
  2   28282     sigdef-category.xml
  1   227537    sigdef-default.xml
```

---

## 🔑 Part 4 – IOS System Image Backup & Recovery

### Backup IOS Image to TFTP

```
R1#show flash                          ! Note the IOS filename
R1#copy flash tftp
Source filename []? c2900-universalk9-mz.SPA.151-4.M4.bin
Address or name of remote host []? 10.10.10.10
Destination filename []?               ! Press Enter to keep same name
[OK - 33591768 bytes]
```

---

### Delete IOS Image (simulate failure)

```
R1#delete flash:c2900-universalk9-mz.SPA.151-4.M4.bin
Delete flash:/c2900-universalk9-mz.SPA.151-4.M4.bin? [confirm]
R1#reload
! Router fails to boot and drops into rommon:
rommon 1 >
```

---

### Recover IOS via TFTP from rommon

```
rommon 1 > tftpdnld             ! Shows required variables

rommon 2 > IP_ADDRESS=10.10.10.1
rommon 3 > IP_SUBNET_MASK=255.255.255.0
rommon 4 > DEFAULT_GATEWAY=10.10.10.1
rommon 5 > TFTP_SERVER=10.10.10.10
rommon 6 > TFTP_FILE=c2900-universalk9-mz.SPA.151-4.M4.bin
rommon 7 > tftpdnld

WARNING: all existing data in all partitions on flash will be lost!
Do you wish to continue? y/n: [n]: y

rommon 8 > reset                ! Reload after download completes
```

> ⚠️ All variable names in rommon must be in **ALL CAPITAL LETTERS**.
> 

> 💡 The `tftpdnld` command erases flash before writing — this is a **disaster recovery** operation only.
> 

---

## 🔑 Part 5 – IOS Image Upgrade (Switch)

### Step 1 – Verify Current IOS Version

```
SW1#show version
Cisco IOS Software, C2960 Software (C2960-LANBASE-M), Version 12.2(25)FX
```

### Step 2 – Copy New IOS from TFTP to Flash

```
SW1#copy tftp flash
Address or name of remote host []? 10.10.10.10
Source filename []? c2960-lanbasek9-mz.150-2.SE4.bin
Destination filename []?           ! Press Enter
[OK - 4670455 bytes]
```

### Step 3 – Verify Both Images in Flash

```
SW1#show flash
  1  c2960-lanbase-mz.122-25.FX.bin       ! Old image
  3  c2960-lanbasek9-mz.150-2.SE4.bin     ! New image
  2  config.text
```

### Step 4 – Set Boot System to New Image

```
SW1(config)#boot system c2960-lanbasek9-mz.150-2.SE4.bin
SW1(config)#end
SW1#copy run start
```

### Step 5 – Reload and Verify

```
SW1#reload
SW1#show version
Cisco IOS Software, C2960 Software (C2960-LANBASEK9-M), Version 15.0(2)SE4
```

---

## 📋 Key Commands Summary

| Command | Purpose |
| --- | --- |
| `write erase` | Erase startup-config (factory reset prep) |
| `reload` | Reboot the device |
| `show flash` | List files stored in flash memory |
| `copy run flash` | Backup running-config to flash |
| `copy start tftp` | Backup startup-config to TFTP server |
| `copy flash tftp` | Backup IOS image to TFTP server |
| `copy tftp flash` | Download file from TFTP to flash |
| `delete flash:<file>` | Delete a file from flash |
| `config-register <value>` | Set boot behaviour register |
| `confreg <value>` | Set config register from rommon |
| `tftpdnld` | Recover IOS image via TFTP from rommon |
| `boot system <filename>` | Specify which IOS image to boot from |
| `show version` | View IOS version and hardware info |
| `enable secret <pw>` | Set encrypted enable password |
| `no enable secret` | Remove enable secret |
| `copy start run` | Load startup-config into running-config |

---

## 🗒️ Key Takeaways

- **Factory reset** = `write erase` + `reload` — wipes NVRAM only.
- **Password recovery** uses `config-register 0x2142` to boot ignoring startup-config — always restore startup-config to running-config before saving, or you'll factory reset the device.
- Router interfaces are **shutdown by default** — `no shutdown` must be explicitly applied after password recovery.
- Backup configs to an **external TFTP server**, not just to flash — flash can fail.
- rommon variables must be in **ALL CAPS** — `IP_ADDRESS`, `TFTP_SERVER`, etc.
- For IOS upgrades, copy the new image to flash first, set `boot system`, save, then reload — keep the old image until you confirm the new one boots correctly.
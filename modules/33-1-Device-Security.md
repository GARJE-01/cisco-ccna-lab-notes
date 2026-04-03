# 33-1 - Cisco Device Security Configuration

Module Number 33-1
Status Completed
Topic Enable Secret, Password Encryption, TelnetSSH, Login Banners, NTP, Console Security

 Lab Type Exercise + Answer Key  Devices R1, SW2, PC1, PC2, NTP Server in Packet Tracer
 

---

## 🧭 Overview

This module secures administrative access to a Cisco router through multiple layers

- Enable password vs enable secret — and why secret always wins
- Password encryption — protecting passwords in config output
- Telnet security — ACL restriction, login banners, vty passwords
- SSH configuration — domain name, RSA key, SSHv2 enforcement
- Local user authentication — usernamepassword database
- Console security — password-protecting physical access
- NTP — synchronising router time to a central server
- Switch management IP — VLAN 1 SVI and default gateway

---

## 📂 Lab Setup

- Download `33-1 Cisco Device Security Configuration.zip`, extract and open in Packet Tracer.
- R1 is connected to PC1 (10.0.0.10) and PC2 (different subnet).
- NTP server at `10.0.1.100`.

---

## 🔑 Part 1 – Secure Privileged Exec Mode

### Enable Password vs Enable Secret

 Command  Encryption  Priority 
 ---  ---  --- 
 `enable password pw`  Plaintext in config (or weak type 7)  Lower — overridden by secret 
 `enable secret pw`  MD5 hashed (type 5) — always  Higher — always takes precedence 

 ⚠️ When both are configured, enable secret always wins. The enable password is completely ignored.
 

### Step 1 – Set Enable Password

```
R1(config)#enable password Flackbox2
```

### Step 2 – Set Enable Secret (Supersedes Password)

```
R1(config)#enable secret Flackbox1
```

 After setting enable secret, `Flackbox2` no longer works — only `Flackbox1` grants privileged access.
 

### Step 3 – View Config — Passwords in Plain Text

```
R1#show run
enable secret 5 $1$mERr$J2XZHMOgpVVXdLjC9lYtE1   ! Hashed (MD5)
enable password Flackbox2                           ! Plain text — insecure!
```

### Step 4 – Encrypt All Passwords

```
R1(config)#service password-encryption
```

After encryption

```
enable secret 5 $1$mERr$J2XZHMOgpVVXdLjC9lYtE1    ! Still MD5
enable password 7 0807404F0A1207180A59             ! Now type 7 encrypted
```

 💡 `service password-encryption` uses type 7 (weak, reversible) encryption for most passwords. The enable secret uses type 5 (MD5) which is much stronger. Always use `enable secret`, never `enable password` alone.
 

---

## 🔑 Part 2 – Secure Remote Telnet Access

### Step 5 – Configure VTY Lines (Logging + Timeout)

```
R1(config)#line console 0
R1(config-line)#logging synchronous     ! Prevent log messages interrupting typing
R1(config-line)#exec-timeout 15         ! Log out after 15 min inactivity

R1(config)#line vty 0 15
R1(config-line)#logging synchronous
R1(config-line)#exec-timeout 15
```

 `line vty 0 15` covers all 16 virtual terminal lines (TelnetSSH sessions).
 

---

### Step 6 – Restrict Telnet to Specific Host (ACL)

```
R1(config)#access-list 1 permit host 10.0.0.10    ! Only allow PC1

R1(config)#line vty 0 15
R1(config-line)#login                              ! Require password
R1(config-line)#password Flackbox3                ! VTY password
R1(config-line)#access-class 1 in                 ! Apply ACL to incoming connections
```

 `access-class` on VTY lines applies the ACL to TelnetSSH source addresses — not to routed traffic.
 

---

### Step 7 – Configure Login Banner

```
R1(config)#banner login 
Authorised users only

```

 The banner appears before the login prompt — used as a legal warning. The character after `banner login` is the delimiter (here ``).
 

---

### Step 8 – Verify Telnet Works from PC1, Blocked from PC2

```
! From PC1 (10.0.0.10)
Ctelnet 10.0.0.1
Authorised users only
Password Flackbox3
R1

! From PC2 (different IP)
Ctelnet 10.0.0.1
% Connection refused by remote host    ! ACL blocks it
```

---

### Step 9 – Add Local UsernamePassword Authentication

```
R1(config)#username admin secret Flackbox4    ! Create local user

R1(config)#line vty 0 15
R1(config-line)#login local                   ! Use local user database instead of vty password
```

 `login` = use vty line password. `login local` = use usernamepassword from local database.
 

---

## 🔑 Part 3 – Configure SSH

### SSH Requirements

1. Hostname must be set (not default Router)
2. Domain name must be configured
3. RSA key pair must be generated (min 768 bits for SSHv2)

### Step 10 – Configure Domain Name and Generate RSA Key

```
R1(config)#ip domain-name flackbox.com
R1(config)#crypto key generate rsa
How many bits in the modulus [512] 768
% Generating 768 bit RSA keys ... [OK]
```

### Step 11 – Verify SSH from PC1

```
Cssh -l admin 10.0.0.1
Password Flackbox4
R1
```

 SSH access is controlled by the same VTY ACL as Telnet — PC2 is still blocked.
 

### Step 12 – Restrict to SSHv2 Only (Disable Telnet)

```
R1(config)#line vty 0 15
R1(config-line)#transport input ssh    ! Only allow SSH, block Telnet

R1(config)#ip ssh version 2            ! Require SSHv2 specifically
```

Verify

```
Ctelnet 10.0.0.1
[Connection to 10.0.0.1 closed by remote host]    ! Telnet blocked

Cssh -l admin 10.0.0.1
Password Flackbox4
R1                                               ! SSH works
```

---

## 🔑 Part 4 – Secure Console Access

### Console vs VTY Lines

 Line  Purpose  Access Method 
 ---  ---  --- 
 `line console 0`  Physical console port  Console cable 
 `line vty 0 15`  Virtual terminal lines  Telnet  SSH 

 Console access is not controlled by VTY config or ACLs — it must be secured separately.
 

### Configure Console Password

```
R1(config)#line console 0
R1(config-line)#login                  ! Require password (not username)
R1(config-line)#password Flackbox5    ! Set the console password
```

Verify on console

```
Authorised users only
User Access Verification
Password Flackbox5
R1enable
Password Flackbox1
R1#
```

---

## 🔑 Part 5 – NTP (Network Time Protocol)

### Why NTP Matters

Accurate time is critical for

- Syslog timestamps (correlating events)
- Certificate validity
- Debugging (matching events across devices)

### Configure NTP Client

```
R1(config)#clock timezone PST -8        ! Set timezone Pacific Standard Time = UTC-8
R1(config)#ntp server 10.0.1.100        ! Synchronise with NTP server
```

### Verify NTP Synchronisation

```
R1#show clock
161936.51 PST Mon Oct 2 2017

R1#show ntp status
Clock is synchronized, stratum 2, reference is 10.0.1.100
nominal freq is 250.0000 Hz
clock offset is 0.00 msec, root delay is 0.00 msec
```

 Stratum indicates NTP hierarchy level — stratum 1 = directly connected to time source, stratum 2 = one hop away, etc.
 

---

## 🔑 Part 6 – Switch Management IP

```
SW2(config)#interface vlan 1
SW2(config-if)#ip address 10.0.1.50 255.255.255.0
SW2(config-if)#no shutdown
SW2(config-if)#exit
SW2(config)#ip default-gateway 10.0.1.1    ! Required for switch to reach other subnets
```

---

## 📊 Security Summary Table

 What  Command  Where 
 ---  ---  --- 
 Privileged access (hashed)  `enable secret pw`  Global config 
 Encrypt all passwords  `service password-encryption`  Global config 
 Console password  `password pw`  • `login`  `line console 0` 
 VTY password  `password pw`  • `login`  `line vty 0 15` 
 Local user auth  `username u secret pw`  • `login local`  Global + VTY 
 ACL restrict VTY  `access-class acl in`  `line vty 0 15` 
 SSH only  `transport input ssh`  `line vty 0 15` 
 SSHv2 enforce  `ip ssh version 2`  Global config 
 Login banner  `banner login text`  Global config 
 Inactivity timeout  `exec-timeout mins`  Line config 
 Sync logging  `logging synchronous`  Line config 
 NTP sync  `ntp server ip`  Global config 
 Timezone  `clock timezone name offset`  Global config 

---

## 📋 Key Commands Summary

 Command  Purpose 
 ---  --- 
 `enable password pw`  Set enable password (plain text — avoid) 
 `enable secret pw`  Set enable secret (MD5 hashed — use this) 
 `service password-encryption`  Encrypt all plain text passwords (type 7) 
 `line vty 0 15`  Enter VTY line config (TelnetSSH) 
 `line console 0`  Enter console line config 
 `login`  Require line password for login 
 `login local`  Require local usernamepassword for login 
 `password pw`  Set line password 
 `username u secret pw`  Create local user with hashed password 
 `access-class acl in`  Apply ACL to VTY incoming connections 
 `exec-timeout mins`  Set idle timeout (0 = never) 
 `logging synchronous`  Prevent log messages disrupting CLI input 
 `banner login text`  Show message before login prompt 
 `ip domain-name d`  Set domain for SSH key generation 
 `crypto key generate rsa`  Generate RSA key pair for SSH 
 `ip ssh version 2`  Enforce SSHv2 only 
 `transport input ssh`  Allow only SSH on VTY lines (block Telnet) 
 `clock timezone n offset`  Set timezone and UTC offset 
 `ntp server ip`  Set NTP server for time sync 
 `show clock`  Display current time 
 `show ntp status`  Verify NTP synchronisation status 

---

## 🗒️ Key Takeaways

- Enable secret always beats enable password — when both exist, password is ignored. Use only `enable secret`.
- `service password-encryption` encrypts passwords with weak type 7 — it's better than nothing but type 5 (secret) is far stronger.
- Secure VTY lines with password + ACL + banner + timeout + SSH only.
- `login` uses the line password; `login local` uses the local usernamepassword database.
- SSH requires hostname set, domain name, RSA key ≥ 768 bits, and `ip ssh version 2`.
- Console is NOT secured by VTY config — must configure `line console 0` separately.
- NTP stratum 2 = one hop from atomic clock source — sufficient for all enterprise needs.
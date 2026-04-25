# 🏥 Aurora Healthcare Group — Network Design & Security Implementation

> **Module:** SEM2 - Networking: Wireless and Routing Concepts - Y2  
> **Institution:** SETU  
> **Instructor:** Keara Barrett  
> **Student:** Ihor Maslovskyi — C00325691  
> **Submission Date:** 2 April 2026  

---

## 📋 Project Overview

This project delivers a **secure, reliable, and scalable LAN-based internetworking solution** for Aurora Healthcare Group — a growing Irish private healthcare provider. The solution connects a Central Campus in **Dublin** to two regional branch clinics in **Limerick** and **Sligo**, with a strong focus on Layer 2 security hardening, redundant routing, and secure remote administration.

---

## 🗺️ Network Topology

```
                        ┌─────────────────────────────┐
                        │     DUBLIN (Central Campus)  │
                        │        Cisco 3560 MLS        │
                        │         (Main_MLS)           │
                        │  • Inter-VLAN Routing (SVI)  │
                        │  • DHCP for VLAN 100 & 200   │
                        │  • SSH Remote Admin          │
                        │  • STP Root Bridge           │
                        └──────────────┬───────────────┘
                                       │
                   ┌───────────────────┼───────────────────┐
                   │                                       │
        ┌──────────▼────────┐                   ┌──────────▼────────┐
        │     LIMERICK       │                   │       SLIGO        │
        │  Cisco ISR 4331   │◄──── Serial ──────►│  Cisco ISR 4331   │
        │  (R1_Limerick)    │   (Failover Link)  │  (R2_Sligo)       │
        │  • Local DHCP     │                    │  • Local DHCP     │
        │  • Flat LAN       │                    │  • Flat LAN       │
        └───────────────────┘                    └───────────────────┘
```

---

## 📡 IP Addressing Scheme

| Site | Device | Network | Subnet | Hosts |
|------|--------|---------|--------|-------|
| Dublin | Main_MLS — VLAN 100 (Clinical) | `10.10.100.0/25` | 255.255.255.128 | Up to 126 |
| Dublin | Main_MLS — VLAN 200 (Admin) | `10.10.200.0/27` | 255.255.255.224 | Up to 30 |
| Limerick | R1_Limerick | `172.16.10.0/27` | 255.255.255.224 | Up to 30 |
| Sligo | R2_Sligo | `172.16.20.0/27` | 255.255.255.224 | Up to 30 |

---

## 🔀 VLAN Design (Dublin Central Campus)

| VLAN ID | Name | Purpose | Ports |
|---------|------|---------|-------|
| **100** | Clinical | Clinical staff — 110 users | Access ports on SW1/SW2/SW3 |
| **200** | Admin | Administrative staff — 25 users | Access ports on SW1/SW2/SW3 |
| **67** | BlackHole | All unused ports — non-routable | Fa0/5–Fa0/24, Gig0/1, Gig0/2 on MLS |

> Inter-VLAN routing is handled by **SVIs (Switched Virtual Interfaces)** on Main_MLS — providing wire-speed hardware routing via ASIC, unlike Router-on-a-Stick which creates a single-port bottleneck.

---

## 🔒 Layer 2 Security Implementation

### 1. Port Security — CAM Table Overflow & VLAN Hopping
- Maximum **2 sticky MAC addresses** per access port
- Violation mode: **restrict** (drops frames, logs alert, port stays up)
- DTP disabled (`switchport nonegotiate`) on all trunk ports
- All access ports hardcoded to access mode — prevents VLAN hopping

```
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 2
SW1(config-if)# switchport port-security mac-address sticky
SW1(config-if)# switchport port-security violation restrict
SW1(config-if)# switchport nonegotiate
```

---

### 2. BPDU Guard — STP Root Bridge Hijack Attack
- **BPDU Guard** enabled on all access ports
- Port enters **err-disable** state immediately upon receiving any BPDU
- Main_MLS hardcoded as Root Bridge with priority **24576**

```
SW1(config-if)# spanning-tree bpduguard enable
Main_MLS(config)# spanning-tree vlan 100 priority 24576
Main_MLS(config)# spanning-tree vlan 200 priority 24576
```

---

### 3. DHCP Snooping — Rogue DHCP Server Attack
- DHCP Snooping enabled globally on VLAN 100 and 200
- **Trusted ports:** trunk uplinks to Main_MLS only
- **Untrusted ports:** all end-user access ports (DHCP Offers dropped)

```
SW1(config)# ip dhcp snooping
SW1(config)# ip dhcp snooping vlan 100
SW1(config)# ip dhcp snooping vlan 200
SW1(config)# no ip dhcp snooping information option
SW1(config-if)# ip dhcp snooping trust   ! On trunk port to MLS only
```

---

### 4. Dynamic ARP Inspection (DAI) — ARP Spoofing / MITM
- DAI validates all ARP packets against the DHCP Snooping binding table
- Invalid MAC-to-IP mappings are silently discarded
- Prevents Man-in-the-Middle attacks via ARP poisoning

```
SW1(config)# ip arp inspection vlan 100
SW1(config)# ip arp inspection vlan 200
```

---

### 5. Blackhole VLAN 67 — Physical Security
- All unused ports assigned to non-routable VLAN 67
- All unused ports administratively shut down
- Prevents unauthorised access via physical wall jacks

```
SW1(config-if-range)# switchport access vlan 67
SW1(config-if-range)# shutdown
```

---

## 🔐 Secure Remote Administration (SSH)

- **SSH v2** configured on Main_MLS — Telnet explicitly blocked
- Domain: `aurora.ie` | RSA key size: **1024-bit**
- Two RBAC accounts:
  - `admin` — Privilege Level **15** (full access)
  - `labuser` — Privilege Level **5** (restricted, view only)

```
Main_MLS(config)# ip domain-name aurora.ie
Main_MLS(config)# crypto key generate rsa modulus 1024
Main_MLS(config)# ip ssh version 2
Main_MLS(config)# username admin privilege 15 secret <password>
Main_MLS(config)# username labuser privilege 5 secret <password>
Main_MLS(config)# line vty 0 4
Main_MLS(config-line)# transport input ssh
Main_MLS(config-line)# login local
```

---

## 🛣️ Static & Floating Routing

Full mesh reachability achieved via static routes. Floating static routes provide automatic failover.

| Route Type | Administrative Distance | Link Used | Status |
|------------|------------------------|-----------|--------|
| Primary static route | **1** | GigabitEthernet (primary WAN) | Active by default |
| Floating static route | **50** | Serial link (backup WAN) | Activates on primary failure |

```
! Primary route (AD=1)
R2_Sligo(config)# ip route 10.10.100.0 255.255.255.128 <next-hop> 1

! Floating route (AD=50) - activates when primary fails
R2_Sligo(config)# ip route 10.10.100.0 255.255.255.128 <serial-next-hop> 50
```

**Failover test:**
```
R2_Sligo(config-if)# shutdown   ! Simulate WAN failure on Gig0/0/0
R2_Sligo# show ip route         ! Floating route now appears in table
```

---

## 🛡️ Attack Mitigation Summary

| Attack | Defence Implemented |
|--------|---------------------|
| CAM Table Overflow | Port Security — max 2 sticky MACs, restrict mode |
| VLAN Hopping | `switchport nonegotiate`, access mode hardcoded |
| STP Root Bridge Hijack | BPDU Guard + Main_MLS priority 24576 |
| Rogue DHCP Server | DHCP Snooping — trusted trunk ports only |
| ARP Spoofing / MITM | Dynamic ARP Inspection (DAI) |
| Unauthorised Physical Access | Blackhole VLAN 67 + shutdown on unused ports |
| Telnet Sniffing / Brute Force | SSH v2, RBAC, Telnet blocked |
| WAN Link Failure | Floating Static Route (AD=50) via Serial link |

---

## ✅ Verification Commands Quick Reference

```bash
# VLAN & IP
Main_MLS# show vlan brief
Main_MLS# show ip interface brief

# DHCP
Main_MLS# show ip dhcp binding
Main_MLS# show ip dhcp pool

# Routing
Main_MLS# show ip route static
R2_Sligo# show ip route

# Port Security
SW1# show port-security
SW1# show port-security interface fa0/1

# STP & BPDU Guard
SW1# show spanning-tree
SW1# show interfaces status err-disabled
Main_MLS# show spanning-tree vlan 100

# DHCP Snooping & DAI
SW1# show ip dhcp snooping
SW1# show ip dhcp snooping binding
SW1# show ip arp inspection statistics

# SSH
Main_MLS# show ip ssh
Main_MLS# show ssh
```

---

## 📐 Design Decisions

**Why SVI over Router-on-a-Stick?**  
SVIs perform inter-VLAN routing at the hardware ASIC level (wire-speed). Router-on-a-Stick creates a bottleneck on a single physical port — unsuitable for 110+ clinical users.

**Why decentralised DHCP at branches?**  
During WAN isolation, local branch users still receive IP addresses from their local edge router. This ensures healthcare operations continue without dependency on Dublin.

**Why a looped triangular topology at Central Campus?**  
The loop is intentional — it activates STP which blocks one redundant path to prevent broadcast storms while keeping it available as an automatic failover.

---

*Project built in Cisco Packet Tracer | SETU — South East Technological University*

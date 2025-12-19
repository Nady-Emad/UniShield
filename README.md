# 🌐 UniShield: Next-Gen Campus Network Infrastructure

![UniShield Banner](https://img.shields.io/badge/Status-Active-success?style=for-the-badge) ![Cisco Packet Tracer](https://img.shields.io/badge/Platform-Cisco%20Packet%20Tracer-blue?style=for-the-badge&logo=cisco) ![Security Level](https://img.shields.io/badge/Security-High-red?style=for-the-badge)

> **"Secure. Segmented. Smart."**  
> An enterprise-grade network architecture designed for modern universities, featuring advanced VLAN segmentation, centralized monitoring, and robust security policies.

---

## 🚀 Project Overview

**UniShield** transforms a traditional flat network into a highly secure, segmented infrastructure. By isolating traffic into dedicated VLANs (Faculty, Students, Admin, Guest), we ensure optimal performance and ironclad security. The entire system is monitored in real-time via a centralized Network Controller Dashboard.

### ✨ Key Features

- **🛡️ VLAN Segmentation:** logical isolation for Faculty, Students, Admins, and Guests.
- **🔒 Guest Isolation:** Dedicated Wi-Fi with internet-only access (Zero Trust for internal resources).
- **⚡ Router-on-a-Stick:** Efficient inter-VLAN routing with Gigabit speeds.
- **📊 Centralized Dashboard:** Real-time health monitoring of all network devices using Cisco Network Controller.
- **🌍 Internet Ready:** Full NAT/PAT configuration for seamless WAN connectivity.

---

## 🗺️ Network Topology Map

| **Zone** | **VLAN ID** | **Subnet** | **Color Code** | **Access Policy** |
| :--- | :---: | :--- | :--- | :--- |
| 🎓 **Faculty** | `10` | `192.168.10.0/24` | 🟣 Purple | Full Internal Access + Internet |
| 📚 **Students** | `20` | `192.168.20.0/24` | 🟢 Green | Restricted Internal Access + Internet |
| 🛠️ **Admin** | `30` | `192.168.30.0/24` | 🟡 Yellow | **Management Access** (Controller & Switches) |
| ☕ **Guest** | `99` | `192.168.99.0/24` | 🔴 Red | **Internet ONLY** (Isolated via ACL) |

> *All zones are routed via the high-performance **R-MAIN Core Router**.*

---

## 🛠️ Full Configuration Guide

Below are the complete configuration scripts for every device in the network. You can copy and paste these commands directly into the CLI.

### 1️⃣ Core Router (R-MAIN)
> **Role:** Inter-VLAN Routing, DHCP Server, NAT, Security ACLs.

```bash
enable
configure terminal
hostname R-MAIN
no ip domain-lookup

! --- Inter-VLAN Routing (ROAS) ---
interface gigabitEthernet 0/0.10
 description Faculty VLAN 10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 ip nat inside
 no shutdown
exit

interface gigabitEthernet 0/0.20
 description Students VLAN 20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 ip nat inside
 no shutdown
exit

interface gigabitEthernet 0/0.30
 description Admin VLAN 30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
 ip nat inside
 no shutdown
exit

interface gigabitEthernet 0/0.99
 description Guest Wi-Fi VLAN 99
 encapsulation dot1Q 99
 ip address 192.168.99.1 255.255.255.0
 ip nat inside
 no shutdown
exit

interface gigabitEthernet 0/0
 no shutdown
exit

! --- WAN Interface ---
interface gigabitEthernet 0/1
 description To ISP
 ip address 203.0.113.2 255.255.255.0
 ip nat outside
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 203.0.113.1

! --- DHCP Pools ---
ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp excluded-address 192.168.20.1 192.168.20.10
ip dhcp excluded-address 192.168.30.1 192.168.30.10
ip dhcp excluded-address 192.168.99.1 192.168.99.10

ip dhcp pool FACULTY-POOL
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
exit

ip dhcp pool STUDENTS-POOL
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 8.8.8.8
exit

ip dhcp pool ADMIN-POOL
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1
 dns-server 8.8.8.8
exit

ip dhcp pool GUEST-POOL
 network 192.168.99.0 255.255.255.0
 default-router 192.168.99.1
 dns-server 8.8.8.8
exit

! --- Security ACLs ---
! Guest Isolation: Block Guest from Internal, Allow Internet
access-list 100 deny ip 192.168.99.0 0.0.0.255 192.168.10.0 0.0.0.255
access-list 100 deny ip 192.168.99.0 0.0.0.255 192.168.20.0 0.0.0.255
access-list 100 deny ip 192.168.99.0 0.0.0.255 192.168.30.0 0.0.0.255
access-list 100 permit ip 192.168.99.0 0.0.0.255 any
interface gigabitEthernet 0/0.99
 ip access-group 100 out
exit

! NAT ACL
access-list 1 permit 192.168.0.0 0.0.255.255
ip nat inside source list 1 interface gigabitEthernet 0/1 overload

! --- SNMP for Controller ---
snmp-server community public ro
snmp-server community private rw
end
write memory
```

### 2️⃣ Main Distribution Switch (MAIN-SWITCH)
> **Role:** Aggregation, Trunking, Management Connection.

```bash
enable
configure terminal
hostname MAIN-SWITCH
no ip domain-lookup

! --- VLAN Creation ---
vlan 10
 name Faculty
vlan 20
 name Students
vlan 30
 name Admin
vlan 99
 name Guest
exit

! --- Trunk Ports (to Access Switches & Router) ---
interface range fa0/1-9, gig0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,99
 switchport trunk native vlan 1
 no shutdown
exit

! --- Connection to Network Controller ---
interface gig0/2
 description Link to Network Controller
 switchport mode access
 switchport access vlan 30
 no shutdown
exit

! --- Management IP ---
interface vlan 30
 ip address 192.168.30.2 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.30.1

! --- SNMP ---
snmp-server community public ro
snmp-server community private rw
end
write memory
```

### 3️⃣ Access Switches (Faculty, Students, Admin)
> **Role:** Provide end-user connectivity for each department.

---

#### **FACULTY-SWITCH**
> Connects all Faculty devices to VLAN 10.

```bash
enable
configure terminal
hostname FACULTY-SWITCH

vlan 10
 name Faculty
exit

! --- Access Ports ---
interface range fa0/1-24
 switchport mode access
 switchport access vlan 10
 no shutdown
exit

! --- Uplink Trunk ---
interface gig0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,99
 no shutdown
exit

! --- Management IP ---
interface vlan 10
 ip address 192.168.10.200 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.10.1

! --- SNMP ---
snmp-server community public ro
snmp-server community private rw
end
write memory
```

---

#### **STUDENTS-SWITCH**
> Connects all Student devices to VLAN 20.

```bash
enable
configure terminal
hostname STUDENTS-SWITCH

vlan 20
 name Students
exit

! --- Access Ports ---
interface range fa0/1-24
 switchport mode access
 switchport access vlan 20
 no shutdown
exit

! --- Uplink Trunk ---
interface gig0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,99
 no shutdown
exit

! --- Management IP ---
interface vlan 20
 ip address 192.168.20.200 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.20.1

! --- SNMP ---
snmp-server community public ro
snmp-server community private rw
end
write memory
```

---

#### **ADMIN-SWITCH**
> Connects all Admin devices to VLAN 30.

```bash
enable
configure terminal
hostname ADMIN-SWITCH

vlan 30
 name Admin
exit

! --- Access Ports ---
interface range fa0/1-24
 switchport mode access
 switchport access vlan 30
 no shutdown
exit

! --- Uplink Trunk ---
interface gig0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,99
 no shutdown
exit

! --- Management IP ---
interface vlan 30
 ip address 192.168.30.200 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.30.1

! --- SNMP ---
snmp-server community public ro
snmp-server community private rw
end
write memory
```

### 4️⃣ Network Controller
> **Role:** Centralized Monitoring Dashboard.

- **Interface:** GigabitEthernet0
- **IP Address:** `192.168.30.10`
- **Subnet Mask:** `255.255.255.0`
- **Default Gateway:** `192.168.30.1`
- **DNS Server:** `8.8.8.8`

---

## 📸 Visual Architecture

The project is visualized with a clear, color-coded topology:
- **Core Layer:** Router & Main Switch (The Backbone)
- **Access Layer:** Dedicated Switches for each department
- **Management Plane:** Network Controller Server (The Eye)

---

## 🚀 Getting Started

1. **Load** the `.pkt` file in Cisco Packet Tracer (v8.0+).
2. **Access** the Controller Dashboard via browser at `http://192.168.30.10`.
3. **Login** with admin credentials to view network health.
4. **Test** connectivity by pinging between VLANs (Faculty <-> Student) and verifying Guest isolation.

---

<div align="center">

**Developed with ❤️ by Nady Emad**  
*Cybersecurity & Networks Student @ SUT University*

[LinkedIn](#) • [GitHub](#) • [Portfolio](#)

</div>

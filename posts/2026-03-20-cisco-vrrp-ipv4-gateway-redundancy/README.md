# How to Configure VRRP for IPv4 Gateway Redundancy on Cisco

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cisco, VRRP, IPv4, Gateway Redundancy, IOS, High Availability, Standards

Description: Configure Virtual Router Redundancy Protocol (VRRP) on Cisco IOS routers for standards-based IPv4 gateway redundancy, with priority, preemption, and object tracking.

## Introduction

VRRP (RFC 5798) is the IETF-standard equivalent of Cisco's proprietary HSRP. It provides the same active/backup gateway redundancy but works across multi-vendor equipment. On Cisco IOS, VRRP configuration closely mirrors HSRP.

## Basic VRRP Configuration

```cisco
! === Master Router (higher priority) ===
interface GigabitEthernet0/0
 description LAN
 ip address 10.1.10.2 255.255.255.0
 !
 vrrp 1 ip 10.1.10.1              ! Virtual IP
 vrrp 1 priority 120              ! Default is 100; highest becomes Master
 vrrp 1 preempt                   ! Reclaim Master after recovery
 vrrp 1 authentication text MyVRRPPass
 vrrp 1 timers advertise msec 200 ! Advertisement interval (default 1s)

! === Backup Router ===
interface GigabitEthernet0/0
 ip address 10.1.10.3 255.255.255.0
 !
 vrrp 1 ip 10.1.10.1
 vrrp 1 priority 90
 vrrp 1 preempt
 vrrp 1 authentication text MyVRRPPass
 vrrp 1 timers advertise msec 200
```

## Verify VRRP State

```cisco
show vrrp
show vrrp brief
show vrrp interface GigabitEthernet0/0

! Example output:
! GigabitEthernet0/0 - Group 1
!   State is Master
!   Virtual IP address is 10.1.10.1
!   Virtual MAC address is 0000.5e00.0101
!   Advertisement interval is 0.200 sec
!   Preemption enabled
!   Priority is 120
```

## Object Tracking Integration

```cisco
! Track WAN uplink — reduce priority if it goes down
track 1 interface GigabitEthernet0/1 line-protocol

interface GigabitEthernet0/0
 vrrp 1 track 1 decrement 30
 ! If WAN fails: priority drops from 120 to 90
 ! Backup (90) will NOT preempt unless master drops below 90
```

## Multi-Group VRRP for Load Sharing

```cisco
! Router A — Master for Group 1 (VLAN 10), Backup for Group 2 (VLAN 20)
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 10.1.10.2 255.255.255.0
 vrrp 10 ip 10.1.10.1
 vrrp 10 priority 120

interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 10.1.20.2 255.255.255.0
 vrrp 20 ip 10.1.20.1
 vrrp 20 priority 80   ! Backup for Group 20
```

## VRRP vs HSRP Comparison

| Feature | VRRP | HSRP |
|---------|------|------|
| Standard | RFC 5798 (open) | Cisco proprietary |
| Election name | Master/Backup | Active/Standby |
| Default priority | 100 | 100 |
| Virtual MAC | 00:00:5e:00:01:XX | 0000.0c07.acXX (v1) |
| Owner router | IP owner = priority 255 | No concept |
| Multicast | 224.0.0.18 | 224.0.0.2 (v1) / 224.0.0.102 (v2) |

## Conclusion

VRRP delivers interoperable gateway redundancy across vendors. Configure a higher priority on the intended master, enable preemption, and use object tracking to trigger failover when uplinks degrade. For Cisco-only environments HSRP is equally valid; VRRP is preferred in multi-vendor deployments.

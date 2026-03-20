# How to Configure IS-IS Authentication for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IS-IS, IPv6, Authentication, Security, Routing

Description: Learn how to configure IS-IS authentication to protect IPv6 routing updates from unauthorized injection on Cisco, Juniper, and FRRouting.

## Overview

IS-IS authentication prevents rogue routers from injecting false LSPs into the network. Authentication can be applied at the interface level (Hello PDUs) or at the area/domain level (LSPs). MD5 and HMAC-SHA are supported on modern implementations.

## Authentication Scope

| Scope | Protects | TLV |
|-------|---------|-----|
| Interface (Hello) | Neighbor formation | TLV 10 or TLV 133 |
| Area (Level-1 LSPs) | L1 routing database | TLV 10 or TLV 133 |
| Domain (Level-2 LSPs) | L2 routing database | TLV 10 or TLV 133 |

## Cisco IOS IS-IS Authentication

```
! Step 1: Create a key chain
Router(config)# key chain ISIS_AUTH
Router(config-keychain)# key 1
Router(config-keychain-key)#  key-string MySecretKey
Router(config-keychain-key)#  cryptographic-algorithm hmac-sha-256   ! or md5

! Step 2: Apply interface authentication (Hello PDUs)
Router(config)# interface GigabitEthernet0/0
Router(config-if)# isis authentication mode md5
Router(config-if)# isis authentication key-chain ISIS_AUTH

! Step 3: Apply area/domain authentication (LSPs and SNPs)
Router(config)# router isis
Router(config-router)# authentication mode md5 level-1    ! Area (L1) LSPs
Router(config-router)# authentication key-chain ISIS_AUTH level-1
Router(config-router)# authentication mode md5 level-2    ! Domain (L2) LSPs
Router(config-router)# authentication key-chain ISIS_AUTH level-2
```

## Juniper IS-IS Authentication

```
# Interface-level authentication (Hello PDUs)
set protocols isis interface ge-0/0/0.0 level 2 hello-authentication-key "secretkey"
set protocols isis interface ge-0/0/0.0 level 2 hello-authentication-type md5

# Area/Domain authentication (LSPs)
set protocols isis level 2 authentication-key "secretkey"
set protocols isis level 2 authentication-type md5
```

## FRRouting IS-IS Authentication

```bash
vtysh
configure terminal

! Key chain for IS-IS
key chain ISIS_KEYS
 key 1
  key-string MySecretKey
  cryptographic-algorithm hmac-sha-256

! Interface hello authentication
interface eth0
 isis authentication mode md5
 isis authentication key-chain ISIS_KEYS

! Area and domain authentication
router isis CORE
 area-password md5 AreaPassword         ! Level-1 LSPs
 domain-password md5 DomainPassword     ! Level-2 LSPs

end
write memory
```

## HMAC-SHA-256 Authentication (Modern)

SHA-256 is more secure than MD5 and is supported on Cisco IOS-XR, FRRouting, and newer Juniper platforms:

```
! Cisco IOS-XR
router isis CORE
 authentication keychain ISIS_SHA_KEY hmac-sha-256
 authentication send-only
```

## Verifying Authentication

```
! Cisco: Check authentication is active
Router# show isis interface GigabitEthernet0/0 | include Auth
  Authentication mode: MD5, key-chain: ISIS_AUTH

! Check that adjacency still forms (authentication working correctly)
Router# show isis neighbors
! If neighbor drops after adding auth → key mismatch

! View authentication TLV in database
Router# show isis database verbose R2.00-00 | include Auth
```

```bash
# FRRouting: Verify authentication
vtysh -c "show isis interface eth0" | grep -i auth

# Check neighbors are still up
vtysh -c "show isis neighbor"
```

## Authentication Transition (Adding Auth to Existing Network)

To add authentication without dropping adjacencies:

```
! Step 1: Configure authentication in "send-only" mode first
! This sends authenticated PDUs but accepts both authenticated and unauthenticated

! Cisco
Router(config-router)# authentication send-only level-2

! Step 2: Once all routers are configured, remove "send-only"
! Now all routers both send and require authentication
Router(config-router)# no authentication send-only level-2
```

## Summary

IS-IS authentication protects Hello PDUs and LSPs from unauthorized injection. Configure key chains for key management, apply authentication per-interface for Hello PDUs, and per-level for LSP authentication. Use HMAC-SHA-256 where available for stronger security. During migration, use `send-only` mode to avoid adjacency drops while gradually enabling authentication across the network.

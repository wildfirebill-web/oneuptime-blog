# How to Configure TACACS+ with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TACACS+, IPv6, AAA, Network Security, Cisco, Authentication, Authorization

Description: Configure TACACS+ servers and network devices to use IPv6 for authentication, authorization, and accounting of network device management access.

## TACACS+ and IPv6 Overview

TACACS+ (TCP port 49) supports IPv6 natively — both the transport (TACACS+ server reachable via IPv6) and attributes (device management via IPv6). Key use cases:
- Network device admin login over IPv6 management network
- TACACS+ server listening on IPv6
- Logging management access from IPv6 source addresses

## TACACS+ Server: tac_plus with IPv6

```bash
# Install tac_plus (TACACS+ daemon)
apt-get install tacacs+   # Debian/Ubuntu
yum install tacacs+       # RHEL/CentOS

# /etc/tacacs+/tac_plus.conf — Basic IPv6 configuration
cat > /etc/tacacs+/tac_plus.conf << 'EOF'
# TACACS+ key
key = mysecretkey

# Listen on IPv6 (and IPv4 via dual-stack)
# Default listens on all interfaces including IPv6

# Define NAS clients by IPv6 address
host = 2001:db8:net::1 {
    key = ciscokey
}

host = 2001:db8:net::2 {
    key = juniperkey
}

# User definitions
user = admin {
    default service = permit
    login = cleartext "adminpass"
    service = exec {
        priv-lvl = 15
    }
}

user = readonly {
    login = cleartext "readpass"
    service = exec {
        priv-lvl = 1
    }
}
EOF

# Start tac_plus
systemctl start tacacs+

# Verify listening on IPv6
ss -6 -t -l -n | grep 49
# LISTEN ... :::49
```

## TACACS+ Server: RADSEC/tac_plus-ng IPv6

```bash
# Modern tac_plus-ng (from ISC/Interlink)
# /etc/tac_plus.conf

listen = {
    address = "::"          # All IPv6 addresses
    port    = 49
}

# Or specific IPv6 address
listen = {
    address = "2001:db8::tacacs"
    port    = 49
}
```

## Cisco IOS: TACACS+ via IPv6

```
! Configure TACACS+ server via IPv6 address

tacacs server IPV6_TACACS
 address ipv6 2001:db8::tacacs
 port 49
 key mysecretkey
 timeout 5

aaa group server tacacs+ MGMT_TACACS
 server name IPV6_TACACS
 ip tacacs source-interface Loopback0

! Enable AAA
aaa new-model
aaa authentication login default group MGMT_TACACS local
aaa authorization exec default group MGMT_TACACS local
aaa accounting exec default start-stop group MGMT_TACACS

! Verify TACACS+ connectivity
test aaa group tacacs+ admin adminpass new-code

! Check statistics
show tacacs
```

## Cisco NX-OS: TACACS+ IPv6

```
! NX-OS TACACS+ with IPv6

tacacs-server host 2001:db8::tacacs key mykey
tacacs-server host 2001:db8::tacacs2 key mykey  ! Backup

aaa group server tacacs+ TACACS_GRP
  server 2001:db8::tacacs
  server 2001:db8::tacacs2
  use-vrf management

aaa authentication login default group TACACS_GRP local
aaa authorization commands default group TACACS_GRP local
aaa accounting default group TACACS_GRP

! Show TACACS+ statistics
show tacacs-server statistics
show aaa authentication login ascii-authentication
```

## Juniper Junos: TACACS+ IPv6

```
# Juniper configuration for IPv6 TACACS+

set system authentication-order [ tacplus password ]

set system tacplus-server 2001:db8::tacacs {
    secret mysecretkey
    port 49
    timeout 5
    single-connection
    source-address 2001:db8:mgmt::router1
}

# Backup TACACS+ server
set system tacplus-server 2001:db8::tacacs2 {
    secret mysecretkey
}

# Authorization
set system login class NETOPS {
    permissions all
}

set system login user admin {
    class NETOPS
    authentication tacplus
}

# Verify
show system tacplus statistics
```

## Arista EOS: TACACS+ IPv6

```
! Arista EOS TACACS+ configuration

tacacs-server host 2001:db8::tacacs key 7 <encrypted-key>
tacacs-server timeout 5

aaa group server tacacs+ TACACS_IPV6
   server 2001:db8::tacacs

aaa authentication login default group TACACS_IPV6 local
aaa authorization exec default group TACACS_IPV6 local
aaa accounting exec default start-stop group TACACS_IPV6

! Source TACACS+ from management IPv6 address
ip tacacs source-interface Management1

! Verify
show tacacs
show aaa methods authentication
```

## TACACS+ Authorization for IPv6 Management Commands

```
# tac_plus.conf — Command authorization
user = netops {
    login = cleartext "netopspass"

    service = exec {
        priv-lvl = 15
    }

    # Authorize specific commands
    cmd = show {
        permit ipv6
        permit ip
        permit interfaces
        deny .*
    }

    cmd = configure {
        permit terminal
        deny .*
    }

    # Authorize IPv6-specific commands
    cmd = ipv6 {
        permit route.*
        permit address.*
        deny .*
    }
}
```

## Testing TACACS+ IPv6 Connectivity

```bash
#!/bin/bash
# test-tacacs-ipv6.sh — Verify TACACS+ reachability over IPv6

TACACS_SERVER="2001:db8::tacacs"
TACACS_PORT=49

echo "Testing TACACS+ server at [${TACACS_SERVER}]:${TACACS_PORT}"

# TCP connectivity test
if nc -6 -z -w3 "${TACACS_SERVER}" "${TACACS_PORT}" 2>/dev/null; then
    echo "PASS: TCP port ${TACACS_PORT} is reachable via IPv6"
else
    echo "FAIL: Cannot reach TACACS+ server"
    echo "  Check: ping6 ${TACACS_SERVER}"
    echo "  Check: ip6tables on server"
fi

# Authentication test using tacc (if available)
# tacc -server ${TACACS_SERVER} -key mysecretkey -user testuser -pass testpass

# Check TACACS+ logs
journalctl -u tacacs+ | grep -i "error\|fail" | tail -5
```

## Conclusion

TACACS+ works over IPv6 with minimal configuration changes — the protocol itself is transport-agnostic. Configure `tac_plus` to listen on `::` (all IPv6 addresses) and define NAS clients by their IPv6 addresses. On Cisco IOS/NX-OS, use `address ipv6` in the `tacacs server` block; on Juniper, specify IPv6 in `tacplus-server`. Always configure a backup TACACS+ server and a local fallback (`local` in the AAA method list) to prevent lockout during network failures. Use `source-interface` or `source-address` to ensure TACACS+ packets originate from the management IPv6 address that the server expects.

# How to Configure IPsec VPN with IPv6 on strongSwan

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPsec, IPv6, strongSwan, VPN, IKEv2, Tunnel

Description: A guide to configuring strongSwan IPsec VPN with IPv6 support using IKEv2, enabling secure IPv6 tunnel establishment and traffic encryption.

strongSwan is a widely used open-source IPsec implementation that supports IPv6 natively. It can establish IPsec tunnels over IPv6 transport, tunnel IPv6 traffic through IPv4 IPsec, or handle both simultaneously. This guide covers configuring strongSwan for IPv6 IPsec using the modern `swanctl` configuration interface.

## Installing strongSwan

```bash
# Debian/Ubuntu
sudo apt-get install strongswan strongswan-swanctl charon-systemd

# Check version (5.x supports IKEv2 and IPv6 well)
strongswan version
```

## IPv6 IPsec with swanctl (Modern Configuration)

### /etc/swanctl/swanctl.conf (Server)

```conf
connections {
    ipv6-vpn {
        # IKEv2 over IPv6
        version = 2

        # Proposals for IKE encryption
        proposals = aes256gcm16-prfsha384-ecp384

        # Server's IPv6 address
        local_addrs = 2001:db8::vpn-server

        # Accept connections from any remote
        remote_addrs = %any

        local {
            id = vpn.example.com
            auth = pubkey
            certs = server-cert.pem
        }

        remote {
            id = %any
            auth = eap-mschapv2
            eap_id = %any
        }

        children {
            ipv6-child {
                # Local traffic selector: all IPv6
                local_ts = ::/0

                # Remote traffic selector: client's tunnel prefix
                remote_ts = ::/0

                # IPsec mode (tunnel)
                mode = tunnel

                # ESP proposals
                esp_proposals = aes256gcm16-ecp384

                # Install routes
                install_routes = yes
            }
        }

        # Assign IPv6 addresses to clients
        pools = ipv6-pool
    }
}

pools {
    ipv6-pool {
        addrs = fd00:ipsec::/64
        dns = 2001:4860:4860::8888
    }
}

secrets {
    eap-client1 {
        id = client1
        secret = "SecurePassword123"
    }
}
```

## IPv4-Transported IPsec Tunneling IPv6 Traffic

```conf
connections {
    ipv4-transport-ipv6-tunnel {
        version = 2

        # Connect over IPv4
        local_addrs = 192.0.2.1
        remote_addrs = 198.51.100.0/24

        local {
            id = server.example.com
            auth = pubkey
            certs = server.pem
        }

        remote {
            id = client.example.com
            auth = pubkey
        }

        children {
            ipv6-child {
                # Tunnel IPv6 traffic
                local_ts = 2001:db8:server::/48
                remote_ts = 2001:db8:client::/48
                mode = tunnel
                esp_proposals = aes256gcm16-sha384
            }
        }
    }
}
```

## Loading and Managing Configuration

```bash
# Load all configurations from swanctl.conf
sudo swanctl --load-all

# List active connections
sudo swanctl --list-conns

# List active SAs (Security Associations)
sudo swanctl --list-sas

# Initiate a connection
sudo swanctl --initiate --child ipv6-child

# Terminate a connection
sudo swanctl --terminate --ike ipv6-vpn
```

## Legacy ipsec.conf Configuration (strongSwan 4.x style)

```conf
# /etc/ipsec.conf
config setup
    charondebug="ike 2, knl 2, cfg 2"

conn ipv6-tunnel
    keyexchange=ikev2
    left=2001:db8::server
    leftsubnet=::/0
    right=%any
    rightsubnet=::/0
    rightdns=2001:4860:4860::8888
    rightsourceip=fd00:ipsec::/64
    auto=add
```

## Verifying IPv6 IPsec

```bash
# Check IKE SAs are established
sudo swanctl --list-sas | grep -A 5 "ipv6"

# Verify IPsec policies are installed
sudo ip -6 xfrm policy show

# Check ESP security associations
sudo ip -6 xfrm state show

# Test connectivity through IPsec tunnel
ping6 -c 3 fd00:ipsec::1
```

## Key Differences: IPv6 IPsec vs IPv4

| Aspect | IPv4 IPsec | IPv6 IPsec |
|---|---|---|
| Header | 20-byte IP + AH/ESP | 40-byte IPv6 + extension headers |
| Fragment | Source or router | Source only |
| IPsec in IPv6 spec | Optional | Originally mandatory |
| NAT Traversal | Required for NAT | Usually not needed |

strongSwan's full IPv6 support makes it an excellent choice for deploying modern IPsec VPN tunnels that protect IPv6 traffic with hardware-accelerated AES-GCM encryption.

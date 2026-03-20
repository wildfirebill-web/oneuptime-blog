# How to Prevent DHCP Starvation Attacks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, Security, Starvation Attack, Network Security, sysadmin

Description: DHCP starvation attacks exhaust the address pool by sending thousands of DHCP requests with fake MAC addresses, and can be mitigated through DHCP snooping, rate limiting, port security, and MAC filtering.

## How DHCP Starvation Works

An attacker uses tools like `dhcpstarv` or `yersinia` to flood the network with DHCP Discovers, each using a different spoofed MAC address. The server assigns an IP to each "fake" device until the pool is exhausted. Legitimate clients then receive DHCPNAK or no response.

## Mitigation 1: DHCP Snooping with Rate Limiting (Cisco)

```
! Enable DHCP snooping
ip dhcp snooping
ip dhcp snooping vlan 10

! Rate-limit DHCP packets on access ports (15 packets/second)
interface range GigabitEthernet0/1-24
  ip dhcp snooping limit rate 15

! Trusted uplinks (no limit)
interface GigabitEthernet0/48
  ip dhcp snooping trust
```

## Mitigation 2: Port Security (Cisco)

Port security limits the number of MAC addresses per port, preventing MAC spoofing:

```
interface GigabitEthernet0/2
  switchport mode access
  switchport port-security
  switchport port-security maximum 2     ! Max 2 MACs per port
  switchport port-security violation restrict
  switchport port-security aging time 5
```

## Mitigation 3: Small Address Pools

While not foolproof, smaller pools mean fewer wasted addresses during an attack:

```
# /etc/dhcp/dhcpd.conf
# Short lease times force faster pool recovery
subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.150;  # Small pool — 50 addresses
    default-lease-time 300;              # 5 min lease — reclaim quickly
    max-lease-time 600;
}
```

## Mitigation 4: MAC-Based Access Control (dhcpd)

Only serve known devices:

```
# /etc/dhcp/dhcpd.conf
subnet 10.0.10.0 netmask 255.255.255.0 {
    deny unknown-clients;           # Only serve known hosts
    option routers 10.0.10.1;
}

host workstation-1 { hardware ethernet aa:bb:cc:dd:ee:01; fixed-address 10.0.10.10; }
host workstation-2 { hardware ethernet aa:bb:cc:dd:ee:02; fixed-address 10.0.10.11; }
```

## Mitigation 5: Monitoring and Alerting

```bash
# Monitor pool utilization and alert when > 80% full
#!/bin/bash
SCOPE="192.168.1.0"
POOL_SIZE=50  # Adjust to your pool size
ACTIVE=$(grep -c "binding state active" /var/lib/dhcp/dhcpd.leases 2>/dev/null || echo 0)
UTIL=$(( ACTIVE * 100 / POOL_SIZE ))

if [ "$UTIL" -gt 80 ]; then
    echo "ALERT: DHCP pool ${UTIL}% full (${ACTIVE}/${POOL_SIZE})" | \
        mail -s "DHCP Pool Alert" admin@example.com
fi
```

## Key Takeaways

- DHCP snooping with rate limiting (15 pkt/s per port) is the most effective defense.
- Port security limits MAC addresses per port, preventing spoofed MAC floods.
- `deny unknown-clients` in dhcpd blocks any device not explicitly registered.
- Monitor pool utilization and alert when it exceeds 80% to detect attacks early.

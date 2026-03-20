# How to Enable GRE Keepalives on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GRE, Keepalive, Linux, Tunnel, Monitoring, IPv4, Networking

Description: Learn about GRE keepalive support on Linux, including why Linux GRE tunnels do not support native keepalives and alternative methods to detect and recover from dead GRE tunnels.

---

GRE keepalives are a Cisco IOS feature where routers send probe packets through the tunnel to detect if the remote end is reachable. Linux GRE tunnels do not have native keepalive support, but there are alternatives.

## Why Linux GRE Has No Native Keepalives

The Linux kernel GRE implementation (`ip_gre` module) does not implement the Cisco-style GRE keepalive mechanism. The kernel cannot automatically bring down a GRE interface when the remote peer is unreachable.

## Alternative 1: BFD (Bidirectional Forwarding Detection)

BFD detects forwarding path failures in milliseconds:

```bash
# Install FRR (includes BFD support)
apt install frr -y

# Enable BFD in FRR
# /etc/frr/daemons
bfdd=yes

# Configure BFD peer for GRE tunnel endpoint
vtysh << 'EOF'
conf t
bfd
 peer 172.16.1.2
  receive-interval 300
  transmit-interval 300
  detect-multiplier 3
 !
EOF
```

## Alternative 2: Dead Peer Detection with ip tunnel

```bash
# Monitor tunnel reachability with a script
# /usr/local/bin/gre-keepalive.sh
#!/bin/bash

TUNNEL_IF="gre1"
REMOTE_INNER="172.16.1.2"
CHECK_INTERVAL=5
FAIL_COUNT=3

failures=0
while true; do
  if ping -c1 -W2 "$REMOTE_INNER" > /dev/null 2>&1; then
    failures=0
    # Ensure tunnel is up
    ip link set "$TUNNEL_IF" up 2>/dev/null
  else
    ((failures++))
    if [ "$failures" -ge "$FAIL_COUNT" ]; then
      echo "$(date): GRE tunnel $TUNNEL_IF appears dead after $failures failures"
      # Take action: bring down tunnel, alert, restart
      logger -t gre-keepalive "Tunnel $TUNNEL_IF unreachable"
      failures=0
    fi
  fi
  sleep "$CHECK_INTERVAL"
done
```

## Alternative 3: Routing Protocol on the Tunnel

Run OSPF or BGP over the GRE tunnel. The routing protocol's hello mechanism detects peer failure:

```bash
# FRR: OSPF on GRE interface
vtysh << 'EOF'
conf t
interface gre1
  ip ospf hello-interval 5
  ip ospf dead-interval 20
EOF
```

When the OSPF neighbor goes down, the route is withdrawn automatically.

## Alternative 4: systemd-networkd Watchdog

```bash
# /etc/systemd/network/gre1.network
[Match]
Name=gre1

[Network]
Address=172.16.1.1/30

[Tunnel]
# No keepalive in networkd, but you can bind to specific interface
BindCarrier=eth0
```

## Monitoring GRE Tunnel Status

```bash
# Check if GRE interface is up
ip link show gre1 | grep "state"

# Check IP connectivity through tunnel
ping -c3 172.16.1.2

# Monitor interface state changes
ip monitor link | grep gre1
```

## Key Takeaways

- Linux kernel GRE does not support native Cisco-style keepalives; use application-layer detection instead.
- Run a routing protocol (OSPF, BGP) over the GRE tunnel for automatic failure detection and route withdrawal.
- BFD provides sub-second failure detection when paired with FRR on both tunnel endpoints.
- A simple ping-based monitoring script can detect dead tunnels and trigger alerts or recovery actions.

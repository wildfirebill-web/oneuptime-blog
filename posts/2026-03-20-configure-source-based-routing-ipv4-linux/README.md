# How to Configure Source-Based Routing for IPv4 on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Networking, Policy Routing, IPv4, Source Routing, ip rule

Description: Configure source-based IPv4 routing on Linux using ip rule and ip route to direct traffic from specific source addresses through different gateways or interfaces.

## Introduction

Standard Linux routing uses only the destination address. Policy-based routing (PBR) adds rules that consider the source address, enabling different traffic flows to use different gateways — essential for multi-homed servers and traffic shaping.

## Use Case: Dual-ISP Linux Router

A server has two ISP uplinks:
- `eth0`: ISP-A gateway `203.0.113.1`, assigned IP `203.0.113.10`
- `eth1`: ISP-B gateway `198.51.100.1`, assigned IP `198.51.100.10`

Without PBR, all traffic exits via the default route, causing asymmetric routing (reply packets leave via the wrong ISP). With PBR, each ISP's traffic stays on its own path.

## Step 1: Create Custom Routing Tables

Linux routing tables are numbered 1–252 (plus special tables 253=default, 254=main, 255=local). Add names for readability:

```bash
# Add table names to /etc/iproute2/rt_tables
sudo tee -a /etc/iproute2/rt_tables << 'EOF'
100 isp-a
200 isp-b
EOF
```

## Step 2: Populate Custom Tables

Each table needs a default route pointing to its ISP:

```bash
# Table isp-a: default via ISP-A gateway
sudo ip route add default via 203.0.113.1 dev eth0 table isp-a

# Table isp-b: default via ISP-B gateway
sudo ip route add default via 198.51.100.1 dev eth1 table isp-b

# Also add the directly-connected routes in each table
sudo ip route add 203.0.113.0/24 dev eth0 table isp-a
sudo ip route add 198.51.100.0/24 dev eth1 table isp-b
```

## Step 3: Add Policy Rules

Rules match traffic and direct it to the appropriate table:

```bash
# Traffic from ISP-A's IP → use isp-a table
sudo ip rule add from 203.0.113.10 table isp-a priority 100

# Traffic from ISP-B's IP → use isp-b table
sudo ip rule add from 198.51.100.10 table isp-b priority 200

# Verify rules
ip rule show
```

Output:

```
0:      from all lookup local
100:    from 203.0.113.10 lookup isp-a
200:    from 198.51.100.10 lookup isp-b
32766:  from all lookup main
32767:  from all lookup default
```

## Verifying Source-Based Routing

```bash
# Test that traffic from each IP uses the correct gateway
ip route get 8.8.8.8 from 203.0.113.10
# Output: 8.8.8.8 from 203.0.113.10 via 203.0.113.1 dev eth0

ip route get 8.8.8.8 from 198.51.100.10
# Output: 8.8.8.8 from 198.51.100.10 via 198.51.100.1 dev eth1
```

## Making Rules Persistent

Add to `/etc/rc.local` or a systemd service:

```bash
sudo tee /etc/systemd/system/pbr.service << 'EOF'
[Unit]
Description=Policy-Based Routing
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash /etc/pbr-setup.sh

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable pbr
```

## Conclusion

Source-based routing with `ip rule` and multiple routing tables ensures traffic from each source address exits via the correct gateway. This is essential for multi-homed servers to maintain symmetric routing and avoid ISP filtering of asymmetrically-routed packets.

# How to Debug Docker IPv6 Networking Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, IPv6, Debugging, Troubleshooting, Network Diagnostics

Description: Diagnose and fix common Docker IPv6 networking problems including containers not receiving IPv6 addresses, routing failures, ip6tables misconfigurations, and connectivity issues between containers.

## Introduction

Docker IPv6 issues commonly fall into four categories: daemon configuration not enabling IPv6, containers not receiving addresses, routing failures preventing internet access, and ip6tables rules blocking traffic. Systematic diagnosis starting from daemon config through network inspection to container-level testing quickly identifies the root cause.

## Diagnostic Checklist

```bash
#!/bin/bash
echo "=== 1. Check Docker daemon IPv6 config ==="
docker info 2>/dev/null | grep -E "IPv6|ip6tables"

echo ""
echo "=== 2. Check daemon.json ==="
cat /etc/docker/daemon.json 2>/dev/null || echo "daemon.json not found"

echo ""
echo "=== 3. Check docker0 bridge IPv6 ==="
ip -6 addr show docker0 2>/dev/null || echo "docker0 not found"

echo ""
echo "=== 4. List networks with IPv6 ==="
docker network ls -q | while read net; do
    NAME=$(docker network inspect "$net" --format "{{.Name}}")
    IPV6=$(docker network inspect "$net" --format "{{.EnableIPv6}}")
    echo "  $NAME: IPv6=$IPV6"
done

echo ""
echo "=== 5. Check ip6tables rules ==="
sudo ip6tables -L DOCKER -n 2>/dev/null | head -20

echo ""
echo "=== 6. Check IPv6 forwarding ==="
cat /proc/sys/net/ipv6/conf/all/forwarding
```

## Fix: Container Has No IPv6 Address

```bash
# Symptom: docker exec container ip -6 addr shows only fe80:: (link-local only)

# Diagnosis 1: Is IPv6 enabled on the network?
docker network inspect mynet | grep EnableIPv6
# If false, recreate network with --ipv6

# Diagnosis 2: Does the network have IPv6 subnet configured?
docker network inspect mynet | grep -A5 "IPAM"
# If no IPv6 subnet in Config, add one

# Fix: Recreate network with IPv6
docker network rm mynet
docker network create \
    --driver bridge \
    --ipv6 \
    --subnet 172.20.0.0/24 \
    --subnet fd00:fix::/64 \
    mynet

# Reconnect container
docker network connect mynet mycontainer

# Verify
docker exec mycontainer ip -6 addr show eth0
```

## Fix: Container Cannot Reach IPv6 Internet

```bash
# Symptom: ping6 2001:4860:4860::8888 fails from container

# Diagnosis 1: Check default IPv6 route inside container
docker exec mycontainer ip -6 route show default
# Should show: default via <gateway> dev eth0

# Diagnosis 2: Check host IPv6 forwarding
cat /proc/sys/net/ipv6/conf/all/forwarding
# Must be 1 for container IPv6 routing

# Fix: Enable IPv6 forwarding
sudo sysctl -w net.ipv6.conf.all.forwarding=1
echo "net.ipv6.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.d/99-docker-ipv6.conf

# Diagnosis 3: Check ip6tables MASQUERADE
sudo ip6tables -t nat -L POSTROUTING -n
# Should show MASQUERADE rule for container subnet

# Fix: Add masquerade rule if missing
sudo ip6tables -t nat -A POSTROUTING -s fd00:fix::/64 ! -o br-<bridge-id> -j MASQUERADE
```

## Fix: IPv6 Not Working After Daemon Restart

```bash
# Docker ip6tables rules are flushed on restart
# Check ip6tables state after restart

# Bad: ip6tables rules missing
sudo ip6tables -L DOCKER -n
# If empty after restart, ip6tables=true may not be set

# Fix: Confirm daemon.json
cat /etc/docker/daemon.json
# Must have: "ip6tables": true

# Apply fix and restart
sudo systemctl restart docker

# Wait for containers to restart and check rules
sudo ip6tables -L DOCKER -n
```

## Debug Container-to-Container IPv6

```bash
# Container 1 cannot reach Container 2 over IPv6

# Step 1: Get Container 2 IPv6 address
C2_IPV6=$(docker inspect container2 \
    --format "{{range .NetworkSettings.Networks}}{{.GlobalIPv6Address}}{{end}}")
echo "Container 2 IPv6: $C2_IPV6"

# Step 2: Are they on the same network?
docker inspect container1 --format "{{range \$k, \$v := .NetworkSettings.Networks}}{{\$k}} {{end}}"
docker inspect container2 --format "{{range \$k, \$v := .NetworkSettings.Networks}}{{\$k}} {{end}}"

# Step 3: Test ping
docker exec container1 ping6 -c 3 "$C2_IPV6"

# Step 4: Check ip6tables ICC (inter-container communication)
sudo ip6tables -L DOCKER -n | grep "$C2_IPV6"

# Fix: Enable ICC on the network
docker network rm shared-net
docker network create \
    --driver bridge \
    --ipv6 \
    --subnet fd00:icc::/64 \
    --opt com.docker.network.bridge.enable_icc=true \
    shared-net
```

## Conclusion

Debug Docker IPv6 by checking in order: daemon.json has `"ipv6": true` and `"ip6tables": true`, the network has `EnableIPv6: true` with an IPv6 subnet, host has `net.ipv6.conf.all.forwarding=1`, ip6tables has MASQUERADE for outbound, and containers are on the same network for inter-container communication. The most common issue is missing `"ipv6": true` in daemon.json — without it, no Docker network or container receives IPv6 addresses.

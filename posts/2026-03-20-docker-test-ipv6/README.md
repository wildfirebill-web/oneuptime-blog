# How to Test Docker Container IPv6 Connectivity

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, IPv6, Testing, Connectivity, Verification

Description: Test IPv6 connectivity in Docker containers using ping6, curl, and custom test scripts, verify dual-stack network behavior, and automate IPv6 connectivity checks for Docker environments.

## Introduction

Testing Docker IPv6 connectivity involves verifying address assignment, routing, DNS resolution, and application-level connectivity. Start with basic ping6 tests, then verify DNS AAAA resolution, HTTP over IPv6, and finally application-specific connectivity. Automated test scripts make it easy to verify IPv6 functionality after configuration changes or deployments.

## Basic IPv6 Connectivity Tests

```bash
# Start a test container
docker run -it --rm \
    --network mynet \
    --name ipv6-test \
    alpine sh

# Inside the container:

# 1. Check IPv6 address assignment
ip -6 addr show eth0
# Should show: inet6 fd00:mynet::2/64 scope global

# 2. Check default route
ip -6 route show
# Should show: default via fd00:mynet::1 dev eth0

# 3. Ping gateway
ping6 -c 3 fd00:mynet::1

# 4. Ping Google DNS (internet IPv6)
ping6 -c 3 2001:4860:4860::8888

# 5. DNS AAAA resolution
apk add --no-cache bind-tools
dig AAAA google.com
nslookup -type=AAAA google.com

# 6. HTTP over IPv6
apk add --no-cache curl
curl -6 -s https://ipv6.icanhazip.com
```

## Automated IPv6 Test Script

```bash
#!/bin/bash
# test_docker_ipv6.sh — automated Docker IPv6 test

NETWORK="mynet"
PASS=0
FAIL=0

run_test() {
    local name="$1"
    local cmd="$2"
    if docker run --rm --network "$NETWORK" alpine sh -c "$cmd" >/dev/null 2>&1; then
        echo "PASS: $name"
        ((PASS++))
    else
        echo "FAIL: $name"
        ((FAIL++))
    fi
}

echo "=== Docker IPv6 Tests for network: $NETWORK ==="

run_test "IPv6 address assigned" \
    "ip -6 addr show eth0 | grep 'scope global'"

run_test "IPv6 default route exists" \
    "ip -6 route show default | grep -v '^$'"

run_test "Ping gateway" \
    "apk add --no-cache iputils-ping -q && ping6 -c 1 -W 5 \$(ip -6 route show default | awk '{print \$3}' | head -1)"

run_test "Ping Google DNS" \
    "apk add --no-cache iputils-ping -q && ping6 -c 1 -W 5 2001:4860:4860::8888"

run_test "DNS AAAA resolution" \
    "apk add --no-cache bind-tools -q && dig AAAA google.com | grep -q AAAA"

run_test "HTTP over IPv6" \
    "apk add --no-cache curl -q && curl -6 -s --max-time 10 https://ipv6.icanhazip.com | grep -q ':'"

echo ""
echo "Results: $PASS passed, $FAIL failed"
[ "$FAIL" -eq 0 ] && exit 0 || exit 1
```

## Container-to-Container IPv6 Test

```bash
#!/bin/bash
# Test IPv6 connectivity between two containers

NETWORK="mynet"

# Start server container
docker run -d \
    --name srv \
    --network "$NETWORK" \
    nginx:latest

# Get server IPv6 address
SRV_IPV6=$(docker inspect srv \
    --format "{{.NetworkSettings.Networks.${NETWORK}.GlobalIPv6Address}}")
echo "Server IPv6: $SRV_IPV6"

# Test client-to-server over IPv6
docker run --rm \
    --network "$NETWORK" \
    alpine sh -c "
        apk add --no-cache curl -q
        curl -6 -s --connect-to ::[$SRV_IPV6] http://nginx/ | grep -q 'Welcome to nginx' && \
        echo 'Container-to-container IPv6: PASS' || \
        echo 'Container-to-container IPv6: FAIL'
    "

# Test by container name (DNS)
docker run --rm \
    --network "$NETWORK" \
    alpine sh -c "
        apk add --no-cache curl -q
        curl -6 -s http://srv/ | grep -q 'Welcome' && \
        echo 'DNS-based IPv6: PASS' || \
        echo 'DNS-based IPv6: FAIL'
    "

# Cleanup
docker rm -f srv
```

## Testing Published IPv6 Ports

```bash
# Run web server with IPv6 port binding
docker run -d \
    --name web \
    -p "[::]:8080:80" \
    nginx:latest

# Test from host via IPv6 loopback
curl -6 http://[::1]:8080/
# Expected: nginx welcome page

# Test via host's global IPv6
HOST_IPV6=$(ip -6 addr show eth0 | grep 'scope global' | awk '{print $2}' | cut -d/ -f1)
curl -6 "http://[$HOST_IPV6]:8080/"

# Cleanup
docker rm -f web
```

## Conclusion

Test Docker IPv6 systematically: first verify address assignment with `ip -6 addr show`, then routing with `ip -6 route show`, then connectivity with `ping6`, then DNS with `dig AAAA`, then HTTP with `curl -6`. Use the automated test script to run all checks at once. For container-to-container tests, verify both direct IPv6 address connectivity and DNS name resolution. Run tests after any Docker configuration change to catch IPv6 regressions early.

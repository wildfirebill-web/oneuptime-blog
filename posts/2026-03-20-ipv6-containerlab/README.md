# How to Build IPv6 Labs with Containerlab

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Containerlab, IPv6, Test Lab, Docker, FRRouting, Networking

Description: Build reproducible IPv6 network labs with Containerlab using topology-as-code YAML definitions and FRRouting containers.

## Containerlab Overview

Containerlab creates network topologies from YAML definitions, orchestrating Docker containers as network nodes. It handles link creation, IP assignment, and lifecycle management — making IPv6 lab setup fully declarative.

```bash
# Install Containerlab
bash -c "$(curl -sL https://get.containerlab.dev)"

# Verify installation
containerlab version
```

## Simple IPv6 Topology Definition

```yaml
# topology.yml
name: ipv6-lab

topology:
  nodes:
    r1:
      kind: linux
      image: frrouting/frr:latest
      exec:
        - ip -6 addr add 2001:db8:1::1/128 dev lo
        - sysctl -w net.ipv6.conf.all.forwarding=1

    r2:
      kind: linux
      image: frrouting/frr:latest
      exec:
        - ip -6 addr add 2001:db8:2::1/128 dev lo
        - sysctl -w net.ipv6.conf.all.forwarding=1

    r3:
      kind: linux
      image: frrouting/frr:latest
      exec:
        - ip -6 addr add 2001:db8:3::1/128 dev lo
        - sysctl -w net.ipv6.conf.all.forwarding=1

  links:
    - endpoints: ["r1:eth1", "r2:eth1"]
    - endpoints: ["r2:eth2", "r3:eth1"]
    - endpoints: ["r1:eth2", "r3:eth2"]
```

```bash
# Deploy the topology
containerlab deploy -t topology.yml

# Show running topology
containerlab inspect -t topology.yml

# Access a node console
docker exec -it clab-ipv6-lab-r1 bash
```

## FRR Configuration via Bind-Mounts

Configure FRR by mounting config files into containers:

```yaml
# topology-with-config.yml
name: ipv6-ospf-lab

topology:
  nodes:
    r1:
      kind: linux
      image: frrouting/frr:latest
      binds:
        - configs/r1/frr.conf:/etc/frr/frr.conf
        - configs/r1/daemons:/etc/frr/daemons

    r2:
      kind: linux
      image: frrouting/frr:latest
      binds:
        - configs/r2/frr.conf:/etc/frr/frr.conf
        - configs/r2/daemons:/etc/frr/daemons

  links:
    - endpoints: ["r1:eth1", "r2:eth1"]
```

```ini
# configs/r1/daemons
ospf6d=yes
bgpd=yes
staticd=yes
```

```
! configs/r1/frr.conf
interface eth1
 ipv6 address 2001:db8:12::1/64
!
interface lo
 ipv6 address 2001:db8:1::1/128
!
ipv6 router ospf6
 router-id 1.1.1.1
!
interface eth1
 ipv6 ospf6 area 0.0.0.0
!
interface lo
 ipv6 ospf6 area 0.0.0.0
 ipv6 ospf6 passive
```

## Multi-Tier Data Center Lab

```yaml
# dc-lab.yml — Spine-Leaf with EVPN over IPv6
name: dc-ipv6

topology:
  nodes:
    spine1:
      kind: linux
      image: frrouting/frr:latest
      exec:
        - ip -6 addr add 2001:db8:0:1::1/128 dev lo
        - sysctl -w net.ipv6.conf.all.forwarding=1

    spine2:
      kind: linux
      image: frrouting/frr:latest
      exec:
        - ip -6 addr add 2001:db8:0:2::1/128 dev lo
        - sysctl -w net.ipv6.conf.all.forwarding=1

    leaf1:
      kind: linux
      image: frrouting/frr:latest
      exec:
        - ip -6 addr add 2001:db8:1::1/128 dev lo
        - sysctl -w net.ipv6.conf.all.forwarding=1

    leaf2:
      kind: linux
      image: frrouting/frr:latest
      exec:
        - ip -6 addr add 2001:db8:2::1/128 dev lo
        - sysctl -w net.ipv6.conf.all.forwarding=1

    host1:
      kind: linux
      image: alpine:latest

    host2:
      kind: linux
      image: alpine:latest

  links:
    - endpoints: ["spine1:eth1", "leaf1:eth1"]
    - endpoints: ["spine1:eth2", "leaf2:eth1"]
    - endpoints: ["spine2:eth1", "leaf1:eth2"]
    - endpoints: ["spine2:eth2", "leaf2:eth2"]
    - endpoints: ["leaf1:eth3", "host1:eth1"]
    - endpoints: ["leaf2:eth3", "host2:eth1"]
```

## Lab Lifecycle Management

```bash
# Deploy
containerlab deploy -t topology.yml

# Verify nodes
containerlab inspect -t topology.yml

# Save current configuration state
containerlab save -t topology.yml

# Destroy (removes containers and links)
containerlab destroy -t topology.yml

# List all running labs
containerlab inspect --all

# Execute command on all nodes
for NODE in r1 r2 r3; do
    echo "=== ${NODE} ==="
    docker exec clab-ipv6-lab-${NODE} \
        ip -6 route show | grep ospf6
done
```

## Automated Testing with Containerlab

```bash
#!/bin/bash
# test-ipv6-lab.sh — Run after containerlab deploy

PASS=0
FAIL=0

check() {
    local TEST=$1
    local CMD=$2
    if eval "${CMD}" &>/dev/null; then
        echo "PASS: ${TEST}"
        ((PASS++))
    else
        echo "FAIL: ${TEST}"
        ((FAIL++))
    fi
}

# Test OSPFv3 adjacency
check "R1-R2 OSPFv3 adjacency" \
    "docker exec clab-ipv6-lab-r1 vtysh -c 'show ipv6 ospf6 neighbor' | grep 'Full'"

# Test end-to-end reachability
check "R1 can reach R3 loopback" \
    "docker exec clab-ipv6-lab-r1 ping6 -c 1 -W 2 2001:db8:3::1"

# Test IPv6 routing table populated
check "R1 has route to 2001:db8:3::" \
    "docker exec clab-ipv6-lab-r1 ip -6 route | grep '2001:db8:3'"

echo ""
echo "Results: ${PASS} passed, ${FAIL} failed"
```

## Conclusion

Containerlab transforms IPv6 lab setup from hours to minutes. Topology-as-code YAML files are version-controllable and reproducible. FRR containers provide full OSPFv3, BGP4+, IS-IS, and static routing support for free. Bind-mounted config files allow version-controlled FRR configurations. The `containerlab deploy/destroy` lifecycle makes it practical to spin up and tear down complex topologies on demand for testing and CI/CD pipelines.

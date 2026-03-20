# How to Use Wildcard Masks in OSPF Configuration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OSPF, Wildcard Mask, Networking, Routing, IPv4, Cisco

Description: OSPF uses wildcard masks in the network statement to specify which interfaces participate in the routing process, matching IP addresses that fall within the defined range for the given area.

## OSPF Network Statement Syntax

```
router ospf <process-id>
  network <address> <wildcard-mask> area <area-id>
```

The wildcard mask defines which bits of the address must match for an interface to be included in the OSPF area.

## Common OSPF Network Statements

```
router ospf 1
  ! Include a single interface (192.168.1.1)
  network 192.168.1.1 0.0.0.0 area 0

  ! Include all interfaces in 10.0.0.0/24
  network 10.0.0.0 0.0.0.255 area 0

  ! Include all interfaces in 10.0.0.0/8
  network 10.0.0.0 0.255.255.255 area 0

  ! Include all interfaces (any IP)
  network 0.0.0.0 255.255.255.255 area 0
```

## Matching Multiple Subnets with One Statement

```
! Include 172.16.0.0/16 through 172.31.0.0/16 (the /12 block)
! Wildcard for /12: 0.15.255.255
network 172.16.0.0 0.15.255.255 area 1
```

## FRRouting OSPF Configuration on Linux

```bash
# Install FRRouting
sudo apt install frr

# Edit /etc/frr/frr.conf
cat >> /etc/frr/frr.conf << 'FRRCFG'
!
router ospf
  ospf router-id 10.0.0.1
  network 10.0.0.0 0.255.255.255 area 0
  network 192.168.1.0 0.0.0.255 area 1
  passive-interface eth0
!
FRRCFG

sudo systemctl restart frr
```

## Python: Generating OSPF Network Statements

```python
import ipaddress

def ospf_network_statement(cidr: str, area: int = 0) -> str:
    """Generate an OSPF network statement for a given CIDR."""
    net = ipaddress.IPv4Network(cidr, strict=False)
    return f"network {net.network_address} {net.hostmask} area {area}"

interfaces = [
    ("10.0.0.0/24", 0),    # Area 0: backbone
    ("10.0.1.0/24", 0),    # Area 0
    ("172.16.10.0/24", 1), # Area 1
    ("192.168.5.0/26", 2), # Area 2
]

print("router ospf 1")
for cidr, area in interfaces:
    print(f"  {ospf_network_statement(cidr, area)}")
```

## Verifying OSPF Neighbors

```bash
# On Cisco IOS
show ip ospf neighbor
show ip ospf interface

# On FRRouting (Linux)
vtysh -c "show ip ospf neighbor"
vtysh -c "show ip ospf interface"
```

## OSPF vs BGP Network Statements

OSPF uses wildcard masks; BGP uses explicit subnet masks:

```
! OSPF (wildcard)
network 10.0.0.0 0.255.255.255 area 0

! BGP (subnet mask)
network 10.0.0.0 mask 255.0.0.0
```

## Key Takeaways

- OSPF `network` statements use wildcard masks, not subnet masks.
- A `0.0.0.0` wildcard matches exactly one IP; `255.255.255.255` matches any.
- Use `0.0.0.0 255.255.255.255 area 0` to include all interfaces in OSPF (simplest config).
- Prefer specific network statements over the catch-all to maintain explicit control over OSPF participation.

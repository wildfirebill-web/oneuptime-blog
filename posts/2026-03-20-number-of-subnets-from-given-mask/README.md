# How to Determine the Number of Subnets from a Given Mask

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Subnetting, Networking, Subnet Count, CIDR

Description: The number of subnets created by applying a longer prefix to a network block equals 2 raised to the number of borrowed bits, where borrowed bits are the difference between the new and original prefix lengths.

## The Formula

```
Borrowed bits = new_prefix - original_prefix
Number of subnets = 2^borrowed_bits
```

## Example: Subnetting a /24 into /26s

Original network: `192.168.1.0/24` (24-bit prefix)
New subnet prefix: `/26` (26-bit prefix)

```
Borrowed bits = 26 - 24 = 2
Subnets = 2^2 = 4
```

The four /26 subnets of 192.168.1.0/24:
- 192.168.1.0/26
- 192.168.1.64/26
- 192.168.1.128/26
- 192.168.1.192/26

## Python: Enumerate Subnets

```python
import ipaddress

def count_and_list_subnets(parent: str, new_prefix: int):
    """
    Show how many subnets are created when applying new_prefix to parent.
    """
    net = ipaddress.IPv4Network(parent, strict=False)
    original_prefix = net.prefixlen
    borrowed = new_prefix - original_prefix

    if borrowed < 0:
        print("Error: new prefix must be longer than original")
        return

    num_subnets = 2 ** borrowed
    print(f"Parent: {parent}")
    print(f"New prefix: /{new_prefix}")
    print(f"Borrowed bits: {borrowed}")
    print(f"Total subnets: {num_subnets}")
    print(f"Hosts per subnet: {2**(32-new_prefix) - 2}")
    print("\nSubnets:")

    for subnet in net.subnets(new_prefix=new_prefix):
        print(f"  {subnet}  "
              f"network={subnet.network_address}  "
              f"broadcast={subnet.broadcast_address}")

count_and_list_subnets("192.168.1.0/24", 26)
```

## Subnet Count Quick Reference

Starting from a /24:

| New Prefix | Borrowed Bits | Subnets | Hosts Each |
|-----------|--------------|---------|-----------|
| /25 | 1 | 2 | 126 |
| /26 | 2 | 4 | 62 |
| /27 | 3 | 8 | 30 |
| /28 | 4 | 16 | 14 |
| /29 | 5 | 32 | 6 |
| /30 | 6 | 64 | 2 |

## Subnetting a /16 for Department VLANs

```python
# Divide 10.0.0.0/16 into /24 subnets (256 departments)
parent = ipaddress.IPv4Network("10.0.0.0/16")
borrowed = 24 - 16  # = 8 bits borrowed
print(f"Subnets: {2**borrowed}")  # 256

# First 5 subnets
for subnet in list(parent.subnets(new_prefix=24))[:5]:
    print(subnet)
```

## Key Takeaways

- Borrowed bits = new prefix length − original prefix length.
- Number of subnets = 2^(borrowed bits).
- Each subnet has 2^(32 − new_prefix) total addresses, minus 2 for network and broadcast.
- Use `ipaddress.IPv4Network.subnets(new_prefix=N)` in Python to enumerate all subnets.

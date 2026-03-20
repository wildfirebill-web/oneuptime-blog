# How to Calculate the Network Address from IPv4 and Subnet Mask in Code

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Networking, Subnetting, C, Python, JavaScript

Description: Calculate the network address from an IPv4 host address and subnet mask using bitwise AND in C, Python, and JavaScript, with support for CIDR notation.

## Introduction

The network address is derived by performing a bitwise AND of the host IP address and the subnet mask. This operation zeroes out the host bits, leaving only the network prefix. Every language that can do integer bitwise operations can compute this in a few lines.

## C Implementation

```c
#include <stdio.h>
#include <stdint.h>
#include <arpa/inet.h>

void calc_network(const char *ip_str, const char *mask_str) {
    struct in_addr ip, mask, net;
    inet_pton(AF_INET, ip_str,   &ip);
    inet_pton(AF_INET, mask_str, &mask);

    net.s_addr = ip.s_addr & mask.s_addr;

    char net_str[INET_ADDRSTRLEN];
    inet_ntop(AF_INET, &net, net_str, INET_ADDRSTRLEN);
    printf("Network: %s\n", net_str);
}

/* CIDR prefix to mask */
uint32_t prefix_to_mask(int prefix) {
    return prefix == 0 ? 0 : htonl(~((1u << (32 - prefix)) - 1));
}

int main(void) {
    calc_network("192.168.10.45", "255.255.255.0");  /* 192.168.10.0 */
    calc_network("10.20.30.100", "255.255.0.0");     /* 10.20.0.0 */

    /* CIDR example */
    struct in_addr ip, mask, net;
    inet_pton(AF_INET, "172.16.5.200", &ip);
    mask.s_addr = prefix_to_mask(20);
    net.s_addr  = ip.s_addr & mask.s_addr;
    char buf[INET_ADDRSTRLEN];
    printf("Network (/20): %s\n",
           inet_ntop(AF_INET, &net, buf, INET_ADDRSTRLEN));
    return 0;
}
```

## Python Implementation

```python
import ipaddress

def network_address(ip: str, mask: str) -> str:
    """Calculate network address from IP and dotted-decimal mask."""
    interface = ipaddress.IPv4Interface(f"{ip}/{mask}")
    return str(interface.network.network_address)

def network_from_cidr(cidr: str) -> str:
    """Return network address from CIDR notation like 192.168.1.45/24."""
    net = ipaddress.IPv4Interface(cidr).network
    return str(net.network_address)

# Examples
print(network_address("192.168.10.45", "255.255.255.0"))   # 192.168.10.0
print(network_address("10.20.30.100",  "255.255.0.0"))     # 10.20.0.0
print(network_from_cidr("172.16.5.200/20"))                # 172.16.0.0
```

## JavaScript Implementation

```javascript
function ipToInt(ip) {
    return ip.split('.').reduce((acc, octet) => (acc << 8) | parseInt(octet), 0) >>> 0;
}

function intToIp(n) {
    return [(n >>> 24) & 0xff, (n >>> 16) & 0xff,
            (n >>> 8)  & 0xff,  n         & 0xff].join('.');
}

function cidrToMask(prefix) {
    return prefix === 0 ? 0 : (~((1 << (32 - prefix)) - 1)) >>> 0;
}

function networkAddress(ip, mask) {
    return intToIp(ipToInt(ip) & ipToInt(mask));
}

function networkFromCidr(cidr) {
    const [ip, prefix] = cidr.split('/');
    return intToIp(ipToInt(ip) & cidrToMask(parseInt(prefix)));
}

console.log(networkAddress("192.168.10.45", "255.255.255.0")); // 192.168.10.0
console.log(networkFromCidr("172.16.5.200/20"));               // 172.16.0.0
```

## Conclusion

The network address calculation is a single bitwise AND operation. Python's `ipaddress` module handles all edge cases natively. In C and JavaScript you perform the AND on the 32-bit integer representations of the IP and mask, then convert back to dotted-decimal notation.

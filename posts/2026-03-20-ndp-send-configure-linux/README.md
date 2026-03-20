# How to Configure SEND on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SEND, Linux, NDP Security, IPv6 Security, Kernel

Description: Configure Secure Neighbor Discovery (SEND) on Linux using available user-space implementations, understand the kernel requirements, and test SEND functionality.

## Introduction

Linux kernel support for SEND is limited and primarily implemented in user space. The kernel provides hooks (via netlink) that allow user-space SEND daemons to intercept NDP messages and add/verify SEND security options. The `sendd` daemon is the most commonly referenced implementation, though it is not widely packaged and deployment remains challenging.

## Linux SEND Support Status

```text
Linux SEND implementation status:

Kernel:
  - Does NOT natively generate SEND options
  - Provides netlink hooks for user-space SEND daemons
  - Will accept SEND-signed NDP if user-space daemon validates them

User-space implementations:
  - sendd (DoCoMo research): Reference implementation, rarely maintained
  - OpenSEND: Alternative implementation
  - Both require significant setup and are not production-ready

Practical conclusion:
  Full SEND on Linux is not available in major distributions as a
  ready-to-install package. RA Guard (switch-level) is the recommended
  alternative for most deployments.

For academic/research use of SEND on Linux:
  Consider using virtual environments and patched kernels
```

## Verifying Kernel SEND Hooks

```bash
# Check if the kernel has CONFIG_IPV6_SIOCSIFADDR hooks

# (required for SEND user-space interaction)
zcat /boot/config-$(uname -r) | grep -E "SEND|CGA"
# Look for CONFIG_IPV6_OPTIMISTIC_DAD and other relevant options

# Check if sendd is installed (unlikely on most distros)
which sendd 2>/dev/null || echo "sendd not installed"

# Alternative: use the ndisc6 toolkit for SEND-like functionality testing
sudo apt-get install ndisc6

# Check kernel module for NDP
lsmod | grep ipv6
modinfo ipv6 | grep -i "send\|cga"
```

## Alternative: RA Guard as the Practical SEND Replacement

```bash
# Since SEND is not practically deployable on Linux,
# use RA Guard + ip6tables to achieve similar protection

# Restrict RA acceptance to known router MAC addresses
# (Poor man's SEND: validate source at L2 level)

# Allow RA only from specific MAC (router's MAC)
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type router-advertisement \
    -m mac --mac-source 00:11:22:33:44:55 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type router-advertisement \
    -j DROP

# Allow RA from specific link-local address
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type router-advertisement \
    -s fe80::1 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type router-advertisement \
    -j DROP

# Similarly restrict Neighbor Advertisements to prevent NA spoofing
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type neighbor-advertisement \
    -m limit --limit 100/sec --limit-burst 200 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type neighbor-advertisement \
    -j DROP
```

## SEND for Testing Environments

```bash
# For testing SEND concepts in a lab environment:
# 1. Use two VMs
# 2. Install development versions of sendd from source
# git clone https://github.com/... (check current repositories)

# Enable kernel experimental features for SEND testing
sudo modprobe ipv6
echo 1 | sudo tee /proc/sys/net/ipv6/conf/eth0/use_optimistic
# Optimistic DAD is related to SEND functionality

# Generate RSA keys for testing CGA
openssl genrsa -out send_key.pem 2048
openssl rsa -in send_key.pem -pubout -out send_pub.pem

# These keys would be used to generate a CGA address
# (actual CGA generation requires the sendd tool)
```

## Conclusion

SEND on Linux is not practically deployable in production environments due to limited user-space implementations and lack of distribution packages. The kernel provides the necessary hooks but no native SEND support. For securing NDP on Linux hosts, the combination of ip6tables RA filtering (allow RA only from trusted sources), rate limiting for NA floods, and switch-level RA Guard provides adequate protection without SEND's complexity. For environments requiring SEND, evaluate dedicated network operating systems that include proper SEND support.

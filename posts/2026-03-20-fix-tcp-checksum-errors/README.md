# How to Fix TCP Checksum Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Checksum, Linux, Networking, Hardware, Offload

Description: Understand why TCP checksum errors appear in packet captures and how to distinguish hardware checksum offload artifacts from genuine checksum failures.

## Introduction

TCP checksum errors in packet captures are almost always false alarms caused by checksum offload - a performance feature where the NIC calculates checksums in hardware rather than the kernel. tcpdump captures packets before the NIC fills in the checksum, showing zeros or incorrect values. True checksum errors from bit corruption are extremely rare on modern wired networks.

## How Checksum Offload Works

```text
Without offload (software checksum):
Kernel calculates checksum → places it in packet → NIC sends packet

With offload (hardware checksum):
Kernel leaves checksum = 0 → NIC calculates checksum → NIC sends packet

tcpdump captures packets IN THE KERNEL before the NIC fills in the checksum
→ tcpdump shows "incorrect checksum" even though the sent packet is correct
```

## Distinguishing False from Real Errors

```bash
# Check if NIC has checksum offload enabled (common case - false errors)

ethtool -k eth0 | grep checksum
# tx-checksumming: on    ← TX offload = tcpdump will show fake errors
# rx-checksumming: on    ← RX offload = NICs validates incoming checksums

# If tx-checksumming is ON:
# Outbound packets captured by tcpdump will show incorrect checksum
# This is expected and NOT a real problem - the NIC corrects it
```

## Disabling Checksum Offload (for Accurate Captures)

```bash
# Temporarily disable TX checksum offload for accurate tcpdump analysis
ethtool -K eth0 tx off

# Now tcpdump will show correct checksums for outbound packets
# Capture your traffic
tcpdump -i eth0 -w /tmp/capture.pcap

# Re-enable offload after capture (offload improves performance significantly)
ethtool -K eth0 tx on
```

## Verifying with Wireshark

```text
# In Wireshark: Edit → Preferences → Protocols → TCP
# Uncheck: "Validate the TCP checksum if possible"
# This suppresses false checksum warnings from captured-before-offload packets

# To identify REAL checksum errors (hardware/bit corruption):
# 1. Disable TX offload
# 2. Capture traffic
# 3. Apply Wireshark filter: tcp.checksum_bad == 1
# Non-zero count after disabling offload = real checksum error
```

## Real TCP Checksum Errors

Real checksum failures are caused by:

```bash
# 1. Hardware malfunction (NIC, cable, memory)
# Check interface hardware errors
ip -s link show eth0 | grep -A1 "RX:"
# Non-zero errors/drops = hardware issue

# 2. Bit flips in memory
# Run memory test if checksum errors are frequent
memtester 512M 1

# 3. Software bug in custom packet processing
# If you're writing raw socket code, verify checksum calculation

# 4. Misconfigured tunnel or overlay
# GRE/VXLAN tunnels may have checksum issues
# Check tunnel checksum settings
ip link show type gre
# "csum" flag = checksums enabled on the tunnel
```

## NIC Driver Configuration

```bash
# Some NICs have checksum offload bugs in certain driver versions
# Check NIC driver version
ethtool -i eth0 | grep -E "driver|version"

# Check if driver updates address checksum issues
# (consult vendor release notes)

# For containers/VMs: check if the hypervisor's virtual NIC supports offload
ethtool -k eth0 | grep "tx-checksumming"
# If using virtio-net: checksum offload is typically supported
```

## Conclusion

TCP checksum errors in packet captures are overwhelmingly false positives from hardware checksum offload. Before investigating further, disable TX offload and recapture - if errors disappear from Wireshark, you were seeing offload artifacts. If errors persist after disabling offload, investigate hardware (NIC, cable, memory) and software (custom packet processing, tunnel configuration). Real checksum errors on modern wired networks are extremely rare.

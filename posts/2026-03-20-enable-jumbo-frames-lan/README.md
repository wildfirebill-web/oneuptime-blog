# How to Enable Jumbo Frames for Local Network Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Jumbo Frames, MTU, Linux, Network Performance, LAN, ethtool

Description: Learn how to enable jumbo frames (9000-byte MTU) on Linux for local network performance improvement, covering NIC configuration, switch requirements, and verification.

## What Are Jumbo Frames?

Standard Ethernet frames have a Maximum Transmission Unit (MTU) of 1500 bytes. Jumbo frames increase this to 9000 bytes (or 9216 for some NICs), allowing larger payloads per frame.

Benefits on LAN:
- Fewer frames to transmit the same data → lower CPU overhead
- Fewer interrupts per MB of data
- Higher throughput for large bulk transfers (NFS, iSCSI, backups)
- Typically 10-20% throughput improvement for large transfers

**Important**: Jumbo frames only work on LAN. All devices in the path (NICs, switches) must support the same MTU.

## Step 1: Verify Hardware Support

```bash
# Check maximum MTU supported by the NIC

ip link show eth0
# look for "maxmtu XXXXX" field (if present)

# Check with ethtool
ethtool -i eth0
# Note the driver name

# Test if jumbo MTU is accepted
sudo ip link set eth0 mtu 9000 2>&1
# If no error, the NIC supports it
```

## Step 2: Enable Jumbo Frames on Linux

```bash
# Set MTU to 9000 on the interface
sudo ip link set eth0 mtu 9000

# Verify
ip link show eth0 | grep mtu
# should show: mtu 9000

# Test with a large ping (9000 - 28 bytes header = 8972 bytes payload)
ping -M do -s 8972 192.168.1.1
# If successful, jumbo frames are working end-to-end
```

## Step 3: Make MTU Persistent

```bash
# Method 1: /etc/network/interfaces (Debian/Ubuntu with ifupdown)
cat >> /etc/network/interfaces << 'EOF'
auto eth0
iface eth0 inet static
    address 192.168.1.10
    netmask 255.255.255.0
    gateway 192.168.1.1
    mtu 9000
EOF

# Method 2: NetworkManager (most modern systems)
nmcli connection modify "Wired connection 1" 802-3-ethernet.mtu 9000
nmcli connection up "Wired connection 1"

# Method 3: netplan (Ubuntu 18.04+)
cat > /etc/netplan/01-netcfg.yaml << 'EOF'
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.1.10/24
      gateway4: 192.168.1.1
      mtu: 9000
EOF
sudo netplan apply
```

## Step 4: Configure Jumbo Frames on the Switch

All switches in the path must support jumbo frames. Example for common platforms:

**Cisco Catalyst IOS:**
```text
! Enable jumbo frames globally (some models)
system mtu jumbo 9000

! Or per-interface
interface GigabitEthernet1/0/1
 mtu 9000
```

**Arista EOS:**
```text
interface Ethernet1
   mtu 9214
```

**Linux bridge:**
```bash
# Set MTU on the bridge interface
ip link set br0 mtu 9000
ip link set eth0 mtu 9000    # Also set on bridge member interfaces
ip link set eth1 mtu 9000
```

## Step 5: Verify End-to-End Jumbo Frame Support

```bash
# Test jumbo ping to remote host (must have jumbo frames configured)
# -M do = don't fragment, -s 8972 = 8972 byte payload (+ 28 header = 9000)
ping -M do -s 8972 192.168.1.100

# If ping fails with "Message too long", jumbo frames are not configured
# on an intermediate device

# Trace the MTU along the path
tracepath -n 192.168.1.100
# Shows the effective MTU at each hop

# Test throughput improvement
# Before jumbo frames:
iperf3 -c 192.168.1.100 -t 30
# After jumbo frames:
iperf3 -c 192.168.1.100 -t 30
```

## Step 6: NFS and iSCSI Jumbo Frame Configuration

Jumbo frames provide the most benefit for storage protocols:

```bash
# NFS mount with jumbo frames
# Ensure both NFS server and client have MTU 9000 on storage interface
mount -t nfs -o rsize=65536,wsize=65536 192.168.1.200:/data /mnt/data

# iSCSI with jumbo frames
# Configure iSCSI to use the jumbo frame interface
iscsiadm -m node --op=update -n node.conn[0].iscsi.MaxXmitDataSegmentLength -v 65536
```

## Conclusion

Jumbo frames improve bulk transfer throughput on LANs by reducing per-packet overhead. Enable with `ip link set eth0 mtu 9000`, verify with a large fragmentation-forbidden ping, and persist through NetworkManager, netplan, or `/etc/network/interfaces`. Ensure all switches in the path are also configured for jumbo frames - a single standard-MTU device in the path will cause large packets to be fragmented or dropped.

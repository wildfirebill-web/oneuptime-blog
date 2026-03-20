# How to Configure a Bond with nmcli

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, nmcli, NetworkManager, Bonding, LACP, Networking

Description: Configure network interface bonding using nmcli, including creating a bond device, adding member interfaces, and setting bonding mode and monitoring parameters.

## Introduction

NetworkManager supports bonding through nmcli. The process involves creating a bond master connection, adding slave (member) connections for each physical interface, and activating them. This provides persistent bonding managed by NetworkManager.

## Step 1: Create the Bond Master

```bash
# Create a bond interface with active-backup mode

nmcli connection add \
    type bond \
    con-name "bond0" \
    ifname bond0 \
    bond.options "mode=active-backup,miimon=100"

# Assign IP to the bond
nmcli connection modify "bond0" \
    ipv4.method manual \
    ipv4.addresses "192.168.1.10/24" \
    ipv4.gateway "192.168.1.1"
```

## Step 2: Add Member Interfaces (Slaves)

```bash
# Add eth0 as a bond member
nmcli connection add \
    type ethernet \
    con-name "bond0-slave-eth0" \
    ifname eth0 \
    master bond0

# Add eth1 as a bond member
nmcli connection add \
    type ethernet \
    con-name "bond0-slave-eth1" \
    ifname eth1 \
    master bond0
```

## Step 3: Activate the Bond

```bash
# Bring up the slaves first, then the master
nmcli connection up "bond0-slave-eth0"
nmcli connection up "bond0-slave-eth1"
nmcli connection up "bond0"
```

## Verify Bond Status

```bash
# Show bond status
cat /proc/net/bonding/bond0

# Check IP address on bond
ip addr show bond0

# Show active connection
nmcli device status
```

## Configure LACP (802.3ad) Bonding

```bash
nmcli connection add \
    type bond \
    con-name "bond-lacp" \
    ifname bond0 \
    bond.options "mode=802.3ad,miimon=100,lacp_rate=fast,xmit_hash_policy=layer3+4"
```

## Configure Round-Robin Bonding

```bash
nmcli connection add \
    type bond \
    con-name "bond-rr" \
    ifname bond0 \
    bond.options "mode=balance-rr,miimon=100"
```

## Change Bond Options on Existing Connection

```bash
nmcli connection modify "bond0" \
    bond.options "mode=active-backup,miimon=200,updelay=400,downdelay=400"

nmcli connection up "bond0"
```

## Delete a Bond

```bash
nmcli connection delete "bond0-slave-eth0"
nmcli connection delete "bond0-slave-eth1"
nmcli connection delete "bond0"
```

## Conclusion

nmcli bond setup requires a master connection (`type bond`) and slave connections (`type ethernet master bond0`). Bonding options like mode and MII monitor interval are set in `bond.options`. Verify with `cat /proc/net/bonding/bond0` to see active/backup state and member interface status.

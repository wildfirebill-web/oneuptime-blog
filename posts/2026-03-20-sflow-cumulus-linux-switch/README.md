# How to Configure sFlow on a Cumulus Linux Switch

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: sFlow, Cumulus Linux, Networking, Monitoring, IPv4, Hsflowd, Traffic Analysis

Description: Learn how to configure sFlow packet sampling on a Cumulus Linux switch using hsflowd to export traffic statistics to an sFlow collector over IPv4.

---

sFlow is a packet sampling protocol that provides visibility into network traffic flows at line rate. On Cumulus Linux, the `hsflowd` daemon handles sFlow configuration and exports sampled data to a remote collector.

## Installing hsflowd

```bash
# Install the Host sFlow daemon

apt update && apt install hsflowd -y

# Verify installation
hsflowd -v
```

## Basic hsflowd Configuration

```ini
# /etc/hsflowd.conf

sflow {
    # sFlow agent IPv4 address (the switch's management IP)
    agent = eth0

    # Sampling rate: 1 in N packets will be sampled
    # For a 10 Gbps link, 1:10000 is typical
    sampling = 10000

    # Polling interval for counters (seconds)
    polling = 30

    # Collector: send sFlow records to this IPv4:port
    collector {
        ip = 10.0.0.50
        port = 6343
        timeout = 60
    }
}
```

## Specifying Per-Interface Sampling Rates

Different interfaces may need different sampling rates based on their speed.

```ini
# /etc/hsflowd.conf

sflow {
    agent = eth0

    # Default sampling rate for all interfaces
    sampling = 10000
    polling = 30

    collector {
        ip = 10.0.0.50
        port = 6343
    }

    # Override for high-speed backbone interfaces
    pcap {
        dev = swp1
        sampling = 1000    # More granular: 1 in 1000 for 1G link
    }

    pcap {
        dev = swp49        # 100G uplink: 1 in 100000
        sampling = 100000
    }
}
```

## Enabling sFlow on Cumulus Switch Ports

```bash
# Apply hsflowd configuration
systemctl enable --now hsflowd

# Verify sFlow is running
systemctl status hsflowd

# Check that hsflowd is sending to the collector
ss -unp | grep hsflowd
tcpdump -i eth0 -nn udp port 6343 -c 5   # Capture outgoing sFlow packets
```

## Verifying Data at the Collector

```bash
# Use sflowtool on the collector server (10.0.0.50) to view incoming sFlow
apt install sflowtool -y
sflowtool -l -p 6343    # Listen on UDP 6343 and print decoded sFlow records

# Example output:
# FLOW 10.0.0.1 2 2 swp1 0:1:1:2:3:4 0:1:1:2:3:5 0x0800 10 1500 10.1.1.1 10.2.2.2 6 80 443
```

## Using ntopng to Visualize sFlow Data

```bash
# Install ntopng and its sFlow receiver (nprobe)
apt install ntopng nprobe -y

# Configure nProbe to receive sFlow from the switch
nprobe --interface="sflow://192.168.1.50:6343" \
       --ntopng="zmq://127.0.0.1:5556"

# Start ntopng to visualize
ntopng -i zmq://127.0.0.1:5556
```

Access ntopng at `http://10.0.0.50:3000` for real-time per-interface and per-flow statistics.

## Key Takeaways

- `hsflowd` is the standard sFlow agent for Cumulus Linux; configure it via `/etc/hsflowd.conf`.
- Set the sampling rate based on link speed: 1:1000 for 1G, 1:10000 for 10G, 1:100000 for 100G.
- sFlow collectors listen on UDP port 6343 by default; use `sflowtool` to verify incoming records.
- Unlike NetFlow (which exports per-flow records), sFlow samples actual packets, providing more accurate traffic characterization.

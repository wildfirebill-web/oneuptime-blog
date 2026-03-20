# How to Configure sFlow on an HP/Aruba Switch

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: sFlow, HP Aruba, Switch, Traffic Monitoring, Network Visibility

Description: Learn how to configure sFlow packet sampling on HP/Aruba switches to export sampled traffic statistics to an sFlow collector for network visibility.

## What Is sFlow?

sFlow (RFC 3176) uses statistical packet sampling—it samples 1 in N packets from each interface and forwards the sample header to a collector. Unlike NetFlow (which records every flow), sFlow scales to any link speed with constant CPU overhead by using sampling instead of flow tracking.

**Key sFlow concepts:**
- **Sampling rate:** 1 in N packets is sampled (e.g., 1 in 1000)
- **Counter polling:** Interface counters are exported periodically alongside samples
- **Agent:** The switch/router performing sampling
- **Collector:** Server receiving and analyzing sFlow datagrams

## Step 1: Configure sFlow on HP Aruba Switch (ArubaOS-Switch/ProCurve)

Log in to the switch CLI and configure sFlow:

```
! Access the switch CLI
Switch# configure terminal

! Enable sFlow globally
Switch(config)# sflow enable

! Configure the sFlow receiver (collector)
Switch(config)# sflow receiver 1 name "sflow-collector" address 192.168.1.200 \
  port 6343 owner "sflow-collector"

! Set the polling interval for counter sampling (in seconds)
Switch(config)# sflow polling receiver 1 interval 30

! Set the packet sampling rate (1 in 512 packets on GigE)
Switch(config)# sflow sampling receiver 1 rate 512
```

## Step 2: Enable sFlow on Specific Interfaces

Apply sFlow sampling to the interfaces you want to monitor:

```
! Enable sFlow on uplink interfaces
Switch(config)# interface 1/1
Switch(config-if)# sflow sampling 512
Switch(config-if)# sflow polling 30

! Enable on multiple interfaces
Switch(config)# interface 1/2
Switch(config-if)# sflow sampling 512
Switch(config-if)# sflow polling 30

! Enable on a trunk/LAG
Switch(config)# interface trk1
Switch(config-if)# sflow sampling 512
```

## Step 3: Configure sFlow on ArubaOS-CX (Aruba CX Switches)

Newer Aruba CX switches use a different CLI:

```
! ArubaOS-CX sFlow configuration
switch(config)# sflow
switch(config-sflow)# agent-ip 10.0.0.1       ! Switch management IP
switch(config-sflow)# collector 1 ip 192.168.1.200 port 6343

! Enable sampling on interfaces
switch(config)# interface 1/1/1
switch(config-if)# sflow enable
switch(config-if)# sflow sampling-rate 512    ! 1 in 512 packets

! Enable counter polling
switch(config-if)# sflow polling-interval 30
```

## Step 4: Verify sFlow Configuration

```
! Show sFlow status
Switch# show sflow

sFlow: enabled
Collector: 192.168.1.200  Port: 6343
Agent IP: 10.0.0.1
Polling interval: 30 seconds

Interface   Sampling Rate  Counters Polling
----------- -------------- -----------------
1/1         512            30
1/2         512            30
```

## Step 5: Set Up an sFlow Collector (sflowtool/nfsen)

Install sflowtool on Linux to receive and decode sFlow data:

```bash
# Install sflowtool
sudo apt-get install -y sflowtool

# Receive and print sFlow datagrams (port 6343)
sflowtool -p 6343

# Output to a file for analysis
sflowtool -p 6343 >> /var/log/sflow/sflow.log &

# Parse specific fields
sflowtool -p 6343 -l | awk '{print $1, $5, $9, $11}' | \
  while read ts srcip dstip bytes; do
    echo "$ts: $srcip -> $dstip ($bytes bytes)"
  done
```

## Step 6: Use ntopng as an sFlow Collector

ntopng provides a web dashboard for sFlow analysis:

```bash
# Install ntopng
sudo apt-get install -y ntopng

# Start ntopng with sFlow collector enabled
ntopng --interface="sflow:192.168.1.200:6343" \
       --http-port 3000

# Access dashboard at http://your-server:3000
```

## Step 7: Choosing the Right Sampling Rate

| Link Speed | Recommended Sampling Rate |
|---|---|
| 1 Gbps | 1 in 512–1000 |
| 10 Gbps | 1 in 2000–5000 |
| 40 Gbps | 1 in 5000–10000 |
| 100 Gbps | 1 in 10000–50000 |

Lower sampling rates are more accurate but use more CPU and bandwidth. Start with 1:1000 and adjust based on CPU load.

## Conclusion

sFlow on HP/Aruba switches provides lightweight, scalable traffic visibility using statistical sampling. Configure the receiver IP and port, set the sampling rate appropriate for your link speed, and enable sFlow on uplink interfaces. Connect the collector to ntopng, ElastiFlow, or sflowtool for traffic analysis dashboards.

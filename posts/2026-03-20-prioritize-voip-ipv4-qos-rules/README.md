# How to Prioritize VoIP IPv4 Traffic Using QoS Rules

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: VoIP, QoS, IPv4, Linux, tc, DSCP

Description: Configure Linux tc and iptables QoS rules to give VoIP IPv4 traffic strict priority queuing, ensuring low latency and jitter for voice calls.

VoIP requires low latency (< 150ms), low jitter (< 30ms), and minimal packet loss (< 1%). Without QoS, large file transfers or streaming can degrade call quality. This guide prioritizes VoIP at the Linux level.

## VoIP Traffic Identification

VoIP typically uses:
- SIP signaling: UDP/TCP port 5060
- SRTP/RTP media: UDP ports 10000-65535 (varies by application)
- Common VoIP platforms: Zoom (8801-8802), Teams (3478-3481, 50000-50019)

## Step 1: Mark VoIP Traffic with iptables

```bash
# Mark SIP signaling packets (port 5060)
sudo iptables -t mangle -A OUTPUT -p udp --dport 5060 -j MARK --set-mark 1
sudo iptables -t mangle -A OUTPUT -p tcp --dport 5060 -j MARK --set-mark 1

# Mark RTP audio packets (common range)
sudo iptables -t mangle -A OUTPUT -p udp --dport 10000:20000 -j MARK --set-mark 1

# Mark DSCP EF on marked packets (for network-wide QoS)
sudo iptables -t mangle -A OUTPUT -m mark --mark 1 -j DSCP --set-dscp-class EF
```

## Step 2: Set Up HTB with Strict Priority for VoIP

```bash
# Create HTB root qdisc
sudo tc qdisc add dev eth0 root handle 1: htb default 30

# Total bandwidth: 100 Mbps
sudo tc class add dev eth0 parent 1: classid 1:1 htb rate 100mbit burst 15k

# VoIP class: strict priority, 10 Mbps guaranteed (prio 0 = highest)
sudo tc class add dev eth0 parent 1:1 classid 1:10 \
  htb rate 10mbit ceil 100mbit burst 3k prio 0

# Standard class: 80 Mbps guaranteed
sudo tc class add dev eth0 parent 1:1 classid 1:20 \
  htb rate 80mbit ceil 100mbit burst 15k prio 1

# Default class: 10 Mbps guaranteed
sudo tc class add dev eth0 parent 1:1 classid 1:30 \
  htb rate 10mbit ceil 100mbit burst 15k prio 2
```

## Step 3: Add Leaf qdiscs

```bash
# Use pfifo for VoIP (minimal buffering = minimal latency)
sudo tc qdisc add dev eth0 parent 1:10 handle 10: pfifo limit 10

# Use fq_codel for other classes (good AQM)
sudo tc qdisc add dev eth0 parent 1:20 handle 20: fq_codel
sudo tc qdisc add dev eth0 parent 1:30 handle 30: fq_codel
```

## Step 4: Filter VoIP to Priority Class

```bash
# Route marked VoIP packets to the VoIP class
sudo tc filter add dev eth0 protocol ip parent 1:0 \
  handle 1 fw flowid 1:10

# Also match DSCP EF
sudo tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 \
  match ip tos 0xb8 0xfc flowid 1:10
```

## Step 5: Verify VoIP Prioritization

```bash
# During an active call, check class statistics
sudo tc -s class show dev eth0

# The VoIP class (1:10) should show minimal drops
# Look for: rate x bps, dropped 0

# Measure latency to VoIP server during a bulk download
wget -O /dev/null http://speedtest.net/largefile.zip &
ping -c 20 <VOIP_SERVER_IP>
# VoIP latency should remain low (< 30ms added by QoS)
```

## For Home Routers (OpenWrt/dd-wrt)

```bash
# On OpenWrt, install and use SQM (Smart Queue Management)
opkg install luci-app-sqm sqm-scripts

# Configure in LuCI: Network → SQM QoS
# Set interface, download/upload speeds, and enable
```

Proper VoIP prioritization ensures call quality remains excellent even when other devices on the network are doing large file transfers or streaming.

# How to Configure Snort IDS for IPv6 Detection

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Snort, IDS, IPv6, Network Security, Intrusion Detection, Linux

Description: Configure Snort intrusion detection system to monitor and detect threats in IPv6 network traffic, including installation, network variables, and IPv6-specific rules.

---

Snort is one of the most widely deployed network intrusion detection systems. Snort 3 supports IPv6 natively, enabling detection of threats in both IPv4 and IPv6 traffic with unified rule syntax.

## Installing Snort 3

```bash
# Ubuntu/Debian - install from source or PPA
sudo apt install snort3 -y

# Or build from source
sudo apt install build-essential cmake libpcap-dev libpcre2-dev \
  libdnet-dev zlib1g-dev pkg-config -y

# Download and compile
wget https://github.com/snort3/snort3/archive/refs/tags/3.1.50.0.tar.gz
tar xf 3.1.50.0.tar.gz
cd snort3-3.1.50.0
mkdir build && cd build
cmake .. -DCMAKE_INSTALL_PREFIX=/usr/local
make -j4
sudo make install
```

## Configuring Snort for IPv6

```lua
-- /etc/snort/snort.lua

-- Network variables including IPv6
HOME_NET = '[192.168.0.0/16,10.0.0.0/8,2001:db8::/32,fd00::/8]'
EXTERNAL_NET = '!HOME_NET'

-- IPv6 specific network groups
IPV6_LOOPBACK = '::1'
IPV6_LINK_LOCAL = 'fe80::/10'

-- Server definitions
HTTP_SERVERS = HOME_NET
DNS_SERVERS = '[8.8.8.8, 8.8.4.4, 2001:4860:4860::8888]'

-- Detection engine configuration
detection = {
  hyperscan_literals = true,
  pcre_to_regex = true
}

-- Include rule files
ips =
{
  include = RULE_PATH .. '/snort3-community.rules',
  include = RULE_PATH .. '/local.rules',
}

-- Log settings
alert_csv =
{
  file = true,
  fields = 'timestamp,src_addr,src_port,dst_addr,dst_port,proto,action,msg'
}
```

## IPv6-Specific Detection Rules

```snort
# /etc/snort/rules/local.rules

# Detect IPv6 header manipulation
alert ip6 any any -> $HOME_NET any \
  (msg:"Snort IPv6 Hop-by-Hop Options Header"; \
   ip6_hdr:hopopts; \
   sid:1000001; rev:1;)

# Detect ICMPv6 redirect (possible MITM)
alert icmp6 any any -> $HOME_NET any \
  (msg:"ICMPv6 Redirect Message - Possible Route Hijack"; \
   itype:137; \
   sid:1000002; rev:1;)

# Detect IPv6 tunneling over DNS
alert udp any any -> $DNS_SERVERS 53 \
  (msg:"Possible IPv6 DNS Tunneling"; \
   dsize:>200; \
   content:"AAAA"; \
   sid:1000003; rev:1;)

# Detect potential IPv6 address scanning
alert tcp any any -> $HOME_NET 22 \
  (msg:"SSH Connection Attempt from IPv6"; \
   flags:S; \
   flow:to_server; \
   sid:1000004; rev:1;)
```

## Running Snort on IPv6 Interface

```bash
# Test configuration
sudo snort -c /etc/snort/snort.lua -T

# Run in detection mode on eth0
sudo snort -c /etc/snort/snort.lua -i eth0 -A alert_csv

# Capture to PCAP for analysis
sudo snort -c /etc/snort/snort.lua -i eth0 -L pcap -l /var/log/snort/

# Read from PCAP file
sudo snort -c /etc/snort/snort.lua -r /path/to/capture.pcap
```

## Systemd Service

```ini
# /etc/systemd/system/snort.service
[Unit]
Description=Snort IDS
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/snort -c /etc/snort/snort.lua \
  -i eth0 -A alert_csv \
  -l /var/log/snort/ \
  --daq-dir /usr/local/lib/daq

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

## Analyzing IPv6 Alerts

```bash
# View Snort alert log
sudo cat /var/log/snort/alert_csv.txt

# Filter IPv6 alerts
sudo grep ":" /var/log/snort/alert_csv.txt | head -20

# Use u2boat to convert unified2 to readable format
sudo u2spewfoo /var/log/snort/snort.u2.* | grep -A3 "IPv6"
```

Snort 3's unified rule syntax handles IPv6 addresses and protocol keywords natively, with `HOME_NET` variable configuration being essential to correctly classify IPv6 address space for alert thresholds and severity tuning.

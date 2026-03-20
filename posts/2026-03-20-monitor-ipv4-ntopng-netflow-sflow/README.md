# How to Monitor IPv4 Traffic with ntopng Using NetFlow and sFlow

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ntopng, NetFlow, sFlow, Traffic Monitoring, IPv4, Network Analytics

Description: Learn how to install ntopng and configure it to receive NetFlow and sFlow data for real-time IPv4 traffic monitoring and visualization.

## What Is ntopng?

ntopng is an open-source network traffic monitoring tool that provides a web-based dashboard for real-time and historical traffic analysis. It can receive NetFlow v5/v9, IPFIX, and sFlow data, as well as perform live packet capture via libpcap.

## Step 1: Install ntopng on Ubuntu/Debian

```bash
# Add ntop repository

sudo apt-get install -y software-properties-common curl
curl https://packages.ntop.org/APT/ntop.key | sudo apt-key add -
echo "deb http://packages.ntop.org/apt-stable/20.04/ x64/" | \
  sudo tee /etc/apt/sources.list.d/ntop.list

sudo apt-get update
sudo apt-get install -y ntopng nprobe

# Start ntopng
sudo systemctl enable ntopng
sudo systemctl start ntopng
```

## Step 2: Configure ntopng to Receive NetFlow

Create or edit the ntopng configuration:

```bash
# Edit /etc/ntopng/ntopng.conf
cat > /etc/ntopng/ntopng.conf << 'EOF'
# Network interfaces to monitor
# For NetFlow: use the netflow protocol with port
-i=netflow:0.0.0.0:2055

# For multiple sources (NetFlow + sFlow)
# -i=netflow:0.0.0.0:2055
# -i=sflow:0.0.0.0:6343

# Web interface port
-w=3000

# Data directory for historical data
-d=/var/lib/ntopng

# Enable community edition features
--community

# Set admin password
--auth-password=Ntopng@Adm1n!

EOF

sudo systemctl restart ntopng
```

## Step 3: Configure ntopng for sFlow

To receive sFlow instead of (or in addition to) NetFlow:

```bash
# Add sFlow interface to ntopng configuration
# In /etc/ntopng/ntopng.conf, add or change the -i line:
-i=sflow:0.0.0.0:6343

# Or for both NetFlow and sFlow simultaneously:
-i=netflow:0.0.0.0:2055,sflow:0.0.0.0:6343
```

## Step 4: Configure Network Devices to Send Flows to ntopng

On your Cisco router, configure NetFlow export to ntopng:

```bash
! Send NetFlow v5 to ntopng server
Router(config)# ip flow-export destination 192.168.1.200 2055
Router(config)# ip flow-export version 5
Router(config)# ip flow-export source Loopback0

! Enable on WAN interface
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ip flow ingress
Router(config-if)# ip flow egress
```

## Step 5: Access the ntopng Dashboard

Open your browser and navigate to `http://server-ip:3000`. Login with `admin` and the password you configured.

Key dashboard sections:
- **Top Talkers:** Hosts generating the most traffic
- **Flow Explorer:** Searchable real-time flow table
- **Alerts:** Configured network anomaly notifications
- **Traffic Analysis:** Protocol breakdown and trends

## Step 6: Set Up Traffic Alerts

Configure ntopng to alert on anomalous traffic via the web UI or configuration file:

```bash
# ntopng supports alert recipients via webhook, email, or syslog
# Configure via the web UI under: Settings > Notifications > Alert Endpoints

# Or via the ntopng scripting engine (Lua scripts in /usr/share/ntopng/scripts/)
# Example: alert when a host exceeds 100 Mbps
cat > /usr/share/ntopng/scripts/callbacks/host_callbacks.lua << 'EOF'
function host.callbackHostTrafficAlert(host_info, flow_info)
  local bytes_rx = host_info["bytes.rcvd"]
  local bytes_tx = host_info["bytes.sent"]
  local mbps = (bytes_rx + bytes_tx) * 8 / 1000000
  if mbps > 100 then
    ntop.sendNotification("High traffic alert: " .. host_info["name"] .. " - " .. mbps .. " Mbps")
  end
end
EOF
```

## Step 7: Verify Data Is Flowing

```bash
# Check ntopng is receiving data
sudo netstat -lunp | grep ntopng

# Check for UDP traffic on port 2055
sudo tcpdump -i any udp port 2055 -n -c 5

# View ntopng logs
sudo journalctl -u ntopng -f
```

The ntopng dashboard should show active flows and top talkers within a few seconds of starting the NetFlow/sFlow export.

## Conclusion

ntopng provides a powerful, free web dashboard for IPv4 traffic monitoring using NetFlow and sFlow data. Install it, configure the appropriate interface mode (`netflow:` or `sflow:`), point your routers and switches to send flows to it, and use the web dashboard to analyze top talkers, protocols, and traffic trends. The community edition is free and sufficient for most monitoring needs.

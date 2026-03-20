# How to Configure Factorio Server with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Factorio, IPv6, Game Server, Steam, Linux, Self-Hosted Gaming

Description: Set up and configure a Factorio dedicated server to accept player connections over IPv6, including installation, server settings, and firewall configuration.

---

Factorio is a factory-building game with dedicated server support. The Factorio server can be configured to listen on IPv6 interfaces, enabling players to connect from IPv6-only or dual-stack networks.

## Installing Factorio Dedicated Server

```bash
# Create server directory

mkdir -p /opt/factorio
cd /opt/factorio

# Download headless server (check factorio.com for latest version)
wget https://factorio.com/get-download/stable/headless/linux64 -O factorio.tar.xz
tar xf factorio.tar.xz

# Create server user
sudo useradd -r -m -s /sbin/nologin factorio
sudo chown -R factorio:factorio /opt/factorio
```

## Configuring Factorio Server for IPv6

```bash
# Generate default server settings
sudo -u factorio /opt/factorio/factorio/bin/x64/factorio \
  --create /opt/factorio/saves/my-world.zip

# Generate server-settings.json
sudo -u factorio /opt/factorio/factorio/bin/x64/factorio \
  --create /opt/factorio/saves/my-world.zip \
  --start-server /opt/factorio/saves/my-world.zip \
  --server-settings /opt/factorio/server-settings.json
```

```json
// /opt/factorio/server-settings.json
{
  "name": "My IPv6 Factorio Server",
  "description": "Factorio server accessible over IPv6",
  "tags": ["ipv6", "vanilla"],
  "max_players": 16,
  "visibility": {
    "public": true,
    "lan": false
  },
  "username": "",
  "password": "",
  "token": "",
  "game_password": "",
  "require_user_verification": true,
  "autosave_interval": 10,
  "autosave_slots": 5,
  "afk_autokick_interval": 0,
  "auto_pause": true,
  "allow_commands": "admins-only",
  "autosave_only_on_server": true,
  "non_blocking_saving": false
}
```

## Binding Factorio to IPv6

```bash
# Start Factorio server - it listens on all interfaces by default (including IPv6)
# Use --bind to specify interface
sudo -u factorio /opt/factorio/factorio/bin/x64/factorio \
  --start-server /opt/factorio/saves/my-world.zip \
  --server-settings /opt/factorio/server-settings.json \
  --bind "::" \
  --port 34197

# For specific IPv6 address
sudo -u factorio /opt/factorio/factorio/bin/x64/factorio \
  --start-server /opt/factorio/saves/my-world.zip \
  --bind "2001:db8::1" \
  --port 34197
```

## Systemd Service for Factorio

```ini
# /etc/systemd/system/factorio.service
[Unit]
Description=Factorio Dedicated Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=factorio
WorkingDirectory=/opt/factorio

ExecStart=/opt/factorio/factorio/bin/x64/factorio \
  --start-server /opt/factorio/saves/my-world.zip \
  --server-settings /opt/factorio/server-settings.json \
  --port 34197

Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now factorio
sudo systemctl status factorio
```

## Firewall Rules for Factorio IPv6

```bash
# Factorio uses UDP port 34197 by default
sudo ip6tables -A INPUT -p udp --dport 34197 -j ACCEPT

# Save rules
sudo ip6tables-save > /etc/ip6tables/rules.v6

# Verify Factorio is listening on IPv6
ss -ulnp | grep 34197
```

## Verifying IPv6 Connectivity

```bash
# Check if server is listening on IPv6
ss -6 -ulnp | grep 34197

# Test UDP reachability over IPv6
nmap -6 -sU -p 34197 2001:db8::factorio

# Check server logs
sudo journalctl -u factorio -f

# Confirm IPv6 address assignment
ip -6 addr show scope global
```

## Server Whitelist and Bans over IPv6

```bash
# /opt/factorio/server-whitelist.json
# Add player names (not IPs) - Factorio uses Steam usernames
["PlayerName1", "PlayerName2"]

# Enable whitelist in server-settings.json
# "use_server_whitelist": true

# Check ban list
cat /opt/factorio/server-banlist.json
```

Factorio's dedicated server listens on all network interfaces including IPv6 when bound to `::`, making it straightforward to serve players on IPv6 networks with standard firewall rules for UDP port 34197.

# How to Configure Counter-Strike 2 Server with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Counter-Strike 2, CS2, IPv6, Game Server, SteamCMD, Source Engine

Description: Configure a Counter-Strike 2 dedicated server to accept player connections over IPv6, including SteamCMD installation, server configuration, and network setup.

---

Counter-Strike 2 (CS2) uses Valve's Source 2 engine which supports IPv6. Setting up a CS2 dedicated server on IPv6 requires configuring the SteamCMD installation, server network binding, and firewall rules.

## Installing SteamCMD

```bash
# Ubuntu/Debian

sudo apt install steamcmd -y

# Or manual install
mkdir -p /opt/steamcmd
cd /opt/steamcmd
wget https://steamcdn-a.akamaihd.net/client/installer/steamcmd_linux.tar.gz
tar xvf steamcmd_linux.tar.gz
```

## Installing CS2 Dedicated Server

```bash
# Create server user
sudo useradd -m -s /bin/bash csgo
sudo su - csgo

# Install CS2 server via SteamCMD
steamcmd +force_install_dir /home/csgo/cs2server \
         +login anonymous \
         +app_update 730 validate \
         +quit
```

## CS2 Server Configuration for IPv6

```bash
# /home/csgo/cs2server/game/csgo/cfg/server.cfg

hostname "My IPv6 CS2 Server"
sv_password ""

# Network settings
sv_lan 0
sv_maxrate 0

# Cheats/security
sv_cheats 0
sv_allowupload 0
sv_allowdownload 1

# Game settings
mp_friendlyfire 0
mp_autoteambalance 1

# Tick rate
sv_minrate 64000
sv_maxrate 786432
sv_mincmdrate 64
sv_maxcmdrate 64
sv_minupdaterate 64
sv_maxupdaterate 64
```

## Starting CS2 Server with IPv6

```bash
# Start CS2 server binding to IPv6
/home/csgo/cs2server/game/bin/linuxsteamrt64/cs2 \
  -dedicated \
  -usercon \
  -console \
  +ip "::" \
  +port 27015 \
  +game_type 0 \
  +game_mode 1 \
  +map de_dust2 \
  +sv_setsteamaccount YOUR_GAME_SERVER_TOKEN

# Force specific IPv6 address
/home/csgo/cs2server/game/bin/linuxsteamrt64/cs2 \
  -dedicated \
  +ip "2001:db8::1" \
  +port 27015 \
  +map de_dust2
```

## Systemd Service

```ini
# /etc/systemd/system/cs2server.service
[Unit]
Description=Counter-Strike 2 Dedicated Server
After=network-online.target

[Service]
Type=simple
User=csgo
WorkingDirectory=/home/csgo/cs2server

ExecStart=/home/csgo/cs2server/game/bin/linuxsteamrt64/cs2 \
  -dedicated \
  -console \
  +ip "::" \
  +port 27015 \
  +game_type 0 +game_mode 1 \
  +map de_dust2

Restart=on-failure
RestartSec=15

[Install]
WantedBy=multi-user.target
```

## Firewall Rules for CS2 IPv6

```bash
# CS2 game port (UDP)
sudo ip6tables -A INPUT -p udp --dport 27015 -j ACCEPT
# CS2 TCP port (for RCON and some connections)
sudo ip6tables -A INPUT -p tcp --dport 27015 -j ACCEPT
# Steam master server communication
sudo ip6tables -A INPUT -p udp --dport 27005 -j ACCEPT

# RCON port (restrict to admin IPs)
sudo ip6tables -A INPUT -p tcp -s 2001:db8::admin --dport 27015 -j ACCEPT

sudo ip6tables-save > /etc/ip6tables/rules.v6
```

## DNS Configuration for IPv6 CS2

```bash
# Add AAAA record for CS2 server
# cs2.example.com. IN AAAA 2001:db8::cs2

# Players connect using:
# connect cs2.example.com:27015

# Verify server is responding
nmap -6 -sU -p 27015 2001:db8::cs2
```

## Game Server Account Token

CS2 requires a Steam Game Server Account Token:

```bash
# Get token at: https://steamcommunity.com/dev/managegameservers
# App ID for CS2: 730

# Use token in startup:
# +sv_setsteamaccount YOUR_TOKEN_HERE

# Verify token is working
grep "Token authentication" /home/csgo/cs2server/game/logs/cs2_*.log
```

CS2 servers support IPv6 through the `+ip` startup parameter, enabling competitive gaming infrastructure accessible to players on IPv6-only connections, particularly relevant as ISP IPv6 deployment continues to grow.

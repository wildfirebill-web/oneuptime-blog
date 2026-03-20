# How to Configure ARK Server with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ARK Survival Evolved, IPv6, Game Server, Steam, Linux, Self-Hosted Gaming

Description: Set up and configure an ARK: Survival Evolved dedicated server to support IPv6 player connections, covering SteamCMD installation, GameUserSettings, and firewall rules.

---

ARK: Survival Evolved (and ARK: Survival Ascended) uses Unreal Engine networking. The dedicated server can be configured to accept player connections over IPv6 through proper system-level binding and firewall configuration.

## Installing ARK Dedicated Server

```bash
# Create server user
sudo useradd -m -s /bin/bash ark
sudo su - ark

# Install via SteamCMD
steamcmd +force_install_dir /home/ark/arkserver \
         +login anonymous \
         +app_update 376030 validate \
         +quit

# List server files
ls /home/ark/arkserver/
```

## Configuring ARK Server Settings

```ini
# /home/ark/arkserver/ShooterGame/Saved/Config/LinuxServer/GameUserSettings.ini

[ServerSettings]
ServerPassword=
ServerAdminPassword=YourAdminPassword
MaxPlayers=70

[SessionSettings]
SessionName=My IPv6 ARK Server

[/Script/ShooterGame.ShooterGameUserSettings]
MaxPlayers=70

[MessageOfTheDay]
Message=Welcome to our IPv6 ARK server!
Duration=20
```

## Starting ARK Server on IPv6

```bash
# ARK uses a startup script
# /home/ark/start_ark.sh
#!/bin/bash

ARK_SERVER="/home/ark/arkserver"
SAVES="/home/ark/arkserver/ShooterGame/Saved"

"$ARK_SERVER/ShooterGame/Binaries/Linux/ShooterGameServer" \
  TheIsland?listen?SessionName="IPv6 ARK" \
  ?ServerPassword="" \
  ?ServerAdminPassword="YourAdminPassword" \
  ?MaxPlayers=70 \
  -server \
  -log \
  -NoBattlEye \
  -Port=7777 \
  -QueryPort=27015 \
  -RCONPort=27020

# ARK binds to all interfaces (including IPv6) automatically
```

## Systemd Service for ARK

```ini
# /etc/systemd/system/ark.service
[Unit]
Description=ARK Dedicated Server
After=network-online.target

[Service]
Type=simple
User=ark
WorkingDirectory=/home/ark/arkserver

ExecStartPre=/usr/games/steamcmd \
  +force_install_dir /home/ark/arkserver \
  +login anonymous \
  +app_update 376030 validate \
  +quit

ExecStart=/home/ark/arkserver/ShooterGame/Binaries/Linux/ShooterGameServer \
  TheIsland?listen?SessionName="IPv6 ARK" \
  ?MaxPlayers=70 \
  ?ServerAdminPassword=YourAdminPassword \
  -server -log \
  -Port=7777 \
  -QueryPort=27015

Restart=on-failure
RestartSec=30
KillSignal=SIGINT
TimeoutStopSec=300

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ark
```

## Firewall Rules for ARK IPv6

```bash
# ARK uses the following ports
# Game port: 7777 UDP
# Query port: 27015 UDP
# RCON port: 27020 TCP

sudo ip6tables -A INPUT -p udp --dport 7777 -j ACCEPT
sudo ip6tables -A INPUT -p udp --dport 27015 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 27020 -j ACCEPT

# Save IPv6 rules
sudo ip6tables-save > /etc/ip6tables/rules.v6
```

## Verifying IPv6 Connectivity

```bash
# Check if ARK is listening on IPv6
ss -6 -ulnp | grep -E "7777|27015"

# Check server logs for IPv6 connections
sudo journalctl -u ark -f | grep -i "connect\|ipv6"

# Test port reachability
nmap -6 -sU -p 7777,27015 2001:db8::ark
```

## ARK Auto-Update Script

```bash
#!/bin/bash
# ark_update.sh

echo "Stopping ARK server..."
sudo systemctl stop ark

echo "Updating ARK..."
/usr/games/steamcmd \
  +force_install_dir /home/ark/arkserver \
  +login anonymous \
  +app_update 376030 validate \
  +quit

echo "Starting ARK server..."
sudo systemctl start ark
echo "ARK server started. Status:"
sudo systemctl status ark --no-pager
```

ARK dedicated servers listen on all network interfaces by default, enabling IPv6 connections from players without additional binding configuration beyond ensuring IPv6 firewall rules permit the game and query ports.

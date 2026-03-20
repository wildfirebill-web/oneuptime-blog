# How to Deploy a Team Fortress 2 Server via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Team Fortress 2, TF2, Game Server, Portainer, Docker, Gaming, Steam

Description: Deploy a Team Fortress 2 dedicated server using Portainer with SourceMod plugin support, custom maps, and automatic updates via SteamCMD.

---

Team Fortress 2 is a free-to-play class-based shooter with a vibrant community. Running your own TF2 server with Portainer lets you customize game modes, install plugins, and build a community around your server.

## Step 1: Deploy TF2 Server via Portainer Stack

```yaml
# tf2-stack.yml

version: "3.8"

services:
  tf2:
    image: cm2network/tf2:latest
    environment:
      # Server settings passed as startup args
      - SRCDS_RCONPW=rcon_password_here
      - SRCDS_PW=join_password_here
      - SRCDS_PORT=27015
      - SRCDS_TV_PORT=27020
      - SRCDS_NET_PUBLIC_ADDRESS=0.0.0.0
      - SRCDS_IP=0.0.0.0
      - SRCDS_MAXPLAYERS=24
      - SRCDS_MAP=cp_dustbowl
      # Game mode (ctf, cp, payload, koth, arena, mvm)
      - SRCDS_GAMETYPE=0
      - SRCDS_GAMEMODE=0
    volumes:
      - tf2-data:/home/steam/tf-dedicated
    ports:
      - "27015:27015/tcp"    # SRCDS TCP
      - "27015:27015/udp"    # SRCDS UDP
      - "27020:27020/udp"    # SourceTV
    restart: unless-stopped

volumes:
  tf2-data:
```

## Step 2: Configure Server with server.cfg

Create a server configuration file in the TF2 data volume:

```bash
# /home/steam/tf-dedicated/tf/cfg/server.cfg
hostname "My TF2 Server | Portainer"
sv_password ""                  # Empty = public server
rcon_password "rcon_password_here"
sv_lan 0
sv_region 1                     # 0=US East, 1=US West, 2=SA, 3=EU, 4=Asia, 5=AU
mp_timelimit 30
mp_maxrounds 5
sv_alltalk 0
sv_cheats 0
tf_weapon_criticals 1
log on
sv_logbans 1
sv_logecho 1
sv_logfile 1
sv_log_onefile 0
```

## Step 3: Install SourceMod and MetaMod

SourceMod is the plugin framework for Source engine games:

```bash
# Run these commands in the TF2 container or place files in the volume

# Download and install MetaMod:Source
cd /home/steam/tf-dedicated/tf
wget https://mms.alliedmods.net/mmsdrop/1.11/mmsource-1.11.0-git1148-linux.tar.gz
tar -xzf mmsource-1.11.0-git1148-linux.tar.gz

# Download and install SourceMod
wget https://sm.alliedmods.net/smdrop/1.12/sourcemod-1.12.0-git6974-linux.tar.gz
tar -xzf sourcemod-1.12.0-git6974-linux.tar.gz
```

## Step 4: Add Custom Maps

Place custom BSP map files in the maps directory:

```bash
# Mount a custom maps directory
volumes:
  - tf2-data:/home/steam/tf-dedicated
  - /opt/tf2/custom-maps:/home/steam/tf-dedicated/tf/maps
```

Create a mapcycle.txt to rotate through your maps:

```text
cp_dustbowl
ctf_2fort
pl_badwater
koth_nucleus
cp_granary
```

## Step 5: Monitor via RCON

Use the RCON protocol to send admin commands:

```bash
# Install rcon client
apt-get install -y mcrcon

# Send commands via RCON
rcon -H localhost -P 27015 -p rcon_password_here status
rcon -H localhost -P 27015 -p rcon_password_here changelevel ctf_2fort
rcon -H localhost -P 27015 -p rcon_password_here sm plugins list
```

## Summary

TF2 dedicated servers on Portainer give community managers full control over game settings, plugin configuration, and map rotation. The container approach means easy updates, consistent configurations, and simple backup/restore of server data.

# How to Deploy a Rust Game Server via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rust, Game Server, Portainer, Docker, Self-Hosted, Gaming, Steam

Description: Deploy a dedicated Rust game server using Portainer with persistent world data, uMod plugin support, and configurable map seeds for your gaming community.

---

Rust is a multiplayer survival game known for its brutal gameplay and active modding community. Running your own server via Portainer gives you full control over wipe schedules, map seeds, and plugin configurations.

## Step 1: Deploy Rust Server Stack

```yaml
# rust-stack.yml
version: "3.8"

services:
  rust:
    image: didstopia/rust-server:latest
    environment:
      # Server identity
      - RUST_SERVER_STARTUP_ARGUMENTS=-batchmode -load +server.port 28015 +server.queryport 28016
      - RUST_SERVER_NAME=My Rust Server
      - RUST_SERVER_DESCRIPTION=A Rust server powered by Portainer
      - RUST_SERVER_URL=https://example.com
      - RUST_SERVER_BANNER_URL=https://example.com/banner.png
      # Connection settings
      - RUST_SERVER_PORT=28015
      - RUST_QUERY_PORT=28016
      - RUST_RCON_PORT=28016
      - RUST_RCON_PASSWORD=rcon_password_here
      - RUST_SERVER_MAXPLAYERS=100
      # World settings
      - RUST_SERVER_WORLDSIZE=4000    # Map size in meters
      - RUST_SERVER_SEED=12345        # Random seed for world generation
      - RUST_SERVER_SAVEINTERVAL=300  # Save every 5 minutes
      # uMod/Oxide plugin framework
      - RUST_OXIDE_ENABLED=1
      # Auto-update
      - RUST_UPDATE_CHECKING=1
      - RUST_UPDATE_BRANCH=public
    volumes:
      - rust-data:/steamcmd/rust
    ports:
      - "28015:28015/tcp"
      - "28015:28015/udp"
      - "28016:28016/udp"    # Query port
    restart: unless-stopped

volumes:
  rust-data:
```

## Step 2: Configure the Server

After first boot, edit `/steamcmd/rust/server/rust/cfg/server.cfg`:

```bash
# server.cfg
server.hostname "My Rust Server"
server.description "A community Rust server. Join our Discord!"
server.url "https://discord.gg/yourserver"
server.maxplayers 100
server.worldsize 4000
server.seed 12345

# Gameplay settings
server.pvp true
decay.scale 0.5              # Half decay rate
airdrop.min_players 25       # Airdrops start at 25 players

# Anti-cheat
antihack.enabled true
antihack.speedhack_protection 1
```

## Step 3: Install uMod (Oxide) Plugins

Place plugin `.cs` files in the plugins directory:

```bash
# The plugins directory is at /steamcmd/rust/oxide/plugins/
# Download popular plugins

# Kits — predefined item kits
wget -O /var/lib/docker/volumes/rust-data/_data/oxide/plugins/Kits.cs \
  https://umod.org/plugins/Kits.cs

# Clans — team/clan system
wget -O /var/lib/docker/volumes/rust-data/_data/oxide/plugins/Clans.cs \
  https://umod.org/plugins/Clans.cs
```

Plugins auto-reload when files are placed in the directory.

## Step 4: Set Up Scheduled Wipes

Rust servers typically wipe on a schedule. Create a wipe script as a Portainer scheduled job:

```bash
#!/bin/bash
# rust-wipe.sh — run monthly via Portainer scheduled job

# Stop server gracefully
docker stop rust_rust_1

# Delete world data (keeps player data/blueprints)
rm -f /var/lib/docker/volumes/rust-data/_data/server/rust/user.seed.map
rm -f /var/lib/docker/volumes/rust-data/_data/server/rust/user.seed.db

# Update the map seed for the new wipe
# This triggers a fresh procedural map generation on next start

# Restart server
docker start rust_rust_1
echo "Wipe completed. Server restarting."
```

## Step 5: Monitor via RCON

```bash
# Connect to RCON and run server commands
rcon -H localhost -P 28016 -p rcon_password_here status
rcon -H localhost -P 28016 -p rcon_password_here server.save
rcon -H localhost -P 28016 -p rcon_password_here say "Server will restart in 10 minutes!"
```

## Summary

Rust dedicated servers via Portainer give gaming communities a manageable, configurable server environment. The containerized approach makes wipe management, plugin updates, and server restarts straightforward, and Portainer's persistent volumes ensure world data survives container rebuilds.

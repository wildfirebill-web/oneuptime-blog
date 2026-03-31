# How to Install Redis on Windows (WSL)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Installation, Windows, WSL, Development

Description: Install and run Redis on Windows using Windows Subsystem for Linux (WSL2), with setup steps, service configuration, and connecting from Windows applications.

---

Redis does not have an official native Windows build for modern versions. The recommended approach on Windows is to use Windows Subsystem for Linux (WSL2), which gives you a full Linux environment where Redis installs and runs just like on a Linux server.

## Step 1: Install WSL2

Open PowerShell as Administrator and run:

```powershell
wsl --install
```

This installs WSL2 with Ubuntu by default. Restart your computer when prompted.

If you already have WSL, ensure you are on version 2:

```powershell
wsl --set-default-version 2
wsl --list --verbose
```

## Step 2: Open Your WSL Terminal

Launch Ubuntu from the Start menu, or open a Windows Terminal and select the Ubuntu profile.

## Step 3: Install Redis

```bash
sudo apt-get update
sudo apt-get install -y redis-server
```

Verify the installation:

```bash
redis-server --version
# Redis server v=7.x.x
```

## Step 4: Start Redis

```bash
# Start Redis in the background
sudo service redis-server start

# Verify it's running
sudo service redis-server status

# Test with redis-cli
redis-cli ping
# PONG
```

## Step 5: Configure Redis to Start Automatically

WSL2 does not run systemd by default (though newer Ubuntu versions on WSL2 support it). Use `/etc/wsl.conf` to enable systemd:

```bash
sudo tee -a /etc/wsl.conf > /dev/null <<'EOF'
[boot]
systemd=true
EOF
```

Restart WSL from PowerShell:

```powershell
wsl --shutdown
```

Then open WSL again and enable the Redis service:

```bash
sudo systemctl enable redis-server
sudo systemctl start redis-server
```

## Step 6: Connect from Windows Applications

Redis running inside WSL2 is accessible from Windows at `localhost` on port 6379 by default. Test this from PowerShell:

```powershell
# If you have redis-cli for Windows (from Redis Stack installer)
redis-cli -h 127.0.0.1 -p 6379 ping
```

Or connect from your Windows application:

```python
import redis
r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)
print(r.ping())
```

## Editing the Redis Configuration

```bash
sudo nano /etc/redis/redis.conf
```

Key settings for development:

```text
bind 127.0.0.1
port 6379
save ""
appendonly no
loglevel notice
```

After changes:

```bash
sudo service redis-server restart
# or with systemd:
sudo systemctl restart redis-server
```

## Using Docker Desktop Instead

An alternative to WSL2 direct installation is Docker Desktop for Windows, which also runs on WSL2:

```powershell
docker run -d -p 6379:6379 --name redis redis:7
```

This avoids managing the Linux service manually and works well for development.

## Summary

Install Redis on Windows by enabling WSL2 with Ubuntu, then using `apt-get install redis-server` inside the Linux environment. Enable systemd in WSL2 for automatic startup via `/etc/wsl.conf`. Redis inside WSL2 is accessible from Windows applications at `localhost:6379`, making it transparent to your development tools.

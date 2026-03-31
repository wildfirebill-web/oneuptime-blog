# How to Install ClickHouse on Windows with WSL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Windows, WSL, Development, Ubuntu

Description: Install ClickHouse on Windows using WSL2 with Ubuntu, configure port forwarding for Windows host access, and set up a local development environment.

---

Windows Subsystem for Linux 2 (WSL2) provides a full Linux environment on Windows, making it possible to run ClickHouse for local development. This guide uses Ubuntu 22.04 LTS inside WSL2.

## Setting Up WSL2

```powershell
# Run in PowerShell as Administrator
wsl --install -d Ubuntu-22.04
wsl --set-default-version 2
```

After installation, launch Ubuntu from the Start menu and set up your username and password.

## Installing ClickHouse in WSL2

Inside the Ubuntu WSL2 terminal:

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL 'https://packages.clickhouse.com/deb/archive/apt/stable.sources' | \
  sudo tee /etc/apt/sources.list.d/clickhouse.sources
sudo apt-get update && sudo apt-get install -y clickhouse-server clickhouse-client
```

## Starting ClickHouse in WSL2

WSL2 does not have systemd enabled by default in older configurations. Start the server manually:

```bash
sudo clickhouse-server --config-file=/etc/clickhouse-server/config.xml &
```

Or enable systemd in WSL2 (Ubuntu 22.04+ supports this):

```bash
# Edit /etc/wsl.conf
sudo tee /etc/wsl.conf << 'EOF'
[boot]
systemd=true
EOF
```

Restart WSL from PowerShell:

```powershell
wsl --shutdown
wsl
```

Then use systemctl:

```bash
sudo systemctl enable --now clickhouse-server
```

## Accessing ClickHouse from Windows

ClickHouse running in WSL2 is accessible from Windows at the WSL2 IP address. Find it with:

```bash
hostname -I
```

Alternatively, configure port forwarding so `localhost` on Windows reaches WSL2:

```powershell
# In PowerShell as Administrator
netsh interface portproxy add v4tov4 listenport=8123 listenaddress=127.0.0.1 connectport=8123 connectaddress=<WSL2-IP>
netsh interface portproxy add v4tov4 listenport=9000 listenaddress=127.0.0.1 connectport=9000 connectaddress=<WSL2-IP>
```

## Configuring ClickHouse to Listen on All Interfaces

By default, ClickHouse listens on `::1` (IPv6 localhost). To allow access from Windows:

```xml
<clickhouse>
  <listen_host>0.0.0.0</listen_host>
</clickhouse>
```

## Using ClickHouse from Windows Applications

With port forwarding configured, connect from Windows tools:

```bash
# DBeaver, DataGrip, or any SQL client
Host: localhost
Port: 8123 (HTTP) or 9000 (native)
Database: default
User: default
Password: (blank by default)
```

Using Python on Windows:

```python
import clickhouse_connect
client = clickhouse_connect.get_client(host='localhost', port=8123)
print(client.query('SELECT version()').first_row)
```

## Persistent Storage

WSL2 mounts the Windows filesystem under `/mnt/c`. Avoid storing ClickHouse data there for performance reasons. The default `/var/lib/clickhouse` inside the Linux filesystem is much faster.

## Summary

Installing ClickHouse on Windows with WSL2 uses the standard Ubuntu APT package installation inside a WSL2 instance. Enable systemd in `/etc/wsl.conf` for proper service management, configure ClickHouse to listen on `0.0.0.0`, and set up Windows port forwarding to access ClickHouse at `localhost` from Windows applications.

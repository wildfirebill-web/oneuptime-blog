# How to Install ClickHouse on Arch Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Arch Linux, Linux, Installation, AUR

Description: Install ClickHouse on Arch Linux using the AUR or official binary, with systemd service configuration and initial setup steps.

---

Arch Linux does not include ClickHouse in the official repositories, but it is available through the AUR (Arch User Repository) or by using the official binary distribution. This guide covers both approaches.

## Option 1: Install from AUR

The `clickhouse` package in the AUR builds ClickHouse from source or packages the official binary. Use an AUR helper like `paru` or `yay`:

```bash
# Using paru
paru -S clickhouse

# Using yay
yay -S clickhouse
```

The AUR package creates the `clickhouse` user, sets up systemd services, and places configuration files in `/etc/clickhouse-server/`.

## Option 2: Install Official Binary

Download the official pre-compiled binary directly:

```bash
curl https://clickhouse.com/ | sh
sudo mv clickhouse /usr/local/bin/
```

Create the system user and directories:

```bash
sudo useradd -r -s /sbin/nologin -d /var/lib/clickhouse clickhouse
sudo mkdir -p /var/lib/clickhouse /var/log/clickhouse-server /etc/clickhouse-server
sudo chown -R clickhouse:clickhouse /var/lib/clickhouse /var/log/clickhouse-server
```

## Creating a systemd Service

If you used the binary install, create a systemd unit file:

```bash
sudo tee /etc/systemd/system/clickhouse-server.service << 'EOF'
[Unit]
Description=ClickHouse Server
After=network.target

[Service]
Type=simple
User=clickhouse
Group=clickhouse
ExecStart=/usr/local/bin/clickhouse server --config-file=/etc/clickhouse-server/config.xml
Restart=on-failure
LimitNOFILE=262144

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now clickhouse-server
```

## Basic Configuration

Create a minimal config for the binary installation:

```xml
<clickhouse>
  <logger>
    <level>warning</level>
    <log>/var/log/clickhouse-server/clickhouse-server.log</log>
    <errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>
  </logger>
  <http_port>8123</http_port>
  <tcp_port>9000</tcp_port>
  <listen_host>127.0.0.1</listen_host>
  <path>/var/lib/clickhouse/</path>
  <tmp_path>/var/lib/clickhouse/tmp/</tmp_path>
</clickhouse>
```

## Verifying the Installation

```bash
clickhouse-client --query "SELECT version()"
```

Or using the binary:

```bash
/usr/local/bin/clickhouse client --query "SELECT version()"
```

## Pacman Configuration

For the AUR package, you can prevent accidental upgrades by adding ClickHouse to the `IgnorePkg` list in `/etc/pacman.conf`:

```text
IgnorePkg = clickhouse
```

This is important for production systems where you want to control upgrade timing.

## Performance Tuning

Arch Linux uses a rolling release model, so the kernel is usually recent. Set `vm.max_map_count` persistently:

```bash
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-clickhouse.conf
sudo sysctl --system
```

## Summary

Installing ClickHouse on Arch Linux works best via the AUR using `paru` or `yay`, which handles user creation, configuration files, and systemd integration. Alternatively, download the official binary and create the systemd unit and user manually. Set `vm.max_map_count` via sysctl for production performance.

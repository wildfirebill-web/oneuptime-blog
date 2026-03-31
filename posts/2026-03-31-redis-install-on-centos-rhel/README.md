# How to Install Redis on CentOS/RHEL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, CentOS, RHEL, Linux, Installation, System Administration

Description: Install and configure Redis on CentOS and RHEL using the Remi repository for the latest version, with systemd service setup, firewall configuration, and basic hardening.

---

## Prerequisites

- CentOS 7/8/9 or RHEL 7/8/9
- Root or sudo access
- Internet connectivity

## Install Redis from Remi Repository (Recommended)

The default CentOS/RHEL repositories often ship older Redis versions. The Remi repository provides the latest stable release.

### CentOS/RHEL 9

```bash
# Install EPEL repository
sudo dnf install -y epel-release

# Install Remi repository
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm

# Enable the Redis module from Remi
sudo dnf module reset redis
sudo dnf module enable redis:remi-7.2

# Install Redis
sudo dnf install -y redis

# Verify installation
redis-server --version
```

### CentOS/RHEL 8

```bash
sudo dnf install -y epel-release
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo dnf module reset redis
sudo dnf module enable redis:remi-7.2
sudo dnf install -y redis
```

### CentOS/RHEL 7

```bash
sudo yum install -y epel-release
sudo yum install -y https://rpms.remirepo.net/enterprise/remi-release-7.rpm
sudo yum --enablerepo=remi install -y redis
```

## Start and Enable Redis Service

```bash
# Start Redis
sudo systemctl start redis

# Enable on boot
sudo systemctl enable redis

# Check status
sudo systemctl status redis
```

## Verify Redis is Running

```bash
redis-cli ping
# PONG

redis-cli info server | grep redis_version
```

## Basic Configuration

Edit `/etc/redis/redis.conf`:

```bash
sudo vi /etc/redis/redis.conf
```

Key settings to adjust:

```text
# Bind to specific interface (restrict access)
bind 127.0.0.1

# Set a password
requirepass your_strong_password_here

# Max memory limit (e.g. 2GB)
maxmemory 2gb
maxmemory-policy allkeys-lru

# Disable persistence for pure cache use
save ""
appendonly no

# Set log level
loglevel notice
logfile /var/log/redis/redis.log
```

After changes, restart Redis:

```bash
sudo systemctl restart redis
```

## Configure Firewall

If Redis needs to be accessible remotely (not recommended without TLS), open port 6379:

```bash
sudo firewall-cmd --permanent --add-port=6379/tcp
sudo firewall-cmd --reload
```

For local-only (recommended default), do NOT open the firewall port.

## Tune OS Settings for Redis

```bash
# Disable THP (Transparent Huge Pages)
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled

# Make THP disable permanent
sudo bash -c 'cat >> /etc/rc.d/rc.local << EOF
echo never > /sys/kernel/mm/transparent_hugepage/enabled
EOF'
sudo chmod +x /etc/rc.d/rc.local

# Increase somaxconn
sudo sysctl -w net.core.somaxconn=65535
echo "net.core.somaxconn = 65535" | sudo tee -a /etc/sysctl.conf

# Set overcommit_memory
sudo sysctl -w vm.overcommit_memory=1
echo "vm.overcommit_memory = 1" | sudo tee -a /etc/sysctl.conf
```

## Configure Redis as a Service with Limits

Edit the systemd override:

```bash
sudo systemctl edit redis
```

Add:

```text
[Service]
LimitNOFILE=65535
LimitNPROC=65535
```

Reload and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart redis
```

## Verify Memory and Connection Settings

```bash
redis-cli -a your_strong_password_here INFO server | grep -E "redis_version|tcp_port|maxmemory"
redis-cli -a your_strong_password_here CONFIG GET maxmemory
redis-cli -a your_strong_password_here CONFIG GET bind
```

## SELinux Configuration

If SELinux is enabled and Redis fails to start:

```bash
# Check SELinux denials
sudo ausearch -m avc -ts recent | grep redis

# Allow Redis to bind to its port
sudo setsebool -P redis_can_network_connect on
sudo semanage port -a -t redis_port_t -p tcp 6379
```

## Summary

Installing Redis on CentOS/RHEL using the Remi repository provides access to the latest stable release. After installation, configure the bind address, authentication password, and memory limits in `/etc/redis/redis.conf`, and apply OS-level tuning for THP, somaxconn, and overcommit_memory. Enable the systemd service and optionally restrict firewall access to keep Redis only accessible locally.

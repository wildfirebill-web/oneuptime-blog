# How to Install Redis on Amazon Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Installation, Amazon Linux, AWS, EC2

Description: Install and configure Redis on Amazon Linux 2 and Amazon Linux 2023 using the package manager or Remi repository, with systemd service setup for production use.

---

Amazon Linux is the default OS for many EC2 instances. Installing Redis on Amazon Linux 2 or Amazon Linux 2023 requires using either the Amazon Linux Extras repository (AL2) or a third-party repository for recent Redis versions, since the default yum repos ship older packages.

## Amazon Linux 2023

Amazon Linux 2023 includes Redis via the standard package manager:

```bash
sudo dnf install -y redis6

# Verify
redis-server --version
# Redis server v=6.2.x
```

For Redis 7.x on AL2023, install from the Remi repository:

```bash
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm
sudo dnf module enable -y redis:remi-7.2
sudo dnf install -y redis

redis-server --version
# Redis server v=7.2.x
```

## Amazon Linux 2

```bash
# Enable the Redis extra
sudo amazon-linux-extras enable redis6

# Install
sudo yum install -y redis

redis-server --version
```

## Start and Enable Redis as a Service

```bash
sudo systemctl start redis
sudo systemctl enable redis
sudo systemctl status redis
```

## Verify the Installation

```bash
redis-cli ping
# PONG

redis-cli info server | grep redis_version
```

## Configure Redis for Production Use

Edit the configuration file:

```bash
sudo nano /etc/redis/redis.conf
# On AL2 the path may be /etc/redis.conf
```

Key production settings:

```text
# Bind to a specific interface
bind 127.0.0.1

# Set a strong password
requirepass YourStrongPassword

# Memory limit
maxmemory 1gb
maxmemory-policy allkeys-lru

# Disable save for a pure cache (re-enable for persistence)
save ""

# Log to file
logfile /var/log/redis/redis.log
loglevel notice
```

Restart after configuration changes:

```bash
sudo systemctl restart redis
```

## Open the Firewall Port (If Needed)

If your application runs on a different host, allow Redis traffic through the security group and firewall:

```bash
# Allow Redis port from within the VPC
sudo firewall-cmd --permanent --add-port=6379/tcp
sudo firewall-cmd --reload
```

In production, Redis should never be exposed to the public internet. Use security groups to restrict access to your application's IP or VPC CIDR only.

## Disable Transparent Huge Pages

THP causes latency spikes in Redis. Disable it permanently:

```bash
sudo tee /etc/rc.d/rc.local > /dev/null <<'EOF'
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
EOF
sudo chmod +x /etc/rc.d/rc.local
sudo /etc/rc.d/rc.local
```

## Installing via Building from Source

If you need a newer version not available in any repository, build from source:

```bash
sudo yum groupinstall -y "Development Tools"
wget https://download.redis.io/releases/redis-7.2.4.tar.gz
tar xzf redis-7.2.4.tar.gz
cd redis-7.2.4
make -j$(nproc)
sudo make install
```

## Summary

On Amazon Linux 2023, install Redis using `dnf install redis6` or enable the Remi repository for Redis 7.x. On Amazon Linux 2, use `amazon-linux-extras enable redis6`. Always set a password, configure `maxmemory`, restrict the bind address to localhost or a private interface, and disable Transparent Huge Pages for consistent low-latency performance in production.

# How to Configure MySQL to Listen on All IPv4 Interfaces

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, IPv4, bind-address, All Interfaces, Database, Configuration

Description: Set MySQL bind-address to 0.0.0.0 to listen on all IPv4 interfaces, while using firewall rules and user grants to control actual access.

## Introduction

Setting `bind-address = 0.0.0.0` makes MySQL accept connections on all network interfaces. This is common in multi-homed servers or containerized environments. Access control is then managed through MySQL user grants and firewall rules—not by binding to a specific IP.

## Configuration

```bash
# /etc/mysql/mysql.conf.d/mysqld.cnf (Debian/Ubuntu)
# /etc/my.cnf (RHEL/CentOS)

[mysqld]
# Listen on all IPv4 interfaces
bind-address = 0.0.0.0

# Disable name resolution for faster connections
skip-name-resolve = ON

# Optional: specify port
port = 3306
```

```bash
# Restart MySQL
sudo systemctl restart mysql

# Verify it's listening on all interfaces
sudo ss -tlnp | grep 3306
# Expected: 0.0.0.0:3306

sudo netstat -tlnp | grep mysqld
# Expected: tcp  0.0.0.0:3306  0.0.0.0:*  LISTEN
```

## MySQL User Grants for Access Control

```bash
# With bind-address = 0.0.0.0, control access through user grants:
sudo mysql -u root -p

-- Specific host only
CREATE USER 'app'@'10.0.0.5' IDENTIFIED BY 'password';
GRANT ALL ON appdb.* TO 'app'@'10.0.0.5';

-- Subnet access
CREATE USER 'webuser'@'192.168.1.%' IDENTIFIED BY 'webpass';
GRANT SELECT, INSERT ON webdb.* TO 'webuser'@'192.168.1.%';

-- Restrict root to localhost only (security)
DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1');
FLUSH PRIVILEGES;

-- Check user hosts
SELECT User, Host FROM mysql.user ORDER BY User;
```

## Firewall Protection

```bash
# With 0.0.0.0 binding, firewall is critical
# Allow MySQL only from trusted networks:

sudo ufw allow from 10.0.0.0/8 to any port 3306
sudo ufw deny 3306

# iptables with logging
sudo iptables -A INPUT -p tcp --dport 3306 -s 10.0.0.0/8 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 3306 -j LOG --log-prefix "MYSQL-BLOCKED: "
sudo iptables -A INPUT -p tcp --dport 3306 -j DROP
```

## Verifying All Interface Listening

```bash
# Check which interfaces MySQL listens on
sudo ss -tlnp | grep mysql

# Check from multiple interfaces
mysql -h 10.0.0.5 -u app -p appdb      # Via internal IP
mysql -h 127.0.0.1 -u root -p          # Via loopback
mysql -h 203.0.113.10 -u app -p appdb  # Via public IP (if granted)

# View current connections
sudo mysql -e "SHOW PROCESSLIST;"
```

## Conclusion

`bind-address = 0.0.0.0` enables MySQL to accept connections on all interfaces. Access control shifts to MySQL user grants (restrict `Host` to specific IPs or subnets) and firewall rules. Add `skip-name-resolve` to avoid DNS lookups on each connection. Never expose MySQL on `0.0.0.0` without firewall rules blocking port 3306 from untrusted networks.

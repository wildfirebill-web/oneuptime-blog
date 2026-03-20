# How to Troubleshoot MySQL "Can't Connect" Errors on IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, IPv4, Troubleshooting, Connection Refused, Debug, Database

Description: Diagnose and fix MySQL connection failures including "Can't connect to MySQL server", "Host not allowed", and "Access denied" errors on IPv4 networks.

## Introduction

MySQL connection failures fall into three categories: network issues (can't reach the server), binding issues (MySQL isn't listening on the right IP), and permission issues (wrong user/host grant). This guide covers all three systematically.

## Error Categories and Meanings

| Error | Likely Cause |
|---|---|
| `Can't connect to MySQL server on '...' (111)` | MySQL not running or not listening on that IP |
| `Can't connect to MySQL server on '...' (110)` | Firewall blocking port 3306 |
| `Host '...' is not allowed to connect` | Missing user grant for that IP |
| `Access denied for user '...'@'...'` | Wrong password or missing privileges |

## Step-by-Step Diagnosis

```bash
# Step 1: Is MySQL running?
sudo systemctl status mysql
sudo systemctl start mysql   # If not running

# Step 2: Is MySQL listening on the right IP?
sudo ss -tlnp | grep 3306
# Expected: 0.0.0.0:3306 or your_server_ip:3306
# If only 127.0.0.1:3306, need to change bind-address

# Step 3: Can you reach the port?
nc -zv 203.0.113.10 3306
# "Connection refused" = firewall or MySQL not listening
# "succeeded" = port is open

# Step 4: Is the firewall blocking it?
sudo iptables -L INPUT -n | grep 3306
sudo ufw status | grep 3306
```

## Fix: MySQL Bound to Localhost Only

```bash
# Check bind-address
sudo grep bind-address /etc/mysql/mysql.conf.d/mysqld.cnf
# If: bind-address = 127.0.0.1

# Change to allow remote:
sudo sed -i 's/bind-address\s*=\s*127.0.0.1/bind-address = 0.0.0.0/' \
  /etc/mysql/mysql.conf.d/mysqld.cnf

sudo systemctl restart mysql
sudo ss -tlnp | grep 3306  # Verify: 0.0.0.0:3306
```

## Fix: Missing User Grant

```bash
sudo mysql -u root -p

-- Check if user@host exists
SELECT User, Host FROM mysql.user WHERE User='appuser';

-- If not found or wrong host, create:
CREATE USER 'appuser'@'10.0.0.5' IDENTIFIED BY 'password';
GRANT ALL ON appdb.* TO 'appuser'@'10.0.0.5';
FLUSH PRIVILEGES;

-- Test login from the client:
mysql -h 203.0.113.10 -u appuser -p appdb
```

## Fix: Firewall Blocking Port 3306

```bash
# Check and add rules
sudo ufw allow from 10.0.0.0/24 to any port 3306
sudo ufw reload

# or iptables:
sudo iptables -A INPUT -p tcp --dport 3306 -s 10.0.0.5 -j ACCEPT

# Verify rule is hit:
sudo iptables -L INPUT -v -n | grep 3306
```

## Advanced Debugging

```bash
# Enable MySQL connection logging
sudo mysql -e "SET GLOBAL general_log = 'ON';"
sudo mysql -e "SET GLOBAL general_log_file='/tmp/mysql-general.log';"

# Watch for connection attempts
sudo tail -f /tmp/mysql-general.log

# Check MySQL error log
sudo tail -30 /var/log/mysql/error.log

# Test with verbose mysql client
mysql -h 203.0.113.10 -u appuser -p --verbose --debug 2>&1 | head -20
```

## Conclusion

MySQL connection issues follow a pattern: verify MySQL is running, confirm it binds to the right IP (check `bind-address`), ensure the firewall allows port 3306 from the client IP, and verify a grant exists for `user@client_ip`. Fix in order: network reachability first, then MySQL binding, then user grants. Use `nc -zv server 3306` to test network connectivity before attempting MySQL-level diagnosis.

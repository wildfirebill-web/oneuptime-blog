# How to Set Up Firewall Rules for MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, Firewall, Network, UFW

Description: Learn how to set up firewall rules for MySQL on Linux using UFW and iptables to restrict access to port 3306 to trusted hosts only.

---

MySQL listens on TCP port 3306 by default. Leaving this port open to the internet is a critical security risk that exposes your database to brute-force attacks, exploitation attempts, and unauthorized access. Firewall rules at the OS level provide an essential layer of network security, allowing only trusted hosts or networks to connect to MySQL.

## Using UFW (Ubuntu/Debian)

UFW (Uncomplicated Firewall) is the default firewall tool on Ubuntu systems.

```bash
# Allow MySQL from a specific application server IP
ufw allow from 192.168.1.100 to any port 3306

# Allow MySQL from an entire subnet
ufw allow from 192.168.1.0/24 to any port 3306

# Deny all other MySQL connections
ufw deny 3306

# Enable UFW if not already active
ufw enable

# Check rules
ufw status verbose
```

## Using firewalld (RHEL/CentOS/Fedora)

```bash
# Allow a specific IP to access MySQL
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.100/32" port protocol="tcp" port="3306" accept'

# Allow a subnet
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/8" port protocol="tcp" port="3306" accept'

# Reload firewall rules
firewall-cmd --reload

# Verify rules
firewall-cmd --list-all
```

## Using iptables Directly

```bash
# Allow MySQL from a specific IP
iptables -A INPUT -p tcp --dport 3306 -s 192.168.1.100 -j ACCEPT

# Allow MySQL from a subnet
iptables -A INPUT -p tcp --dport 3306 -s 10.0.0.0/8 -j ACCEPT

# Drop all other MySQL connections
iptables -A INPUT -p tcp --dport 3306 -j DROP

# Save rules (Ubuntu/Debian)
iptables-save > /etc/iptables/rules.v4
```

## Restricting MySQL to Localhost Only

If your application runs on the same server as MySQL, block external access entirely:

```bash
# UFW: allow only local connections
ufw allow from 127.0.0.1 to any port 3306

# Deny all external connections
ufw deny 3306
```

Also configure MySQL's `bind-address` to match:

```text
[mysqld]
bind-address = 127.0.0.1
```

## Allowing MySQL for Docker Containers

If MySQL runs in Docker, use the host's bridge network range:

```bash
# Allow Docker subnet to reach MySQL
ufw allow from 172.17.0.0/16 to any port 3306
```

## Verifying Firewall Rules Are Working

```bash
# From an unauthorized machine, this should time out or be refused
nc -zv db-server.example.com 3306

# From an authorized machine, this should succeed
mysql -h db-server.example.com -u app_user -p
```

## Using AWS Security Groups (Cloud)

For MySQL on AWS EC2 or RDS:

```text
Inbound rule:
  Type: MySQL/Aurora
  Protocol: TCP
  Port: 3306
  Source: 10.0.1.0/24 (your app subnet)
```

## Summary

Firewall rules for MySQL should restrict port 3306 to only the IP addresses or subnets that legitimately need database access. Use UFW on Ubuntu, firewalld on RHEL-family systems, or cloud security groups for managed instances. Always pair firewall rules with MySQL's `bind-address` setting to create defense in depth at both the network and application layers.

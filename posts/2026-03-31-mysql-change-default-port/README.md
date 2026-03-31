# How to Change the MySQL Default Port from 3306

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Configuration, Port, Security, Administration

Description: Change the MySQL listening port from the default 3306 to a custom port by editing my.cnf, updating firewall rules, and reconnecting clients on the new port.

---

## How It Works

MySQL listens on TCP port 3306 by default. The port is controlled by the `port` variable in `my.cnf` under the `[mysqld]` section. After changing the port, you must restart MySQL, update firewall rules, and update all client connection strings.

```mermaid
flowchart LR
    A[Edit my.cnf: port = 3307] --> B[Restart mysqld]
    B --> C[mysqld binds to port 3307]
    C --> D[Update firewall rules]
    D --> E[Update client connection strings]
    E --> F[Clients connect on 3307]
```

## Prerequisites

- MySQL server installed and running
- User with `sudo` access
- Client applications whose connection strings you can update

## Step 1 - Find the Current Configuration File

```bash
mysql -u root -p -e "SHOW VARIABLES LIKE 'port';"
```

```text
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| port          | 3306  |
+---------------+-------+
```

Find the active configuration file.

```bash
mysql -u root -p -e "SHOW VARIABLES LIKE 'datadir';"
mysql --help | grep -A1 "Default options"
```

```text
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf ~/.my.cnf
```

## Step 2 - Edit the Configuration File

Open the configuration file (path varies by distribution).

```bash
# Ubuntu / Debian
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf

# RHEL / CentOS / Rocky / AlmaLinux
sudo nano /etc/my.cnf
```

Find or add the `port` directive under `[mysqld]`.

```ini
[mysqld]
port = 3307
```

Also update the `[client]` section so that command-line tools default to the new port.

```ini
[client]
port = 3307
```

## Step 3 - Restart MySQL

```bash
# Ubuntu / Debian
sudo systemctl restart mysql

# RHEL-based
sudo systemctl restart mysqld
```

## Step 4 - Verify the New Port

```bash
sudo ss -tlnp | grep mysqld
```

```text
LISTEN   0   151   127.0.0.1:3307   0.0.0.0:*   users:(("mysqld",...))
```

Confirm from inside MySQL.

```bash
mysql -u root -p --port=3307 -e "SHOW VARIABLES LIKE 'port';"
```

```text
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| port          | 3307  |
+---------------+-------+
```

## Step 5 - Update Firewall Rules

### UFW (Ubuntu / Debian)

```bash
# Remove old rule
sudo ufw delete allow 3306/tcp

# Add new rule
sudo ufw allow 3307/tcp
sudo ufw reload
```

### firewalld (RHEL-based)

```bash
# Remove old port
sudo firewall-cmd --permanent --remove-port=3306/tcp

# Add new port
sudo firewall-cmd --permanent --add-port=3307/tcp
sudo firewall-cmd --reload
```

### SELinux (RHEL-based only)

If SELinux is enforcing, you must allow the new port.

```bash
sudo semanage port -a -t mysqld_port_t -p tcp 3307
```

Verify:

```bash
sudo semanage port -l | grep mysqld_port_t
```

```text
mysqld_port_t   tcp   1186, 3306, 3307, 33060, 33062
```

## Step 6 - Update Client Connection Strings

### Command-line client

```bash
mysql -u root -p --host=127.0.0.1 --port=3307
```

Or set in `~/.my.cnf` to avoid typing `--port` every time.

```ini
[client]
port = 3307
```

### Application connection strings

PHP PDO:

```php
$dsn = 'mysql:host=127.0.0.1;port=3307;dbname=myapp';
```

Python (mysql-connector):

```python
conn = mysql.connector.connect(host='127.0.0.1', port=3307, user='app', password='pass', database='myapp')
```

Node.js (mysql2):

```javascript
const pool = mysql.createPool({ host: '127.0.0.1', port: 3307, user: 'app', password: 'pass', database: 'myapp' });
```

Java JDBC:

```text
jdbc:mysql://127.0.0.1:3307/myapp
```

## Security Note

Changing the default port provides minimal security by obscurity and is not a substitute for a firewall, strong passwords, and TLS encryption. It does reduce noise from automated scans targeting port 3306.

## Summary

Changing the MySQL port requires editing `my.cnf` to set `port` under `[mysqld]`, restarting the MySQL service, updating firewall rules to allow the new port, and for SELinux systems adding the port to the `mysqld_port_t` context. Update all client application connection strings after the port change to restore connectivity.

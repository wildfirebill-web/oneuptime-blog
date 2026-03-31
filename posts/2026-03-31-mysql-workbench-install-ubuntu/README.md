# How to Install MySQL Workbench on Ubuntu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Workbench, Ubuntu, Installation, GUI

Description: Install MySQL Workbench on Ubuntu 22.04 or 24.04 using the official MySQL APT repository or the downloaded DEB package, and connect to a local or remote MySQL server.

---

## How It Works

MySQL Workbench is the official GUI tool for database design, SQL development, and server administration. On Ubuntu, it can be installed from the official MySQL APT repository or directly from a downloaded `.deb` package.

```mermaid
flowchart LR
    A[Add MySQL APT repo] --> B[apt install mysql-workbench-community]
    B --> C[Launch Workbench]
    C --> D[Create connection profile]
    D --> E[Connect to MySQL server]
    E --> F[Query / Design / Administer]
```

## Prerequisites

- Ubuntu 22.04 or 24.04 (64-bit)
- Desktop environment (GNOME, KDE, etc.)
- MySQL server running locally or remotely
- User with `sudo` privileges

## Method 1 - Install via MySQL APT Repository

### Add the Repository

```bash
wget https://dev.mysql.com/get/mysql-apt-config_0.8.29-1_all.deb
sudo dpkg -i mysql-apt-config_0.8.29-1_all.deb
```

In the dialog, select **MySQL Tools & Connectors** and ensure **Enabled** is selected. Select **Ok**.

```bash
sudo apt update
```

### Install Workbench

```bash
sudo apt install -y mysql-workbench-community
```

## Method 2 - Install from Downloaded DEB Package

If you prefer not to add the repository, download the DEB directly.

```bash
wget https://dev.mysql.com/get/Downloads/MySQLGUITools/mysql-workbench-community_8.0.36-1ubuntu22.04_amd64.deb
sudo apt install -y ./mysql-workbench-community_8.0.36-1ubuntu22.04_amd64.deb
```

The `apt install` command automatically resolves dependencies from the Ubuntu repos.

## Step 2 - Launch MySQL Workbench

Open MySQL Workbench from the application launcher or from the terminal.

```bash
mysql-workbench
```

The Workbench home screen shows existing connection profiles and lets you create new ones.

## Step 3 - Create a Connection Profile

1. Click the **+** icon next to "MySQL Connections."
2. Fill in the connection details:

```text
Connection Name:  Local MySQL
Connection Method: Standard (TCP/IP)
Hostname:         127.0.0.1
Port:             3306
Username:         root
```

3. Click **Store in Vault** to save the password.
4. Click **Test Connection** to verify.
5. Click **OK** to save the profile.

## Step 4 - Connect via SSH Tunnel (Remote Server)

For remote servers not exposed directly to the internet, use the SSH tunnel method.

1. Click **+** to create a new connection.
2. Set **Connection Method** to **Standard TCP/IP over SSH**.

```text
SSH Hostname:     your-server.example.com:22
SSH Username:     ubuntu
SSH Key File:     ~/.ssh/id_rsa
MySQL Hostname:   127.0.0.1
MySQL Port:       3306
Username:         appuser
```

## Using the SQL Editor

Once connected, open a query tab with `Ctrl+T`.

```sql
-- Show all databases
SHOW DATABASES;

-- Use a database
USE myapp;

-- Show tables
SHOW TABLES;

-- Run a query
SELECT id, username, created_at
FROM users
ORDER BY created_at DESC
LIMIT 20;
```

Execute with `Ctrl+Shift+Enter` (all statements) or `Ctrl+Enter` (statement at cursor).

## Using the Visual Explain Plan

1. Write a query in the SQL editor.
2. Click the **Explain** button (lightning bolt with magnifying glass).
3. Select **Visual Explain** to see the query execution plan as a diagram.

This helps identify missing indexes and full table scans.

## Updating MySQL Workbench

If installed via the APT repository:

```bash
sudo apt update
sudo apt upgrade mysql-workbench-community
```

## Uninstalling

```bash
sudo apt remove --purge mysql-workbench-community
sudo apt autoremove
```

## Troubleshooting

### Workbench crashes on startup

Install missing GNOME keyring dependencies.

```bash
sudo apt install -y libsecret-1-0 libsecret-common
```

### Cannot connect to MySQL on Ubuntu 24.04

Ubuntu 24.04 uses AppArmor profiles that may restrict Workbench. Check the journal.

```bash
journalctl -xe | grep apparmor | tail -20
```

## Summary

MySQL Workbench on Ubuntu can be installed from the official MySQL APT repository or from a downloaded DEB package. After installation, create a connection profile pointing to your MySQL server. For remote servers, use the built-in SSH tunnel feature to avoid exposing MySQL port 3306 to the internet. The visual SQL editor and explain plan viewer make Workbench a practical tool for daily database development work.

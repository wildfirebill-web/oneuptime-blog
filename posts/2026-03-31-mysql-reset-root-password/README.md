# How to Reset the MySQL Root Password

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, Authentication, Root, Password, Administration

Description: Learn how to reset the MySQL root password on Linux and Windows when you are locked out, using safe-mode and the ALTER USER statement.

---

## Overview

If you have lost or forgotten the MySQL root password, you can reset it by starting MySQL in a special mode that bypasses authentication. The exact steps differ slightly between MySQL 5.7 and MySQL 8.0, and between Linux and Windows.

## Method 1: Using --skip-grant-tables (MySQL 5.7 and 8.0)

### Step 1 - Stop the MySQL Service

```bash
# Linux (systemd)
sudo systemctl stop mysql

# macOS (Homebrew)
brew services stop mysql
```

### Step 2 - Start MySQL with --skip-grant-tables

```bash
sudo mysqld_safe --skip-grant-tables --skip-networking &
```

`--skip-networking` prevents remote connections while the server runs without authentication.

### Step 3 - Connect Without a Password

```bash
mysql -u root
```

### Step 4 - Reset the Password

For MySQL 8.0:

```sql
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'NewStrongPassword!';
FLUSH PRIVILEGES;
```

For MySQL 5.7:

```sql
UPDATE mysql.user
SET authentication_string = PASSWORD('NewStrongPassword!')
WHERE User = 'root' AND Host = 'localhost';
FLUSH PRIVILEGES;
```

### Step 5 - Restart MySQL Normally

```bash
sudo systemctl start mysql
```

## Method 2: Using --init-file (MySQL 8.0 Recommended)

This method is cleaner because it avoids running the server without authentication for more than a moment.

### Step 1 - Create a Reset SQL File

```bash
cat > /tmp/mysql_reset.sql << 'EOF'
ALTER USER 'root'@'localhost' IDENTIFIED BY 'NewStrongPassword!';
EOF
```

### Step 2 - Stop MySQL and Start with --init-file

```bash
sudo systemctl stop mysql
sudo mysqld --init-file=/tmp/mysql_reset.sql --user=mysql &
```

MySQL executes the file on startup, sets the password, and enters normal operation.

### Step 3 - Restart Normally

```bash
sudo pkill mysqld
sudo systemctl start mysql
```

### Step 4 - Clean Up the Password File

```bash
rm /tmp/mysql_reset.sql
```

## Method 3: Resetting on Windows

Open an elevated Command Prompt:

```cmd
net stop MySQL80
```

Create `C:\mysql_reset.sql` with:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'NewStrongPassword!';
```

Start MySQL with the init file:

```cmd
"C:\Program Files\MySQL\MySQL Server 8.0\bin\mysqld.exe" --init-file=C:\mysql_reset.sql --console
```

After MySQL starts, stop it and restart the service:

```cmd
net start MySQL80
```

## Verifying the New Password

```bash
mysql -u root -p
```

Enter the new password at the prompt. A successful login confirms the reset worked.

## Changing the Root Password When You Know It

If you still have access but want to change the password:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'NewPassword!';
FLUSH PRIVILEGES;
```

## Security Best Practices After Reset

- Delete any temporary SQL files used during the reset.
- Review all user accounts with `SELECT User, Host FROM mysql.user;`.
- Enable the `validate_password` component to enforce strong passwords.
- Restrict root login to localhost only - never allow `root@'%'`.

## Summary

To reset a lost MySQL root password, stop the server, restart it with `--skip-grant-tables` and `--skip-networking` (or with `--init-file` pointing to an `ALTER USER` statement), connect without a password, run `ALTER USER` to set the new password, then restart MySQL normally. Always delete temporary files and re-verify remote access restrictions after a password reset.

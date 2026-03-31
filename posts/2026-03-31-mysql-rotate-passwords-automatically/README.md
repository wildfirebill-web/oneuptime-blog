# How to Rotate MySQL Passwords Automatically

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, Password Rotation, Automation, Credential

Description: Automate MySQL password rotation using shell scripts, cron jobs, or Vault's static secret rotation to keep credentials fresh without manual intervention.

---

## Why Rotate MySQL Passwords?

Security standards like PCI-DSS and SOC 2 require periodic credential rotation. Even without compliance requirements, rotating passwords limits the window of exposure if a credential is compromised. Automating this process removes the human delay and inconsistency that makes manual rotation unreliable.

## Option 1 - Shell Script with Cron

A simple rotation script generates a new password, updates MySQL, and writes the new credential to a secure file or secret store.

```bash
#!/bin/bash
set -euo pipefail

DB_HOST="localhost"
DB_USER="appuser"
CURRENT_PASS_FILE="/etc/mysql/secrets/appuser.pass"
NEW_PASS=$(openssl rand -base64 32 | tr -d '/+=\n' | head -c 32)

CURRENT_PASS=$(cat "$CURRENT_PASS_FILE")

mysql -h "$DB_HOST" -u root -p"$(cat /etc/mysql/secrets/root.pass)" \
  -e "ALTER USER '${DB_USER}'@'%' IDENTIFIED BY '${NEW_PASS}';"

echo "$NEW_PASS" > "$CURRENT_PASS_FILE"
chmod 600 "$CURRENT_PASS_FILE"

echo "[$(date)] Password rotated for ${DB_USER}" >> /var/log/mysql-rotation.log
```

Schedule with cron:

```bash
sudo crontab -e
```

```text
0 2 * * 0 /usr/local/bin/rotate-mysql-password.sh >> /var/log/mysql-rotation.log 2>&1
```

## Option 2 - Vault Static Secret Rotation

If you use HashiCorp Vault, it can manage static credentials and rotate them automatically:

```bash
vault write database/static-roles/appuser-static \
  db_name=myapp-mysql \
  username=appuser \
  rotation_period=86400 \
  rotation_statements="ALTER USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';"
```

Trigger an immediate rotation:

```bash
vault write -force database/rotate-role/appuser-static
```

Read the current password:

```bash
vault read database/static-creds/appuser-static
```

## Option 3 - AWS Secrets Manager Rotation (RDS)

For RDS MySQL, AWS Secrets Manager automates rotation with Lambda:

```bash
aws secretsmanager create-secret \
  --name prod/mysql/appuser \
  --secret-string '{"username":"appuser","password":"InitialPass!","host":"mydb.rds.amazonaws.com","port":3306}'

aws secretsmanager rotate-secret \
  --secret-id prod/mysql/appuser \
  --rotation-rules AutomaticallyAfterDays=30
```

The managed Lambda rotation function handles the dual-user rotation pattern: it creates a new password on the secondary user, tests connectivity, then updates the primary.

## Zero-Downtime Rotation with Dual-User Pattern

For applications that cannot tolerate any gap, maintain two accounts and rotate them alternately:

```sql
-- Both accounts have identical privileges
CREATE USER 'appuser_a'@'%' IDENTIFIED BY 'PassA';
CREATE USER 'appuser_b'@'%' IDENTIFIED BY 'PassB';
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp.* TO 'appuser_a'@'%';
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp.* TO 'appuser_b'@'%';
```

```bash
# Rotation: update the inactive account, then switch the app to it
mysql -u root -p -e "ALTER USER 'appuser_b'@'%' IDENTIFIED BY '${NEW_PASS}';"
# Update app config to use appuser_b
# On next rotation, update appuser_a
```

## Verifying Rotation Success

```sql
-- Check when password was last changed
SELECT User, Host, password_last_changed
FROM mysql.user
WHERE User LIKE 'appuser%';
```

## Summary

Automated MySQL password rotation keeps credentials fresh and reduces the blast radius of a credential compromise. Shell scripts with cron work for simple setups, HashiCorp Vault static roles provide centralized management with audit trails, and AWS Secrets Manager handles RDS rotation natively. The dual-user pattern ensures zero-downtime rotation for production systems.

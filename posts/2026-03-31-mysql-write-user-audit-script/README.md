# How to Write a MySQL User Audit Script

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, Script

Description: Write a MySQL user audit script that lists all users, their privileges, password age, and flags accounts with overly broad permissions or no password.

---

MySQL databases accumulate user accounts over time: old service accounts, developer accounts from departed team members, and overly privileged test users. A user audit script helps you identify and remediate these security risks before they are exploited.

## What to Audit

A thorough MySQL user audit should check:

1. All user accounts and their hosts
2. Users with SUPER, GRANT OPTION, or global ALL PRIVILEGES
3. Users with no password or empty password
4. Accounts that have never been used
5. Password age (accounts with old passwords)

## The Audit Script

```bash
#!/bin/bash
# mysql_user_audit.sh
MYSQL="mysql -u root -p${MYSQL_ROOT_PASSWORD} -se"
REPORT="/tmp/mysql_user_audit_$(date +%Y-%m-%d).txt"

log() { echo "$1" | tee -a "$REPORT"; }

log "=== MySQL User Audit Report - $(date) ==="
log ""

# 1. List all users
log "--- All MySQL Users ---"
$MYSQL "SELECT CONCAT(User, '@', Host) AS account FROM mysql.user ORDER BY User, Host;" | \
  tee -a "$REPORT"
log ""

# 2. Users with no password
log "--- Users with No Password (SECURITY RISK) ---"
$MYSQL "
SELECT CONCAT(User, '@', Host) AS account
FROM mysql.user
WHERE authentication_string = '' OR authentication_string IS NULL
ORDER BY User;" | tee -a "$REPORT"
log ""

# 3. Users with SUPER privilege
log "--- Users with SUPER Privilege ---"
$MYSQL "
SELECT CONCAT(User, '@', Host) AS account
FROM mysql.user
WHERE Super_priv = 'Y'
ORDER BY User;" | tee -a "$REPORT"
log ""

# 4. Users with global ALL PRIVILEGES
log "--- Users with Global ALL PRIVILEGES ---"
$MYSQL "
SELECT CONCAT(User, '@', Host) AS account
FROM mysql.user
WHERE Grant_priv = 'Y'
   OR (Select_priv = 'Y' AND Insert_priv = 'Y' AND Update_priv = 'Y'
       AND Delete_priv = 'Y' AND Create_priv = 'Y' AND Drop_priv = 'Y')
ORDER BY User;" | tee -a "$REPORT"
log ""

# 5. Per-database grants - overly broad
log "--- Per-Database ALL PRIVILEGES Grants ---"
$MYSQL "
SELECT CONCAT(User, '@', Host) AS account, Db
FROM mysql.db
WHERE Select_priv = 'Y' AND Insert_priv = 'Y'
  AND Update_priv = 'Y' AND Delete_priv = 'Y'
  AND Drop_priv = 'Y'
ORDER BY User, Db;" | tee -a "$REPORT"
log ""

# 6. Password age check (MySQL 8 - password_last_changed in mysql.user)
log "--- Password Age (accounts not changed in 90+ days) ---"
$MYSQL "
SELECT
  CONCAT(User, '@', Host)    AS account,
  password_last_changed,
  DATEDIFF(NOW(), password_last_changed) AS days_since_change
FROM mysql.user
WHERE password_last_changed IS NOT NULL
  AND DATEDIFF(NOW(), password_last_changed) > 90
ORDER BY days_since_change DESC;" | tee -a "$REPORT"
log ""

log "=== Audit complete. Report saved to: $REPORT ==="
```

## Running the Audit

```bash
chmod +x mysql_user_audit.sh
MYSQL_ROOT_PASSWORD=yourpassword ./mysql_user_audit.sh
```

## Locking a Suspicious Account

```sql
-- Lock account without deleting it (reversible)
ALTER USER 'old_dev_user'@'%' ACCOUNT LOCK;

-- Revoke dangerous global privileges
REVOKE SUPER ON *.* FROM 'app_user'@'%';

-- Drop account confirmed as unused
DROP USER 'stale_user'@'localhost';
```

## Scheduling Monthly Audits

```text
0 6 1 * * /opt/scripts/mysql_user_audit.sh | mail -s "MySQL User Audit" security@example.com
```

## Summary

A MySQL user audit script should detect accounts with no password, overly broad global privileges, and stale credentials. Run it monthly at minimum and after any team change. Lock accounts that look suspicious before dropping them - locking is reversible if the account turns out to be needed.

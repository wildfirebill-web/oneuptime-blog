# Redis Runbook: Handling Security Incidents

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Security, Runbook

Description: Runbook for responding to Redis security incidents - unauthorized access, exposed instances, suspicious commands, and remediation steps.

---

Redis was historically deployed without authentication in trusted networks. With internet exposure and misconfigured cloud deployments, security incidents have become more common. This runbook covers how to detect, contain, and remediate Redis security incidents.

## Step 1: Detect the Incident

Check for unauthorized client connections:

```bash
redis-cli CLIENT LIST
```

Look for unfamiliar IP addresses. Check recent command history:

```bash
redis-cli MONITOR
```

Note: `MONITOR` impacts performance. Use it briefly to capture suspicious activity.

## Step 2: Check for Known Attack Patterns

Attackers often write SSH keys or cron jobs to Redis. Check for suspicious keys:

```bash
redis-cli KEYS "*authorized_keys*"
redis-cli KEYS "*crontab*"
redis-cli KEYS "crackit"
```

Check if the RDB dir was changed to a writable system path:

```bash
redis-cli CONFIG GET dir
redis-cli CONFIG GET dbfilename
```

If `dir` points to `/root/.ssh/` or `/var/spool/cron/`, your system may be compromised.

## Step 3: Contain the Incident

Immediately bind Redis to localhost if it is currently exposed:

```bash
redis-cli CONFIG SET bind "127.0.0.1"
```

Enable authentication immediately:

```bash
redis-cli CONFIG SET requirepass "strong-random-password-here"
```

Block external access with firewall rules:

```bash
sudo ufw deny 6379
```

## Step 4: Kill Suspicious Clients

Kill all external connections:

```bash
redis-cli CLIENT KILL ADDR <suspicious-ip>:0 SKIPME no
```

## Step 5: Audit ACLs

Check current ACL configuration:

```bash
redis-cli ACL LIST
redis-cli ACL WHOAMI
```

Disable the default user if it has full permissions:

```bash
redis-cli ACL SETUSER default off
```

Create a scoped user for your application:

```bash
redis-cli ACL SETUSER appuser on >apppassword ~* +@read +@write -@dangerous
```

## Step 6: Rename Dangerous Commands

Rename or disable dangerous commands:

```bash
# redis.conf
rename-command CONFIG ""
rename-command FLUSHALL ""
rename-command DEBUG ""
rename-command SLAVEOF ""
```

## Step 7: Post-Incident Review

After containing the incident, check system files for unauthorized changes, rotate all credentials, enable TLS for Redis connections, and review audit logs:

```bash
redis-cli CONFIG GET loglevel
redis-cli CONFIG SET loglevel verbose
```

## Summary

Redis security incidents often involve unauthorized writes to system files via an exposed Redis instance. Immediate containment involves binding to localhost, enabling authentication, and blocking the port. Long-term remediation requires ACL configuration, dangerous command renaming, and enabling TLS.

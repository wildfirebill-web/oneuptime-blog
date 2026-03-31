# Redis Security Hardening Best Practices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Security, Hardening, Authentication, Production

Description: Learn how to harden Redis security in production - covering authentication, network binding, TLS encryption, ACLs, dangerous command restrictions, and OS-level protections.

---

Redis was originally designed as a trusted internal service, but production deployments require explicit security hardening. A misconfigured Redis instance exposed to the internet can result in remote code execution and complete server compromise. This guide covers essential security measures.

## Bind to Private Interfaces Only

Never expose Redis on `0.0.0.0`. Bind to the loopback or your private network interface:

```text
# redis.conf
bind 127.0.0.1 10.0.0.5
```

This prevents Redis from accepting connections from external networks. If you must expose Redis over a network, use a VPN or private subnet.

## Require Authentication

Set a strong password using the `requirepass` directive:

```text
# redis.conf
requirepass YourVeryStrongPasswordHere123!@#
```

Clients must now authenticate before issuing any commands:

```bash
redis-cli -a YourVeryStrongPasswordHere123!@# ping
```

In Redis 6+, prefer ACL-based authentication over `requirepass`.

## Use ACLs for Least-Privilege Access

Redis ACLs allow you to create users with specific permissions:

```bash
# Create a read-only user
ACL SETUSER reader on >readpassword ~* &* +@read

# Create an app user with write access to specific key patterns
ACL SETUSER app on >apppassword ~session:* ~cache:* +@write +@read

# Save ACL configuration
ACL SAVE
```

Different application components should use different users with only the permissions they need.

## Enable TLS Encryption

For any cross-network Redis traffic, enable TLS:

```text
# redis.conf
tls-port 6380
port 0

tls-cert-file /etc/redis/redis.crt
tls-key-file /etc/redis/redis.key
tls-ca-cert-file /etc/redis/ca.crt
tls-auth-clients yes
```

Generate certificates with your internal CA and rotate them regularly.

## Disable or Rename Dangerous Commands

Commands like `FLUSHALL`, `CONFIG`, `DEBUG`, and `EVAL` can be exploited:

```text
# redis.conf - disable commands entirely
rename-command FLUSHALL ""
rename-command CONFIG ""
rename-command DEBUG ""

# Or rename to a hard-to-guess string
rename-command FLUSHDB "FLUSHDB_7f3a9e2b"
```

In Redis 7+, use ACLs to restrict command access instead of renaming.

## Protect the Configuration File

The Redis configuration file contains passwords. Restrict file permissions:

```bash
chmod 640 /etc/redis/redis.conf
chown redis:redis /etc/redis/redis.conf
```

## Run Redis as a Non-Root User

Never run Redis as root. Create a dedicated service account:

```bash
useradd -r -s /sbin/nologin redis
chown -R redis:redis /var/lib/redis /var/log/redis
```

## Use a Firewall

Add firewall rules to restrict Redis port access:

```bash
# Only allow connections from your application servers
ufw allow from 10.0.0.0/24 to any port 6379
ufw deny 6379
```

## Enable Protected Mode

Protected mode is on by default and refuses connections from non-loopback addresses unless authentication is configured:

```text
# redis.conf
protected-mode yes
```

Do not disable this unless you have explicit firewall rules in place.

## Summary

Redis security hardening requires multiple layers: bind to private interfaces, require strong authentication via ACLs, encrypt traffic with TLS, restrict dangerous commands, and use OS-level controls like firewall rules and non-root execution. Each layer independently reduces your attack surface, and together they make your Redis deployment significantly more resistant to compromise.

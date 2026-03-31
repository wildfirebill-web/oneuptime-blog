# How to Use ACL GENPASS in Redis to Generate Secure Passwords

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, ACL, Security, Password, Authentication

Description: Learn how to use Redis ACL GENPASS to generate cryptographically secure random passwords for Redis users and ACL configurations.

---

The `ACL GENPASS` command in Redis generates cryptographically secure random passwords suitable for use with the Redis ACL system. Rather than relying on weak or manually created passwords, this built-in command produces high-entropy strings that can be immediately used when creating or updating Redis users.

## Basic Usage

The simplest form generates a 64-character hexadecimal string (256 bits of entropy):

```bash
127.0.0.1:6379> ACL GENPASS
"9f8e2d1c4b7a6f3e0d9c5b8a2e1f4d7c0b3a6e9f2c5d8b1a4e7f0c3d6b9e2f5"
```

You can specify the number of bits explicitly to get shorter or longer output:

```bash
127.0.0.1:6379> ACL GENPASS 128
"4a7b2f9c1e6d8a3f5b0c7e2d4f1a9b6e"

127.0.0.1:6379> ACL GENPASS 64
"3f8c1a5e9b2d6f04"
```

## Using Generated Passwords with ACL SETUSER

The primary use case is creating user accounts with secure passwords:

```bash
# Generate a password and immediately use it
127.0.0.1:6379> ACL GENPASS
"a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2"

127.0.0.1:6379> ACL SETUSER alice on >a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2 ~* &* +@all
OK
```

You can also pass the hash of the password using the `#` prefix if you want to store a SHA-256 hash rather than the plaintext:

```bash
127.0.0.1:6379> ACL SETUSER bob on #<sha256_of_password> ~cache:* +GET +SET +DEL
OK
```

## Automating Password Generation in Scripts

In shell scripts, use `ACL GENPASS` to automate user provisioning:

```bash
#!/bin/bash
REDIS_HOST="localhost"
REDIS_PORT="6379"
USERNAME="app_user"

# Generate a secure password
PASSWORD=$(redis-cli -h $REDIS_HOST -p $REDIS_PORT ACL GENPASS)

# Create the user with the generated password
redis-cli -h $REDIS_HOST -p $REDIS_PORT \
  ACL SETUSER $USERNAME on ">$PASSWORD" "~app:*" "+GET" "+SET" "+DEL" "+EXPIRE"

echo "Created user: $USERNAME"
echo "Password: $PASSWORD"
```

## Generating Passwords for Multiple Users

When provisioning multiple users at once:

```bash
#!/bin/bash
USERS=("reader" "writer" "admin")

for USER in "${USERS[@]}"; do
  PASS=$(redis-cli ACL GENPASS)
  echo "$USER: $PASS" >> credentials.txt
  redis-cli ACL SETUSER $USER on ">$PASS" "~*" "+@all"
done
```

## Entropy Considerations

The default 256 bits of entropy from `ACL GENPASS` provides excellent security for most use cases. For highly sensitive environments, the default is more than sufficient. For scenarios with lower security requirements, 128 bits remains strong. The output is always lowercase hexadecimal.

```text
Bits  | Output length | Use case
------|---------------|------------------------
64    | 16 chars      | Internal low-risk tokens
128   | 32 chars      | Standard application keys
256   | 64 chars      | High-security passwords (default)
```

## Best Practices

Never hardcode Redis passwords in source code. Use `ACL GENPASS` at provisioning time and store the result in a secrets manager like HashiCorp Vault or AWS Secrets Manager. Rotate passwords periodically using this same command.

```bash
# Rotate a user's password
NEW_PASS=$(redis-cli ACL GENPASS)
redis-cli ACL SETUSER alice resetpass ">$NEW_PASS"
echo "Rotated password for alice: $NEW_PASS"
```

## Summary

`ACL GENPASS` is a simple but important tool in the Redis security toolkit. It generates cryptographically secure random passwords that integrate directly with the Redis ACL system, making it easy to provision users with strong credentials without relying on external tools or weak manual passwords.

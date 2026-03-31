# How to Use YAML Syntax for MongoDB Configuration Files

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, YAML, Configuration, mongod, Syntax

Description: Learn how to write valid YAML syntax for mongod.conf, including indentation rules, data types, comments, and common formatting mistakes to avoid.

---

MongoDB configuration files use YAML (YAML Ain't Markup Language) format. While YAML is human-readable, its whitespace-sensitive syntax causes frequent errors. Understanding how YAML works in the context of `mongod.conf` prevents misconfiguration and hard-to-diagnose startup failures.

## YAML Fundamentals

YAML represents configuration as key-value pairs, nested mappings (objects), and sequences (lists). Indentation uses spaces, not tabs.

```yaml
# This is a comment
net:
  port: 27017        # integer
  bindIp: 127.0.0.1  # string

storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true    # boolean
```

## Indentation Rules

Every level of nesting requires consistent indentation. Two spaces per level is the MongoDB convention.

```yaml
# Correct - 2-space indentation
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4

# Wrong - mixing tabs and spaces causes parse errors
storage:
	wiredTiger:
		engineConfig:
			cacheSizeGB: 4
```

Never use tabs. Configure your editor to insert spaces when you press Tab in YAML files.

## Scalar Data Types

YAML infers types from the value format.

```yaml
net:
  port: 27017              # integer
  ipv6: false              # boolean
  maxIncomingConnections: 100

storage:
  dbPath: /var/lib/mongodb  # string (no quotes needed unless special chars)
  journal:
    enabled: true
    commitIntervalMs: 100
```

Strings that contain colons or special characters must be quoted.

```yaml
systemLog:
  path: "/var/log/mongodb/mongod.log"  # quotes required if path has special chars
```

## Multi-Value Strings

The `bindIp` option accepts a comma-separated string. Write it as a plain YAML string.

```yaml
net:
  bindIp: 127.0.0.1,10.0.1.50
```

Do not use a YAML list here - MongoDB expects a comma-separated string for this field.

## YAML Lists for Multi-Value Options

Some options accept YAML sequences.

```yaml
net:
  compression:
    compressors:
      - snappy
      - zlib
      - zstd
```

Check the MongoDB documentation to determine whether a field expects a comma-separated string or a YAML list.

## Validating mongod.conf Syntax

MongoDB does not validate YAML syntax until startup. Use a standalone YAML parser to catch errors before restarting the daemon.

```bash
python3 -c "import yaml, sys; yaml.safe_load(open(sys.argv[1]))" /etc/mongod.conf
echo $?
```

A zero exit code means the YAML is syntactically valid. MongoDB may still reject it if a key is unrecognized or a value is out of range, which you will see in the `mongod` startup error output.

## Full Annotated Example

```yaml
# /etc/mongod.conf

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true

net:
  port: 27017
  bindIp: 127.0.0.1

security:
  authorization: enabled

replication:
  replSetName: rs0
```

## Summary

MongoDB configuration files use YAML with strict whitespace rules. Always use 2-space indentation, never tabs. Know whether a field expects a plain string, a comma-separated value, or a YAML list. Validate syntax with a Python YAML parser before restarting `mongod` to catch formatting errors early and avoid accidental downtime.

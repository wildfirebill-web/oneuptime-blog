# How to Install Redis on macOS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Installation, macOS, Homebrew, Development

Description: Install Redis on macOS using Homebrew, configure it as a background service, and set up your development environment with common client tools.

---

The fastest way to get Redis running on macOS is with Homebrew. It handles the build, installs the binaries, and provides a service manager so Redis starts automatically on login.

## Prerequisites

Install Homebrew if you do not have it:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

## Install Redis

```bash
brew install redis
```

Homebrew installs `redis-server` and `redis-cli` to `/opt/homebrew/bin/` (Apple Silicon) or `/usr/local/bin/` (Intel).

## Start Redis

Start Redis as a background service that launches at login:

```bash
brew services start redis
```

Or start Redis manually for a single session (runs in the foreground):

```bash
redis-server
```

To start with a custom config file:

```bash
redis-server /opt/homebrew/etc/redis.conf
```

## Verify the Installation

```bash
redis-cli ping
# PONG

redis-cli info server | grep redis_version
# redis_version:7.2.x
```

## Configuration File

Homebrew places the default config at:

```text
/opt/homebrew/etc/redis.conf     # Apple Silicon
/usr/local/etc/redis.conf        # Intel Mac
```

Edit it with your preferred editor:

```bash
nano /opt/homebrew/etc/redis.conf
```

Useful settings for local development:

```text
# Bind to localhost only
bind 127.0.0.1

# Disable persistence for dev (faster)
save ""
appendonly no

# Set a password (optional for local dev)
requirepass yourpassword
```

After editing, restart the service:

```bash
brew services restart redis
```

## Managing the Redis Service

```bash
# Check status
brew services info redis

# Stop Redis
brew services stop redis

# Restart Redis
brew services restart redis

# View logs
tail -f /opt/homebrew/var/log/redis.log
```

## Connecting with redis-cli

```bash
# Connect to the default local instance
redis-cli

# Connect with password
redis-cli -a yourpassword

# Run a single command
redis-cli SET hello world
redis-cli GET hello
```

## Running Multiple Redis Instances

For testing different configurations, start additional instances on different ports:

```bash
# Start a second instance on port 6380 with no persistence
redis-server --port 6380 --save "" --appendonly no --daemonize yes --logfile /tmp/redis-6380.log
```

Stop it with:

```bash
redis-cli -p 6380 shutdown
```

## Upgrading Redis

```bash
brew upgrade redis
brew services restart redis
```

## Summary

Install Redis on macOS with `brew install redis` and manage it as a background service with `brew services`. The config file lives in the Homebrew etc directory and can be edited to disable persistence for local development. For multiple isolated instances during testing, start additional `redis-server` processes on different ports directly from the command line.

# How to Use SSH Escape Sequences to Manage IPv4 Port Forwards at Runtime

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SSH, Escape Sequences, IPv4, Port Forwarding, Runtime, CLI

Description: Learn how to use SSH escape sequences to add, list, and remove IPv4 port forwards in an active SSH session without reconnecting.

---

SSH escape sequences are special key combinations that let you interact with the SSH session itself — not the remote shell. They allow you to add or remove port forwards, check connection status, and even background the session, all from an active interactive session.

## Entering the SSH Escape Prompt

The default escape character is `~`. To activate it, press `Enter` then `~` then a command character.

> **Important:** The `~` must follow a newline. Press Enter first, then `~`.

## Escape Sequence Reference

| Sequence | Action |
|----------|--------|
| `~C` | Open SSH command-line prompt (to manage port forwards) |
| `~#` | List all forwarded connections |
| `~?` | Show help for all escape sequences |
| `~.` | Disconnect from the SSH session |
| `~Z` | Suspend the SSH session (background it) |
| `~&` | Background SSH and return to local shell |

## Adding a Local Port Forward at Runtime

```bash
# In an active SSH session, press Enter then ~C to open the command prompt
# You'll see:
ssh>

# Add a local port forward: local 8080 → remote 192.168.10.5:80
ssh> -L 8080:192.168.10.5:80
Forwarding port.

# Test the new forward from your local machine
# (in another terminal)
curl http://localhost:8080/
```

## Adding a Remote Port Forward at Runtime

```bash
# In the SSH escape prompt:
ssh> -R 9090:localhost:3000
Forwarding port.
# Now port 9090 on the remote server forwards to localhost:3000 on your machine
```

## Canceling a Port Forward

```bash
# Cancel a local forward on port 8080
ssh> -KL 8080

# Cancel a remote forward on port 9090
ssh> -KR 9090
```

## Listing Active Forwards

```bash
# List all currently active port forwards
ssh> # (type ~ then # in the session)
The following connections are open:
  #0 client-session (t4 r0 i0/0 o0/0 e[write]/0 fd 4/5/6 sock 4 cc -1)
  #1 direct-tcpip: listening port 8080 for 192.168.10.5 port 80
```

## Practical Workflow: Dynamic Tunnels During a Session

```bash
# 1. SSH into the bastion
ssh user@203.0.113.10

# 2. Discover an internal service while exploring
#    You find a web app on 10.0.0.30:3000

# 3. Add a local forward without disconnecting
# Press Enter, then type: ~C
ssh> -L 7777:10.0.0.30:3000
Forwarding port.

# 4. From another terminal, access the internal service
curl http://localhost:7777/

# 5. Remove the forward when done
# Press Enter, then: ~C
ssh> -KL 7777
```

## Key Takeaways

- `~C` opens the SSH command prompt for managing port forwards without reconnecting.
- `-L local_port:remote_host:remote_port` adds a local forward; `-KL local_port` cancels it.
- `-R remote_port:local_host:local_port` adds a remote forward; `-KR remote_port` cancels it.
- `~#` lists all open channels and active port forwards in the current session.

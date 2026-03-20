# How to Fix Portainer Firewall Issues After Synology DSM Updates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Synology, NAS, Docker, Firewall, Troubleshooting, DSM

Description: Troubleshoot and fix common Portainer connectivity issues caused by Synology DSM firewall rule resets after system updates.

## Introduction

After Synology DSM updates, firewall rules are sometimes reset or reordered, which can block access to Portainer and other Docker containers. This guide covers how to diagnose and permanently fix these issues so your Portainer installation survives future updates.

## Understanding the Problem

Synology DSM's built-in firewall manages access to the NAS at the system level. Docker containers expose ports through the host network, so they are subject to the same firewall rules. After DSM updates:

- Custom firewall rules may be reset to defaults
- Rule ordering may change, with deny-all rules taking precedence
- Docker's iptables rules may conflict with DSM firewall rules

## Step 1: Diagnose the Issue

### Check Container Status

First, verify Portainer is actually running:

```bash
# SSH into Synology

ssh admin@<synology-ip>

# Check container status
sudo docker ps | grep portainer

# Check container logs for errors
sudo docker logs portainer --tail 50
```

### Check Port Binding

Verify the port is bound on the host:

```bash
# Check if port 9000 is listening
sudo netstat -tlnp | grep 9000
# or
sudo ss -tlnp | grep 9000
```

### Test Local Connectivity

Test from the Synology itself:

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:9000
```

If this returns `200` but you can't access it from your network, the issue is the firewall.

## Step 2: Fix the DSM Firewall

### Via DSM Web Interface

1. Go to **Control Panel > Security > Firewall**
2. Click **Edit Rules** for the active profile
3. Check if a deny-all rule exists at the top of the list
4. Look for rules allowing ports 9000 and 9443

If the rules are missing, add them:

1. Click **Create**
2. Set **Ports**: Custom, TCP, `9000`
3. Set **Source IP**: Specific IP or subnet (e.g., `192.168.1.0/24`)
4. Set **Action**: Allow
5. Repeat for port `9443`
6. **Drag** the allow rules above any deny-all rule

### Understanding Rule Order

DSM firewall rules are evaluated top-to-bottom. The first matching rule wins:

```text
# Correct order:
1. ALLOW TCP 9000 from 192.168.1.0/24  ← Must be ABOVE the deny rule
2. ALLOW TCP 9443 from 192.168.1.0/24
3. DENY ALL (default)
```

## Step 3: Fix Docker iptables Conflicts

DSM updates can reset iptables rules that Docker relies on. Fix this by restarting Docker:

```bash
# Restart Container Manager to restore Docker iptables rules
sudo synoservicectl --restart pkgctl-ContainerManager
```

Or via DSM:
1. Open **Package Center**
2. Click on **Container Manager**
3. Click **Stop**, wait 10 seconds, then **Start**

## Step 4: Create a Persistent Firewall Fix Script

Create a Task Scheduler script that runs after boot to ensure firewall rules are correct:

```bash
#!/bin/bash
# Fix Docker container firewall rules after DSM updates
# Run this as a boot-up task in Task Scheduler (root user)

# Wait for network services to start
sleep 15

# Add iptables rules to allow Portainer ports
# These supplement DSM firewall rules at the kernel level
iptables -I INPUT -p tcp --dport 9000 -j ACCEPT
iptables -I INPUT -p tcp --dport 9443 -j ACCEPT

# Also allow Docker daemon API port (internal use only)
iptables -I INPUT -s 172.17.0.0/16 -j ACCEPT

echo "Firewall rules applied at $(date)" >> /var/log/docker-firewall.log
```

Add this as a boot-up task in Task Scheduler:
- Task name: `Fix Docker Firewall`
- User: `root`
- Event: Boot-up

## Step 5: Prevent Future Issues with Firewall Profile

Create a dedicated firewall profile that you can reapply after updates:

1. In **Control Panel > Security > Firewall**, click **Create profile**
2. Name it `Docker Services`
3. Add all your Docker port rules to this profile
4. Before applying a DSM update, note which profile is active
5. After the update, reapply your profile if it was reset

## Step 6: Fix Docker Network Interface Issues

Sometimes DSM updates remove Docker's bridge network interface:

```bash
# Check if docker0 interface exists
ip link show docker0

# If missing, restart Docker
sudo synoservicectl --restart pkgctl-ContainerManager

# Verify Docker networks are restored
sudo docker network ls
```

## Step 7: Verify All Container Ports

After any DSM update, do a quick audit:

```bash
# List all running containers with their ports
sudo docker ps --format "table {{.Names}}\t{{.Ports}}"

# Test connectivity to each
for port in 9000 9443 3000 8080; do
    if curl -s -o /dev/null --connect-timeout 3 http://localhost:$port; then
        echo "Port $port: OK"
    else
        echo "Port $port: FAILED"
    fi
done
```

## Conclusion

Firewall issues after Synology DSM updates are a common frustration for home lab users. The key is understanding that DSM firewall rules may be reset and that Docker's iptables rules need to be in the correct state. By creating persistent boot-up tasks and a dedicated firewall profile, you can ensure Portainer and your other containers remain accessible after every update.

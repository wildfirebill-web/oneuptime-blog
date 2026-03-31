# How to Set Up Edge Agent Behind a NAT or Firewall - Portainer Behind

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Edge Agent, NAT, Firewall, Network, Remote Access

Description: Deploy and configure Portainer Edge Agents in NAT or firewalled environments where devices don't have public IP addresses.

## Introduction

NAT (Network Address Translation) is ubiquitous - most edge devices sit behind a home router, office NAT gateway, or corporate firewall without a public IP. The Portainer Edge Agent is specifically designed for this scenario: it initiates outbound connections, making it transparent to NAT devices.

## How Edge Agent Works Through NAT

The key design principle:

```yaml
[Edge Device behind NAT]                [Internet]           [Portainer Server]
    |                                                              |
    |------ TCP SYN to portainer.example.com:8000 ------------>  |
    |<----- TCP SYN-ACK (NAT tracks this connection) ----------  |
    |                                                              |
    |  [Persistent outbound connection - NAT allows return traffic]
    |                                                              |
    Portainer sends commands via this established connection
```

No inbound ports needed on the edge device's NAT gateway.

## Requirements

From the edge device (outbound only):
- **Port 443 outbound** → Portainer HTTPS for registration and API
- **Port 8000 outbound** → Portainer Tunnel Server for command channel

## Verifying Outbound Connectivity

```bash
# Test from behind NAT

curl -sv https://portainer.example.com/api/system/version 2>&1 | head -20
# Should succeed if port 443 is allowed outbound

nc -zv portainer.example.com 8000
# Should succeed if port 8000 is allowed outbound
```

## Deploying Edge Agent Behind NAT

```bash
# Standard edge agent deployment (works through NAT automatically)
docker run -d \
  --name portainer_edge_agent \
  --restart unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -v /var/run/portainer:/var/run/portainer \
  -e EDGE=1 \
  -e EDGE_ID=nat-device-001 \
  -e EDGE_KEY=portainer-edge-key \
  portainer/agent:latest
```

No special NAT configuration needed. The agent initiates the connection.

## Handling Strict Corporate Firewalls

Some corporate firewalls:
- Block all outbound except HTTP/HTTPS (ports 80/443)
- Inspect HTTPS traffic (SSL inspection)
- Block non-standard ports like 8000

### If Port 8000 is Blocked

Configure Portainer to use port 443 for the tunnel server:

```bash
# On Portainer Server - expose tunnel server on port 443
# Via docker run
docker run -d \
  --name portainer \
  -p 443:9443 \
  -p 443:8000 \   # This conflicts - use separate port mappings
  portainer/portainer-ce:latest
```

Or configure a reverse proxy to forward port 443 to the tunnel server:

```nginx
# nginx.conf - Route based on SNI/path
stream {
    upstream tunnel {
        server portainer:8000;
    }
    
    server {
        listen 443;
        proxy_pass tunnel;
    }
}
```

Then set the tunnel server address to use port 443:
```bash
-e EDGE_KEY=key-with-443-tunnel-address
```

## Multi-Layer NAT

For devices behind multiple NAT layers (common in enterprise):

```text
Device → Office NAT → ISP NAT → Portainer
```

As long as outbound TCP to portainer.example.com:8000 is allowed at each layer, the edge agent works transparently.

## Conclusion

The Edge Agent's outbound-connection model makes it inherently NAT-friendly. The most common issue in NAT environments isn't NAT itself but corporate firewalls blocking outbound port 8000. If the standard ports are blocked, configure the Portainer tunnel server to listen on port 443 (HTTPS) which is almost universally allowed outbound.

# How to Configure the Tunnel Server Address for Edge Agents - Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Edge Agent, Tunnel Server, Configuration, Network

Description: Configure the Portainer tunnel server address and port that Edge Agents use to establish their management connection.

## Introduction

The Portainer tunnel server is the component that handles communication between edge agents and the Portainer server. When running Portainer in a custom network environment, you may need to configure a specific address or port for the tunnel server - particularly when using a reverse proxy or when running on non-standard ports.

## Default Tunnel Server Configuration

By default, Portainer's tunnel server:
- Listens on port **8000**
- Uses the same hostname as the Portainer server
- Is included automatically in edge keys

## Configuring the Tunnel Server Address

When starting Portainer, specify the tunnel server address:

```bash
docker run -d \
  --name portainer \
  --restart always \
  -p 443:9443 \
  -p 8000:8000 \    # Expose tunnel server port
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --tunnel-addr=portainer.example.com \
  --tunnel-port=8000
```

```yaml
# docker-compose.yml

services:
  portainer:
    image: portainer/portainer-ce:latest
    ports:
      - "443:9443"
      - "8000:8000"
    command:
      - "--tunnel-addr=portainer.example.com"
      - "--tunnel-port=8000"
```

## Using a Non-Standard Tunnel Port

If port 8000 is in use or blocked, use a different port:

```bash
docker run -d \
  -p 443:9443 \
  -p 18000:18000 \
  portainer/portainer-ce:latest \
  --tunnel-addr=portainer.example.com \
  --tunnel-port=18000
```

Edge agents receive the new port in their edge key.

## Tunnel Server Behind a Reverse Proxy

When Portainer is behind Nginx, the tunnel server needs TCP passthrough (not HTTP proxying):

```nginx
# nginx.conf - stream block for TCP tunnel passthrough
stream {
    server {
        listen 8000;
        proxy_pass portainer:8000;
        proxy_timeout 1d;    # Keep connections alive
    }
}
```

For Traefik with TCP support:

```yaml
# traefik.yml additions
tcp:
  routers:
    portainer-tunnel:
      rule: "HostSNI(`portainer.example.com`)"
      service: portainer-tunnel-svc
      entryPoints:
        - tunnel
      tls:
        passthrough: true
  services:
    portainer-tunnel-svc:
      loadBalancer:
        servers:
          - address: "portainer:8000"

entryPoints:
  tunnel:
    address: ":8000"
```

## Viewing the Tunnel Server Address in Edge Keys

```bash
# Decode an edge key to see the tunnel server configuration
EDGE_KEY="your-edge-key"
echo $EDGE_KEY | base64 -d 2>/dev/null | python3 -m json.tool

# Look for "tunnelServerAddr" field
```

## Checking Tunnel Server Availability

```bash
# Test tunnel server port
nc -zv portainer.example.com 8000

# Test from inside Docker network
docker run --rm alpine nc -zv portainer:8000

# Check Portainer logs for tunnel server status
docker logs portainer 2>&1 | grep -i "tunnel"
```

## Conclusion

The tunnel server address configuration is critical for edge agent connectivity. When Portainer is behind a reverse proxy or running on custom ports, ensure the tunnel server address in the edge key matches the publicly accessible address. Test port 8000 connectivity from representative edge device locations before deploying agents at scale.

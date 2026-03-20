# How to Set Up Cloudflare Tunnel for Portainer Access

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Cloudflare, Tunnel, Zero Trust, Security

Description: Learn how to use Cloudflare Tunnel (formerly Argo Tunnel) to securely expose Portainer without opening any inbound firewall ports, using Cloudflare's global network as the access point.

## What Is Cloudflare Tunnel?

Cloudflare Tunnel creates an outbound-only connection from your server to Cloudflare's network. Cloudflare then routes HTTPS traffic from your domain to Portainer through this tunnel — no ports need to be opened on your firewall.

```
Browser → Cloudflare (your domain) → Tunnel → cloudflared → Portainer
                                    (no inbound ports needed)
```

## Benefits Over Port Forwarding

- No open inbound firewall ports
- DDoS protection via Cloudflare
- Built-in HTTPS with Cloudflare certificates
- Integrates with Cloudflare Access for zero-trust auth
- Works behind NAT and CGNAT

## Prerequisites

- Cloudflare account with your domain added
- `cloudflared` CLI or Cloudflare Zero Trust dashboard access

## Method 1: Using cloudflared CLI

```bash
# Install cloudflared
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o cloudflared.deb
dpkg -i cloudflared.deb

# Authenticate (opens browser)
cloudflared tunnel login

# Create a tunnel
cloudflared tunnel create portainer-tunnel

# Configure the tunnel
cat > ~/.cloudflared/config.yml << 'EOF'
tunnel: <TUNNEL_ID>
credentials-file: /root/.cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: portainer.yourdomain.com
    service: http://localhost:9000
  - service: http_status:404
EOF

# Create DNS record
cloudflared tunnel route dns portainer-tunnel portainer.yourdomain.com

# Run the tunnel
cloudflared tunnel run portainer-tunnel
```

## Method 2: Using Cloudflare Zero Trust Dashboard (Recommended)

1. Go to [Cloudflare Zero Trust](https://one.dash.cloudflare.com) → **Networks → Tunnels**
2. Click **Create Tunnel**
3. Select **Cloudflared**
4. Name it `portainer`
5. Copy the install command shown (includes your token):

```bash
# Cloudflare provides this command — run it on your server
cloudflared service install <YOUR_TOKEN>
```

6. Under **Public Hostname**:
   - Subdomain: `portainer`
   - Domain: `yourdomain.com`
   - Service: `http://localhost:9000`

Cloudflare automatically creates the DNS record and the tunnel is active.

## Accessing Portainer Through the Tunnel

After configuration, Portainer is available at:
`https://portainer.yourdomain.com`

No port 9000 needs to be published on the Docker container:

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    # No ports needed — tunnel handles external access
    restart: unless-stopped
```

## Running cloudflared as a Docker Container

```yaml
# Add to your docker-compose.yml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}
    restart: unless-stopped
    environment:
      - CLOUDFLARE_TUNNEL_TOKEN=${CLOUDFLARE_TUNNEL_TOKEN}
    networks:
      - default    # Same network as Portainer

  portainer:
    image: portainer/portainer-ce:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    restart: unless-stopped
```

Set `CLOUDFLARE_TUNNEL_TOKEN` in Portainer's environment variables.

## Security Note

Combine with Cloudflare Access for authentication before users reach Portainer:

1. In Zero Trust → **Access → Applications → Add Application**
2. Type: **Self-hosted**
3. Application domain: `portainer.yourdomain.com`
4. Create a policy: require specific email domains or identity provider

## Conclusion

Cloudflare Tunnel is one of the most secure ways to expose Portainer externally. With no open inbound ports, DDoS protection, and optional Cloudflare Access MFA, it provides enterprise-grade security for home labs and production environments alike.

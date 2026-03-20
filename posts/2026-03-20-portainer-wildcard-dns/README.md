# How to Set Up Wildcard DNS for Portainer Services

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, DNS, Wildcard, Traefik, SSL

Description: Configure wildcard DNS records and wildcard SSL certificates to automatically cover all subdomains for Portainer-managed services.

## Introduction

Wildcard DNS allows a single DNS record (`*.example.com`) to match any subdomain. Combined with wildcard SSL certificates, this eliminates the need to create individual DNS records and certificates for each new service you deploy via Portainer. New services get automatic HTTPS access just by adding Traefik labels.

## How Wildcard DNS Works

A wildcard A record like `*.services.example.com → 192.168.1.50` means:
- `app.services.example.com` → resolves to `192.168.1.50`
- `db-admin.services.example.com` → resolves to `192.168.1.50`
- `new-service.services.example.com` → resolves to `192.168.1.50`

Your reverse proxy (Traefik) then routes each hostname to the correct container.

## Step 1: Create Wildcard DNS Record

```bash
# Using Cloudflare API

CF_ZONE_ID="your-zone-id"
CF_API_TOKEN="your-api-token"
SERVER_IP="203.0.113.10"

curl -X POST \
  "https://api.cloudflare.com/client/v4/zones/$CF_ZONE_ID/dns_records" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"type\": \"A\",
    \"name\": \"*.services.example.com\",
    \"content\": \"$SERVER_IP\",
    \"ttl\": 300,
    \"proxied\": false
  }"

# Also add the root record if needed
curl -X POST \
  "https://api.cloudflare.com/client/v4/zones/$CF_ZONE_ID/dns_records" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"type\": \"A\",
    \"name\": \"services.example.com\",
    \"content\": \"$SERVER_IP\",
    \"ttl\": 300
  }"
```

## Step 2: Obtain Wildcard SSL Certificate

Let's Encrypt supports wildcard certificates via DNS-01 challenge:

```bash
# Install certbot with Cloudflare plugin
sudo apt-get install -y python3-certbot-dns-cloudflare

# Create Cloudflare credentials file
mkdir -p ~/.secrets
tee ~/.secrets/cloudflare.ini << 'EOF'
dns_cloudflare_api_token = your-cloudflare-api-token
EOF
chmod 600 ~/.secrets/cloudflare.ini

# Request wildcard certificate
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials ~/.secrets/cloudflare.ini \
  -d "services.example.com" \
  -d "*.services.example.com" \
  --non-interactive \
  --agree-tos \
  -m admin@example.com

# Verify certificate
sudo certbot certificates
```

## Step 3: Configure Traefik with Wildcard Certificates

```yaml
# traefik-stack.yml
version: '3.8'
services:
  traefik:
    image: traefik:v3.0
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    environment:
      CF_API_TOKEN: "${CF_API_TOKEN}"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik-certs:/letsencrypt
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
    networks:
      - proxy

volumes:
  traefik-certs:

networks:
  proxy:
    external: true
```

```yaml
# traefik.yml - static configuration
api:
  dashboard: true

providers:
  docker:
    exposedByDefault: false
    network: proxy

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
  websecure:
    address: ":443"

certificatesResolvers:
  cloudflare:
    acme:
      email: admin@example.com
      storage: /letsencrypt/acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"
```

## Step 4: Deploy Services Under Wildcard Domain

With wildcard DNS and certificates in place, deploying a new service is simple:

```yaml
# new-service-stack.yml
version: '3.8'
services:
  my-new-service:
    image: my-service:latest
    labels:
      - traefik.enable=true
      # Automatically uses wildcard cert
      - traefik.http.routers.my-service.rule=Host(`my-new-service.services.example.com`)
      - traefik.http.routers.my-service.entrypoints=websecure
      - traefik.http.routers.my-service.tls=true
      # Use the wildcard certificate resolver
      - traefik.http.routers.my-service.tls.certresolver=cloudflare
      - traefik.http.services.my-service.loadbalancer.server.port=3000
    networks:
      - proxy

networks:
  proxy:
    external: true
```

## Step 5: Automatic Certificate Renewal

```bash
# Set up automatic renewal
sudo tee /etc/cron.d/certbot-renewal << 'EOF'
0 0 * * * root certbot renew --quiet --deploy-hook "docker exec traefik kill -s HUP 1"
EOF

# Or use a systemd timer
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer

# Test renewal
sudo certbot renew --dry-run
```

## Testing Wildcard DNS

```bash
# Test DNS resolution
dig new-service.services.example.com
nslookup another-service.services.example.com

# Test HTTPS
curl -v https://new-service.services.example.com

# Verify certificate covers wildcard
echo | openssl s_client -connect new-service.services.example.com:443 2>/dev/null \
  | openssl x509 -noout -subject -issuer
```

## Conclusion

Wildcard DNS and certificates with Portainer and Traefik create a zero-configuration service discovery system. Every new service you deploy only needs a Traefik label with the desired subdomain-no DNS records to create, no certificates to request. This makes deploying new services a one-step process, perfect for rapid iteration and microservice architectures.

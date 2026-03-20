# How to Fix Portainer Self-Signed Certificate Warnings

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, SSL, Self-Signed Certificate, TLS, Troubleshooting

Description: Learn how to fix browser and API client warnings about Portainer's self-signed certificate with trusted certificates or proper CA configuration.

---

Out of the box, Portainer generates a self-signed TLS certificate. While it provides encryption, browsers and tools like `curl` display trust warnings because no recognized CA has signed the certificate.

## Understanding the Warning

The warning `NET::ERR_CERT_AUTHORITY_INVALID` or "Your connection is not private" appears because:
- Portainer's auto-generated certificate is self-signed
- No public CA has vouched for it
- The certificate may not match the domain/IP you're using

## Solution 1: Use a Trusted CA Certificate (Best)

Replace the self-signed cert with one from Let's Encrypt or your commercial CA:

```bash
# Obtain a Let's Encrypt certificate (requires public domain)
sudo certbot certonly --standalone \
  -d portainer.example.com \
  --email admin@example.com \
  --agree-tos

# Run Portainer with the trusted certificate
docker stop portainer && docker container rm portainer

docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  -v /etc/letsencrypt:/letsencrypt:ro \
  portainer/portainer-ce:latest \
  --ssl \
  --sslcert /letsencrypt/live/portainer.example.com/fullchain.pem \
  --sslkey /letsencrypt/live/portainer.example.com/privkey.pem
```

## Solution 2: Create a CA-Signed Internal Certificate

For internal deployments without a public domain, create a local CA:

```bash
# Create an internal CA
openssl genrsa -out internal-ca.key 4096
openssl req -x509 -new -nodes -key internal-ca.key -sha256 -days 3650 \
  -out internal-ca.crt -subj "/C=US/O=Internal/CN=Internal CA"

# Create a certificate signed by the internal CA
openssl genrsa -out portainer.key 2048

# Create CSR with SAN for your IP/hostname
cat > portainer.cnf << EOF
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = req_ext

[dn]
C = US
O = Internal
CN = portainer.internal

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = portainer.internal
DNS.2 = localhost
IP.1 = 192.168.1.100
EOF

openssl req -new -key portainer.key -out portainer.csr -config portainer.cnf
openssl x509 -req -in portainer.csr -CA internal-ca.crt -CAkey internal-ca.key \
  -CAcreateserial -out portainer.crt -days 365 -extensions req_ext -extfile portainer.cnf

# Trust the internal CA on all clients
sudo cp internal-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

## Solution 3: Add Certificate Exception in the Browser

For development environments only — permanently trust the certificate:

**Chrome/Edge:**
1. Click "Advanced" on the warning page
2. Click "Proceed to localhost (unsafe)"
3. For permanent trust: visit `chrome://flags/#allow-insecure-localhost`

**Firefox:**
1. Click "Advanced..."
2. Click "Accept the Risk and Continue"
3. This creates a permanent exception for this site

## Solution 4: Trust the Self-Signed Cert via OS

Extract and trust Portainer's generated certificate on all clients:

```bash
# Extract the certificate Portainer is currently using
openssl s_client -connect localhost:9443 </dev/null 2>/dev/null | \
  openssl x509 -outform PEM > /tmp/portainer-cert.pem

# Trust it on Ubuntu/Debian
sudo cp /tmp/portainer-cert.pem /usr/local/share/ca-certificates/portainer.crt
sudo update-ca-certificates

# Trust it on RHEL/CentOS
sudo cp /tmp/portainer-cert.pem /etc/pki/ca-trust/source/anchors/portainer.pem
sudo update-ca-trust
```

---

*Monitor SSL certificate health automatically with [OneUptime](https://oneuptime.com) before issues reach users.*

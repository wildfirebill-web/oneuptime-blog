# How to Configure GitLab Container Registry with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GitLab, Container Registry, IPv6, Docker, DevOps, CI/CD

Description: Configure GitLab's integrated container registry to accept connections over IPv6, enabling Docker image storage and CI/CD pipelines on IPv6-enabled infrastructure.

---

GitLab includes a built-in container registry for storing Docker images. Enabling IPv6 access requires configuring both GitLab's Nginx proxy and the registry service to listen on IPv6 interfaces.

## GitLab Omnibus IPv6 Configuration

For GitLab Omnibus (all-in-one installation):

```ruby
# /etc/gitlab/gitlab.rb

# Enable container registry
registry_external_url 'https://registry.example.com'

# Configure registry to listen on IPv6
registry['enable'] = true

# Nginx for registry (includes IPv6 listening)
registry_nginx['listen_addresses'] = ['0.0.0.0', '::']
registry_nginx['listen_port'] = 5050

# Main GitLab Nginx with IPv6
nginx['listen_addresses'] = ['0.0.0.0', '::']
nginx['listen_port'] = 80
nginx['listen_https_port'] = 443

# GitLab URL
external_url 'https://gitlab.example.com'
```

```bash
# Apply configuration
sudo gitlab-ctl reconfigure

# Verify registry is running
sudo gitlab-ctl status registry

# Check listening ports
ss -tlnp | grep :5050
ss -tlnp | grep :443
```

## Docker Compose GitLab with IPv6

For Docker Compose GitLab deployments:

```yaml
# docker-compose.yml
version: '3.8'

services:
  gitlab:
    image: 'gitlab/gitlab-ce:latest'
    restart: always
    hostname: 'gitlab.example.com'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.example.com'
        registry_external_url 'https://registry.example.com'
        nginx['listen_addresses'] = ['0.0.0.0', '::']
        registry_nginx['listen_addresses'] = ['0.0.0.0', '::']
    ports:
      - '80:80'
      - '[::]:80:80'
      - '443:443'
      - '[::]:443:443'
      - '5050:5050'
      - '[::]:5050:5050'
    volumes:
      - '/srv/gitlab/config:/etc/gitlab'
      - '/srv/gitlab/logs:/var/log/gitlab'
      - '/srv/gitlab/data:/var/opt/gitlab'

networks:
  default:
    enable_ipv6: true
    ipam:
      config:
        - subnet: "2001:db8:gitlab::/80"
        - subnet: "172.25.0.0/16"
```

## Configuring DNS for GitLab Registry IPv6

```bash
# Add AAAA records for GitLab and registry
# registry.example.com. IN AAAA 2001:db8::30
# gitlab.example.com.   IN AAAA 2001:db8::30

# Verify DNS
dig AAAA registry.example.com +short
dig AAAA gitlab.example.com +short
```

## TLS Certificate for Registry with IPv6 SAN

```bash
# Generate cert with both hostname and IPv6 SAN
cat > /tmp/registry-cert.cnf << 'EOF'
[req]
default_bits = 2048
prompt = no
distinguished_name = dn
x509_extensions = v3_req

[dn]
CN = registry.example.com

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = registry.example.com
DNS.2 = gitlab.example.com
IP.1  = 2001:db8::30
EOF

# Or use Let's Encrypt with DNS-01 challenge (preferred for production)
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  --domain registry.example.com \
  --domain gitlab.example.com
```

## Authenticating and Using the Registry over IPv6

```bash
# Login to GitLab Container Registry
docker login registry.example.com \
  -u yourusername \
  -p your_access_token

# Tag and push an image
docker tag myapp:latest registry.example.com/mygroup/myproject/myapp:latest
docker push registry.example.com/mygroup/myproject/myapp:latest

# Use in GitLab CI/CD .gitlab-ci.yml
```

```yaml
# .gitlab-ci.yml
build:
  image: docker:24
  services:
    - docker:24-dind
  variables:
    # Docker registry login
    REGISTRY: $CI_REGISTRY
    IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $IMAGE .
    - docker push $IMAGE
```

## Troubleshooting GitLab Registry IPv6

```bash
# Check registry logs
sudo gitlab-ctl tail registry

# Check if registry is listening on IPv6
ss -tlnp | grep "5050\|registry"

# Test registry endpoint over IPv6
curl -6 https://registry.example.com/v2/ \
  -H "Authorization: Basic $(echo -n 'user:token' | base64)"

# Check Nginx configuration for registry
sudo cat /var/opt/gitlab/nginx/conf/gitlab-registry.conf | grep "listen"
```

GitLab's container registry with IPv6 support enables CI/CD pipelines to build and store container images over IPv6, supporting modern dual-stack deployment patterns for containerized applications.

# How to Configure Nexus Repository with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nexus Repository, IPv6, Maven, Docker, Artifact Management, Sonatype, DevOps

Description: Configure Sonatype Nexus Repository Manager to listen on IPv6 interfaces and serve Maven, Docker, npm, and other artifact types to IPv6 clients.

---

Sonatype Nexus Repository Manager is a popular artifact repository supporting Maven, Docker, npm, PyPI, and more. Enabling IPv6 requires configuring the Jetty web server's listener and optional Nginx reverse proxy.

## Installing Nexus Repository

```bash
# Download Nexus Repository Manager

wget https://download.sonatype.com/nexus/3/latest-unix.tar.gz
tar xvf latest-unix.tar.gz
sudo mv nexus-3* /opt/nexus
sudo mv sonatype-work /opt/sonatype-work

# Create nexus user
sudo useradd -r -M -s /sbin/nologin nexus
sudo chown -R nexus:nexus /opt/nexus /opt/sonatype-work

# Create systemd service
cat > /etc/systemd/system/nexus.service << 'EOF'
[Unit]
Description=Nexus Repository Manager
After=network.target

[Service]
Type=forking
User=nexus
ExecStart=/opt/nexus/bin/nexus start
ExecStop=/opt/nexus/bin/nexus stop
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now nexus
```

## Configuring Nexus to Listen on IPv6

Nexus uses Jetty. Edit `nexus-default.properties`:

```properties
# /opt/nexus/etc/nexus-default.properties

# Change the listen address to support IPv6
# Default listens on 0.0.0.0 (IPv4 only on some systems)
# For dual-stack support:
application-port=8081
application-host=::

# For HTTPS (if using SSL directly)
# application-port-ssl=8443
```

For Jetty XML configuration:

```xml
<!-- /opt/nexus/etc/jetty/jetty.xml or jetty-http.xml -->
<New class="org.eclipse.jetty.server.ServerConnector">
  <Arg name="server"><Ref id="Server"/></Arg>
  <Set name="host">
    <!-- Listen on all interfaces including IPv6 -->
    <!-- Empty string or null = all interfaces -->
  </Set>
  <Set name="port"><Property name="application-port" default="8081"/></Set>
</New>
```

```bash
# Restart Nexus to apply changes
sudo systemctl restart nexus

# Verify Nexus is listening on IPv6
ss -tlnp | grep :8081
# Expected: [::]:8081
```

## Nginx Reverse Proxy for Nexus IPv6

For production, use Nginx as a reverse proxy:

```nginx
# /etc/nginx/sites-available/nexus
upstream nexus {
    server 127.0.0.1:8081;
}

server {
    listen 80;
    listen [::]:80;

    server_name nexus.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name nexus.example.com;

    ssl_certificate     /etc/letsencrypt/live/nexus.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/nexus.example.com/privkey.pem;

    # Required for large artifact uploads
    client_max_body_size 5G;

    location / {
        proxy_pass http://nexus;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Using Nexus Docker Registry over IPv6

```bash
# Login to Nexus Docker registry
docker login nexus.example.com:8082 \
  -u admin \
  -p password

# Or using the repository-specific URL
docker login nexus.example.com \
  -u admin \
  -p password

# Push image
docker tag myapp:latest nexus.example.com/docker-hosted/myapp:latest
docker push nexus.example.com/docker-hosted/myapp:latest

# Pull through proxy (npm, Maven, Docker)
docker pull nexus.example.com/docker-proxy/nginx:latest
```

## Configuring Maven to Use Nexus over IPv6

```xml
<!-- ~/.m2/settings.xml -->
<settings>
  <mirrors>
    <mirror>
      <id>nexus</id>
      <mirrorOf>*</mirrorOf>
      <url>https://nexus.example.com/repository/maven-public/</url>
      <name>Nexus Maven Mirror</name>
    </mirror>
  </mirrors>

  <servers>
    <server>
      <id>nexus</id>
      <username>admin</username>
      <password>password</password>
    </server>
  </servers>
</settings>
```

## Verifying IPv6 Access to Nexus

```bash
# Check status over IPv6
curl -6 https://nexus.example.com/service/rest/v1/status

# Test Docker registry API
curl -6 -u admin:password \
  https://nexus.example.com/v2/ \
  -H "Accept: application/json"

# Test Maven artifact retrieval
curl -6 -u admin:password \
  "https://nexus.example.com/repository/maven-central/junit/junit/4.13.2/junit-4.13.2.pom"
```

Nexus Repository's Jetty-based architecture makes IPv6 configuration straightforward through the `application-host` property, enabling IPv6-accessible artifact management for all supported repository formats.

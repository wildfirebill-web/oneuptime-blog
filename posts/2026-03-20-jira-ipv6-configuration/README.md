# How to Configure Jira with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Jira, IPv6, Atlassian, Project Management, Tomcat, Linux

Description: Configure Jira Software or Jira Service Management to accept connections from IPv6 clients by updating Tomcat server configuration and reverse proxy settings.

---

Jira runs on Apache Tomcat which supports IPv6. Configuring Jira for IPv6 access involves updating the Tomcat connector settings and, if using a reverse proxy (Nginx/Apache), ensuring the proxy also listens on IPv6.

## Jira Tomcat IPv6 Configuration

```xml
<!-- /opt/atlassian/jira/conf/server.xml -->

<!-- Connector listening on all interfaces including IPv6 -->
<Connector port="8080"
           relaxedPathChars="[]|"
           relaxedQueryChars="[]|{}^&#x5c;&#x60;&quot;&lt;&gt;"
           maxThreads="150"
           minSpareThreads="25"
           connectionTimeout="20000"
           enableLookups="false"
           maxHttpHeaderSize="8192"
           protocol="HTTP/1.1"
           useBodyEncodingForURI="true"
           redirectPort="8443"
           acceptCount="100"
           disableUploadTimeout="true"
           address="::"
           URIEncoding="UTF-8"/>

<!-- HTTPS connector on IPv6 -->
<!--
<Connector port="8443"
           address="::"
           protocol="org.apache.coyote.http11.Http11NioProtocol"
           .../>
-->
```

```bash
# Restart Jira after configuration change

sudo systemctl restart jira

# Verify Jira is listening on IPv6
ss -6 -tlnp | grep 8080
```

## Nginx Reverse Proxy for Jira IPv6

```nginx
# /etc/nginx/sites-available/jira
server {
    listen 80;
    listen [::]:80;
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name jira.example.com;

    ssl_certificate /etc/ssl/certs/jira.crt;
    ssl_certificate_key /etc/ssl/private/jira.key;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Update Jira Base URL for IPv6

```text
Jira Administration > System > General Configuration

Base URL: https://jira.example.com
(Use FQDN with AAAA record, not raw IPv6 address)

This ensures:
- Correct links in emails and webhooks
- Proper OAuth callback URLs
- Correct redirect URLs for SAML/SSO
```

## Jira Database Connection over IPv6

```xml
<!-- /opt/atlassian/jira/dbconfig.xml -->

<jira-database-config>
  <name>defaultDS</name>
  <database-type>postgres72</database-type>
  <jdbc-datasource>
    <url>jdbc:postgresql://[2001:db8::postgres]:5432/jiradb</url>
    <driver-class>org.postgresql.Driver</driver-class>
    <username>jira</username>
    <password>dbpassword</password>
    <pool-min-size>20</pool-min-size>
    <pool-max-size>20</pool-max-size>
  </jdbc-datasource>
</jira-database-config>
```

## Jira System Properties for IPv6

```bash
# /opt/atlassian/jira/bin/setenv.sh

# Force Jira/JVM to prefer IPv6
export JAVA_OPTS="$JAVA_OPTS -Djava.net.preferIPv6Addresses=true"

# Or prefer IPv4 stack (default JVM behavior)
# export JAVA_OPTS="$JAVA_OPTS -Djava.net.preferIPv4Stack=true"

# For IPv6-only environments
export JAVA_OPTS="$JAVA_OPTS -Djava.net.preferIPv6Addresses=true -Djava.net.preferIPv4Stack=false"
```

## Firewall for Jira IPv6

```bash
# Allow Jira port over IPv6
sudo ip6tables -A INPUT -p tcp --dport 8080 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 443 -j ACCEPT

# If behind Nginx reverse proxy, only allow from localhost
sudo ip6tables -A INPUT -p tcp -s ::1 --dport 8080 -j ACCEPT
sudo ip6tables -A INPUT -p tcp \
  -s 2001:db8:internal::/48 \
  --dport 8080 -j ACCEPT

sudo ip6tables-save > /etc/ip6tables/rules.v6
```

## Testing Jira IPv6 Access

```bash
# Verify listening on IPv6
ss -6 -tlnp | grep 8080

# Test HTTP access over IPv6
curl -6 -L http://[2001:db8::jira]:8080/jira/

# Test via proxy
curl -6 https://jira.example.com/ -I

# Check Jira application logs for IPv6 connections
sudo tail -f /opt/atlassian/jira/logs/atlassian-jira.log | grep "2001:"
```

Jira's Tomcat-based architecture enables IPv6 support through the `address="::"` attribute in server.xml, with most deployments accessing Jira via a reverse proxy like Nginx that can independently handle IPv6 listener configuration.

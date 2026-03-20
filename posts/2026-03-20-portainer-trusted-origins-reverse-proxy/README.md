# How to Configure Trusted Origins in Portainer for Reverse Proxies

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Reverse Proxy, Security, CSRF, Configuration

Description: Learn how to properly configure the --trusted-origins flag in Portainer to allow secure access through reverse proxies and custom domains.

## Introduction

Portainer's `--trusted-origins` configuration is a security mechanism that controls which HTTP origins are allowed to make requests to the Portainer API. When deploying behind a reverse proxy, configuring this correctly is essential for both security and functionality.

## Understanding Trusted Origins

The trusted origins check operates as follows:

1. The browser sends an `Origin` header with every cross-origin request
2. Portainer compares this header against its list of trusted origins
3. If the origin is not in the list, the request is rejected with "access denied"

The default trusted origin is derived from the URL Portainer is accessed on directly. Once behind a proxy with a different hostname, you must explicitly add the proxy URL.

## Basic Configuration

### Via Command Flag

```bash
# Single origin
docker run -d portainer/portainer-ce:latest \
  --trusted-origins=https://portainer.example.com

# Multiple origins (comma-separated, no spaces)
docker run -d portainer/portainer-ce:latest \
  --trusted-origins=https://portainer.example.com,https://portainer.internal.com

# Include port if non-standard
docker run -d portainer/portainer-ce:latest \
  --trusted-origins=https://portainer.example.com:8443
```

### Via Docker Compose

```yaml
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    command:
      # List each origin on its own line for readability
      - "--trusted-origins=https://portainer.example.com,https://portainer.backup.example.com"
    volumes:
      portainer_data:
```

### Via Environment Variable

In some deployment scenarios, you can use environment variables to pass configuration:

```yaml
  portainer:
    image: portainer/portainer-ce:latest
    environment:
      # Use the environment variable form
      - PORTAINER_TRUSTED_ORIGINS=https://portainer.example.com
```

Note: Not all versions support environment variable configuration. CLI flags are the most reliable approach.

## Multi-Environment Scenarios

### Development + Production

```bash
docker run -d portainer/portainer-ce:latest \
  --trusted-origins=https://portainer.example.com,http://localhost:9000,http://dev.example.internal:9000
```

### Multiple Subdomains

```bash
# Each subdomain must be listed explicitly - wildcards in origins are not supported
docker run -d portainer/portainer-ce:latest \
  --trusted-origins=https://portainer.us.example.com,https://portainer.eu.example.com,https://portainer.ap.example.com
```

### Subpath Deployment

When Portainer is served at a subpath (e.g., `https://example.com/portainer/`), the origin is still just the scheme + host:

```bash
# For https://example.com/portainer/, the origin is https://example.com
docker run -d portainer/portainer-ce:latest \
  --base-url=/portainer \
  --trusted-origins=https://example.com
```

## Verifying Your Configuration

### Check Current Configuration

```bash
# View the Portainer process arguments
docker inspect portainer | python3 -c "import sys,json; c=json.load(sys.stdin); print(c[0]['Config']['Cmd'])"
```

### Test Origin Validation

```bash
# Simulate what the browser sends
curl -v -X OPTIONS \
  -H "Origin: https://portainer.example.com" \
  -H "Access-Control-Request-Method: POST" \
  https://portainer.example.com/api/auth

# Look for "Access-Control-Allow-Origin" in response headers
```

### Monitor Portainer Logs

```bash
# Watch for origin rejection messages
docker logs portainer -f 2>&1 | grep -i "origin\|access denied\|trusted"
```

## Common Mistakes

| Mistake | Correct Approach |
|---------|-----------------|
| `--trusted-origins=portainer.example.com` | Must include scheme: `https://portainer.example.com` |
| `--trusted-origins=https://portainer.example.com/` | No trailing slash |
| Missing port: `https://portainer.example.com` | If custom port: `https://portainer.example.com:8443` |
| Wildcard: `https://*.example.com` | Not supported; list each origin |

## Security Considerations

- Never use `--trusted-origins='*'` in production — it disables CSRF protection
- List only origins your users actually access Portainer from
- Review and update the trusted origins list when changing DNS or proxy configurations
- Treat this list as part of your security configuration

## Conclusion

Properly configured trusted origins are the bridge between Portainer's CSRF protection and your reverse proxy setup. By explicitly listing every URL that users access Portainer through, you maintain security while enabling seamless proxy access. Always use the full URL with scheme and hostname, matching exactly what appears in users' browser address bars.

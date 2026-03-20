# How to Set a Custom Logo in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Customization, Branding, Logo, White-Label

Description: A guide to replacing the default Portainer logo with a custom logo for white-labeling or corporate branding purposes.

## Overview

Portainer Business Edition supports custom logo branding to replace the default Portainer logo with your organization's logo. This is useful for managed service providers, enterprises deploying Portainer as an internal tool, or any organization wanting consistent branding. The CE version does not support logo customization via the UI but can be achieved through other methods.

## Prerequisites

- Portainer Business Edition (for UI-based logo customization)
- A logo image (PNG or SVG recommended, max 800x150px recommended)
- Admin access to Portainer

## Method 1: Custom Logo via Portainer BE UI

1. Log in to Portainer as admin
2. Navigate to **Settings** → **Appearance**
3. In the **Custom branding** section, toggle **Use custom logo**
4. Enter the URL of your logo image:
   - The logo must be accessible from the user's browser (not just from the server)
   - Use a CDN, web server, or public URL
5. Click **Save**

```text
Logo URL example:
https://cdn.example.com/logos/company-logo.png
https://yourcompany.com/assets/portainer-logo.png
```

## Method 2: Host Logo with Nginx

```bash
# Create a simple logo server alongside Portainer

mkdir -p /opt/portainer-branding

# Place your logo in the directory
cp company-logo.png /opt/portainer-branding/

# Run Nginx to serve it
docker run -d \
  --name portainer-branding \
  -p 8080:80 \
  -v /opt/portainer-branding:/usr/share/nginx/html/logos:ro \
  nginx:alpine

# Logo URL to use in Portainer:
# http://your-server:8080/logos/company-logo.png
```

## Method 3: Serve Logo from the Same Nginx Reverse Proxy

```nginx
# Add logo path to existing Nginx config
server {
    listen 443 ssl;
    server_name portainer.example.com;

    # Serve Portainer
    location / {
        proxy_pass https://localhost:9443;
        proxy_ssl_verify off;
    }

    # Serve custom logo
    location /branding/ {
        alias /opt/portainer-branding/;
        expires 7d;
        add_header Cache-Control "public, immutable";
    }
}
```

```bash
# Place logo at /opt/portainer-branding/logo.png
# Reference in Portainer: https://portainer.example.com/branding/logo.png
```

## Logo Guidelines

| Property | Recommendation |
|---|---|
| Format | PNG or SVG |
| Background | Transparent (PNG) |
| Width | 155px or less |
| Height | 55px or less |
| Max file size | 100KB |
| Color mode | RGB |

## Step: Configure in Portainer Settings

```bash
# Verify logo is accessible before configuring
curl -I https://cdn.example.com/logos/company-logo.png
# Expected: HTTP/2 200

# Set via API (BE)
TOKEN="your-admin-token"
curl -X PUT \
  "https://portainer.example.com:9443/api/settings" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"LogoURL": "https://cdn.example.com/logos/company-logo.png"}'
```

## Reverting to Default Portainer Logo

```bash
# Via UI: Settings → Appearance → Disable "Use custom logo"

# Via API
curl -X PUT \
  "https://portainer.example.com:9443/api/settings" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"LogoURL": ""}'
```

## Troubleshooting

```bash
# If logo doesn't appear:
# 1. Verify URL is publicly accessible
curl -I "https://your-logo-url.com/logo.png"

# 2. Check CORS headers (logo must allow cross-origin)
curl -I -H "Origin: https://portainer.example.com" \
  "https://your-logo-url.com/logo.png"

# 3. Check browser console for errors
# Open DevTools → Console → Look for CORS or 404 errors
```

## Conclusion

Custom logo branding in Portainer Business Edition provides a professional, consistent experience for end users. The key requirement is that the logo URL is accessible from user browsers (not just the server). Use a CDN or co-located web server for best performance. For CE users, custom branding requires maintaining a fork or using browser extensions, making BE the practical choice for white-labeling.

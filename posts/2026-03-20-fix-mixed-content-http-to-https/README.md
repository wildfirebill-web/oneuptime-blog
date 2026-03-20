# How to Fix Mixed Content Warnings When Migrating from HTTP to HTTPS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: HTTPS, Mixed Content, TLS, SSL, Web Security, CSP, Nginx, Apache

Description: Learn how to identify and fix mixed content warnings that appear when migrating a website from HTTP to HTTPS, including server-side redirects, CSP headers, and content scanning techniques.

---

Mixed content occurs when an HTTPS page loads resources (images, scripts, stylesheets) over HTTP. Browsers block or warn about these requests, breaking functionality.

## Types of Mixed Content

- **Active mixed content** (scripts, iframes, XHR): Blocked by all modern browsers.
- **Passive mixed content** (images, video, audio): Shown with a warning but often loaded.

## Step 1: Identify Mixed Content

```bash
# Use curl to check for HTTP references in page HTML

curl -sk https://example.com | grep -Eo 'src="http://[^"]*"' | head -20

# Use the browser console: open DevTools → Console tab
# Look for: "Mixed Content: The page was loaded over HTTPS..."
```

```javascript
// In-browser scan using fetch to enumerate mixed content
document.querySelectorAll('[src^="http:"], [href^="http:"]').forEach(el => {
  console.warn("Mixed content:", el.src || el.href);
});
```

## Step 2: Redirect All HTTP to HTTPS at the Server

**Nginx:**
```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
```

**Apache:**
```apache
<VirtualHost *:80>
    ServerName example.com
    Redirect permanent / https://example.com/
</VirtualHost>
```

## Step 3: Use Content Security Policy to Upgrade HTTP Requests

```nginx
# Upgrade insecure requests automatically (where possible)
add_header Content-Security-Policy "upgrade-insecure-requests";
```

This CSP directive tells the browser to fetch HTTP resources over HTTPS automatically without blocking them.

## Step 4: Fix Hardcoded HTTP URLs in HTML/CSS/JS

```bash
# Find hardcoded http:// references in web root
grep -r 'http://' /var/www/html --include="*.html" --include="*.php" --include="*.js" --include="*.css" -l

# Replace http:// with // (protocol-relative) or https://
sed -i 's|http://static.example.com|https://static.example.com|g' /var/www/html/index.html
```

## Step 5: Fix Mixed Content in Databases (CMS)

For WordPress and similar CMS platforms:

```sql
-- Update WordPress database (run in MySQL)
UPDATE wp_options SET option_value = REPLACE(option_value, 'http://example.com', 'https://example.com') WHERE option_name IN ('siteurl', 'home');
UPDATE wp_posts SET post_content = REPLACE(post_content, 'http://example.com', 'https://example.com');
UPDATE wp_postmeta SET meta_value = REPLACE(meta_value, 'http://example.com', 'https://example.com');
```

## Step 6: Validate with HSTS Preload

Once all mixed content is resolved, add HSTS:

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

## Verifying the Fix

```bash
# Check for remaining HTTP references
curl -sk https://example.com | grep -c 'http://'

# Use online tools:
# https://www.whynopadlock.com/
# https://www.ssllabs.com/ssltest/
```

## Key Takeaways

- Use `upgrade-insecure-requests` CSP header as a quick mitigation while fixing underlying URLs.
- Replace all hardcoded `http://` references in HTML, CSS, JavaScript, and database content.
- For CMS platforms like WordPress, use database search-and-replace to update stored URLs.
- Add HSTS after confirming zero mixed content to enforce HTTPS for all future requests.

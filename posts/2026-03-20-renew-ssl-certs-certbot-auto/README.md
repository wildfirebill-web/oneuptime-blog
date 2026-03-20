# How to Renew SSL/TLS Certificates Automatically with Certbot

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Let's Encrypt, Certbot, SSL, Auto-Renewal, HTTPS, Certificates

Description: Learn how to configure automatic SSL certificate renewal with Certbot, including renewal hooks to reload web servers and monitoring for renewal failures.

## Why Automatic Renewal Is Critical

Let's Encrypt certificates expire after 90 days. Manual renewal is error-prone—it's easy to forget, and an expired certificate causes HTTPS errors for all visitors. Certbot provides automatic renewal that runs twice daily and renews certificates 30 days before expiry.

## Step 1: Verify Renewal Is Already Set Up

When you install Certbot via snap, it automatically creates a renewal timer:

```bash
# Check the renewal systemd timer
sudo systemctl status snap.certbot.renew.timer

# Or if installed via apt
sudo systemctl status certbot.timer

# Verify the cron job (older installations)
sudo crontab -l | grep certbot
# or
cat /etc/cron.d/certbot
```

## Step 2: Test Renewal Without Actually Renewing

Always dry-run first to catch configuration issues:

```bash
# Dry run - simulates renewal without making changes
sudo certbot renew --dry-run

# Expected output:
# Congratulations, all simulated renewals succeeded:
#   /etc/letsencrypt/live/example.com/fullchain.pem (success)
```

If the dry run fails, fix the issue before the actual expiry deadline.

## Step 3: Manual Renewal

Force renewal regardless of expiry date:

```bash
# Renew all certificates due for renewal
sudo certbot renew

# Force renewal even if not due (e.g., to switch from RSA to ECDSA)
sudo certbot renew --force-renewal

# Renew a specific domain only
sudo certbot renew --cert-name example.com
```

## Step 4: Configure Renewal Hooks

Renewal hooks run before or after renewal to reload services. Certbot automatically reloads Apache/Nginx if configured with `--apache` or `--nginx`, but for custom setups use hooks:

```bash
# Create a deploy hook to reload Nginx after renewal
sudo mkdir -p /etc/letsencrypt/renewal-hooks/deploy

cat > /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh << 'EOF'
#!/bin/bash
# Reload Nginx after certificate renewal
systemctl reload nginx
echo "Nginx reloaded after certificate renewal at $(date)" >> /var/log/certbot-deploy.log
EOF

sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

Hook directory types:
- `pre/` - run before renewal attempt
- `deploy/` - run if renewal succeeded
- `post/` - run after renewal attempt (success or failure)

## Step 5: Send Alerts on Renewal Failure

Create a post-hook that sends an alert if renewal fails:

```bash
cat > /etc/letsencrypt/renewal-hooks/post/alert-on-failure.sh << 'EOF'
#!/bin/bash
# Send alert if Certbot renewal failed
EXIT_CODE=$?

if [ $EXIT_CODE -ne 0 ]; then
    echo "CRITICAL: Certbot renewal failed at $(date)" | \
        mail -s "SSL Renewal Failure: $(hostname)" admin@example.com

    # Or send to Slack webhook
    curl -s -X POST "https://hooks.slack.com/services/xxx/yyy/zzz" \
         -H "Content-Type: application/json" \
         -d "{\"text\": \"SSL renewal FAILED on $(hostname)! Check /var/log/letsencrypt/letsencrypt.log\"}"
fi
EOF

chmod +x /etc/letsencrypt/renewal-hooks/post/alert-on-failure.sh
```

## Step 6: Configure Renewal for Multiple Domains

Each certificate has its own renewal configuration in `/etc/letsencrypt/renewal/`:

```bash
# List all certificates and their renewal configurations
sudo certbot certificates

# Edit a specific renewal config
sudo cat /etc/letsencrypt/renewal/example.com.conf

# The [renewalparams] section shows how the cert was originally obtained
# Modify if you need to change renewal method
```

## Step 7: Monitor Certificate Expiry

Set up monitoring to alert before certificates expire:

```bash
#!/bin/bash
# /usr/local/bin/check-cert-expiry.sh
# Run daily with cron

DOMAINS=("example.com" "api.example.com" "shop.example.com")
WARN_DAYS=30
CRITICAL_DAYS=7

for DOMAIN in "${DOMAINS[@]}"; do
    EXPIRY=$(openssl s_client -connect "${DOMAIN}:443" -servername "${DOMAIN}" \
              2>/dev/null | openssl x509 -noout -enddate 2>/dev/null | \
              sed 's/notAfter=//')
    EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s)
    NOW_EPOCH=$(date +%s)
    DAYS_LEFT=$(( (EXPIRY_EPOCH - NOW_EPOCH) / 86400 ))

    if [ "$DAYS_LEFT" -lt "$CRITICAL_DAYS" ]; then
        echo "CRITICAL: ${DOMAIN} expires in ${DAYS_LEFT} days!"
    elif [ "$DAYS_LEFT" -lt "$WARN_DAYS" ]; then
        echo "WARNING: ${DOMAIN} expires in ${DAYS_LEFT} days"
    fi
done
```

Add to crontab:
```bash
0 8 * * * /usr/local/bin/check-cert-expiry.sh | mail -s "SSL Expiry Check" admin@example.com
```

## Step 8: View Renewal Logs

```bash
# View Certbot renewal logs
sudo tail -100 /var/log/letsencrypt/letsencrypt.log

# Check the last renewal attempt
sudo grep -A5 "Renewal" /var/log/letsencrypt/letsencrypt.log | tail -20
```

## Conclusion

Certbot's automatic renewal via systemd timer or cron handles most renewal scenarios without manual intervention. Verify the renewal mechanism is active with `systemctl status certbot.timer`, test with `certbot renew --dry-run`, and add deploy hooks to reload your web server after renewal. Implement expiry monitoring as a safety net to catch any renewal failures before they impact production.

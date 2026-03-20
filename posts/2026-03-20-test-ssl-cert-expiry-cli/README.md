# How to Test SSL/TLS Certificate Expiry with Command-Line Tools

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SSL, TLS, Certificate Expiry, OpenSSL, Monitoring, Command Line

Description: Learn how to check SSL certificate expiry dates using openssl, curl, and custom scripts for monitoring certificates across multiple domains.

## Quick Expiry Check with openssl s_client

```bash
# Check certificate expiry for a live server

openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | \
  openssl x509 -noout -dates

# Output:
# notBefore=Jan  1 00:00:00 2026 GMT
# notAfter=Apr  1 00:00:00 2026 GMT
                # ^^^^^^^^^^^^^^^^^ This is the expiry date
```

## Step 1: Check Days Until Expiry

```bash
# Get the expiry date as a Unix timestamp
EXPIRY=$(openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | \
  openssl x509 -noout -enddate | sed 's/notAfter=//')

# Calculate days remaining
EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s)
NOW=$(date +%s)
DAYS=$(( (EXPIRY_EPOCH - NOW) / 86400 ))
echo "Certificate expires in ${DAYS} days (${EXPIRY})"

# Use checkend to directly test if cert expires within N seconds
# 0 = check if already expired
# 2592000 = check if expires within 30 days
openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | \
  openssl x509 -noout -checkend 2592000 && echo "OK (>30 days)" || echo "WARNING: expires within 30 days"
```

## Step 2: Check a Local Certificate File

```bash
# Check expiry of a certificate file
openssl x509 -enddate -noout -in /etc/ssl/certs/example.com.crt

# Days remaining for a local file
CERT="/etc/letsencrypt/live/example.com/cert.pem"
openssl x509 -enddate -noout -in "$CERT"

# Check if local cert expires within 30 days
openssl x509 -noout -checkend 2592000 -in "$CERT" && \
  echo "OK" || echo "Renew soon!"
```

## Step 3: Batch Check Multiple Domains

```bash
#!/bin/bash
# check_certs.sh - Check SSL expiry for multiple domains

DOMAINS=(
    "example.com:443"
    "api.example.com:443"
    "mail.example.com:993"
    "smtp.example.com:587:smtp"
)

WARNING_DAYS=30
CRITICAL_DAYS=7

echo "====================================="
echo "SSL Certificate Expiry Report"
echo "Generated: $(date)"
echo "====================================="

for ENTRY in "${DOMAINS[@]}"; do
    HOST="${ENTRY%%:*}"
    REST="${ENTRY#*:}"
    PORT="${REST%%:*}"
    PROTO="${REST##*:}"

    # Build openssl command (with STARTTLS if needed)
    if [ "$PROTO" = "smtp" ]; then
        STARTTLS="-starttls smtp"
    elif [ "$PROTO" = "imap" ]; then
        STARTTLS="-starttls imap"
    else
        STARTTLS=""
    fi

    # Get certificate expiry
    RESULT=$(openssl s_client $STARTTLS \
              -connect "${HOST}:${PORT}" \
              -servername "${HOST}" \
              2>/dev/null | openssl x509 -noout -enddate 2>/dev/null | \
              sed 's/notAfter=//')

    if [ -z "$RESULT" ]; then
        echo "ERROR: ${HOST}:${PORT} - Could not retrieve certificate"
        continue
    fi

    # Calculate days remaining
    EXPIRY_EPOCH=$(date -d "$RESULT" +%s 2>/dev/null)
    if [ -z "$EXPIRY_EPOCH" ]; then
        EXPIRY_EPOCH=$(date -jf "%b %d %H:%M:%S %Y %Z" "$RESULT" +%s 2>/dev/null)
    fi
    NOW=$(date +%s)
    DAYS=$(( (EXPIRY_EPOCH - NOW) / 86400 ))

    # Determine status
    if [ "$DAYS" -lt 0 ]; then
        STATUS="EXPIRED"
    elif [ "$DAYS" -lt "$CRITICAL_DAYS" ]; then
        STATUS="CRITICAL"
    elif [ "$DAYS" -lt "$WARNING_DAYS" ]; then
        STATUS="WARNING"
    else
        STATUS="OK"
    fi

    printf "%-30s %8s  %3d days  %s\n" "${HOST}:${PORT}" "$STATUS" "$DAYS" "$RESULT"
done
```

## Step 4: Use curl to Check Certificate

```bash
# curl shows certificate info in verbose mode
curl -vI https://example.com/ 2>&1 | grep -E "expire|issuer|subject"

# Check with curl's --head option
curl --head --silent https://example.com/ \
  --write-out "%{ssl_verify_result} expires: %{ssl_certificate_expiry}\n" \
  --output /dev/null

# Note: ssl_certificate_expiry requires curl 7.52.0+
```

## Step 5: Nagios/Check_MK Plugin for Expiry Monitoring

```bash
#!/bin/bash
# check_ssl_expiry.sh - Nagios-compatible check

HOST="${1:-example.com}"
PORT="${2:-443}"
WARNING="${3:-30}"
CRITICAL="${4:-7}"

EXPIRY=$(openssl s_client -connect "${HOST}:${PORT}" -servername "${HOST}" \
          2>/dev/null | openssl x509 -noout -enddate | sed 's/notAfter=//')
DAYS=$(( ($(date -d "$EXPIRY" +%s) - $(date +%s)) / 86400 ))

if [ "$DAYS" -lt 0 ]; then
    echo "CRITICAL: ${HOST} certificate EXPIRED ${DAYS#-} days ago"
    exit 2
elif [ "$DAYS" -lt "$CRITICAL" ]; then
    echo "CRITICAL: ${HOST} certificate expires in ${DAYS} days"
    exit 2
elif [ "$DAYS" -lt "$WARNING" ]; then
    echo "WARNING: ${HOST} certificate expires in ${DAYS} days"
    exit 1
else
    echo "OK: ${HOST} certificate valid for ${DAYS} days (expires ${EXPIRY})"
    exit 0
fi
```

## Step 6: Check OCSP Revocation Status

```bash
# Get the OCSP responder URL from the certificate
OCSP_URL=$(openssl s_client -connect example.com:443 2>/dev/null | \
  openssl x509 -noout -text | grep "OCSP.*http" | awk '{print $NF}')

# Check revocation status via OCSP
openssl ocsp -issuer /etc/ssl/certs/intermediate.pem \
  -cert /etc/ssl/certs/example.com.crt \
  -url "$OCSP_URL" \
  -text 2>&1 | grep -E "good|revoked|error"
```

## Conclusion

Command-line SSL certificate expiry checking with `openssl` is fast and scriptable. Use `openssl x509 -checkend` for quick pass/fail checks, write batch scripts to check multiple domains, and integrate with Nagios/Zabbix using the Nagios-compatible exit codes. Run expiry checks daily from cron and alert when certificates fall below 30 days remaining.

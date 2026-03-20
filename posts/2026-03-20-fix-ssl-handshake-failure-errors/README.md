# How to Fix SSL Handshake Failure Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SSL, TLS, Handshake Failure, Troubleshooting, OpenSSL, HTTPS

Description: Learn how to diagnose and fix SSL/TLS handshake failure errors caused by protocol version mismatches, cipher suite incompatibilities, and certificate issues.

## What Causes SSL Handshake Failures?

The SSL/TLS handshake is the negotiation phase where client and server agree on a protocol version, cipher suite, and exchange certificates. Failures occur when:

1. No common protocol version (e.g., server only accepts TLS 1.3, client only supports TLS 1.2)
2. No common cipher suite
3. Server certificate is invalid, expired, or has chain errors
4. Client certificate required but not provided (mTLS)
5. SNI mismatch
6. Certificate/key mismatch on the server

## Step 1: Capture the Exact Error

```bash
# Get verbose handshake output
openssl s_client -connect example.com:443 -debug 2>&1 | head -50

# Test with specific protocol version
openssl s_client -connect example.com:443 -tls1_3 2>&1 | grep -E "handshake|error|alert"
openssl s_client -connect example.com:443 -tls1_2 2>&1 | grep -E "handshake|error|alert"
openssl s_client -connect example.com:443 -tls1_1 2>&1 | grep -E "handshake|error|alert"

# Check what the server supports
nmap --script ssl-enum-ciphers -p 443 example.com
```

## Step 2: Fix Protocol Version Mismatch

If the client only supports old TLS versions and server only accepts TLS 1.3:

```bash
# For Nginx - accept both TLS 1.2 and 1.3 for broader compatibility
ssl_protocols TLSv1.2 TLSv1.3;

# For Apache
SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1

# For curl (as a client) - force a specific version
curl --tlsv1.2 --tls-max 1.3 https://example.com
```

## Step 3: Fix Cipher Suite Mismatch

If no common cipher suite exists:

```bash
# See which cipher suites client and server have in common
openssl s_client -connect example.com:443 -cipher ALL 2>&1 | grep Cipher

# Add more cipher suites to Nginx (for legacy client compatibility)
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-SHA256;
```

## Step 4: Fix Certificate/Key Mismatch

A very common cause of handshake failure is a mismatch between the certificate and its private key:

```bash
# Compare the modulus of the cert and key - they must match
openssl x509 -noout -modulus -in example.com.crt | openssl md5
openssl rsa -noout -modulus -in example.com.key | openssl md5

# If MD5 hashes differ, the key doesn't match the certificate
# You need to either get the correct key or regenerate the certificate

# Check the certificate and key are paired
openssl x509 -noout -text -in example.com.crt | grep "Public Key"
openssl rsa -text -noout -in example.com.key | grep "Private-Key"
```

## Step 5: Fix SNI Mismatch

If the client connects to an IP directly but the certificate is for a hostname:

```bash
# Test with explicit SNI
openssl s_client -connect 203.0.113.10:443 -servername example.com

# Without -servername, the default certificate (may be wrong) is served
# Make sure your reverse proxy/load balancer is passing the correct SNI
```

In Nginx, ensure `server_name` matches the certificate:

```nginx
server {
    listen 443 ssl;
    server_name example.com www.example.com;  # Must match certificate CN/SAN
    ssl_certificate /etc/ssl/certs/example.com.crt;
}
```

## Step 6: Handle Java/Old Client Handshake Failures

Java clients older than Java 11 may fail with TLS 1.3:

```bash
# Force Java to use TLS 1.2
java -Djdk.tls.client.protocols="TLSv1.2" -jar your-application.jar

# Or add TLS 1.2 back to your Nginx config alongside TLS 1.3
ssl_protocols TLSv1.2 TLSv1.3;
```

## Step 7: Enable TLS Debugging in Application Logs

For Node.js:

```bash
NODE_DEBUG=tls node app.js 2>&1 | grep -E "TLS|SSL|error"
```

For Java:

```bash
java -Djavax.net.debug=ssl:handshake your-app
```

For Python:

```python
import ssl
import logging
logging.basicConfig(level=logging.DEBUG)
ssl.SSLContext.set_alpn_protocols  # Enable SSL debug logging
```

## Common Error Codes Reference

| Error | Common Cause | Fix |
|---|---|---|
| `ssl_error_rx_record_too_long` | Connecting to HTTP port with HTTPS | Use port 443 |
| `certificate_unknown` | Certificate not trusted | Install correct chain |
| `handshake_failure` | No common protocol/cipher | Expand TLS versions/ciphers |
| `unknown_ca` | Self-signed or unknown CA | Add CA to trust store |
| `bad_certificate` | Certificate parsing error | Verify PEM format |

## Conclusion

SSL handshake failures require systematic diagnosis: start with `openssl s_client` to see the exact error, verify certificate/key matching with modulus comparison, check for protocol and cipher suite compatibility, and ensure SNI is configured correctly. Most handshake failures fall into one of these categories and can be fixed within minutes once the root cause is identified.

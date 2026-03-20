# How to Configure DMARC for IPv6 Mail Servers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DMARC, IPv6, Email, SPF, DKIM, Mail Security, Deliverability

Description: Configure DMARC policy records for domains sending email from IPv6 servers, including monitoring, quarantine, and reject policies with reporting setup.

## Introduction

DMARC (Domain-based Message Authentication, Reporting, and Conformance) builds on SPF and DKIM to give domain owners control over how their email is treated when authentication fails. For IPv6 mail servers, DMARC is crucial because IPv6 reputation systems are newer and more likely to require strong authentication signals.

## How DMARC Works with IPv6

DMARC validation is independent of IP version. The DMARC policy is evaluated based on:
1. **SPF alignment**: The SPF-authenticated domain aligns with the From header domain
2. **DKIM alignment**: The DKIM-signed domain aligns with the From header domain

For IPv6 senders, SPF must include `ip6:` mechanisms, and DKIM must be configured as covered in prerequisite guides.

## DMARC Record Structure

A DMARC record is a DNS TXT record at `_dmarc.<your-domain>`:

```text
v=DMARC1; p=<policy>; rua=<aggregate-report-address>; ruf=<forensic-report-address>; pct=<percentage>
```

## Publishing a DMARC Record

Start with a monitoring-only policy and progressively tighten it:

### Phase 1: Monitor (p=none)

```dns
; No action taken on failures, but reports are sent
_dmarc.example.com.  300  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com; ruf=mailto:dmarc-forensic@example.com; fo=1"
```

### Phase 2: Quarantine (p=quarantine)

```dns
; Failed messages go to spam folder
_dmarc.example.com.  300  IN  TXT  "v=DMARC1; p=quarantine; pct=25; rua=mailto:dmarc@example.com"
```

Increase `pct` from 25 to 100 over several weeks as you verify reports.

### Phase 3: Reject (p=reject)

```dns
; Failed messages are rejected outright
_dmarc.example.com.  300  IN  TXT  "v=DMARC1; p=reject; pct=100; rua=mailto:dmarc@example.com"
```

## Understanding DMARC Tags

| Tag | Description | Example |
|-----|-------------|---------|
| `p` | Policy for the domain | `p=reject` |
| `sp` | Policy for subdomains | `sp=quarantine` |
| `pct` | Percentage of messages to apply policy to | `pct=100` |
| `rua` | Aggregate report destination | `rua=mailto:dmarc@example.com` |
| `ruf` | Forensic report destination | `ruf=mailto:dmarc-forensic@example.com` |
| `adkim` | DKIM alignment mode (r=relaxed, s=strict) | `adkim=r` |
| `aspf` | SPF alignment mode | `aspf=r` |
| `fo` | Failure options for forensic reports | `fo=1` |

## Verifying the DMARC Record

```bash
# Check DMARC record is published correctly

dig TXT _dmarc.example.com +short

# Expected output:
# "v=DMARC1; p=reject; pct=100; rua=mailto:dmarc@example.com"

# Test from a specific tool
curl -s "https://api.mxtoolbox.com/api/v1/lookup/dmarc/example.com" | python3 -m json.tool
```

## Reading DMARC Aggregate Reports

Aggregate reports arrive as XML attachments from receiving servers. Use a parser:

```bash
# Install a DMARC report analyzer
pip3 install parsedmarc

# Analyze a DMARC report file
parsedmarc /path/to/report.xml.gz

# Or set up continuous monitoring with the JSON output
parsedmarc /path/to/reports/ --json-path /var/log/dmarc/
```

A typical aggregate report shows:

```xml
<record>
  <row>
    <source_ip>2001:db8::10</source_ip>
    <count>150</count>
    <policy_evaluated>
      <disposition>none</disposition>
      <dkim>pass</dkim>
      <spf>pass</spf>
    </policy_evaluated>
  </row>
</record>
```

## Testing End-to-End DMARC Validation

```bash
# Send a test email to Google's check-auth service
swaks --from sender@example.com \
      --to check-auth@verifier.port25.com \
      --server [2001:db8::10]:25 \
      --header "From: sender@example.com"

# The reply will include authentication results:
# DMARC: pass (p=reject dis=none) header.from=example.com
```

## DMARC for Subdomains

Control subdomain behavior with the `sp` tag:

```dns
; Strict for main domain, quarantine for subdomains
_dmarc.example.com.  300  IN  TXT  "v=DMARC1; p=reject; sp=quarantine; rua=mailto:dmarc@example.com"
```

## Conclusion

DMARC policy setup for IPv6 mail servers follows the same process as IPv4 - publish a `_dmarc` TXT record, start with `p=none` to collect reports, verify your IPv6 SPF `ip6:` entries and DKIM signatures pass, then progressively enforce `quarantine` and `reject` policies. Proper DMARC is essential for IPv6 mail deliverability to major providers like Google, Microsoft, and Yahoo.

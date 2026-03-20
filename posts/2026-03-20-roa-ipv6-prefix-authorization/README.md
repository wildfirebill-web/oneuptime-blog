# How to Create ROA (Route Origin Authorization) for IPv6 Prefixes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RPKI, ROA, IPv6, BGP, Routing Security

Description: Step-by-step instructions for creating Route Origin Authorizations (ROAs) for IPv6 prefixes through your Regional Internet Registry portal.

## What is a ROA?

A Route Origin Authorization (ROA) is a digitally signed object that states which Autonomous System (AS) is authorized to originate a specific IP prefix. ROAs are the core building blocks of RPKI-based BGP security.

A ROA contains:
- The prefix (e.g., `2001:db8::/32`)
- The maximum prefix length
- The authorized origin ASN

## Step 1: Log Into Your RIR Portal

Each RIR has a portal for managing ROAs:

| RIR | Portal URL |
|-----|-----------|
| RIPE NCC | https://my.ripe.net |
| ARIN | https://account.arin.net |
| APNIC | https://myapnic.net |
| LACNIC | https://milacnic.lacnic.net |
| AFRINIC | https://my.afrinic.net |

## Step 2: Navigate to RPKI/ROA Management

In RIPE NCC as an example:
1. Log in → go to **My Resources**
2. Select **IPv6** → choose your prefix
3. Click **RPKI** → **Create ROA**

## Step 3: Define ROA Parameters

When creating a ROA, specify these fields:

```text
Prefix:       2001:db8::/32
Max Length:   48           (allows announcements up to /48)
Origin ASN:   AS64496
Valid Until:  2027-12-31   (certificate expiry)
```

**Important notes on max length:**
- Setting max length equal to prefix length restricts to exact prefix only
- A larger max length allows sub-prefix announcements
- Too permissive max lengths can enable route hijacking of sub-prefixes

## Step 4: Create Multiple ROAs for Redundancy

If you announce from multiple ASNs (e.g., for multihoming), create a ROA for each:

```bash
# ROA 1: Primary upstream ASN

Prefix: 2001:db8::/32, Max-Length: 48, Origin: AS64496

# ROA 2: Secondary upstream ASN
Prefix: 2001:db8::/32, Max-Length: 48, Origin: AS64497

# ROA 3: More specific prefix for traffic engineering
Prefix: 2001:db8:1::/48, Max-Length: 48, Origin: AS64496
```

## Step 5: Verify ROA Propagation

After creation, ROAs take up to a few hours to propagate. Verify using public RPKI validators:

```bash
# Check ROA validity using Cloudflare's RPKI validator
curl "https://rpki.cloudflare.com/api/v1/validity/AS64496/2001:db8::/32"

# Check via RIPE's validator
curl "https://stat.ripe.net/data/rpki-validation/data.json?resource=AS64496&prefix=2001:db8::/32"

# Use routinator locally
routinator validate --asn AS64496 --prefix 2001:db8::/32
```

## Step 6: Check for ROA Conflicts

Before creating a ROA, check that no conflicting ROA already exists:

```bash
# List existing ROAs for your prefix
curl "https://stat.ripe.net/data/announced-prefixes/data.json?resource=AS64496"

# Use Routinator to check coverage
routinator dump | grep "2001:db8"
```

## Automation with RIPE NCC API

For large networks, automate ROA creation via the RIPE NCC API:

```bash
# Create a ROA via RIPE NCC API
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  https://rest.db.ripe.net/ripe/route6 \
  -d '{
    "objects": {
      "object": [{
        "type": "route6",
        "attributes": {
          "attribute": [
            {"name": "route6", "value": "2001:db8::/32"},
            {"name": "descr", "value": "My IPv6 prefix"},
            {"name": "origin", "value": "AS64496"},
            {"name": "mnt-by", "value": "MY-MNT"}
          ]
        }
      }]
    }
  }'
```

## Monitoring ROA Health

Monitor ROA expiry and validity using [OneUptime](https://oneuptime.com). Set up HTTP monitors against RPKI validator APIs to alert you before ROAs expire or if your prefixes become INVALID.

## Conclusion

Creating ROAs is the first step to deploying RPKI. Always create ROAs before your peers or upstreams start enforcing RPKI filtering. Use conservative max-length values and remember to renew certificates before expiry.

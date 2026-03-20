# How to Use bgpq4 to Auto-Generate BGP Prefix Filters from IRR Data

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, bgpq4, IRR, Prefix Filters, Route Security, Automation

Description: Learn how to use bgpq4 to automatically generate accurate BGP prefix filter lists from Internet Routing Registry (IRR) data, reducing manual effort and improving routing security.

## What Is bgpq4 and Why Use It?

Maintaining accurate BGP prefix lists manually is error-prone and time-consuming. When a customer adds new prefixes, you need to update filters immediately—or traffic gets dropped. bgpq4 queries Internet Routing Registry (IRR) databases and automatically generates prefix-list configurations from AS-SET and route objects.

## Step 1: Install bgpq4

```bash
# On Ubuntu/Debian
sudo apt-get install bgpq4

# Or build from source
git clone https://github.com/bgp/bgpq4.git
cd bgpq4
./configure && make && sudo make install
```

## Step 2: Query a Single AS for Its Prefixes

Generate a Cisco-format prefix list for AS 65100:

```bash
# Generate an IOS prefix list for all prefixes in AS 65100
bgpq4 -4 -F "%n/%l\n" AS65100

# Generate a full Cisco IOS prefix-list configuration
bgpq4 -4 -l CUSTOMER_65100 AS65100

# Example output:
# ip prefix-list CUSTOMER_65100 seq 5 permit 192.168.1.0/24
# ip prefix-list CUSTOMER_65100 seq 10 permit 192.168.2.0/24
```

## Step 3: Generate a Prefix List for an AS-SET

Most ISPs register their prefixes under an AS-SET object, which includes their customers' ASes. Query the AS-SET instead of individual ASes:

```bash
# Generate prefix list from AS-SET (includes all member ASes)
bgpq4 -4 -l UPSTREAM_FILTER AS-EXAMPLE

# Generate with maximum prefix length of /24 (reject more-specifics)
bgpq4 -4 -l UPSTREAM_FILTER -R 24 AS-EXAMPLE

# Generate for IPv6
bgpq4 -6 -l UPSTREAM_FILTER_V6 AS-EXAMPLE
```

## Step 4: Use bgpq4 in an Automation Script

Create a shell script that regenerates and applies prefix filters for all customers:

```bash
#!/bin/bash
# regenerate_bgp_filters.sh - Auto-generate and apply BGP prefix filters

# List of customers: AS_number:description
CUSTOMERS=(
    "AS65100:customer-a"
    "AS65200:customer-b"
    "AS-EXAMPLE:upstream-isp"
)

OUTPUT_FILE="/tmp/bgp_filters.conf"
> "$OUTPUT_FILE"

for ENTRY in "${CUSTOMERS[@]}"; do
    AS="${ENTRY%%:*}"
    NAME="${ENTRY##*:}"
    LIST_NAME="FILTER_${NAME//-/_}"

    echo "! Auto-generated filter for $AS on $(date)" >> "$OUTPUT_FILE"

    # Generate IPv4 prefix list
    bgpq4 -4 -l "$LIST_NAME" -R 24 "$AS" >> "$OUTPUT_FILE"

    echo "" >> "$OUTPUT_FILE"
done

echo "Generated filters saved to $OUTPUT_FILE"

# Apply to router via SSH (requires sshpass or SSH keys)
# ssh router.example.com < "$OUTPUT_FILE"
```

## Step 5: Generate Filters for Different Router Formats

bgpq4 supports multiple router formats:

```bash
# Cisco IOS format (default)
bgpq4 -4 -l MYFILTER AS65100

# Cisco IOS XR format
bgpq4 -4 -S MYFILTER -X AS65100

# BIRD format
bgpq4 -4 -f AS65100 -b

# OpenBGPD format
bgpq4 -4 -f AS65100 -O

# JSON output for programmatic use
bgpq4 -4 -j AS65100
```

## Step 6: Specify IRR Servers

By default, bgpq4 queries RADB. Specify alternative IRR servers for different RIRs:

```bash
# Query RIPE IRR
bgpq4 -h whois.ripe.net -4 -l RIPE_FILTER AS-RIPE

# Query ARIN
bgpq4 -h rr.arin.net -4 -l ARIN_FILTER AS-ARIN

# Query multiple servers
bgpq4 -h whois.radb.net,rr.ntt.net -4 -l FILTER AS65100
```

## Step 7: Automate with Cron

Refresh prefix filters daily to pick up customer changes automatically:

```bash
# Add to crontab (run at 2am daily)
# crontab -e
0 2 * * * /usr/local/bin/regenerate_bgp_filters.sh >> /var/log/bgp_filters.log 2>&1
```

## Conclusion

bgpq4 eliminates the manual work of maintaining BGP prefix filters by generating them directly from IRR data. Run it as part of a scheduled automation pipeline to ensure filters stay current as customers update their IRR objects. Always set a maximum prefix length (`-R 24`) to prevent overly specific prefixes from being permitted.

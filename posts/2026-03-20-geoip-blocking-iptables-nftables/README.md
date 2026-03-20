# How to Set Up GeoIP Blocking with iptables or nftables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GeoIP, iptables, nftables, Linux, Security, Firewall

Description: Block traffic from specific countries using GeoIP databases with iptables or nftables to reduce attack surface from high-risk geographic regions.

GeoIP blocking restricts traffic by country of origin, reducing the attack surface when your services have no legitimate users in certain regions. It doesn't stop sophisticated attackers who use proxies, but it eliminates a large percentage of automated attacks.

## Method 1: GeoIP with xtables-addons (iptables)

```bash
# Install xtables-addons (provides geoip module for iptables)

sudo apt install xtables-addons-common libtext-csv-xs-perl -y

# Download and install GeoIP database
sudo mkdir -p /usr/share/xt_geoip
cd /usr/share/xt_geoip

# Download MaxMind GeoLite2 database (requires free registration)
# Then convert to xtables format:
sudo /usr/lib/xtables-addons/xt_geoip_build -D /usr/share/xt_geoip \
  GeoLite2-Country-Blocks-IPv4.csv GeoLite2-Country-Locations-en.csv
```

## Block Countries with xtables geoip

```bash
# Block all traffic from country code CN (China)
sudo iptables -A INPUT -m geoip --src-cc CN -j DROP

# Block multiple countries
sudo iptables -A INPUT -m geoip --src-cc CN,RU,KP -j DROP

# Log and block
sudo iptables -A INPUT -m geoip --src-cc CN \
  -j LOG --log-prefix "GEOIP-DROP-CN: "
sudo iptables -A INPUT -m geoip --src-cc CN -j DROP
```

## Method 2: ipset + Country IP Lists

Use publicly available country CIDR lists with ipset:

```bash
#!/bin/bash
# block-country.sh - Block a country by downloading its IP ranges

COUNTRY="CN"  # ISO country code
IPSET_NAME="block-${COUNTRY}"

# Get IP ranges from ipdeny.com
URL="https://www.ipdeny.com/ipblocks/data/countries/${COUNTRY,,}.zone"

# Create ipset
sudo ipset create -exist "$IPSET_NAME" hash:net family inet

# Download and add ranges
curl -s "$URL" | while read cidr; do
    sudo ipset add -exist "$IPSET_NAME" "$cidr"
done

# Apply iptables rule
sudo iptables -A INPUT -m set --match-set "$IPSET_NAME" src -j DROP

echo "Blocked $(sudo ipset list $IPSET_NAME | grep 'Number of entries')"
```

## Method 3: nftables with GeoIP

With nftables, use a set loaded from a file:

```bash
# Create a file with blocked CIDR ranges
curl -s https://www.ipdeny.com/ipblocks/data/countries/cn.zone > /tmp/cn-ranges.txt

# nftables ruleset
sudo tee /etc/nftables-geoip.conf << 'EOF'
table inet geoip {
    set blocked_countries {
        type ipv4_addr
        flags interval
        # Ranges loaded via nft add element
    }

    chain input {
        type filter hook input priority 0; policy accept;
        ip saddr @blocked_countries drop
    }
}
EOF

sudo nft -f /etc/nftables-geoip.conf

# Add IP ranges to the set
while read cidr; do
    sudo nft add element inet geoip blocked_countries "{ $cidr }"
done < /tmp/cn-ranges.txt
```

## Allow Your Country, Block Everything Else

For services only used domestically:

```bash
#!/bin/bash
# Allowlist: only accept traffic from US
COUNTRY="US"
IPSET_NAME="allow-${COUNTRY}"

sudo ipset create -exist "$IPSET_NAME" hash:net family inet

curl -s "https://www.ipdeny.com/ipblocks/data/countries/${COUNTRY,,}.zone" \
  | while read cidr; do sudo ipset add -exist "$IPSET_NAME" "$cidr"; done

# Allow traffic from US, drop everything else
sudo iptables -A INPUT -p tcp --dport 443 \
  -m set --match-set "$IPSET_NAME" src -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j DROP
```

## Update GeoIP Databases Regularly

Country IP allocations change frequently; schedule updates:

```bash
# /etc/cron.weekly/update-geoip
#!/bin/bash
# Re-run the block-country.sh script to refresh IP ranges
bash /opt/scripts/block-country.sh
```

GeoIP blocking is most valuable for services like SSH or admin panels where you know all legitimate users are in specific countries, dramatically reducing the attack surface.

# How to Handle IPv6 in GeoIP Lookups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, GeoIP, MaxMind, Geolocation, Networking, Web Development

Description: Perform accurate GeoIP lookups for IPv6 addresses using MaxMind GeoLite2 and GeoIP2 databases across multiple programming languages.

## Introduction

GeoIP databases map IP addresses to geographic locations. IPv6 support in GeoIP has matured significantly, with MaxMind's GeoLite2 and GeoIP2 databases covering millions of IPv6 prefixes. This guide covers how to perform IPv6 GeoIP lookups correctly in applications.

## Downloading GeoIP2 Databases

MaxMind provides free GeoLite2 databases and paid GeoIP2 databases:

```bash
# Sign up for a free MaxMind account at https://dev.maxmind.com/geoip/geolite2-free-geolocation-data/
# Then download with your license key:

# GeoLite2-City (most comprehensive)
curl -fL "https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-City&license_key=YOUR_KEY&suffix=tar.gz" \
    -o GeoLite2-City.tar.gz
tar xzf GeoLite2-City.tar.gz

# Or use geoipupdate tool
sudo apt install geoipupdate
# Configure /etc/GeoIP.conf with your license key and run:
sudo geoipupdate
```

## Python: GeoIP2 IPv6 Lookup

```python
import geoip2.database
import geoip2.errors
import ipaddress

def geoip_lookup(ip_str: str, db_path: str = '/usr/share/GeoIP/GeoLite2-City.mmdb') -> dict:
    """
    Look up geolocation for an IPv4 or IPv6 address.
    Returns dict with country, city, latitude, longitude.
    """
    result = {
        'ip': ip_str,
        'version': None,
        'country_code': None,
        'country_name': None,
        'city': None,
        'latitude': None,
        'longitude': None,
        'error': None,
    }

    # Normalize and validate the IP
    try:
        # Handle IPv4-mapped IPv6 addresses
        clean = ip_str.replace('::ffff:', '').split('%')[0]
        addr = ipaddress.ip_address(clean)
        result['version'] = addr.version
        lookup_ip = str(addr)
    except ValueError:
        result['error'] = 'Invalid IP address'
        return result

    # Perform lookup
    try:
        with geoip2.database.Reader(db_path) as reader:
            response = reader.city(lookup_ip)

            result['country_code'] = response.country.iso_code
            result['country_name'] = response.country.name
            result['city'] = response.city.name
            result['latitude'] = float(response.location.latitude or 0)
            result['longitude'] = float(response.location.longitude or 0)
            result['timezone'] = response.location.time_zone

    except geoip2.errors.AddressNotFoundError:
        result['error'] = 'Address not found in database'
    except Exception as e:
        result['error'] = str(e)

    return result

# Test with IPv4 and IPv6 addresses
test_ips = [
    '8.8.8.8',                    # Google DNS IPv4
    '2001:4860:4860::8888',       # Google DNS IPv6
    '::ffff:8.8.8.8',             # IPv4-mapped IPv6
    '2606:4700:4700::1111',       # Cloudflare IPv6
]

for ip in test_ips:
    info = geoip_lookup(ip)
    print(f"IPv{info['version']} {ip}: {info['country_name']}, {info['city']}")
    if info['error']:
        print(f"  Error: {info['error']}")
```

## Node.js: GeoIP2 IPv6 Lookup

```javascript
const maxmind = require('maxmind');
const net = require('net');

let reader;

async function initGeoIP(dbPath) {
  reader = await maxmind.open(dbPath || '/usr/share/GeoIP/GeoLite2-City.mmdb');
}

function geoipLookup(ipStr) {
  // Clean the IP address
  let cleanIP = ipStr.replace(/^::ffff:/, '').split('%')[0].replace(/[\[\]]/g, '');

  const isV6 = net.isIPv6(cleanIP);

  try {
    const result = reader.get(cleanIP);
    if (!result) {
      return { ip: cleanIP, version: isV6 ? 6 : 4, error: 'Not found' };
    }

    return {
      ip: cleanIP,
      version: isV6 ? 6 : 4,
      countryCode: result.country?.iso_code,
      countryName: result.country?.names?.en,
      city: result.city?.names?.en,
      latitude: result.location?.latitude,
      longitude: result.location?.longitude,
      timezone: result.location?.time_zone,
    };
  } catch (e) {
    return { ip: cleanIP, error: e.message };
  }
}

// Usage
async function main() {
  await initGeoIP();

  const ips = ['2001:4860:4860::8888', '8.8.8.8', '2606:4700:4700::1111'];
  for (const ip of ips) {
    const info = geoipLookup(ip);
    console.log(`${ip}: ${info.countryName}, ${info.city}`);
  }
}

main();
```

## Nginx GeoIP2 Module for IPv6

Configure Nginx to use GeoIP2 for IPv6 traffic:

```nginx
# Install ngx_http_geoip2_module first
# apt install libnginx-mod-http-geoip2

load_module modules/ngx_http_geoip2_module.so;

geoip2 /usr/share/GeoIP/GeoLite2-Country.mmdb {
    auto_reload 5m;
    $geoip2_metadata_country_build metadata build_epoch;
    $geoip2_data_country_code default=XX source=$remote_addr country iso_code;
    $geoip2_data_country_name country names en;
}

# Use in access control or logging
server {
    # Log country for both IPv4 and IPv6 clients
    log_format geoip '$remote_addr - [$geoip2_data_country_code] '
                     '"$request" $status';

    access_log /var/log/nginx/geoip.log geoip;

    # Block requests from a specific country
    if ($geoip2_data_country_code = "XX") {
        return 403;
    }
}
```

## Handling Private and Reserved IPv6 Addresses

```python
import ipaddress

def should_geoip_lookup(ip_str: str) -> bool:
    """Return False for IPs that don't need GeoIP lookup."""
    try:
        addr = ipaddress.ip_address(ip_str.split('%')[0])
    except ValueError:
        return False

    # Skip private, loopback, link-local addresses
    if addr.is_loopback or addr.is_private or addr.is_link_local:
        return False

    # Skip documentation ranges
    if isinstance(addr, ipaddress.IPv6Address):
        if ipaddress.IPv6Network('2001:db8::/32').supernet_of(
                ipaddress.IPv6Network(f'{addr}/128')):
            return False

    return True
```

## Conclusion

GeoIP lookups for IPv6 use the same MaxMind GeoIP2 API as IPv4. The key considerations are stripping IPv4-mapped prefixes before lookup, skipping GeoIP for private/link-local addresses, and ensuring your database is kept up to date with `geoipupdate`. Both GeoLite2-City and GeoLite2-Country databases include substantial IPv6 coverage.

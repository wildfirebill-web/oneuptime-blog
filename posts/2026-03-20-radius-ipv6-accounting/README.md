# How to Configure RADIUS IPv6 Accounting

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RADIUS, IPv6, Accounting, FreeRADIUS, AAA, Session Management, SQL

Description: Configure RADIUS accounting for IPv6 subscribers including session tracking, IPv6 attribute logging, interim updates, and usage reporting with FreeRADIUS and SQL.

## RADIUS Accounting Overview

RADIUS accounting tracks subscriber sessions with three packet types:
- **Acct-Status-Type = Start**: session begins, IPv6 prefix assigned
- **Acct-Status-Type = Interim-Update**: periodic usage update
- **Acct-Status-Type = Stop**: session ends, final byte counts

IPv6 accounting extends this with additional attributes to log the assigned IPv6 prefix.

## Accounting Attributes for IPv6 Sessions

```text
Typical Accounting-Request (Start) with IPv6:

  Acct-Status-Type:       Start
  User-Name:              alice
  NAS-IPv6-Address:       2001:db8:nas::1
  NAS-Port:               100
  Framed-IPv6-Prefix:     2001:db8:1::10/128
  Delegated-IPv6-Prefix:  2001:db8:home:a::/56
  Acct-Session-Id:        5e8a1f00-00000001
  Acct-Unique-Session-Id: a1b2c3d4e5f6a7b8
  Event-Timestamp:        1709000000
```

## FreeRADIUS: SQL Accounting with IPv6

```sql
-- Enhanced radacct schema with IPv6 columns
ALTER TABLE radacct
    ADD COLUMN nasipv6address      VARCHAR(50)   DEFAULT NULL,
    ADD COLUMN framedipv6prefix    VARCHAR(50)   DEFAULT NULL,
    ADD COLUMN delegatedipv6prefix VARCHAR(50)   DEFAULT NULL;

-- Index for IPv6 prefix lookups
CREATE INDEX idx_framedipv6 ON radacct(framedipv6prefix);
CREATE INDEX idx_delegatedipv6 ON radacct(delegatedipv6prefix);
```

```sql
# /etc/freeradius/3.0/mods-config/sql/main/mysql/queries.conf

# Update accounting queries to include IPv6 attributes

accounting_start_query = "\
    INSERT INTO ${....acct_table1} \
        (acctsessionid, acctuniqid, username, realm, nasipaddress, \
         nasipv6address, nasportid, nasporttype, acctstarttime, \
         framedipaddress, framedipv6prefix, delegatedipv6prefix, \
         acctauthentic, connectinfo_start, calledstationid, \
         callingstationid, servicetype, framedprotocol, acctstartdelay) \
    VALUES \
        ('%{Acct-Session-Id}', '%{Acct-Unique-Session-Id}', \
         '%{SQL-User-Name}', '%{Realm}', '%{NAS-IP-Address}', \
         '%{NAS-IPv6-Address}', '%{NAS-Port-Id}', '%{NAS-Port-Type}', \
         '%S', '%{Framed-IP-Address}', '%{Framed-IPv6-Prefix}', \
         '%{Delegated-IPv6-Prefix}', '%{Acct-Authentic}', \
         '%{Connect-Info}', '%{Called-Station-Id}', \
         '%{Calling-Station-Id}', '%{Service-Type}', \
         '%{Framed-Protocol}', '%{Acct-Delay-Time}')"
```

## FreeRADIUS: Detail File Accounting

```text
# /etc/freeradius/3.0/mods-enabled/detail
# Log all accounting to detail files (including IPv6 attrs)

detail {
    # Include NAS IPv6 address in filename for log organization
    filename = ${radacctdir}/%{%{NAS-IPv6-Address}:-%{NAS-IP-Address}}/detail-%Y%m%d

    permissions = 0600
    locking = yes

    # Log all attributes including IPv6
    log_packet_header = yes
}
```

## Accounting Start/Interim/Stop Testing

```bash
# Test accounting Start
radclient -x [2001:db8::radius]:1813 acct testing123 << 'EOF'
Acct-Status-Type = Start
User-Name = "alice"
NAS-IPv6-Address = "2001:db8:nas::1"
NAS-Port = 100
Acct-Session-Id = "test-session-001"
Framed-IPv6-Prefix = "2001:db8:1::10/128"
Delegated-IPv6-Prefix = "2001:db8:home:a::/56"
Acct-Authentic = RADIUS
Event-Timestamp = 1709000000
EOF

# Test Interim-Update (with byte counts)
radclient -x [2001:db8::radius]:1813 acct testing123 << 'EOF'
Acct-Status-Type = Interim-Update
User-Name = "alice"
NAS-IPv6-Address = "2001:db8:nas::1"
Acct-Session-Id = "test-session-001"
Framed-IPv6-Prefix = "2001:db8:1::10/128"
Acct-Input-Octets = 1073741824
Acct-Output-Octets = 5368709120
Acct-Input-Gigawords = 0
Acct-Output-Gigawords = 1
Acct-Session-Time = 3600
EOF

# Test Accounting Stop
radclient -x [2001:db8::radius]:1813 acct testing123 << 'EOF'
Acct-Status-Type = Stop
User-Name = "alice"
NAS-IPv6-Address = "2001:db8:nas::1"
Acct-Session-Id = "test-session-001"
Acct-Session-Time = 7200
Acct-Input-Octets = 2147483648
Acct-Output-Octets = 10737418240
Acct-Terminate-Cause = User-Request
EOF
```

## Usage Reporting Queries

```sql
-- Daily usage per IPv6 subscriber
SELECT
    username,
    framedipv6prefix,
    delegatedipv6prefix,
    DATE(acctstarttime) AS date,
    SUM(acctinputoctets + acctoutputoctets +
        (acctinputgigawords + acctoutputgigawords) * 4294967296) AS total_bytes,
    SUM(acctsessiontime) AS total_seconds,
    COUNT(*) AS sessions
FROM radacct
WHERE framedipv6prefix IS NOT NULL
  AND acctstarttime >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY username, framedipv6prefix, DATE(acctstarttime)
ORDER BY username, date;

-- Top IPv6 bandwidth users (last 24h)
SELECT
    username,
    framedipv6prefix,
    SUM(acctinputoctets + acctoutputoctets) / (1024*1024*1024) AS total_gb
FROM radacct
WHERE acctstarttime > DATE_SUB(NOW(), INTERVAL 24 HOUR)
  AND framedipv6prefix IS NOT NULL
GROUP BY username, framedipv6prefix
ORDER BY total_gb DESC
LIMIT 10;
```

## Detecting Accounting Gaps

```bash
#!/bin/bash
# Find sessions missing Stop records (potential accounting gaps)

mysql -u radius -p${MYSQL_PASS} -s radius << 'EOF'
-- Sessions older than 2h with no Stop and no recent interim update
SELECT
    username,
    framedipv6prefix,
    nasipv6address,
    acctstarttime,
    TIMESTAMPDIFF(HOUR, acctstarttime, NOW()) AS hours_open,
    acctinputoctets + acctoutputoctets AS bytes
FROM radacct
WHERE acctstoptime IS NULL
  AND acctstarttime < DATE_SUB(NOW(), INTERVAL 2 HOUR)
  AND (acctinteriminterval IS NULL
       OR acclastupdatetime < DATE_SUB(NOW(), INTERVAL 30 MINUTE))
ORDER BY acctstarttime;
EOF
```

## Cisco NAS: Accounting Configuration

```text
! Cisco BNG - send accounting with IPv6 attributes

aaa accounting network default start-stop group RADIUS_GRP

! Ensure IPv6 attributes are included in accounting
radius-server attribute 97 include-in-access-req  ! Framed-IPv6-Prefix
radius-server attribute 123 include-in-acct-req   ! Delegated-IPv6-Prefix

! Interim updates every 10 minutes
aaa accounting update periodic 10

! Verify accounting is being sent
debug radius accounting
show aaa accounting
```

## Conclusion

RADIUS IPv6 accounting captures session details including `Framed-IPv6-Prefix` and `Delegated-IPv6-Prefix` in Start, Interim-Update, and Stop packets. Extend the FreeRADIUS SQL schema with `nasipv6address`, `framedipv6prefix`, and `delegatedipv6prefix` columns, and update accounting queries to populate them. Test with `radclient` sending accounting packets to UDP port 1813. Use SQL queries on `radacct` for usage reports - remember to account for Gigawords overflow (`acctinputgigawords * 4294967296`) for high-bandwidth sessions. Monitor for accounting gaps (sessions without Stop records and no recent interim update) which indicate NAS reachability issues.

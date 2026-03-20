# How to Create an IPv6 Migration Checklist

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Migration, Checklist, Project Management, DevOps

Description: A comprehensive IPv6 migration checklist covering pre-migration, per-service, and post-migration tasks with pass/fail criteria for each item.

## Introduction

A migration checklist provides a shared source of truth for all teams involved in IPv6 enablement. It transforms the abstract roadmap into actionable tasks with clear completion criteria. This guide presents a reusable checklist organized by phase and team.

## Pre-Migration Checklist

### Address Planning
- [ ] IPv6 address block allocated from RIR or ISP (/32 for organizations, /48–/56 for sites)
- [ ] Hierarchical addressing scheme designed (region → site → VLAN → host)
- [ ] Subnets assigned: all VLANs have /64 assignments
- [ ] Loopback addresses assigned to all routers and key servers
- [ ] Management network assigned a dedicated IPv6 /64
- [ ] IPAM system (NetBox or equivalent) populated with the address plan

### Network Equipment
- [ ] All core routers support IPv6 hardware forwarding
- [ ] All switches support IPv6 RA Guard and DHCPv6 snooping
- [ ] Firewalls support IPv6 stateful inspection
- [ ] Load balancers support IPv6 virtual IPs
- [ ] DNS servers support AAAA record creation and DNSSEC signing

### Security
- [ ] IPv6 firewall default policies reviewed (deny all, allow established)
- [ ] RA Guard configured on all access switch ports
- [ ] DHCPv6 snooping enabled on access switches
- [ ] IPv6 ACLs mirror equivalent IPv4 ACLs
- [ ] Fail2Ban or equivalent configured for IPv6 ban actions

## Per-Service Checklist

Use this for each service being migrated:

### Service: ____________________

**Pre-Migration**
- [ ] Service runs in staging with IPv6 connectivity
- [ ] AAAA DNS record prepared (not yet published)
- [ ] Load balancer IPv6 VIP configured
- [ ] Firewall rules permit IPv6 to this service
- [ ] SSL certificate covers same hostnames (certificates are not IP-version specific)
- [ ] Application binds to `::` instead of `0.0.0.0`
- [ ] X-Forwarded-For / X-Real-IP handling updated for IPv6
- [ ] Health checks updated to use IPv6 endpoints

**Go-Live**
- [ ] Deploy updated application configuration
- [ ] Publish AAAA DNS record (low TTL: 60 seconds)
- [ ] Verify service responds: `curl -6 https://service.example.com`
- [ ] Verify IPv6 address appears in access logs
- [ ] Monitoring shows IPv6 traffic

**Post Go-Live**
- [ ] Increase DNS TTL to normal value (300–3600 seconds)
- [ ] IPAM updated with service IPv6 addresses
- [ ] Runbook updated with IPv6 troubleshooting steps
- [ ] Alert thresholds reviewed for IPv6 traffic patterns

## Network Layer Checklist

```bash
#!/bin/bash
# run_migration_checks.sh — automated checklist verification

FAIL=0
PASS=0

check() {
    local desc="$1"
    shift
    if "$@" &>/dev/null; then
        echo "  [PASS] $desc"
        PASS=$((PASS+1))
    else
        echo "  [FAIL] $desc"
        FAIL=$((FAIL+1))
    fi
}

echo "=== Network Layer IPv6 Checks ==="
check "IPv6 enabled on all interfaces" sysctl net.ipv6.conf.all.disable_ipv6
check "IPv6 default route present" ip -6 route show default
check "IPv6 DNS works" dig AAAA google.com +short
check "IPv6 external reach" ping6 -c 2 2001:4860:4860::8888
check "IPv6 NDP cache populated" ip -6 neigh show
check "IPv6 forwarding enabled (router)" sysctl -n net.ipv6.conf.all.forwarding

echo ""
echo "=== Service Layer IPv6 Checks ==="
# Check each service listens on IPv6
for port in 80 443 22 25 53; do
    check "Port $port listening on IPv6" bash -c "ss -tlnp | grep -q '\\[::'$port"
done

echo ""
echo "Result: PASS=$PASS FAIL=$FAIL"
[ $FAIL -eq 0 ] && exit 0 || exit 1
```

## Post-Migration Checklist

- [ ] All services have AAAA DNS records published
- [ ] IPv6 traffic visible in dashboards (not zero)
- [ ] IPv6 error rate comparable to IPv4 error rate
- [ ] Monitoring alerts tested with IPv6 addresses
- [ ] Incident runbooks updated for IPv6 troubleshooting
- [ ] All firewall rules reviewed: no IPv4-only rules that should apply to IPv6
- [ ] IPAM reflects current deployed state
- [ ] Stakeholder sign-off on IPv6 acceptance criteria

## Conclusion

An IPv6 migration checklist prevents the most common failure modes: services that were not enabled, DNS records not published, firewall rules that block IPv6, and monitoring that does not see IPv6 traffic. Organize the checklist by phase (pre-migration, per-service, post-migration) and assign items to specific teams. Automate what you can — shell scripts that verify service listening, connectivity, and DNS records catch issues before they cause incidents.

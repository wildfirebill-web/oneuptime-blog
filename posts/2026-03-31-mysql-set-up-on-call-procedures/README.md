# How to Set Up MySQL On-Call Procedures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, On-Call, SRE, Alerting, Incident Response

Description: How to design and implement effective MySQL on-call procedures including alert routing, escalation policies, and engineer preparation.

---

## Foundations of MySQL On-Call

On-call for MySQL means being the first responder when database alerts fire outside business hours. Effective on-call procedures reduce fatigue, improve response times, and ensure incidents are handled consistently. The foundation is a combination of good monitoring, clear runbooks, and a sensible escalation policy.

## Defining Alert Severity Levels

Not all MySQL alerts require immediate action. Define clear severity levels:

```yaml
# Example alert severity definitions
severity:
  P1_critical:
    description: "Database unavailable, writes failing, or data loss risk"
    response_time: 5 minutes
    examples:
      - "MySQL process down"
      - "max_connections reached"
      - "Replication lag > 10 minutes"

  P2_high:
    description: "Degraded performance with customer impact"
    response_time: 30 minutes
    examples:
      - "Slow query rate > 50/minute"
      - "CPU > 90% for 5 minutes"
      - "Disk usage > 85%"

  P3_medium:
    description: "Warning thresholds, no immediate customer impact"
    response_time: next business day
    examples:
      - "Replication lag > 30 seconds"
      - "Connection count > 70% of max"
```

## Configuring Alerts in MySQL

Set up MySQL-level monitoring with Performance Schema:

```sql
-- Create a monitoring user with minimal privileges
CREATE USER 'oncall_monitor'@'10.0.0.%'
  IDENTIFIED BY 'monitor_password';

GRANT SELECT ON performance_schema.* TO 'oncall_monitor'@'10.0.0.%';
GRANT SELECT ON information_schema.* TO 'oncall_monitor'@'10.0.0.%';
GRANT PROCESS ON *.* TO 'oncall_monitor'@'10.0.0.%';
GRANT REPLICATION CLIENT ON *.* TO 'oncall_monitor'@'10.0.0.%';
```

## On-Call Preparation Checklist

Every engineer joining the MySQL on-call rotation should complete:

```text
Before your first shift:
[ ] Read the MySQL operational runbook end-to-end
[ ] Verify you have SSH access to all database servers
[ ] Confirm you can connect to MySQL as the on-call admin user
[ ] Test your PagerDuty mobile app is working
[ ] Shadow the previous on-call engineer for one week
[ ] Complete a dry-run of at least two incident scenarios in staging

Access requirements:
[ ] SSH key added to db-primary, db-replica1, db-replica2
[ ] MySQL on-call user credentials in the team password manager
[ ] ProxySQL admin access configured
[ ] Access to the monitoring dashboard (Grafana/OneUptime)
[ ] Permission to create PagerDuty incidents
```

## Incident Response Workflow

When an alert fires, follow this workflow:

```text
1. Acknowledge the alert in PagerDuty within the SLA
2. Open the alert in the monitoring dashboard
3. Identify which MySQL server is affected
4. Check the relevant runbook section for the alert type
5. Begin diagnosis using the runbook commands
6. Post an update in #incidents Slack channel every 15 minutes
7. Escalate to DBA lead if not resolved within 30 minutes (P1)
8. Resolve alert once incident is confirmed resolved
9. Create a post-incident report within 24 hours
```

## On-Call Handoff Template

Use a consistent handoff format:

```text
MySQL On-Call Handoff - 2026-03-31

Incidents this week:
- 2026-03-28 02:15 UTC: High connection count on db-primary
  Root cause: Connection pool misconfiguration in orders-service
  Status: Resolved - fix deployed 2026-03-28 09:00 UTC

Known issues:
- Replication lag on db-replica2 spiking to 15s during peak traffic
  Tracking issue: INFRA-4521 (next sprint)

Upcoming maintenance:
- 2026-04-02 01:00 UTC: MySQL minor version upgrade (8.0.35 -> 8.0.37)
  Runbook: /docs/mysql-upgrade-runbook.md
  Estimated downtime: 30 seconds (rolling replica update)
```

## Reducing On-Call Toil

High-quality on-call means reducing unnecessary pages:

- Tune alert thresholds to eliminate flapping alerts
- Automate remediation for known-safe actions (e.g., killing idle connections)
- Add `runbook_url` annotations to every alert so engineers can find procedures instantly
- Track alert frequency and eliminate alerts that never require action
- Conduct monthly on-call reviews to identify repeat incidents

## Summary

Effective MySQL on-call procedures require clear severity definitions, properly scoped monitoring accounts, and a structured incident response workflow. Engineers should be prepared before joining the rotation with access verification and staging dry-runs. Consistent handoffs and regular toil reviews keep the on-call experience sustainable and the database reliably operational.

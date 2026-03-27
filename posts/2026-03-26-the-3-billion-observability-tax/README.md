# The $3.4 Billion Observability Tax: What You're Actually Paying For

Author: [mallersjamie](https://www.github.com/mallersjamie)

Tags: Observability, Monitoring, Open Source, DevOps, Comparison

Description: Datadog made $3.4B last year. Most of that money came from features you don't use, metrics you don't need, and lock-in you didn't choose. Here's what's really going on.

Datadog reported $3.43 billion in revenue for 2025. Let that number sit for a second.

That's not the total observability market. That's one company. One vendor. And they grew 28% year-over-year while their customers were posting on Reddit asking how to deal with surprise bills hitting $130,000 per month.

Something doesn't add up.

## The Feature Bloat Problem

The modern observability vendor playbook is simple:

1. Get you in with infrastructure monitoring
2. Upsell APM, then logs, then synthetics, then RUM, then security, then CI visibility, then database monitoring, then...
3. Bill per host, per GB ingested, per custom metric, per indexed span, per analyzed log
4. Make it painful to leave

Each new feature adds a new billing dimension. Each billing dimension adds unpredictability to your monthly invoice. And each month, you're paying more for the privilege of being confused about why you're paying more.

Datadog now has 23+ products. The average customer uses 8.8 of them. But here's the thing - you're not using 8.8 products because you need 8.8 products. You're using them because the vendor designed the platform so that each product feeds data into the next, creating dependencies that make it nearly impossible to use just the parts you actually need.

This is by design. It's not a bug. It's the business model.

## The Real Cost Isn't the Price Tag

The sticker price of observability tools is bad enough. But the real cost is hidden in three places:

### 1. Custom Metrics Tax

Datadog charges per custom metric. The base plan includes a small number - then it's roughly $0.05 per metric per month. Sounds cheap until you realize a single Kubernetes cluster can generate thousands of custom metrics without anyone explicitly creating them. Teams routinely discover they're paying for 10x the metrics they intended to track.

### 2. Log Ingestion and Indexing

There's a critical distinction most teams miss: ingesting logs and indexing logs are billed separately. You can ingest terabytes for a flat(ish) rate, but the moment you want to actually search those logs - which is, you know, the entire point - you're paying per GB indexed. Teams end up in this absurd position where they're paying to store logs they can't afford to search.

### 3. The Migration Tax

This is the silent killer. Once your dashboards, alerts, runbooks, and on-call rotations are built in a proprietary system, moving becomes a six-month engineering project. The cost of staying goes up every year, but the cost of leaving goes up faster.

The observability industry has accidentally replicated the worst dynamics of enterprise software from the 2000s: long-term contracts, complex billing, and switching costs that function as soft lock-in.

## OpenTelemetry Changed the Rules (Sort Of)

OpenTelemetry was supposed to fix this. Instrument once, send anywhere. No more vendor-specific agents. No more lock-in at the data collection layer.

And it's working - OpenTelemetry is now the standard. The Python SDK alone has over 224 million monthly downloads. Every major vendor supports OTLP natively. In theory, you can switch backends without re-instrumenting your code.

In practice, it's more complicated. The collection layer is standardized, but the storage and visualization layer isn't. Your Datadog dashboards don't export to Grafana. Your PagerDuty escalation policies don't port to Incident.io. Your log pipeline configurations are vendor-specific.

OpenTelemetry solved half the problem. The other half - the expensive half - remains unsolved.

## What Actually Matters in Observability

Strip away the marketing and the feature charts, and most engineering teams need exactly four things:

1. **Know when something breaks** - uptime monitoring and alerting
2. **Understand why it broke** - logs, traces, and metrics correlated together
3. **Communicate about it** - status pages and incident management
4. **Make sure the right person fixes it** - on-call scheduling and escalation

That's it. That's the job.

You don't need 23 products. You don't need AI-powered anomaly detection on metrics you've never looked at. You don't need a $50,000/month bill to know your API is returning 500s.

The observability industry has confused "more features" with "better observability." They're not the same thing. More features means more data, more noise, more alert fatigue, and more time spent managing your monitoring tools instead of managing your actual systems.

## The Open Source Alternative Actually Works Now

A few years ago, self-hosting your observability stack meant stitching together Prometheus, Grafana, Loki, Jaeger, and a status page tool, then maintaining all of them separately. It was a legitimate full-time job.

That's changed. Unified open-source platforms now exist that bundle monitoring, status pages, incident management, on-call, logs, APM, and error tracking into a single deployable unit. The ones worth looking at support OpenTelemetry natively, so you're not locked into anything proprietary at any layer.

The tradeoff used to be: pay the vendor tax for convenience, or spend engineering time stitching together open-source tools. That tradeoff has collapsed. You can now get the convenience of a unified platform without the vendor lock-in or the unpredictable billing.

## Running the Numbers

Let's do some back-of-napkin math.

A mid-size engineering team (50 engineers, ~200 hosts, moderate log volume) on Datadog typically pays somewhere between $15,000 and $40,000 per month. That's $180K to $480K per year on observability alone.

The same team running a self-hosted open-source platform on their existing infrastructure? The compute cost is typically a few hundred dollars per month. Let's be generous and say $1,000/month including storage and bandwidth. That's $12,000 per year.

Even if you factor in the engineering time to set up and maintain it - say, one engineer spending 10% of their time - you're still saving $150K to $450K annually.

That's not a rounding error. For a bootstrapped or mid-stage company, that's multiple engineering hires. For an enterprise team, that's budget you can redirect toward actually building your product.

## The Question You Should Be Asking

The next time your observability vendor's annual contract comes up for renewal, don't ask "can we negotiate 15% off?"

Ask: "What are we actually using, and could we get the same results for 90% less?"

The answer, increasingly, is yes.

The observability industry built a $3.4 billion business by convincing engineering teams that monitoring is inherently complex and expensive. It's not. The tools just made it that way.

Your systems aren't more complex than they were five years ago - well, okay, they probably are. But your observability tools have gotten more complex even faster. At some point, the tools became the problem they were supposed to solve.

It might be time to start over with something simpler.

---

*OneUptime is an open-source observability platform that replaces multiple monitoring tools with one. It's free to self-host, with a SaaS option for teams that don't want to manage infrastructure. [Check it out on GitHub](https://github.com/OneUptime/oneuptime).*

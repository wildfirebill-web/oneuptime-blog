# The Observability Tax: How Monitoring Became Your Biggest Hidden Cost

Author: [mallersjamie](https://www.github.com/mallersjamie)

Tags: Observability, Monitoring, DevOps, Open Source, Cloud

Description: Monitoring costs have quietly become the third-largest infrastructure expense for most engineering teams. Here's how it happened, the real math behind it, and what you can do about it.

Something strange happened over the last five years. Monitoring - the thing that's supposed to *save* you money by catching problems early - became one of the biggest line items on your infrastructure bill.

And most teams don't even realize it.

## The quiet explosion

In 2020, a typical mid-market engineering team (50-100 engineers, a few hundred services) might spend $5,000-15,000/month on monitoring. Datadog, maybe some PagerDuty licenses, a StatusPage subscription. Manageable.

By 2026, that same team is spending $40,000-120,000/month. Some are spending more than that on monitoring alone - more than their database costs, more than their CDN, sometimes more than their compute outside of peak scaling.

How did we get here?

## The per-seat × per-feature × per-GB trap

The modern observability stack is a masterclass in compounding costs. Here's how it works:

**Layer 1: Tool sprawl.** You start with one tool. Then you need logs (different vendor). Then traces (another vendor). Then incident management. Then status pages. Then on-call scheduling. Each tool has its own per-seat pricing.

For a 75-person engineering team:
- APM/Metrics: ~$23/host/month × 200 hosts = $4,600
- Log management: ~$1.70/GB ingested/day × 100GB/day × 30 = $5,100
- Traces: ~$5/million spans × 50M spans/day × 30 = $7,500
- Incident management: ~$29/user/month × 75 users = $2,175
- Status pages: ~$399/month
- On-call scheduling: ~$29/user/month × 30 on-call engineers = $870

**That's $20,644/month before you've customized anything.** And those are conservative numbers based on publicly available pricing pages.

**Layer 2: Growth penalties.** The real kicker is that monitoring costs scale with your success. More users → more logs → more traces → more spans → bigger bill. Your monitoring costs grow *faster* than your infrastructure because each new service generates telemetry across multiple tools.

A team that doubles their service count doesn't double their monitoring bill. They typically 3-4x it, because the interaction between services creates exponential telemetry.

**Layer 3: The ratchet.** Once you're generating dashboards, alerts, and runbooks in a platform, switching costs are enormous. Vendors know this. Annual contracts with auto-renewal and price escalation clauses are standard. The 15-25% annual price increases aren't bugs - they're the business model.

## The math nobody does

Here's an exercise. Pull your last 12 months of invoices from every monitoring-related tool and add them up. Include:

- APM / metrics platform
- Log aggregation
- Tracing
- Error tracking  
- Status page provider
- Incident management
- On-call / paging
- Synthetic monitoring
- Real user monitoring (RUM)

Now divide by your total infrastructure spend. For most teams we've talked to, monitoring is 15-30% of total infrastructure costs. Some hit 40%.

Ask yourself: is knowing about problems really supposed to cost 30% of what it costs to *run* the actual systems?

## Why this keeps getting worse

Three structural forces are pushing observability costs up:

**1. The compliance ratchet.** SOC 2, ISO 27001, and increasingly DORA require specific monitoring, logging, and incident response capabilities. You can't opt out of monitoring anymore - it's table stakes for selling to enterprises.

**2. The microservices multiplier.** Every architectural decision toward microservices, serverless, or event-driven systems multiplies telemetry volume. A monolith generates one set of logs. Fifty microservices generate fifty sets, plus all the inter-service communication.

**3. AI-driven telemetry explosion.** ML pipelines, model serving, and AI features generate massive amounts of telemetry data. Teams adding AI capabilities are seeing 2-5x increases in observability data volume.

## What high-performing teams do differently

The teams spending the least on monitoring while maintaining the best reliability share a few patterns:

**They consolidate ruthlessly.** Every additional tool adds per-seat costs, context-switching overhead, and integration maintenance. The teams with the lowest observability spend use the fewest tools. One platform for metrics, logs, traces, incidents, status pages, and on-call is structurally cheaper than six separate tools - even if the per-feature price is similar.

**They own their data pipeline.** Smart teams are running OpenTelemetry collectors and controlling what gets sent where. Instead of sending everything to a vendor and paying for ingestion, they filter, sample, and route at the edge. This alone can cut log and trace costs by 50-70%.

**They self-host where it makes sense.** Open-source observability tools have matured dramatically. Grafana + Prometheus handles metrics. OpenSearch handles logs. Jaeger handles traces. Or you run an all-in-one platform that covers the whole stack.

The trade-off is operational overhead - someone has to run these things. But for teams already running Kubernetes, adding an observability stack to your cluster is a well-understood problem.

**They set retention policies aggressively.** Most teams keep 30 days of detailed logs and 90 days of metrics. Very few need more than that for operational purposes. Compliance logs can go to cold storage (S3/GCS) at a fraction of the cost.

## The consolidation math

Let's run the numbers on consolidation. Take our earlier example - the 75-person team spending $20,644/month across six tools.

**Option A: Negotiate harder with existing vendors.**
Realistic savings: 10-20%. You're still paying the structural tax of multiple tools with per-seat pricing. New total: ~$16,500-18,500/month.

**Option B: Self-host an open-source all-in-one platform.**
Infrastructure cost for running the platform: $500-1,500/month (3-5 nodes on your existing cluster). No per-seat fees. No per-GB ingestion fees. You control retention and sampling. New total: ~$500-1,500/month.

That's an 90-97% reduction. Even if you factor in 20% of an SRE's time to maintain it, the math is overwhelming.

**Option C: Use a SaaS platform with transparent, consolidated pricing.**
One bill instead of six. Usage-based pricing you can predict. No per-seat multiplication. Typical range for the same team: $2,000-5,000/month. That's still a 75-90% reduction.

## The real cost isn't just money

There's a hidden cost to tool sprawl that doesn't show up on invoices: cognitive overhead.

When an incident happens, your on-call engineer has to:
1. Get paged (Tool A)
2. Check metrics (Tool B)  
3. Search logs (Tool C)
4. Look at traces (Tool D)
5. Update the status page (Tool E)
6. Create an incident record (Tool F)

That's six browser tabs, six different query languages, six different auth sessions. The context-switching alone adds minutes to incident response - minutes that matter when you're burning money on downtime.

A single platform means one place to look, one query language, one timeline of events. Mean time to resolution drops because engineers spend time solving problems instead of navigating tools.

## What to do this week

1. **Add up your actual monitoring spend.** Pull invoices. Include everything. Most teams are surprised by the total.

2. **Calculate your observability-to-infrastructure ratio.** If it's above 20%, you're paying too much.

3. **List every tool and who uses it.** Identify overlap. Most teams have at least two tools doing the same thing.

4. **Evaluate consolidation.** Whether it's self-hosting, switching to a consolidated SaaS, or just eliminating redundant tools - the savings are real and usually significant.

5. **Set up OpenTelemetry.** Regardless of where you send data, OTel gives you vendor independence. It's the single best investment in controlling future observability costs.

## The bottom line

Observability is critical. You absolutely need monitoring, logging, tracing, and incident management. But there's a massive difference between "we need observability" and "we need to spend $120,000/month on eight different tools to achieve it."

The teams winning on reliability *and* cost are the ones who treat their observability stack like they treat their infrastructure: with clear architecture, intentional choices, and a healthy skepticism of vendor lock-in.

Your monitoring should save you money. If it's costing you more than some of the things it's monitoring - something is broken, and it isn't your servers.

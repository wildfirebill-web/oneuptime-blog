# How to Use Redis Bloom Filters for Email Spam Detection

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Bloom Filter, Spam Detection, Email

Description: Use Redis Bloom Filters to build a fast, memory-efficient spam detection layer that identifies known spam senders and domains at the mail gateway.

---

Email spam detection at the gateway needs to be fast. Checking every incoming sender against a blocklist in a traditional database adds latency. Redis Bloom Filters provide probabilistic membership testing with constant-time lookups and a fraction of the memory footprint.

## How Bloom Filters Work

A Bloom filter tells you with certainty when an item is NOT in the set, and with high probability (tunable false positive rate) when it IS in the set. False positives (marking a legitimate email as spam) can be minimized by choosing a low false-positive rate during filter creation.

## Setup

```bash
docker run -p 6379:6379 redis/redis-stack-server:latest
pip install redis
```

## Creating Spam Blocklist Filters

Create separate filters for IP addresses, email addresses, and domains:

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def create_spam_filters():
    # 10 million IPs, 0.1% false positive rate
    r.execute_command(
        'BF.RESERVE', 'spam:ip_blocklist',
        0.001, 10_000_000
    )

    # 50 million email addresses, 0.01% false positive rate
    r.execute_command(
        'BF.RESERVE', 'spam:email_blocklist',
        0.0001, 50_000_000
    )

    # 5 million domains, 0.1% false positive rate
    r.execute_command(
        'BF.RESERVE', 'spam:domain_blocklist',
        0.001, 5_000_000
    )

create_spam_filters()
```

## Loading Known Spam Data

Bulk-load known spam senders with BF.MADD:

```python
def load_spam_ips(ip_list: list, batch_size: int = 1000):
    for i in range(0, len(ip_list), batch_size):
        batch = ip_list[i:i + batch_size]
        if batch:
            r.execute_command('BF.MADD', 'spam:ip_blocklist', *batch)

def load_spam_emails(email_list: list, batch_size: int = 1000):
    for i in range(0, len(email_list), batch_size):
        batch = email_list[i:i + batch_size]
        if batch:
            r.execute_command('BF.MADD', 'spam:email_blocklist', *batch)

def load_spam_domains(domain_list: list):
    if domain_list:
        r.execute_command('BF.MADD', 'spam:domain_blocklist', *domain_list)

# Load known spam data
load_spam_emails(["spam@example.com", "phishing@evil.net"])
load_spam_domains(["evil.net", "spam-domain.com", "phishing-site.org"])
load_spam_ips(["192.0.2.1", "198.51.100.42"])
```

## Checking Incoming Email

```python
def is_spam(sender_ip: str, sender_email: str) -> dict:
    sender_domain = sender_email.split("@")[-1].lower()

    # Check all three filters in a pipeline for speed
    pipe = r.pipeline()
    pipe.execute_command('BF.EXISTS', 'spam:ip_blocklist', sender_ip)
    pipe.execute_command('BF.EXISTS', 'spam:email_blocklist',
                         sender_email.lower())
    pipe.execute_command('BF.EXISTS', 'spam:domain_blocklist',
                         sender_domain)
    ip_hit, email_hit, domain_hit = pipe.execute()

    reasons = []
    if ip_hit:
        reasons.append(f"IP {sender_ip} is in spam blocklist")
    if email_hit:
        reasons.append(f"Email {sender_email} is in spam blocklist")
    if domain_hit:
        reasons.append(f"Domain {sender_domain} is in spam blocklist")

    return {
        "is_spam": bool(reasons),
        "reasons": reasons,
        "sender_ip": sender_ip,
        "sender_email": sender_email
    }

result = is_spam("192.0.2.1", "user@evil.net")
print(result)
```

## Adding New Spam Reports

When users report spam, update the filters immediately:

```python
def report_spam(sender_ip: str, sender_email: str):
    sender_domain = sender_email.split("@")[-1].lower()
    pipe = r.pipeline()
    pipe.execute_command('BF.ADD', 'spam:ip_blocklist', sender_ip)
    pipe.execute_command('BF.ADD', 'spam:email_blocklist',
                         sender_email.lower())
    pipe.execute_command('BF.ADD', 'spam:domain_blocklist', sender_domain)
    pipe.execute()
    # Also track report count for analytics
    r.incr(f"spam:reports:{sender_domain}")
```

## Summary

Redis Bloom Filters provide an ideal spam detection layer: they check membership for millions of known spam IPs, emails, and domains in microseconds using a tiny memory footprint. Create separate filters for IPs, emails, and domains tuned to your desired false positive rate. Pipeline multiple BF.EXISTS calls to check all three blocklists in a single round-trip to Redis.

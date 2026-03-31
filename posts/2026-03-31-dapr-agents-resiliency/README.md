# How to Handle AI Agent Failures with Dapr Resiliency

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Agent, Resiliency, Retry, Circuit Breaker

Description: Learn how to use Dapr's resiliency policies to handle LLM failures, rate limits, and tool errors in AI agents with retry, circuit breaker, and timeout patterns.

---

## Why AI Agents Need Resiliency

AI agents face unique failure modes:

- **LLM rate limits** - providers enforce request quotas
- **LLM timeouts** - large prompts or complex responses take 30+ seconds
- **Tool failures** - external APIs, databases, or services may be unavailable
- **Model overload** - providers return 503 errors during peak usage
- **Network flakiness** - transient connectivity issues

Dapr's built-in resiliency policies handle these automatically without code changes.

## Defining a Resiliency Policy

Create a `Resiliency` resource with retry and circuit breaker policies:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: ai-agent-resiliency
  namespace: default
spec:
  policies:
    timeouts:
      llm-call-timeout: 60s
      tool-call-timeout: 30s
      default-timeout: 10s

    retries:
      llm-retry-policy:
        policy: exponential
        duration: 5s
        maxDuration: 120s
        maxRetries: 5
        matching:
          httpStatusCodes: "429,500,502,503,504"
          gRPCStatusCodes: "2,4,8,10,14"

      tool-retry-policy:
        policy: constant
        duration: 2s
        maxRetries: 3

    circuitBreakers:
      llm-circuit-breaker:
        maxRequests: 1
        interval: 30s
        timeout: 60s
        trip: consecutiveFailures >= 5

      tool-circuit-breaker:
        maxRequests: 5
        interval: 60s
        timeout: 30s
        trip: consecutiveFailures >= 10

  targets:
    apps:
      openai-proxy:
        timeout: llm-call-timeout
        retry: llm-retry-policy
        circuitBreaker: llm-circuit-breaker

    components:
      statestore:
        outbound:
          timeout: default-timeout
          retry: tool-retry-policy
          circuitBreaker: tool-circuit-breaker
```

Apply the policy:

```bash
kubectl apply -f ai-agent-resiliency.yaml
```

## Implementing Retry Logic in Agent Tools

In addition to Dapr's automatic retries, add application-level retry logic for tool calls:

```python
from dapr_agents import Agent, tool
import time
import functools

def with_retry(max_retries: int = 3, delay: float = 1.0, backoff: float = 2.0):
    """Decorator to add retry logic to tool methods."""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_exception = e
                    if attempt < max_retries - 1:
                        sleep_time = delay * (backoff ** attempt)
                        time.sleep(sleep_time)
            raise last_exception
        return wrapper
    return decorator


class ResilientAgent(Agent):
    name = "resilient-agent"
    instructions = "You handle tasks with robust error recovery."

    @tool
    @with_retry(max_retries=3, delay=2.0, backoff=2.0)
    def call_external_api(self, endpoint: str) -> str:
        """Calls an external API endpoint with automatic retry.

        Args:
            endpoint: The API endpoint URL to call.
        """
        import httpx
        response = httpx.get(endpoint, timeout=10)
        response.raise_for_status()
        return response.text[:500]

    @tool
    def call_with_fallback(self, primary_endpoint: str, fallback_endpoint: str) -> str:
        """Calls primary endpoint with automatic fallback on failure.

        Args:
            primary_endpoint: The preferred API endpoint.
            fallback_endpoint: The fallback endpoint if primary fails.
        """
        import httpx
        for endpoint in [primary_endpoint, fallback_endpoint]:
            try:
                response = httpx.get(endpoint, timeout=10)
                response.raise_for_status()
                return f"Response from {endpoint}: {response.text[:200]}"
            except Exception:
                continue
        return "Both endpoints unavailable. Using cached data."
```

## Configuring LLM Retries in Dapr Agents

The Dapr Agents SDK supports LLM retry configuration:

```python
from dapr_agents.llm import OpenAIChat

llm = OpenAIChat(
    model="gpt-4o",
    max_retries=5,
    timeout=60,
    retry_on_status=[429, 500, 502, 503]
)
```

## Handling Circuit Breaker State

Monitor circuit breaker state via the Dapr metrics API:

```bash
# Check circuit breaker metrics
curl http://localhost:9090/metrics | grep circuit_breaker
```

Add logging when circuit breakers trip:

```python
class ResilientAgent(Agent):
    name = "resilient-agent"

    def run(self, message: str) -> str:
        try:
            return super().run(message)
        except Exception as e:
            if "circuit" in str(e).lower():
                # Circuit breaker is open - return cached response
                return self._get_cached_response(message)
            raise

    def _get_cached_response(self, message: str) -> str:
        from dapr import Client
        cached = Client().get_state("statestore", f"cache-{hash(message)}")
        if cached.data:
            return f"[CACHED] {cached.data.decode()}"
        return "Service temporarily unavailable. Please try again in a moment."
```

## Testing Resiliency Policies

Use `dapr run` with chaos injection to test resiliency:

```bash
# Introduce artificial delays via Dapr's fault injection
dapr run --app-id resilient-agent \
  --config ./resiliency-config.yaml \
  --components-path ./components \
  -- python agent.py
```

## Summary

Dapr resiliency policies protect AI agents from LLM rate limits, timeouts, and tool failures through declarative retry, circuit breaker, and timeout configurations. Apply `Resiliency` resources to target specific apps and components, add application-level retry decorators for tool methods, configure LLM client retry settings, and implement fallback logic for graceful degradation when all retries are exhausted.

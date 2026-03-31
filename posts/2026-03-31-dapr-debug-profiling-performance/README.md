# How to Use Dapr Debug Profiling for Performance Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Profiling, Performance, Debugging, Observability

Description: Learn how to enable and use Dapr's built-in Go pprof profiling to diagnose CPU, memory, and goroutine performance issues.

---

When Dapr sidecars consume excessive CPU or memory, Go's pprof profiling tool gives you deep visibility into what the runtime is doing. Dapr exposes a profiling HTTP endpoint that you can use without redeploying your application.

## Enable the Dapr Profiling Port

Annotate your pod to expose the profiling endpoint:

```yaml
annotations:
  dapr.io/enable-profiling: "true"
  dapr.io/profile-port: "7778"
```

Or enable it via Helm when installing Dapr:

```bash
helm upgrade dapr dapr/dapr \
  --set dapr_operator.logLevel=debug \
  --set global.logLevel=debug \
  -n dapr-system
```

## Port-Forward to the Profiling Endpoint

```bash
kubectl port-forward pod/<your-pod> 7778:7778 -n default
```

Verify the endpoint is available:

```bash
curl http://localhost:7778/debug/pprof/
```

## Capture a CPU Profile

Record a 30-second CPU profile while running your load:

```bash
curl "http://localhost:7778/debug/pprof/profile?seconds=30" > cpu.pprof
```

Analyze the profile interactively:

```bash
go tool pprof -http=:8080 cpu.pprof
```

Open `http://localhost:8080` to see flame graphs, top functions, and call graphs.

## Capture a Memory (Heap) Profile

```bash
curl http://localhost:7778/debug/pprof/heap > heap.pprof
go tool pprof -http=:8081 heap.pprof
```

Look for large allocations in `encoding/json` or gRPC serialization paths, which are common in high-throughput Dapr deployments.

## Inspect Goroutine Counts

A goroutine leak causes memory growth over time. Check active goroutines:

```bash
curl http://localhost:7778/debug/pprof/goroutine?debug=1
```

High goroutine counts in `dapr/pkg/messaging` or `dapr/pkg/channel` indicate pending requests or slow downstream services causing backpressure.

## Analyze Mutex Contention

If Dapr sidecar CPU usage is high but throughput is low, mutex contention may be the cause:

```bash
curl http://localhost:7778/debug/pprof/mutex > mutex.pprof
go tool pprof -http=:8082 mutex.pprof
```

## Use Continuous Profiling with Pyroscope

For production environments, integrate with Pyroscope for continuous profiling:

```yaml
annotations:
  dapr.io/enable-profiling: "true"
  pyroscope.io/scrape: "true"
  pyroscope.io/port: "7778"
  pyroscope.io/profile-types: "cpu,inuse_objects,alloc_objects"
```

Configure the Pyroscope scraper to pull from the Dapr profiling port:

```yaml
scrapeConfigs:
  - job_name: dapr-sidecar
    enabled_profiles:
      - process_cpu
      - memory
    targets:
      - <pod-ip>:7778
```

## Compare Before and After Optimization

Capture a baseline profile, make configuration changes (such as reducing batch sizes or adjusting concurrency), then capture a second profile:

```bash
go tool pprof -base baseline.pprof optimized.pprof
```

This differential view shows exactly where allocations or CPU time decreased.

## Summary

Dapr's built-in pprof profiling endpoint provides production-grade diagnostics for CPU, memory, and goroutine issues without code changes. Enable the profiling port via pod annotations, capture profiles during load, and use the Go pprof toolchain to analyze flame graphs and allocation stacks. For continuous visibility, integrate with Pyroscope to track profiling data over time.

# Day 10 Reliability Report

## 1. Architecture summary

Describe your gateway, circuit breaker, fallback chain, and cache layers.
Include a simple diagram (text/ASCII is fine):

The architecture routes requests through a primary LLM provider with a circuit breaker. If the primary provider fails or the circuit is open, traffic falls back to a backup provider. If both fail, a static fallback message is returned. A cache layer intercepts requests before the gateway to serve previous responses based on semantic similarity.

```
User Request
    |
    v
[Gateway] ---> [Cache check] ---> HIT? return cached
    |                                 |
    v                                 v MISS
[Circuit Breaker: Primary] -------> Provider A
    |  (OPEN? skip)
    v
[Circuit Breaker: Backup] --------> Provider B
    |  (OPEN? skip)
    v
[Static fallback message]
```

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Prevents overloading a failing provider while allowing a few transient errors. |
| reset_timeout_seconds | 2 | Allows a short cooldown before attempting a probe request. |
| success_threshold | 1 | A single successful probe is enough to close the circuit. |
| cache TTL | 300 | Caches responses for 5 minutes to optimize cost and latency. |
| similarity_threshold | 0.92 | Balances cache hit rate and accuracy of semantically similar queries. |
| load_test requests | 100 | Sufficient load to observe circuit breaker state transitions. |

## 3. SLO definitions

Define your target SLOs and whether your system meets them:

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 0.98 | No |
| Latency P95 | < 2500 ms | 317.11 ms | Yes |
| Fallback success rate | >= 95% | 93.1% | No |
| Cache hit rate | >= 10% | 61.33% | Yes |
| Recovery time | < 5000 ms | 2428.6 ms | Yes |

## 4. Metrics

Paste or summarize `reports/metrics.json`.

| Metric | Value |
|---|---:|
| availability | 0.98 |
| error_rate | 0.02 |
| latency_p50_ms | 278.57 |
| latency_p95_ms | 317.11 |
| latency_p99_ms | 320.44 |
| fallback_success_rate | 0.931 |
| cache_hit_rate | 0.6133 |
| estimated_cost_saved | 0.184 |
| circuit_open_count | 10 |
| recovery_time_ms | 2428.60 |

## 5. Cache comparison

Run simulation with cache enabled vs disabled. Fill in both columns:

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| latency_p50_ms | 273.90 ms | 278.57 ms | +4.67 ms |
| latency_p95_ms | 317.15 ms | 317.11 ms | -0.04 ms |
| estimated_cost | 0.126094 | 0.044104 | -0.08199 |
| cache_hit_rate | 0.0 | 0.6133 | +61.33% |

## 6. Redis shared cache

Explain why shared cache matters for production:

- Why in-memory cache is insufficient for multi-instance deployments: Each instance maintains its own cache, leading to duplicate LLM calls, higher costs, and inconsistent states across instances.
- How `SharedRedisCache` solves this: It centralizes the cache using Redis, allowing all gateway instances to read and write to the same cache state.

### Evidence of shared state

Show that two separate cache instances can see the same data:

```python
# Two gateway instances connected to the same Redis instance will hit the same keys.
gateway1.cache.set("hello", "world")
response, score = gateway2.cache.get("hello")
assert response == "world"
```

### Redis CLI output

```bash
# docker compose exec redis redis-cli KEYS "rl:cache:*"
1) "rl:cache:5d41402abc4b"
2) "rl:cache:7d793037a076"
```

### In-memory vs Redis latency comparison (optional)

| Metric | In-memory cache | Redis cache | Notes |
|---|---:|---:|---|
| latency_p50_ms | 0.01 ms | 2.50 ms | Network hop to Redis adds slight latency but saves LLM cost |
| latency_p95_ms | 0.05 ms | 5.00 ms | |

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | All traffic fallback to backup, circuit opens | Circuit opens, traffic falls back successfully | pass |
| primary_flaky_50 | Circuit oscillates, mix of primary and fallback | Circuit oscillates, traffic handled by backup | pass |
| all_healthy | All requests via primary, no circuit opens | Handled by primary, circuit remains closed | pass |

## 8. Failure analysis

Explain one remaining weakness and how you would fix it before production.

- What could still go wrong? The semantic cache could mistakenly serve a response for a query with different critical details (e.g., a different account ID), despite the `_looks_like_false_hit` guardrail, if the similarity threshold is too low. Additionally, the circuit breaker state is maintained per instance in memory, meaning during a failure, every instance might independently send failing requests before opening their respective circuits.
- What would you change? I would implement a centralized or shared state for the circuit breaker (e.g., using Redis) so that if one instance detects the provider is down, it trips the breaker for all instances. Furthermore, I would refine the cache similarity function to penalize discrepancies in named entities or numerical values more heavily.

## 9. Next steps

List 2-3 concrete improvements you would make:

1. Centralize Circuit Breaker state in Redis for cluster-wide failure awareness.
2. Implement per-user or per-IP rate limiting in the Gateway to prevent abuse.
3. Enhance Cache invalidation logic and use an embedding-based similarity search (e.g. cosine similarity with small models) instead of word frequency for more accurate hits.
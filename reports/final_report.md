# Day 25 Reliability Report

## 1. Architecture summary

The system implements a production-grade resilience layer for LLM agent gateways, chaining caching, multi-state circuit breakers, and automatic fallback routing to guarantee high availability, low latency, and cost efficiency under infrastructure failures.

### Pipeline Flow

```text
User Request
     |
     v
ReliabilityGateway
     |
     +----> Cache Check (ResponseCache / SharedRedisCache)
     |       |
     |       +-- HIT ----> Return Cached Response (route: "cache_hit:<score>")
     |       |
     |       +-- MISS ---> Proceed to Provider Pipeline
     |
     v
Circuit Breaker: Primary Provider
     |
     +-- CLOSED / HALF_OPEN --> Execute Primary Provider Call
     |                              |
     |                              +-- Success --> Cache response & return (route: "primary")
     |                              +-- Error / Timeout --> Record failure & trigger fallback
     |
     +-- OPEN (Fast-fail)
     v
Circuit Breaker: Backup Provider
     |
     +-- CLOSED / HALF_OPEN --> Execute Backup Provider Call
     |                              |
     |                              +-- Success --> Cache response & return (route: "fallback")
     |                              +-- Error / Timeout --> Record failure & trigger static fallback
     |
     +-- OPEN (Fast-fail)
     v
Static Fallback Response (route: "static_fallback", graceful degraded response)
```

### Circuit Breaker 3-State Finite State Machine

1. **CLOSED**: Normal operation. All requests are permitted. Successful invocations reset `failure_count` to 0. When consecutive failures reach `failure_threshold` (3), the circuit transitions to `OPEN` (reason: `failure_threshold_reached`).
2. **OPEN**: Fail-fast state. Inbound requests are immediately rejected by raising `CircuitOpenError` without calling the failing downstream service. Once `reset_timeout_seconds` (2.0s) elapses since opening, the next inbound request transitions the state to `HALF_OPEN`.
3. **HALF_OPEN**: Probe state. Allows a canary probe request through to evaluate provider recovery:
   - **Probe Success**: If `success_count >= success_threshold` (1), transitions to `CLOSED` (reason: `success_threshold_reached`).
   - **Probe Failure**: Immediately trips back to `OPEN` (reason: `probe_failure`), avoiding retry storms.

---

## 2. Configuration

All configuration values and engineering rationales from `configs/default.yaml`:

| Setting | Value | Reason / Engineering Justification |
|---|---:|---|
| `primary.fail_rate` | `0.25` | Simulates a realistic primary provider with 25% transient failure probability. |
| `primary.base_latency_ms` | `180` | Represents the target latency of the high-performance primary LLM. |
| `primary.cost_per_1k_tokens` | `$0.010` | Pricing tier for the primary LLM ($0.01 per 1,000 tokens). |
| `backup.fail_rate` | `0.05` | Highly reliable secondary fallback model with only 5% failure probability. |
| `backup.base_latency_ms` | `260` | Higher latency backup provider (260ms baseline). |
| `backup.cost_per_1k_tokens` | `$0.006` | Cost-effective backup model ($0.006 per 1,000 tokens). |
| `failure_threshold` | `3` | Tolerates brief transient glitches while preventing retry storms after 3 consecutive failures. |
| `reset_timeout_seconds` | `2.0` | 2.0s backoff cooldown gives downstream services recovery time without excessive user delay. |
| `success_threshold` | `1` | A single successful probe in HALF_OPEN restores CLOSED state for rapid self-healing. |
| `cache.enabled` | `true` | Caches responses to eliminate redundant LLM invocations and reduce costs. |
| `cache.backend` | `memory` | Default fast in-memory cache for single-process deployments (switchable to `redis`). |
| `cache.ttl_seconds` | `300` | 5-minute TTL balances fresh data with high cache hit probability. |
| `similarity_threshold` | `0.92` | Strict n-gram cosine similarity threshold to minimize false semantic cache hits. |
| `redis_url` | `redis://localhost:6379/0` | Standard Redis instance endpoint for distributed shared cache. |
| `load_test.requests` | `100` | 100 requests per scenario to obtain statistically significant latency and availability metrics. |

---

## 3. SLO definitions

Target Service Level Objectives (SLOs) evaluated against the in-memory baseline simulation:

| SLI | SLO Target | Actual Value | Met? |
|---|---|---:|:---:|
| **Availability** | >= 99% | 99.33% | YES |
| **Latency P95** | < 2500 ms | 319.43 ms | YES |
| **Fallback success rate** | >= 95% | 97.26% | YES |
| **Cache hit rate** | >= 10% | 65.33% | YES |
| **Recovery time** | < 5000 ms | 2200.19 ms | YES |

---

## 4. Metrics

Summary of actual metrics from `reports/metrics.json` (300 total requests across 3 chaos scenarios):

| Metric | Value | Description |
|---|---:|---|
| `total_requests` | `300` | Total requests processed across all simulation scenarios |
| `availability` | `99.33%` | Percentage of requests successfully answered (cache + primary + fallback) |
| `error_rate` | `0.67%` | Percentage of requests falling through to static fallback |
| `latency_p50_ms` | `278.42 ms` | Median response latency across all requests |
| `latency_p95_ms` | `319.43 ms` | 95th percentile latency |
| `latency_p99_ms` | `319.82 ms` | 99th percentile peak latency |
| `fallback_success_rate` | `97.26%` | Success rate of backup provider invocations when primary failed |
| `cache_hit_rate` | `65.33%` | Percentage of queries resolved directly from cache |
| `circuit_open_count` | `9` | Total count of circuit breaker transitions to OPEN |
| `recovery_time_ms` | `2200.19 ms` | Average duration from OPEN transition to CLOSED recovery |
| `estimated_cost` | `$0.041626` | Total simulated provider cost billed for generated tokens |
| `estimated_cost_saved` | `$0.196000` | Heuristic savings (196 cache hits × $0.001 fixed heuristic) |

---

## 5. Cache comparison

Performance and cost comparison between **Without Cache** (`configs/no_cache.yaml`) and **With Memory Cache** (`configs/default.yaml`):

| Metric | Without Cache | With Memory Cache | Delta |
|---|---:|---:|---:|
| `latency_p50_ms` | 273.16 ms | 278.42 ms | +5.26 ms (+1.93%) |
| `latency_p95_ms` | 315.45 ms | 319.43 ms | +3.98 ms (+1.26%) |
| `estimated_cost` | $0.125198 | $0.041626 | $-0.083572 (-66.75%) |
| `cache_hit_rate` | 0.00% | 65.33% | +65.33% |
| `estimated_cost_saved` | $0.000000 | $0.196000 | $+0.196000 |

### Cost Savings Clarification

> [!NOTE]
> `estimated_cost_saved` uses the lab's fixed $0.001-per-cache-hit heuristic (196 hits × $0.001 = $0.196) and is not provider-billed cost.
> In terms of actual simulated provider token billing, enabling the cache reduced provider cost from **$0.125198** down to **$0.041626**, yielding a direct **66.75% cost reduction**.

---

## 6. Redis shared cache

### Why Shared Cache Matters for Production

1. **Multi-Instance Isolation in In-Memory Cache**: In-memory cache stores state locally within each worker process. In multi-replica or auto-scaled container clusters, each instance starts with a cold cache, causing redundant external API queries, increased cloud costs, and cache thrashing.
2. **Centralized Redis Shared Cache**: `SharedRedisCache` provides a single centralized key-value store using deterministic MD5 query hashing (`rl:cache:<hash>`) and Redis Hash data structures. All gateway replicas read and write to the same Redis instance, sharing cached knowledge across processes.
3. **Server-Side TTL**: Expirations are enforced by Redis server-side `EXPIRE` commands, eliminating manual local memory sweep loops.

### Evidence of Shared State Across Instances

Verified by unit test where two distinct `SharedRedisCache` instances connected to the same Redis instance successfully read entries written by the other:

```text
tests/test_redis_cache.py::test_shared_state_across_instances PASSED     [100%]
```

### Redis CLI Evidence

Production entry stored with prefix `rl:cache:` and inspected via `redis-cli`:

```bash
$ redis-cli KEYS "rl:cache:*"
rl:cache:ca706c1d0178

$ redis-cli HGETALL "rl:cache:ca706c1d0178"
query
What is circuit breaker?
response
Circuit breaker prevents repeated calls to a failing service.

$ redis-cli TTL "rl:cache:ca706c1d0178"
(integer) 288
```

### Guardrails Verification

- **Privacy Guardrail**: `test_privacy_query_not_cached` PASSED. Queries matching sensitive patterns (e.g., `account balance for user 123`) are intercepted by `_is_uncacheable()` and never written to Redis.
- **False-Hit Guardrail**: `test_false_hit_different_years` PASSED. Queries with conflicting 4-digit numbers (e.g., `refund policy for 2024` vs `refund policy for 2026`) are rejected and recorded in `false_hit_log` with reason `date_or_number_mismatch`.

---

## 7. In-memory vs Redis cache comparison

| Metric | In-Memory Cache | Redis Shared Cache | Notes |
|---|---:|---:|---|
| **Availability** | 99.33% | 99.67% | Both backends achieve >= 99% availability |
| **Latency P50** | 278.42 ms | 284.52 ms | Redis adds minor TCP network roundtrip overhead (~9ms) |
| **Latency P95** | 319.43 ms | 315.67 ms | Tail latency is dominated by downstream provider execution |
| **Latency P99** | 319.82 ms | 318.95 ms | Consistent peak latency bound |
| **Cache Hit Rate** | 65.33% | 68.67% | Both backends deliver ~65-69% semantic cache hits |
| **Estimated Cost** | $0.041626 | $0.038874 | Consistent token cost reduction |
| **Circuit Open Count** | 9 | 9 | Circuit trips triggered under flaky chaos scenario |
| **Recovery Time** | 2200.19 ms | 2355.29 ms | Rapid automatic recovery within 2.5s |

> [!NOTE]
> These runs were independently randomized across queries and provider failure seeds, so minor variances in availability, hit rate, circuit opens, and cost cannot be causally attributed to the cache backend alone. The primary advantage of Redis in production is multi-instance state sharing and centralized lifecycle management.

---

## 8. Chaos scenarios

Detailed results from the 3 evaluated chaos test scenarios:

| Scenario | Expected Behavior | Observed Behavior | Pass/Fail |
|---|---|---|:---:|
| `primary_timeout_100` | Primary provider fails 100%. Primary breaker trips to OPEN after 3 failures; all traffic seamlessly routes to backup provider. | 0 static fallbacks. All non-cached requests routed to backup provider (`route="fallback"`). Primary breaker remained OPEN. | **PASS** |
| `primary_flaky_50` | Primary provider fails 50% randomly. Circuit breaker oscillates between CLOSED, OPEN, and HALF_OPEN. | Circuit tripped multiple times, probing recovered during HALF_OPEN canary requests, backup caught failures during outages. | **PASS** |
| `all_healthy` | Both providers operate at baseline health. Maximum traffic handled by primary; zero circuit breaker trips. | High throughput, high cache hit rate, zero circuit breaker openings, minimal latency. | **PASS** |

---

## 9. Failure analysis

Technical review of current architecture limitations and production remediation plans:

### Weakness 1: Redis Semantic Scan Scalability ($O(N)$ Overhead)
- **Problem**: `SharedRedisCache.get()` executes `SCAN prefix*` and fetches `query` per key to compute cosine similarity locally in Python. As cache grows to tens of thousands of entries, this causes excessive Redis roundtrips and CPU saturation.
- **Production Fix**: Implement vector similarity search (VSS) using Redis RediSearch or an external vector database (Qdrant / Milvus) with dense embeddings (`text-embedding-3-small`) to enable sub-millisecond O(log N) ANN vector retrieval.

### Weakness 2: Process-Local Circuit Breaker State
- **Problem**: While cache is shared via Redis, `CircuitBreaker` state machines currently reside in local Python process memory. When a downstream provider suffers an outage, each gateway replica must independently fail 3 times before opening its circuit.
- **Production Fix**: Store circuit breaker counters and state in Redis using atomic Lua scripts or distributed locks/gossip, allowing a failure detected by one replica to immediately protect all other replicas.

### Weakness 3: Heuristic-Only False Hit Detection
- **Problem**: `_looks_like_false_hit()` only detects 4-digit number (year) discrepancies. Subtle semantic differences (e.g. negation, entity names, or version tags like "v1" vs "v2") could yield false hits if token overlap is high.
- **Production Fix**: Integrate a lightweight cross-encoder reranker model (or strict entity/intent constraint filters) to validate semantic equivalence before returning cached responses.

---

## 10. Next steps

1. **Vector-Indexed Semantic Caching**: Replace prefix scan with Redis Vector Similarity Search (RediSearch) with dense embedding index.
2. **Distributed Circuit Breaker State**: Synchronize failure counters and breaker states across all gateway nodes via Redis key expiry and pub/sub notifications.
3. **Adaptive Cost-Budget Routing**: Implement budget tracking in `ReliabilityGateway` to automatically steer non-critical requests to cheaper models when budget reaches 80% capacity.

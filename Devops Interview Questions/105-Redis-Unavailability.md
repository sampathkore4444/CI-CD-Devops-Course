# 105. Redis Becomes Unavailable - Application Impact

## Scenario

At 3:15 AM, the Redis cluster used by the Java/Spring Boot microservices platform becomes completely unavailable. The Redis cluster serves three critical functions for the platform: session storage (all user login sessions are stored in Redis), caching (database query results, API responses, and computed values are cached to reduce database load), and rate limiting (a sliding window rate limiter implemented in Redis prevents API abuse). The platform runs on Kubernetes with 12 microservices, each connecting to the same Redis cluster.

Within minutes of Redis going down, the application experiences cascading failures: all logged-in users are abruptly logged out and must re-authenticate (session loss), the database CPU spikes from 30% to 95% as every request now hits the database directly (cache stampede / thundering herd), and the rate limiter stops working entirely (API abuse becomes possible — no protection against brute force or DDoS). The on-call engineer receives 3 simultaneous alerts: session service errors (all session lookups failing), database CPU critical (95% utilization), and rate limiter health check failed. The Redis cluster was running on a 3-node Kubernetes StatefulSet with 8GB memory per node. You are the DevOps engineer responding to this multi-impact incident.

## Interviewer Question

"Redis has gone completely unavailable. Your application is experiencing simultaneous session loss, cache stampede, and rate limiter failure. Walk me through the immediate response to contain the damage, the investigation to find the root cause, the recovery process, and the long-term prevention strategy."

## What I Should Think About

- Cache stampede (thundering herd) is the most dangerous impact — the database will crash under the combined load of all microservices hitting it simultaneously
- Session loss affects all active users — they must re-authenticate, adding even more load to the already overwhelmed database
- Rate limiter failure means API abuse is possible — need immediate alternative protection (API gateway rate limiting, local rate limiting)
- Database connection pool may be exhausted by the stampede — connection pool exhaustion means requests queue, timeouts increase, and the application becomes unresponsive
- Redis may be recoverable quickly (restart, failover, eviction) — don't make it worse by adding load during recovery
- Cache warming strategy is needed after Redis recovers — cold cache means continued DB pressure until the cache is repopulated
- Long-term fix requires Redis HA (Sentinel or Cluster), not just restart — single Redis instance is a single point of failure
- Kubernetes resource limits and network policies affect Redis behavior — check if Redis was OOM killed or network-isolated
- Sticky sessions vs distributed sessions vs JWT — session architecture matters for resilience
- The cache-aside pattern must include a fallback strategy when the cache is unavailable

## Ideal Answer

**Immediate Response (first 5 minutes):**

1. **Protect the database first — it's the most critical and hardest to recover.** Enable application-level rate limiting as a stopgap to reduce total request volume. Set a hard limit on database connections from the application to prevent the connection pool from being overwhelmed. If the database crashes, the entire platform goes down.
2. **Activate fallback rate limiting:** Switch to API gateway rate limiting or local in-memory rate limiting (per-pod, sliding window) as a temporary replacement for Redis-based rate limiting.
3. **Assess Redis status:** Check if Redis is down entirely or just the primary node (Cluster failover may be possible). Check Kubernetes pod status, OOM kills, and network connectivity.

**Investigation (5-30 minutes):**

1. Check Redis cluster health: nodes, memory, CPU, network connectivity, persistence files
2. Identify the failure mode: OOM kill, network partition, disk full (if persistence enabled), or hardware failure
3. Check Kubernetes events for node failures, resource pressure, or pod eviction
4. Review Redis logs for the root cause of the outage

**Recovery (30-60 minutes):**

1. Restart Redis or promote a replica if primary failed
2. Implement cache warming: preload hot keys from the database in batches
3. Monitor database CPU as cache hits begin reducing load
4. Re-enable Redis-based rate limiting once Redis is restored
5. Verify session service is accepting new logins and existing sessions are恢复

**Prevention:**

1. Deploy Redis with Sentinel for automatic failover (3 sentinels + 1 primary + 2 replicas)
2. Implement cache-aside pattern with fallback to database
3. Add connection retry with exponential backoff and jitter to Redis client
4. Implement local rate limiting as a backup
5. Use JWT for stateless sessions instead of Redis-stored sessions
6. Implement L1 (in-process) + L2 (Redis) caching for hot keys

## Investigation

1. Check Redis cluster pod status and events: `kubectl get pods -l app=redis` and `kubectl describe pods -l app=redis`
2. Review Redis container logs for OOM kills, connection errors, or persistence failures
3. Check if Redis was OOM killed by Kubernetes: `kubectl get pod redis-0 -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'`
4. Verify network connectivity between application pods and Redis: `kubectl exec -it <app-pod> -- redis-cli -h redis-cluster -p 6379 ping`
5. Check Kubernetes events for node failures, resource pressure, or eviction: `kubectl get events --sort-by=.lastTimestamp`
6. Review database connection pool metrics to assess cache stampede impact
7. Check application logs for Redis connection errors and retry attempts
8. Verify rate limiter health endpoint
9. Assess session service error rate and total user impact
10. Check PersistentVolumeClaim status if Redis uses persistence

## Commands

```bash
# Check Redis cluster status
kubectl get pods -l app=redis -n production -o wide
kubectl describe pods -l app=redis -n production

# Check Redis health from application pod
kubectl exec -it <app-pod> -n production -- redis-cli -h redis-cluster -p 6379 ping

# Check Redis memory and connections
kubectl exec -it redis-0 -n production -- redis-cli INFO memory
kubectl exec -it redis-0 -n production -- redis-cli INFO clients
kubectl exec -it redis-0 -n production -- redis-cli INFO stats
kubectl exec -it redis-0 -n production -- redis-cli INFO replication

# Check Redis logs for failure cause
kubectl logs -l app=redis -n production --tail=200 | grep -i "error\|oom\|kill\|fail\|denied"

# Check if Redis was OOM killed
kubectl get pod redis-0 -n production -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'

# Check database CPU (post-stampede)
kubectl top pods -l app=database -n production
kubectl exec -it postgres-0 -n production -- \
  psql -c "SELECT count(*) as active_connections FROM pg_stat_activity WHERE state = 'active';"
kubectl exec -it postgres-0 -n production -- \
  psql -c "SELECT datname, numbackends, xact_commit, xact_rollback FROM pg_stat_database;"

# Check application-level connection pool to Redis
kubectl exec -it <app-pod> -n production -- \
  curl -s localhost:8080/actuator/metrics/redis.connection.active

# Monitor cache hit rate (will be 0% during outage)
kubectl exec -it <app-pod> -n production -- \
  curl -s localhost:8080/actuator/metrics/cache.hit.rate

# Check rate limiter status
kubectl exec -it <app-pod> -n production -- \
  curl -s localhost:8080/actuator/health/rate-limiter

# Enable fallback rate limiting (application config patch)
kubectl patch configmap app-config -n production -p \
  '{"data":{"rate.limiter.type":"LOCAL"}}'
kubectl rollout restart deployment/<app-name> -n production

# Limit database connections to prevent crash
kubectl exec -it postgres-0 -n production -- \
  psql -c "ALTER SYSTEM SET max_connections = 200;"
kubectl rollout restart statefulset/postgres -n production

# Check Redis Sentinel status (if deployed)
kubectl exec -it redis-sentinel-0 -n production -- redis-cli -p 26379 SENTINEL masters

# Warm cache after Redis recovery (batch loading)
kubectl exec -it cache-warmer -n production -- \
  java -jar warmer.jar --keys=hot-keys.txt --batch-size=100 --delay=10ms

# Monitor database CPU after cache warming
watch -n 2 "kubectl top pods -l app=database -n production"

# Check Redis persistence files
kubectl exec -it redis-0 -n production -- ls -la /data/dump.rdb

# Monitor Redis memory after recovery
kubectl exec -it redis-0 -n production -- redis-cli INFO memory | grep used_memory_human
```

## Architecture

```
Normal State:
=============

┌──────────┐    ┌─────────┐    ┌──────────────┐
│  Client  │───>│   API   │───>│   App Pod    │
│          │    │ Gateway │    │  (Spring)    │
└──────────┘    └─────────┘    └──────┬───────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                  │
              ┌─────┴─────┐   ┌──────┴──────┐   ┌──────┴──────┐
              │   Redis    │   │   Redis     │   │   Redis     │
              │  Sessions  │   │   Cache     │   │ Rate Limit  │
              │  (user:xx) │   │  (query:xx) │   │  (ip:xx)    │
              │  Hit: 99%  │   │  Hit: 85%   │   │  Window: 60s│
              └───────────┘   └──────┬──────┘   └─────────────┘
                                     │ (15% miss)
                              ┌──────┴──────┐
                              │  Database    │
                              │  (30% CPU)   │
                              └─────────────┘

During Redis Outage:
====================

┌──────────┐    ┌─────────┐    ┌──────────────┐
│  Client  │───>│   API   │───>│   App Pod    │
│  (login  │    │ Gateway │    │  (Spring)    │
│  again!) │    └─────────┘    └──────┬───────┘
└──────────┘                         │
                              ┌───────┼────────┐
                              │       │        │
                        ┌─────┴┐  ┌──┴───┐  ┌─┴────┐
                        │Redis │  │Redis │  │Redis │
                        │ FAIL │  │ FAIL │  │ FAIL │
                        └──────┘  └──┬───┘  └──────┘
                                     │ (ALL requests miss)
                              ┌──────┴──────┐
                              │  Database    │
                              │  (95% CPU!)  │
                              │  CONNECTIONS │
                              │  EXHAUSTED   │
                              └─────────────┘

Cache Stampede (Thundering Herd):
=================================

Request Rate to DB
    │
    │                        ████ Cache stampede
    │  ████ Normal          ████████
    │  ████ (15%)          ████████████
    │  ████               ████████████████
    │  ████────┐          ████████████████████████
    │  ████    │          ████████████████████████ ← DB overloaded
    │  ████    │ Redis     ████████████████████████   95% CPU
    │  ████    │ Down!     ████████████████████████
    │  ████    │          ████████████████████████
    └────────────────────────────────────────── Time
         Normal    Redis    Stampede    DB Crash?
                  Down     Begins     Risk!

Impact Cascade:
===============
Redis Down → 100% cache miss → DB CPU spike
          → Session lookup fail → User logout → Re-auth storm → More DB load
          → Rate limiter fail → API abuse possible → Even more DB load
          → DB connection pool exhaustion → Application unresponsive
```

## Root Cause

**Immediate:** Redis cluster unavailable — possible causes include OOM kill (memory limit exceeded by Kubernetes), node failure, network partition, or disk full (if RDB/AOF persistence enabled and disk ran out of space).

**Impact Chain:**
1. **Session loss:** Redis stores HTTP sessions. When Redis is down, session lookup fails → all users logged out → re-authentication storm adds massive load to the database
2. **Cache stampede:** Every database query that was previously cached now hits the database directly. With 12 microservices, this multiplies database load by 5-10x (typical cache hit rate is 80-90%)
3. **Rate limiter failure:** Redis-based sliding window rate limiting stops working → no API abuse protection → attackers can flood the system with requests, adding even more load
4. **Database overload:** Combined effect of cache miss storm + re-authentication storm + potential API abuse drives CPU from 30% to 95%

**Contributing Factors:**
- No Redis Sentinel/Cluster for automatic failover — single Redis instance is a single point of failure
- Application has no fallback when Redis is unavailable — no graceful degradation
- No local rate limiting as backup — rate limiting depends entirely on Redis
- Cache-aside pattern without degradation strategy — no fallback when cache is unavailable
- Session management depends entirely on Redis — no JWT fallback for stateless sessions
- No connection retry with backoff — application may give up on Redis too quickly or too slowly

## Immediate Mitigation

1. Enable application-level rate limiting (in-memory, per-pod) immediately to reduce total request volume
2. Set a hard cap on database connections to prevent the pool from being overwhelmed
3. Restart Redis or promote a replica if available
4. Once Redis is back, implement cache warming for the top 100 hot keys in batches
5. Monitor database CPU as cache hit rate gradually recovers
6. Re-enable Redis-based rate limiting once Redis is confirmed stable

## Permanent Fix

1. Deploy Redis with Sentinel (3 sentinels + 1 primary + 2 replicas) for automatic failover within 15 seconds
2. Implement connection retry with exponential backoff and jitter for the Redis client (Spring Redis lettuce configuration)
3. Switch to JWT-based stateless sessions to eliminate the session dependency on Redis entirely
4. Implement local rate limiting (in-memory sliding window per pod) as a backup at the application level
5. Add cache warming job that preloads hot keys after any Redis restart
6. Implement circuit breaker for Redis — if Redis is down, serve degraded responses instead of crashing
7. Set Redis memory limits with 30% headroom and implement LFU eviction policy
8. Use Redis Cluster for horizontal scaling if single-instance memory is insufficient
9. Implement database connection pool limits as a safety net against cache stampede
10. Consider L1 (in-process Caffeine cache) + L2 (Redis) caching for ultra-hot keys

## Monitoring

- **Redis health:** Primary/replica status, memory usage, connection count, replication lag, persistence status
- **Cache hit rate:** Alert if cache hit rate drops below 80% for more than 2 minutes (indicates Redis issues or cache invalidation storm)
- **Database CPU:** Alert at 70% (early warning for cache stampede before it reaches critical levels)
- **Session service:** Alert on session lookup failure rate > 1%
- **Rate limiter:** Alert if rate limiter health check fails
- **Redis memory:** Alert at 80% memory usage before OOM kill occurs
- **Connection retries:** Monitor Redis connection retry count — high values indicate instability
- **Pod eviction events:** Monitor for Kubernetes pod eviction events targeting Redis pods

## Security

- Rate limiter failure during Redis outage exposes the system to API abuse — implement immediate fallback at the API gateway level
- Cached data in Redis must be encrypted at rest — if Redis is compromised, sensitive data (session tokens, query results) is exposed
- Session tokens must be invalidated properly after Redis recovery to prevent session fixation attacks
- Redis AUTH must be configured with strong passwords — unauthenticated Redis is a critical security risk
- Network policies must restrict Redis access to only authorized services — no external access
- Redis persistence files (RDB/AOF) must be encrypted and access-controlled

## Production Considerations

- **High Availability:** Redis Sentinel provides automatic failover in ~15 seconds. Redis Cluster provides sharding and replication. For session storage, Sentinel is essential. For caching, Cluster provides better scaling. For rate limiting, either works.
- **Scalability:** Redis single instance has memory limits (typically 25-50GB practical). Cluster allows horizontal scaling across multiple nodes. Cache key distribution must be monitored for hotspots that could overload a single shard.
- **Cost:** Redis Sentinel requires 3 additional sentinel nodes (lightweight). Redis Cluster requires at least 6 nodes (3 masters + 3 replicas). This cost ($500-2000/month) is justified by the impact of Redis unavailability on the entire platform.
- **Persistence:** RDB snapshots + AOF persistence prevent data loss on restart, but add write latency (typically 1-5ms). For sessions, persistence is optional — losing sessions means users re-login. For rate limiting counters, it depends on the business requirement.
- **Operational:** Redis is single-threaded for commands — one slow command (KEYS, SORT, large SMEMBERS) can block all others. Monitor slowlog. Use connection pooling in the application. Never use KEYS in production — use SCAN.
- **Kubernetes:** Redis must use PersistentVolumeClaims for data durability. Resource limits must be set to prevent OOM kills (set memory limit to 80% of available). Pod disruption budgets ensure minimum availability during node maintenance.

## Senior-Level Answer

"The immediate priority is protecting the database from cache stampede — I would enable application-level rate limiting and cap database connections immediately to prevent the database from crashing under the combined load of cache misses, re-authentication, and potential API abuse. Then I would restart Redis and implement cache warming for hot keys in batches to gradually repopulate the cache without overwhelming the database. Long-term, I would deploy Redis with Sentinel for automatic failover, switch to JWT for stateless sessions to eliminate the session dependency, and implement local rate limiting as a backup. The key lesson is that every Redis-dependent feature must have a degradation strategy — when Redis is unavailable, the application must degrade gracefully, not cascade into database overload."

## Architect-Level Answer

"This incident reveals that Redis is a single point of failure for three critical capabilities: sessions, caching, and rate limiting. The architectural response is three-fold: First, implement Redis HA with Sentinel or Cluster depending on the use case — Sentinel for session storage failover, Cluster if caching requires more than single-instance memory. Second, reduce coupling to Redis by adopting stateless JWT sessions and local rate limiting at the application level. Third, implement a cache-aside pattern with circuit breaker — when Redis is unavailable, the application serves degraded responses (cached data where available, reduced functionality) instead of cascading to the database. The cache warming strategy must be part of the Redis recovery runbook — cold cache after recovery can be as dangerous as the original outage. We should also consider a tiered caching architecture: L1 (in-process Caffeine cache for ultra-hot keys — survives Redis outage), L2 (Redis for shared cache across pods), L3 (database as source of truth). During Redis outage, L1 still provides protection for the most critical queries, buying time for recovery."

## Summary: Impact and Recovery Timeline

```
Time      │ Event                        │ Impact                │ Action
──────────┼──────────────────────────────┼───────────────────────┼──────────────────────
T+0min    │ Redis goes down              │ Sessions fail         │ Alert fires
T+1min    │ Cache misses start           │ DB CPU rising         │ Enable local rate limit
T+3min    │ All users logged out         │ Re-auth storm         │ Cap DB connections
T+5min    │ DB CPU at 95%                │ App degradation       │ Restart Redis
T+10min   │ Redis restarted              │ New connections OK     │ Start cache warming
T+15min   │ Cache warming in progress    │ DB CPU decreasing     │ Monitor metrics
T+30min   │ Cache 80% warm               │ Near-normal           │ Verify session restore
T+60min   │ Full recovery                │ Normal operations     │ Post-incident review
```

## Follow-Up Questions

1. "How would you design a cache warming strategy that doesn't itself overload the database during recovery?"
2. "What is the difference between Redis Sentinel and Redis Cluster, and when would you choose one over the other?"
3. "How do you implement a sliding window rate limiter in Redis, and what happens to the sliding window counter when Redis restarts?"
4. "If you switch to JWT for sessions, how do you handle session invalidation — for example, when a user changes password or an admin forces logout?"
5. "How would you implement a 'degraded mode' for each microservice that depends on Redis — what does each service do when Redis is unavailable?"

## Multi-Layer Caching Strategy

For maximum resilience, implement a multi-layer caching strategy:

**L1 — In-Process Cache (Caffeine/Guava):**
- Size: 1,000-10,000 entries depending on JVM heap
- TTL: 30-60 seconds (shorter than Redis)
- Survives: Redis outage, network partition
- Hit rate: 60-80% of total cache hits for hot keys
- Use case: User profiles, frequently accessed configuration, session data

**L2 — Redis Cache:**
- Size: Depends on Redis memory allocation
- TTL: 5-60 minutes depending on data freshness requirements
- Survives: Application restart, pod rescheduling
- Hit rate: 20-40% of total cache hits
- Use case: Shared data across pods, larger datasets, rate limiting counters

**L3 — Database Query Cache:**
- Size: PostgreSQL shared_buffers or query plan cache
- TTL: Until data changes (invalidation-based)
- Survives: Redis outage, application restart
- Hit rate: Baseline protection when L1 and L2 miss
- Use case: Read-heavy queries, reference data

**Cache-Aside with Fallback:**
```java
// L1 check
Object value = l1Cache.get(key);
if (value != null) return value;

// L2 check (with circuit breaker)
try {
    value = redis.get(key);
    if (value != null) {
        l1Cache.put(key, value);
        return value;
    }
} catch (RedisConnectionException e) {
    // Redis unavailable — skip to L3
}

// L3 — database query (with rate limiting)
value = database.query(key);
l1Cache.put(key, value);
// Don't write to Redis during outage — wait for recovery
return value;
```

## Spring Boot Redis Configuration for Resilience

The Java/Spring Boot application's Redis configuration is critical for resilience:

**Connection Factory Configuration:** Use Lettuce (default) with connection pooling. Configure max-active connections, max-idle connections, and min-idle connections. Set connection timeout to 5 seconds and command timeout to 3 seconds.

**Retry Configuration:** Implement retry with exponential backoff and jitter. Spring Retry provides @Retryable annotation that can be applied to Redis operations. Configure: max attempts = 3, initial interval = 100ms, multiplier = 2, max interval = 5 seconds.

**Circuit Breaker Integration:** Use Resilience4j circuit breaker around Redis operations. Configure: failure rate threshold = 50%, slow call duration = 3 seconds, slow call rate threshold = 80%, wait duration in open state = 30 seconds.

**Cache Configuration:** Use @Cacheable with a custom CacheManager that implements fallback behavior. When Redis is unavailable, the fallback can return null (cache miss) or a default value, rather than throwing an exception.

**Session Configuration:** If using Spring Session with Redis, configure spring.session.store-type=redis and spring.session.timeout=1800. Consider switching to spring.session.store-type=none with JWT for stateless sessions that don't depend on Redis.

## Redis Persistence and Recovery Considerations

Redis persistence configuration affects how quickly the system recovers after a restart:

**RDB Snapshots:** Periodic point-in-time snapshots. If Redis crashes, data since the last snapshot is lost. Recovery is fast — load the RDB file on startup. Configure: `save 900 1` (save if 1 key changed in 900 seconds).

**AOF (Append-Only File):** Every write operation is logged. More durable than RDB but slower recovery (must replay the entire AOF). Configure: `appendfsync everysec` for balance between durability and performance.

**RDB + AOF Combined:** Use both for maximum durability. Redis loads RDB first, then replays AOF. This provides the fastest recovery with the least data loss.

**Kubernetes Persistence:** For Redis on Kubernetes, use PersistentVolumeClaims (PVC) with ReadWriteOnce access mode. Ensure the PVC survives pod restarts by using a StorageClass that supports dynamic provisioning.

**Recovery Time Objective (RTO):** For a payment service, the Redis RTO should be under 30 seconds. This requires: fast persistence (AOF with `appendfsync everysec`), pre-warmed connection pools, and an automated cache warming job that starts immediately after Redis recovers.

## Cache Stampede Prevention Strategies

The cache stampede (thundering herd) is the most dangerous consequence of Redis unavailability. Here are strategies to prevent it:

**Probabilistic Early Expiration:** Instead of all requests hitting the cache simultaneously when a key expires, randomly stagger expiration checks. This distributes cache misses over time rather than concentrating them in a single instant.

**Request Coalescing:** When multiple requests for the same uncached key arrive simultaneously, only one request actually queries the database. The others wait for the result and share it. This prevents 100 concurrent requests from generating 100 database queries.

**Cache Warming on Startup:** When the application starts or Redis recovers, proactively load hot keys into the cache before accepting traffic. This reduces the cold-start cache miss rate.

**Circuit Breaker with Fallback:** When Redis is unavailable, the circuit breaker opens and the application switches to a degraded mode — serving cached data where available, accepting transactions into a queue, and limiting database queries to prevent overload.

**Tiered Caching (L1 + L2):** Use in-process caching (Caffeine/Guava) as L1 for ultra-hot keys, and Redis as L2 for shared cache. During Redis outage, L1 still protects the most critical queries. L1 cache hit rate is typically 60-80% of total cache hits.

**Database Query Caching:** For read-heavy queries, implement query result caching at the database level (PostgreSQL pg_prewarm, query plan caching). This provides a third layer of protection when both L1 and L2 caches are unavailable.

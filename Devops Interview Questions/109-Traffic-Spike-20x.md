# 109. Unexpected 20x Traffic Spike - System Under Load

## Scenario

At 2:00 PM on a Friday, traffic to your e-commerce platform suddenly increases 20x. A viral social media post by a major influencer mentioned your product, driving massive traffic. The application starts timing out. Database connections are maxed out at 500/500. CDN cache miss rate jumped from 5% to 70% because the traffic is hitting dynamic API endpoints, not static assets. Auto-scaling hasn't kicked in yet — the HPA is configured but the Cluster Autoscaler needs 5 minutes to provision new nodes. Customers are seeing 503 errors. The on-call engineer is getting flooded with alerts. How do you handle the immediate crisis and build long-term resilience?

## Interviewer Question

"You're hit with a 20x traffic spike that's overwhelming your platform. Walk me through your immediate triage, the decisions you make in the first 15 minutes, and your long-term strategy to prevent this from causing an outage again."

## What I Should Think About

- Immediate triage: what to check first — error rates, latency, database connections, CDN hit rate
- CDN cache miss: why CDN isn't absorbing the spike — dynamic content, cache-busting headers, query parameters
- Auto-scaling lag: why HPA and Cluster Autoscaler didn't scale fast enough
- Database protection: connection limiting, query queuing, read replicas, connection pooling
- Rate limiting: API gateway level, per-client, per-endpoint
- Load shedding: gracefully rejecting excess requests with proper error responses
- Caching strategy: Redis, in-memory caching, CDN tuning for dynamic content
- Auto-scaling optimization: warm pools, target tracking, step scaling,预热 (pre-warming)
- Content optimization: static vs dynamic, edge compute (CloudFront Functions, Lambda@Edge)
- Long-term: load testing, capacity planning, burstable infrastructure
- Kubernetes: HPA configuration, Cluster Autoscaler, Karpenter
- AWS: Auto Scaling warm pools, Spot instances for burst capacity

## Ideal Answer

**First 5 minutes — Triage and Stabilize:**
Check the dashboard: error rates, latency, database connections, CDN hit rate. Identify the bottleneck — in this case, it's database connections maxed out at 500/500. Immediately increase database connection limits if possible. Enable rate limiting at the API gateway to shed excess traffic. Check if CDN is absorbing static traffic — if not, tune cache headers for dynamic responses.

**Next 10 minutes — Protect the Database:**
Database is the critical bottleneck. Enable connection pooling (PgBouncer, ProxySQL) if not already in place. Deploy read replicas to distribute read traffic. Implement query queuing to prevent thundering herd on database. Enable circuit breakers on non-critical database calls.

**Next 30 minutes — Scale and Optimize:**
Force HPA to scale by temporarily lowering utilization targets. Pre-warm Auto Scaling groups by manually increasing desired capacity. Enable CDN caching for dynamic responses where possible (product pages, search results). Deploy rate limiting per client to prevent abuse.

**Long-term — Build Resilience:**
Implement a multi-layer caching strategy: CDN for static, Redis for hot data, application-level caching for computed results. Optimize auto-scaling: use warm pools, pre-provision capacity for expected traffic, configure Karpenter for faster node provisioning. Load test with 30x expected traffic to validate capacity. Implement graceful degradation — serve cached responses when database is overloaded.

## Architecture

```
Traffic Spike Defense Architecture
====================================

                    Viral Traffic (20x)
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│                    CDN Layer                         │
│  CloudFront / Cloudflare                            │
│  - Static: 95% hit rate; Dynamic: cache headers,   │
│    TTL, stale-while-revalidate                      │
│  - Strip query params to avoid cache-busting        │
└──────────────────────────┬──────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────┐
│              API Gateway / Load Balancer             │
│  - Rate limiting: 1000 req/s per client             │
│  - Request throttling, circuit breakers, WAF rules  │
└──────────────────────────┬──────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────┐
│              Application Layer (Kubernetes)          │
│  HPA: 60% CPU target, min=10, max=200 pods         │
│  Cluster Autoscaler / Karpenter (30s provisioning)  │
│  Warm pool: 5 pre-provisioned nodes, Spot for burst │
│  Pod Pools: API / Worker / Cache all auto-scaling   │
└──────────────────────────┬──────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────┐
│              Data Layer                              │
│  PgBouncer: connection pooling (500 → 2000)         │
│  Read Replicas (3): distribute read traffic         │
│  Redis Cache: hot data, session, rate-limit counters│
└─────────────────────────────────────────────────────┘
```

## Investigation

1. **Check Error Dashboard**: What's the error rate? 503s indicate the system is shedding load — this is actually good (better than timeouts)
2. **Check Database Connections**: Database at 500/500 connections — this is the primary bottleneck
3. **Check CDN Cache Hit Rate**: 70% miss rate — dynamic content isn't being cached, CDN isn't absorbing the spike
4. **Check Auto-scaling Status**: HPA wants to scale but Cluster Autoscaler needs 5 minutes to provision nodes
5. **Check Application Logs**: What errors are being logged? Timeout errors? Connection refused?
6. **Check Request Latency**: p99 latency probably > 10s — indicate database connection wait time
7. **Check Per-Endpoint Breakdown**: Which endpoints are getting hit hardest? Product pages? Search? Checkout?
8. **Check Infrastructure Metrics**: CPU, memory, network — identify resource exhaustion points
9. **Check CDN Configuration**: Cache headers, TTL values, query parameter handling
10. **Check Rate Limiting**: Is rate limiting enabled? What are the current limits?

## Commands

```bash
# Check current error rates
kubectl get pods -n production | grep -E "CrashLoop|Error|OOMKilled"

# Check database connection pool
kubectl exec -it postgres-pool-0 -- psql -c "SELECT count(*) FROM pg_stat_activity;"

# Check HPA status
kubectl get hpa -n production
kubectl describe hpa order-service-hpa -n production

# Check Cluster Autoscaler status
kubectl get nodes
kubectl describe node <node-name> | grep -A 5 "Conditions:"

# Check CDN cache hit rate (CloudFront)
aws cloudfront get-metrics --distribution-id E1234567890 --start-time 2024-01-01T14:00:00Z --end-time 2024-01-01T14:30:00Z

# Force HPA to scale by lowering targets
kubectl patch hpa order-service-hpa -n production -p '{"spec":{"metrics":[{"type":"Resource","resource":{"name":"cpu","target":{"type":"Utilization","averageUtilization":30}}}]}}'

# Manually scale deployment
kubectl scale deployment order-service --replicas=50 -n production

# Increase database connection limits
kubectl exec -it postgres-0 -- psql -c "ALTER SYSTEM SET max_connections = 2000;"
kubectl exec -it postgres-0 -- psql -c "SELECT pg_reload_conf();"

# Enable rate limiting at API gateway
cat > rate-limit-policy.yaml <<EOF
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: ClientTrafficPolicy
metadata:
  name: rate-limit
  namespace: production
spec:
  rateLimit:
    global:
      rules:
        - clientSelectors:
            - headers:
                - name: "x-api-key"
                  type: "Distinct"
          limit:
            requests: 1000
            unit: "Second"
EOF

# Check Redis cache hit rate
kubectl exec -it redis-0 -- redis-cli INFO stats | grep keyspace

# Pre-warm Auto Scaling group (AWS)
aws autoscaling set-desired-capacity --auto-scaling-group-name k8s-worker-pool --desired-capacity 20

# Check Karpenter provisioner status
kubectl get provisioners
kubectl describe provisioner default

# Enable warm pool for faster scaling
aws autoscaling put-warm-pool --auto-scaling-group-name k8s-worker-pool --min-size 5 --max-size 10

# Monitor real-time traffic
kubectl top pods -n production
kubectl logs -f -l app=order-service -n production --tail=100
```

## Root Cause

1. **Insufficient CDN Caching**: Dynamic content not cached due to missing cache headers, query parameters causing cache-busting
2. **Auto-scaling Lag**: HPA and Cluster Autoscaler take 5+ minutes to provision new capacity
3. **Database Connection Exhaustion**: Fixed connection pool size (500) insufficient for 20x traffic
4. **No Rate Limiting**: All traffic accepted without throttling, overwhelming downstream services
5. **No Load Shedding**: System tries to serve all requests instead of gracefully rejecting excess
6. **Missing Caching Layers**: No Redis or application-level caching for frequently accessed data
7. **Under-provisioned Infrastructure**: Base capacity designed for 2x normal traffic, not 20x bursts

## Immediate Mitigation

1. **Enable Rate Limiting**: Configure API gateway to limit per-client requests to 1000/s
2. **Increase Database Connections**: Raise max_connections from 500 to 2000, deploy connection pooling
3. **Force Scale**: Manually scale deployments and node pools to handle immediate traffic
4. **Enable CDN Caching**: Add cache headers for dynamic responses, enable stale-while-revalidate
5. **Deploy Read Replicas**: Spin up 3 read replicas to distribute database read traffic
6. **Implement Circuit Breakers**: Protect database from non-critical requests
7. **Communicate**: Status page update, social media response, customer communication

## Permanent Fix

1. **Multi-layer Caching**: CDN (static) → Redis (hot data) → Application cache (computed) → Database (source of truth)
2. **Pre-provisioned Capacity**: Warm pools, pre-scaled node groups, Karpenter for fast provisioning
3. **Database Optimization**: Connection pooling (PgBouncer), read replicas, query optimization, connection limiting
4. **Auto-scaling Tuning**: Target tracking at 60% CPU, aggressive scale-up policies, predictive scaling
5. **Load Testing**: Monthly load tests at 30x expected traffic to validate capacity
6. **Graceful Degradation**: Serve cached responses when database is overloaded, queue non-critical operations
7. **Capacity Planning**: Review and update capacity plans quarterly based on growth projections

## Monitoring

- **Error Rate Dashboard**: Real-time 503 rate, timeout rate, connection refused errors
- **Database Metrics**: Connection pool utilization, query latency, queue depth, replication lag
- **CDN Metrics**: Cache hit rate, origin requests, error rate by edge location
- **Auto-scaling Metrics**: Desired vs actual capacity, scaling events, node provisioning time
- **Application Metrics**: Request rate, latency percentiles, throughput per endpoint
- **Infrastructure Metrics**: CPU, memory, network utilization across all nodes
- **Business Metrics**: Orders/minute, cart abandonment rate, revenue impact

## Security

- **Rate Limiting**: Prevent abuse and DDoS by limiting per-client request rates
- **Bot Detection**: Identify and block bot traffic that amplifies spikes
- **WAF Rules**: Deploy rules to block known attack patterns during traffic spikes
- **Authentication**: Ensure rate limits apply per authenticated user, not just per IP
- **Data Protection**: During overload, prioritize protecting sensitive operations (payments, PII)
- **DDoS Protection**: CloudFront Shield, Cloudflare, or AWS Shield for volumetric attacks
- **Incident Response**: Communication plan for customers during extended outages

## Production Considerations

- **HA**: Traffic spikes don't respect business hours — auto-scaling must work 24/7
- **Scalability**: Design for 30x normal traffic, not 2x — burst capacity is unpredictable
- **Reliability**: Graceful degradation beats total failure — serve cached responses when overloaded
- **Cost**: Pre-provisioned capacity costs money during normal traffic — balance cost vs risk
- **Compliance**: Payment processing must maintain SLAs during traffic spikes — prioritize these requests
- **Operational**: Runbooks for traffic spike scenarios, automated scaling, and manual intervention procedures
- **CDN Strategy**: Cache everything possible at the edge — product pages, search results, recommendations
- **Database Strategy**: Read replicas, connection pooling, query optimization — database is always the bottleneck

## Senior-Level Answer

I'd triage in three phases: first, stabilize by enabling rate limiting and increasing database connections; second, scale by forcing HPA and pre-warming node pools; third, optimize by enabling CDN caching and deploying read replicas. Long-term, I'd implement a multi-layer caching strategy (CDN → Redis → app cache), optimize auto-scaling with warm pools and Karpenter, and run monthly load tests at 30x expected traffic. The key insight is that database connections are always the bottleneck — I'd deploy connection pooling and read replicas as standard infrastructure.

## Architect-Level Answer

A 20x traffic spike reveals infrastructure gaps that only appear under extreme load. I'd architect for burst capacity: multi-layer caching (CDN, Redis, application), connection pooling with read replicas, and auto-scaling with pre-provisioned warm pools. The strategic insight is that traffic spikes are unpredictable — you can't predict when they'll happen, but you can design systems that absorb them gracefully. Key design decisions: serve cached responses during overload (graceful degradation), prioritize critical business transactions (payments over recommendations), and implement predictive scaling based on historical patterns. The goal is not just surviving the spike — it's maintaining business continuity.

## Kubernetes HPA and Cluster Autoscaler Deep Dive

```
HPA Configuration for Traffic Spikes

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 10
  maxReplicas: 200
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 10
          periodSeconds: 30
        - type: Percent
          value: 100
          periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

Key HPA tuning points:
- `scaleUp.stabilizationWindowSeconds: 30` — react within 30 seconds to traffic increases; `selectPolicy: Max` — use the most aggressive scale-up policy
- `scaleUp.policies` — allow 10 pods OR 100% increase every 30-60 seconds; `minReplicas: 10` — keep warm baseline capacity; target 60% CPU — headroom for bursts

Cluster Autoscaler vs Karpenter:
- Cluster Autoscaler: Provisions new nodes in 3-5 minutes, uses ASG, conservative scaling
- Karpenter: Provisions new nodes in 30-60 seconds, uses EC2 directly, more aggressive, better for burst

## CDN Optimization for Dynamic Content

Cache-bearing strategy for dynamic endpoints: `Cache-Control: public, max-age=60, stale-while-revalidate=300` — serve cached content for 60s, then serve stale while revalidating in background for up to 300s.

- Product pages: Cache 5 minutes with stale-while-revalidate (mostly static product data)
- Search results: Cache 30 seconds (personalized but cacheable)
- Cart/checkout: No-cache (must be real-time)
- User profiles: Cache 60 seconds (semi-static)

CloudFront Functions apply per-URI cache rules at the edge:

```javascript
function handler(event) {
    var request = event.request;
    var uri = request.uri;
    if (uri.startsWith('/products/')) {
        request.headers['cache-control'] = { value: 'public, max-age=300, stale-while-revalidate=600' };
    } else if (uri.startsWith('/search')) {
        request.headers['cache-control'] = { value: 'public, max-age=30, stale-while-revalidate=60' };
    } else if (uri.startsWith('/checkout')) {
        request.headers['cache-control'] = { value: 'no-store, no-cache' };
    }
    return request;
}
```

Also strip tracking/analytics query parameters from cache keys — they cause cache-busting and destroy hit rates.

## Emergency Response Checklist

```
Traffic Spike Response (First 15 Minutes)
===========================================

0-2 min — TRIAGE
  Check error rate dashboard (503s rising?), DB connection pool (maxed?),
  CDN cache hit rate (dropping?), auto-scaling status (pods/nodes scaling?)

2-5 min — STABILIZE
  Enable rate limiting at API gateway (1000 req/s per client),
  increase DB max_connections (500 → 2000), deploy PgBouncer,
  force scale: kubectl scale deployment order-service --replicas=50

5-10 min — SCALE
  Pre-warm ASG (set-desired-capacity), enable CDN caching for dynamic
  responses, deploy read replicas, enable circuit breakers

10-15 min — OPTIMIZE
  Enable stale-while-revalidate, deploy Redis caching for hot data,
  implement DB query queuing, verify error rate is decreasing
```

## Follow-Up Questions

1. How do you distinguish between a legitimate traffic spike and a DDoS attack?
2. What's the optimal auto-scaling strategy for unpredictable traffic patterns — target tracking vs step scaling vs predictive?
3. How do you handle database connection pooling at scale — PgBouncer vs ProxySQL vs application-level pooling?
4. How do you implement graceful degradation when the database is overloaded — what do you sacrifice first?
5. How do you load test for 20x traffic spikes without actually causing an outage?

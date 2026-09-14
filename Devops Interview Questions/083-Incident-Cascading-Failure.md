# 83. Cascading Failure Across Microservices

## Scenario

At 11:00 AM, Service A (Product Catalog) starts returning HTTP 500 errors due to a database connection leak. Service B (Search) depends on Service A and starts timing out after 30 seconds. Service C (Recommendations) depends on Service B and starts queuing requests. Service D (Shopping Cart) depends on A, B, and C and starts failing. Service E (Checkout) depends on all services and becomes completely unavailable. Within 10 minutes, 6 services are down. The root cause is a single database connection leak in Service A that was introduced in a deployment 2 hours ago. The platform serves 100,000 concurrent users.

## Interviewer Question

"A database connection leak in Service A cascaded to take down 6 services in 10 minutes. How do you prevent cascading failures in a microservices architecture?"

## What I Should Think About

- Cascading failures are the #1 risk in microservices
- Need circuit breakers, bulkheads, timeouts, retries
- Service dependency mapping and visualization
- Health checks and graceful degradation
- Load shedding and rate limiting
- Chaos engineering to test resilience
- Incident response for cascading failures

## Ideal Answer

**1. Prevention: Resilience Patterns**
- **Circuit Breaker**: Stop calling failing service after threshold
- **Bulkhead**: Isolate service dependencies (thread pools per dependency)
- **Timeout**: Set aggressive timeouts on all outbound calls
- **Retry with Backoff**: Exponential backoff with jitter
- **Rate Limiting**: Protect services from overload
- **Load Shedding**: Drop low-priority requests under load

**2. Detection: Monitoring**
- Service dependency map (real-time)
- Error rate per service (alert on > 1%)
- Latency per service (alert on p95 > 1s)
- Connection pool metrics per service
- Queue depth metrics

**3. Response: Incident Management**
- Identify root cause service quickly
- Isolate failing service (feature flag, traffic shaping)
- Graceful degradation (return cached data, simplified responses)
- Scale healthy services to handle rerouted traffic

**4. Recovery: Post-Incident**
- Fix root cause (connection leak)
- Add missing circuit breakers
- Update service dependency map
- Run chaos engineering tests

## Architecture

```
┌─────────────────────────────────────────────────┐
│        CASCADING FAILURE SEQUENCE                 │
│                                                   │
│  11:00 AM: Service A fails (DB connection leak)  │
│            ┌──────────┐                           │
│            │Service A │ ← Root cause              │
│            │ (500s)   │                           │
│            └────┬─────┘                           │
│                 │                                  │
│  11:02 AM: Service B times out                   │
│            ┌──────────┐                           │
│            │Service B │ ← Depends on A            │
│            │ (timeouts)│                          │
│            └────┬─────┘                           │
│                 │                                  │
│  11:05 AM: Services C, D fail                    │
│            ┌──────────┐  ┌──────────┐            │
│            │Service C │  │Service D │             │
│            │ (queued) │  │ (500s)   │             │
│            └────┬─────┘  └────┬─────┘            │
│                 │              │                   │
│  11:10 AM: Service E (Checkout) completely down  │
│            ┌──────────┐                           │
│            │Service E │ ← All dependencies failed│
│            │ (DOWN)   │                           │
│            └──────────┘                           │
│                                                   │
│  WITH CIRCUIT BREAKERS:                           │
│            ┌──────────┐                           │
│            │Service A │ ← Fails, circuit opens   │
│            └────┬─────┘                           │
│                 │ (blocked by circuit breaker)    │
│            ┌────┴─────┐  ┌──────────┐           │
│            │Service B │  │Service C │ ← Healthy │
│            │ (degraded)│  │ (healthy)│           │
│            └──────────┘  └──────────┘           │
└─────────────────────────────────────────────────┘
```

## Investigation

1. **Map service dependencies:**
   ```bash
   # Check service dependency map (if using Istio)
   istioctl proxy-config clusters service-a -n production
   
   # Check service mesh topology
   kubectl get virtualservices -n production -o json | jq '.items[] | {name: .metadata.name, hosts: .spec.hosts, routes: .spec.http[].route[].destination.host}'
   
   # Check service health
   for svc in service-a service-b service-c service-d service-e; do
     echo "$svc: $(curl -s -o /dev/null -w "%{http_code}" http://$svc.production.svc.cluster.local/health)"
   done
   ```

2. **Identify connection leak:**
   ```sql
   -- Check active connections per service
   SELECT application_name, count(*), state
   FROM pg_stat_activity
   GROUP BY application_name, state;
   
   -- Check for idle connections
   SELECT application_name, count(*)
   FROM pg_stat_activity
   WHERE state = 'idle'
   GROUP BY application_name;
   
   -- Check connection age
   SELECT application_name, count(*), max(now() - state_change)
   FROM pg_stat_activity
   WHERE state = 'idle'
   GROUP BY application_name;
   ```

3. **Check circuit breaker status:**
   ```bash
   # Check Envoy circuit breaker stats (if using Istio)
   kubectl exec -it service-b-pod -n production -c istio-proxy -- \
     curl -s localhost:15000/stats | grep circuit
   
   # Check for open circuits
   kubectl exec -it service-b-pod -n production -c istio-proxy -- \
     curl -s localhost:15000/stats | grep "circuit_breakers.*default.*open"
   ```

## Commands

```bash
# 1. Implement circuit breaker in Istio
cat > destination-rule.yml << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: service-a
  namespace: production
spec:
  host: service-a
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 5s
      http:
        h2UpgradePolicy: DEFAULT
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
        maxRetries: 3
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 30
EOF

kubectl apply -f destination-rule.yml

# 2. Add timeout to service calls
cat > virtual-service.yml << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: service-b
  namespace: production
spec:
  hosts:
    - service-b
  http:
    - timeout: 5s
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: 5xx,reset,connect-failure
      route:
        - destination:
            host: service-b
EOF

kubectl apply -f virtual-service.yml

# 3. Implement rate limiting
cat > quota.yml << 'EOF'
apiVersion: config.istio.io/v1alpha2
kind: QuotaSpecBinding
metadata:
  name: service-a-ratelimit
  namespace: production
spec:
  quotaSpecRef:
    name: service-a-quota
  bindings:
    - kind: VirtualService
      name: service-a
      namespace: production
      default:
        quota: service-a-requests
        charge: 1
EOF

# 4. Fix the connection leak in Service A
# In application code (Python example)
cat > db.py << 'EOF'
import psycopg2
from contextlib import contextmanager

@contextmanager
def get_db_connection():
    conn = psycopg2.connect(DATABASE_URL)
    try:
        yield conn
    finally:
        conn.close()  # Always close connection

# Usage
with get_db_connection() as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM products")
        results = cur.fetchall()
# Connection automatically closed
EOF

# 5. Scale healthy services
kubectl scale deployment/service-b --replicas=5 -n production
kubectl scale deployment/service-c --replicas=5 -n production

# 6. Enable load shedding
cat > envoy-filter.yml << 'EOF'
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: load-shedding
  namespace: production
spec:
  workloadSelector:
    labels:
      app: service-a
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.local_ratelimit
          typed_config:
            "@type": type.googleapis.com/udpa.type.v1.TypedStruct
            type_url: type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
            value:
              stat_prefix: http_local_rate_limiter
              token_bucket:
                max_tokens: 1000
                tokens_per_fill: 100
                fill_interval: 1s
              filter_enabled:
                runtime_key: local_rate_limit_enabled
                default_value:
                  numerator: 100
                  denominator: HUNDRED
EOF
```

## Root Cause

| Root Cause | Evidence | Prevention |
|---|---|---|
| Database connection leak in Service A | Idle connections accumulating | Connection pooling, context managers |
| No circuit breakers | Service B calling failing Service A | Implement Istio circuit breakers |
| No timeouts on outbound calls | Service B waiting 30s for Service A | Set aggressive timeouts (5s) |
| No bulkhead isolation | All services sharing connection pool | Isolate dependencies with bulkheads |
| No load shedding | Services accepting unlimited requests | Implement rate limiting |
| No chaos testing | Resilience patterns not validated | Regular chaos engineering |

## Immediate Mitigation

1. **Fix connection leak** — close connections properly
2. **Restart Service A** — clear leaked connections
3. **Add circuit breakers** — prevent cascade to other services
4. **Scale healthy services** — handle rerouted traffic

## Permanent Fix

1. Implement circuit breakers on all service-to-service calls
2. Set aggressive timeouts (5s) on all outbound calls
3. Implement bulkhead isolation for critical dependencies
4. Add load shedding and rate limiting
5. Run chaos engineering tests regularly
6. Create service dependency map and monitor

## Monitoring

- **Service dependency map** — real-time visualization
- **Circuit breaker status** — alert on open circuits
- **Connection pool metrics** — alert on exhaustion
- **Error rate per service** — alert on > 1%
- **Latency per service** — alert on p95 > 1s

## Security

- Circuit breakers should not expose internal service details
- Rate limiting should be per-client to prevent abuse
- Load shedding should prioritize critical services
- Service mesh should encrypt all inter-service communication

## Production Considerations

- **100,000 concurrent users** — cascading failures affect massive traffic
- Circuit breakers add latency (~1ms per call)
- Consider using service mesh (Istio/Linkerd) for automatic resilience
- Cost: Istio requires additional CPU/memory on each pod
- Chaos engineering should be done in staging first

## Senior-Level Answer

"I'd implement a layered defense strategy: (1) Prevention — circuit breakers, bulkheads, timeouts on all service calls, (2) Detection — real-time service dependency map with health monitoring, (3) Response — automated isolation of failing services, graceful degradation, (4) Recovery — fix root cause, update resilience patterns, run chaos tests. The key insight is that cascading failures are the #1 risk in microservices — resilience must be built in, not added after incidents."

## Architect-Level Answer

"At the architectural level, I'd establish: (1) Resilience Standards — all services must implement circuit breakers, bulkheads, and timeouts, (2) Service Mesh Strategy — use Istio/Linkerd for automatic resilience without code changes, (3) Chaos Engineering Program — regular failure injection to validate resilience, (4) Service Dependency Governance — documented and monitored dependency map, (5) Incident Response — automated isolation and graceful degradation as standard patterns."

## Follow-Up Questions

1. "How do you implement circuit breakers in a non-service-mesh environment?"
2. "What's the difference between circuit breakers and bulkheads?"
3. "How do you implement graceful degradation for a payment service?"
4. "How would you design a chaos engineering program for microservices?"
5. "How do you handle cascading failures in event-driven architectures (Kafka)?"

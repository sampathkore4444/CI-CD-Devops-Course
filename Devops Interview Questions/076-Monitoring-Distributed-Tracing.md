# 76. Distributed Tracing Across Microservices

## Scenario

A user reports that checkout is slow on your e-commerce platform. The request flows through: API Gateway → Auth Service → Order Service → Payment Service → Notification Service. Each service is written by a different team. When latency increases, each team blames the others. You have Jaeger installed but nobody uses it. Traces are sampled at 1% so most slow requests aren't captured. You need to implement distributed tracing with OpenTelemetry and make it useful for debugging.

## Interviewer Question

"How do you implement distributed tracing across 5 microservices so that when a slow request comes in, you can immediately identify which service is the bottleneck?"

## What I Should Think About

- OpenTelemetry as the instrumentation standard
- Trace context propagation (W3C Trace Context headers)
- Sampling strategies: head-based vs. tail-based
- Service mesh integration (Istio/Linkerd for automatic tracing)
- Trace-to-log and trace-to-metrics correlation
- Sampling rate optimization (too low = no data, too high = cost)
- Span naming conventions and attributes
- Jaeger/Tempo configuration for storage and querying

## Ideal Answer

**1. Instrument with OpenTelemetry**
- Add OpenTelemetry SDK to each service (auto-instrumentation)
- Use OTLP exporter to send traces to collector
- Propagate trace context via W3C Trace Context headers

**2. Configure Sampling**
- Head-based sampling at 10% for normal traffic
- Always sample errors and slow requests (> 1s)
- Use tail-based sampling at collector for intelligent sampling

**3. Standardize Span Attributes**
- `service.name`, `http.method`, `http.url`, `http.status_code`
- `db.system`, `db.statement`, `db.duration`
- Custom attributes: `user.id`, `order.id`, `payment.gateway`

**4. Correlate with Logs and Metrics**
- Inject trace ID into log messages
- Create RED metrics from trace data
- Use trace ID to jump from metric → log → trace

**5. Set Up Jaeger**
- Deploy Jaeger with Elasticsearch/Cassandra backend
- Configure retention policy (7 days traces)
- Set up alerts on trace latency anomalies

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│            DISTRIBUTED TRACING ARCHITECTURE              │
│                                                          │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐           │
│  │ API       │──►│ Auth     │──►│ Order    │           │
│  │ Gateway   │   │ Service  │   │ Service  │           │
│  │ [span 1]  │   │ [span 2] │   │ [span 3] │           │
│  └──────────┘   └──────────┘   └────┬─────┘           │
│                                      │                   │
│                    ┌─────────────────┼──────────┐       │
│                    ▼                 ▼          ▼       │
│              ┌──────────┐    ┌──────────┐  ┌──────┐   │
│              │ Payment  │    │ Notif.   │  │ DB   │   │
│              │ Service  │    │ Service  │  │      │   │
│              │ [span 4] │    │ [span 5] │  │[spn6]│   │
│              └──────────┘    └──────────┘  └──────┘   │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │              OPENTELEMETRY COLLECTOR              │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐      │    │
│  │  │ Receives │  │ Samples  │  │ Exports  │      │    │
│  │  │ OTLP     │  │ (Tail)   │  │ to Jaeger│      │    │
│  │  └──────────┘  └──────────┘  └──────────┘      │    │
│  └─────────────────────┬───────────────────────────┘    │
│                        ▼                                 │
│              ┌──────────────────┐                        │
│              │     Jaeger       │                        │
│              │  (Query & UI)    │                        │
│              └──────────────────┘                        │
└─────────────────────────────────────────────────────────┘
```

## Investigation

1. **Check trace sampling rate:**
   ```bash
   # Check Jaeger sampling configuration
   kubectl get configmap jaeger-config -n observability -o yaml | grep -A5 "sampling"
   
   # Check how many traces are being collected
   curl -s http://jaeger:16686/api/traces?service=order-service&limit=10 | jq '.data | length'
   ```

2. **Verify trace context propagation:**
   ```bash
   # Check if services are sending traces
   curl -s http://jaeger:16686/api/services | jq '.data'
   
   # Check trace in Jaeger UI
   # Navigate to: http://jaeger:16686 → Order Service → Find Traces
   ```

3. **Check OpenTelemetry collector metrics:**
   ```bash
   # Check collector received/exported spans
   curl -s http://otel-collector:8888/metrics | grep "otelcol_receiver_accepted_spans_total"
   curl -s http://otel-collector:8888/metrics | grep "otelcol_exporter_sent_spans_total"
   ```

4. **Query slow traces:**
   ```
   # In Jaeger UI:
   Service: order-service
   Operation: checkout
   Duration: > 1000ms
   Tags: http.status_code=200
   ```

## Commands

```bash
# 1. Deploy OpenTelemetry Collector
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: observability
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      batch:
        timeout: 5s
        send_batch_size: 1000
      tail_sampling:
        decision_wait: 10s
        policies:
          - name: slow-requests
            type: latency
            latency:
              threshold_ms: 1000
          - name: errors
            type: status_code
            status_code:
              status_codes: [ERROR]
          - name: normal-traffic
            type: probabilistic
            probabilistic:
              sampling_percentage: 10
    exporters:
      jaeger:
        endpoint: jaeger-collector:14250
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch, tail_sampling]
          exporters: [jaeger]
EOF

# 2. Add OpenTelemetry to a Python service
cat > requirements.txt << 'EOF'
opentelemetry-api==1.22.0
opentelemetry-sdk==1.22.0
opentelemetry-exporter-otlp==1.22.0
opentelemetry-instrumentation-flask==0.43b0
opentelemetry-instrumentation-requests==0.43b0
EOF

cat > tracing.py << 'EOF'
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import SERVICE_NAME, Resource
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

resource = Resource(attributes={SERVICE_NAME: "order-service"})
provider = TracerProvider(resource=resource)
processor = BatchSpanProcessor(OTLPSpanExporter(endpoint="otel-collector:4317"))
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

# Auto-instrument Flask
FlaskInstrumentor().instrument()
RequestsInstrumentor().instrument()

tracer = trace.get_tracer(__name__)

@app.route("/checkout")
def checkout():
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        span.set_attribute("user.id", user_id)
        # ... business logic
        result = process_payment(order_id)
        span.set_attribute("payment.status", result.status)
        return result
EOF

# 3. Add trace ID to logs
import logging
from opentelemetry import trace

class TraceContextFilter(logging.Filter):
    def filter(self, record):
        span = trace.get_current_span()
        record.trace_id = format(span.get_span_context().trace_id, '032x')
        record.span_id = format(span.get_span_context().span_id, '016x')
        return True

handler.addFilter(TraceContextFilter())
formatter = logging.Formatter(
    '%(asctime)s [%(trace_id)s] [%(span_id)s] %(levelname)s %(message)s'
)

# 4. Query traces via Jaeger API
curl -s "http://jaeger:16686/api/traces?service=order-service&minDuration=1000ms&limit=5" | jq '.data[].spans[].operationName'
```

## Root Cause

| Root Cause | Elimination |
|---|---|
| No distributed tracing implemented | Deploy OpenTelemetry + Jaeger |
| Sampling at 1% misses slow requests | Use tail-based sampling for slow/erroneous requests |
| No trace context propagation | Implement W3C Trace Context headers |
| No correlation between traces and logs | Inject trace ID into log messages |
| Each team blames others | Traces provide objective evidence of bottlenecks |
| No span naming conventions | Standardize span attributes across services |

## Immediate Mitigation

1. **Increase sampling rate to 10%** for all services
2. **Always sample errors and slow requests** (latency > 1s)
3. **Check Jaeger** for existing traces of slow requests
4. **Deploy OTel Collector** with tail-based sampling

## Permanent Fix

1. Implement OpenTelemetry auto-instrumentation in all services
2. Configure tail-based sampling at collector level
3. Standardize span naming and attributes across all services
4. Correlate traces with logs (inject trace ID into log messages)
5. Create dashboards showing latency by service from trace data
6. Set up alerts on trace latency anomalies

## Monitoring

- **Trace sampling rate** — ensure 100% of slow/error traces captured
- **Trace completeness** — verify all services in chain are instrumented
- **Span latency distribution** — per service, per operation
- **Trace-to-log correlation** — verify trace IDs appear in logs
- **Collector metrics** — spans received, processed, exported

## Security

- Trace data may contain sensitive information (user IDs, order details)
- Implement span attribute filtering to remove PII
- Restrict Jaeger UI access to authorized personnel
- Retention policies for trace data (7-30 days)
- Consider anonymizing user IDs in traces

## Production Considerations

- **5 services** — need consistent instrumentation across all teams
- OpenTelemetry is vendor-neutral — can switch backends (Jaeger, Tempo, X-Ray)
- Tail-based sampling requires collector resources (CPU/memory)
- Cost: Jaeger storage ~1GB/day for 10% sampling at 1000 RPS
- Consider service mesh (Istio) for automatic instrumentation without code changes

## Senior-Level Answer

"I'd implement OpenTelemetry as the standard instrumentation layer: (1) Deploy OTel Collector with tail-based sampling to capture 100% of slow/error traces while sampling normal traffic at 10%, (2) Auto-instrument all services with OTel SDK, (3) Standardize span naming and attributes across teams, (4) Correlate traces with logs by injecting trace IDs, (5) Create latency analysis dashboards from trace data. The key insight is that tracing should be automatic — teams shouldn't need to write instrumentation code."

## Architect-Level Answer

"At the organizational level, I'd establish: (1) Observability Standards — all services must use OpenTelemetry, (2) Service Mesh Strategy — consider Istio/Linkerd for automatic instrumentation without code changes, (3) Trace Governance — span naming conventions, attribute standards, retention policies, (4) Cost Optimization — tiered sampling (100% for errors, 10% for normal, 1% for health checks), (5) Platform Team — dedicated team managing observability infrastructure and helping teams instrument their services."

## Follow-Up Questions

1. "What's the difference between head-based and tail-based sampling? When would you use each?"
2. "How do you handle distributed tracing in asynchronous messaging (Kafka, SQS)?"
3. "How do you implement trace propagation across HTTP and gRPC boundaries?"
4. "How would you use distributed tracing to implement SLOs per service?"
5. "What's the impact of distributed tracing on application performance?"

# 82. Third-Party Payment Gateway Outage

## Scenario

At 3:00 PM, your primary payment gateway (Stripe) experiences a major outage. 40% of your customers cannot complete payments. Your e-commerce platform processes $2M in daily revenue. You have a secondary payment gateway (Adyen) available but it's not integrated — it was set up as a backup but never connected to the application. The Stripe outage is ongoing with no ETA for resolution. Your CEO is asking "when will payments work?" Your customers are complaining on social media. How do you handle the business impact while technically switching to the backup gateway?

## Interviewer Question

"Stripe is down. 40% of customers can't pay. You have Adyen as backup but it's not integrated. How do you handle the business impact while technically switching to the backup?"

## What I Should Think About

- Business impact: $2M daily revenue, 40% = $800K/day at risk
- Customer experience: payments failing, cart abandonment
- Technical integration: Adyen not connected to application
- Communication: customers, social media, internal stakeholders
- Rollback: ability to switch back to Stripe when it recovers
- Compliance: PCI-DSS, payment security
- Data consistency: ensure no double charges or lost transactions

## Ideal Answer

**Phase 1: Immediate Business Response (First 30 Minutes)**
- Communicate with customers: "We're experiencing payment issues, working on resolution"
- Update status page
- Monitor Stripe status page for updates
- Assess: how long has Stripe been down? ETA for recovery?

**Phase 2: Technical Assessment (30-60 Minutes)**
- Check Adyen integration status — what's already built?
- Identify integration gaps — what needs to be connected?
- Estimate time to integrate: hours vs. days
- Check if Stripe is degrading or fully down

**Phase 3: Rapid Integration (1-4 Hours)**
- Deploy Adyen payment module to production
- Route traffic to Adyen using feature flag
- Test with small percentage of traffic first
- Monitor error rates and transaction success

**Phase 4: Full Cutover (If Stripe outage persists)**
- Route 100% of traffic to Adyen
- Monitor transaction success rates
- Verify no double charges
- Update payment reconciliation

**Phase 5: Rollback Preparation**
- Keep Stripe integration ready for rollback
- Monitor Stripe status page
- Plan for gradual traffic shift back to Stripe

## Architecture

```
┌─────────────────────────────────────────────────┐
│          PAYMENT GATEWAY FAILOVER                 │
│                                                   │
│  BEFORE (Stripe Down):                           │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │ Customer │───►│ App      │───►│ Stripe   │  │
│  │ (40%     │    │ Server   │    │ (DOWN)   │  │
│  │  failing)│    │          │    │          │  │
│  └──────────┘    └──────────┘    └──────────┘  │
│                                                   │
│  AFTER (Adyen Active):                           │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │ Customer │───►│ App      │───►│ Adyen    │  │
│  │ (100%    │    │ Server   │    │ (ACTIVE) │  │
│  │  working)│    │          │    │          │  │
│  └──────────┘    └──────────┘    └──────────┘  │
│                      │                           │
│                      │ (ready for rollback)      │
│                      ▼                           │
│                ┌──────────┐                      │
│                │ Stripe   │                      │
│                │ (STANDBY)│                      │
│                └──────────┘                      │
└─────────────────────────────────────────────────┘
```

## Investigation

1. **Check Stripe status:**
   ```bash
   # Check Stripe API status
   curl -s https://status.stripe.com/api/v2/status.json | jq '.status'
   
   # Check Stripe API health
   curl -s -o /dev/null -w "%{http_code}" https://api.stripe.com/v1/health
   
   # Check Stripe incidents
   curl -s https://status.stripe.com/api/v2/incidents.json | jq '.incidents[] | {name: .name, status: .status}'
   ```

2. **Assess Adyen integration status:**
   ```bash
   # Check Adyen API availability
   curl -s -o /dev/null -w "%{http_code}" https://checkout-test.adyen.com/v71/payments
   
   # Check existing Adyen integration code
   find . -name "*.py" -o -name "*.java" -o -name "*.go" | xargs grep -l "adyen" | head -10
   
   # Check Adyen credentials
   kubectl get secret adyen-credentials -n production -o jsonpath='{.data.api-key}' | base64 -d
   ```

3. **Monitor business impact:**
   ```bash
   # Check transaction success rate
   curl -s "http://prometheus:9090/api/v1/query" \
     --data-urlencode "query=sum(rate(payment_transactions_total{status=\"success\"}[5m])) / sum(rate(payment_transactions_total[5m])) * 100"
   
   # Check revenue impact
   curl -s "http://prometheus:9090/api/v1/query" \
     --data-urlencode "query=sum(rate(payment_revenue_total[5m])) * 3600"
   ```

## Commands

```bash
# 1. Deploy Adyen payment module
cat > adyen-deployment.yml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service-adyen
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
      gateway: adyen
  template:
    metadata:
      labels:
        app: payment-service
        gateway: adyen
    spec:
      containers:
        - name: payment-service
          image: payment-service:adyen-latest
          env:
            - name: PAYMENT_GATEWAY
              value: "adyen"
            - name: ADYEN_API_KEY
              valueFrom:
                secretKeyRef:
                  name: adyen-credentials
                  key: api-key
EOF

kubectl apply -f adyen-deployment.yml

# 2. Route traffic using feature flag
cat > feature-flag.yml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
  namespace: production
data:
  USE_ADYEN_GATEWAY: "true"
  ADYEN_TRAFFIC_PERCENTAGE: "100"
  STRIPE_TRAFFIC_PERCENTAGE: "0"
EOF

kubectl apply -f feature-flag.yml

# 3. Configure traffic splitting (if using Istio)
cat > virtual-service.yml << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
  namespace: production
spec:
  hosts:
    - payment-service
  http:
    - route:
        - destination:
            host: payment-service-adyen
          weight: 100
        - destination:
            host: payment-service-stripe
          weight: 0
EOF

kubectl apply -f virtual-service.yml

# 4. Monitor Adyen transaction success
curl -s "http://prometheus:9090/api/v1/query" \
  --data-urlencode "query=sum(rate(payment_transactions_total{gateway=\"adyen\",status=\"success\"}[5m])) / sum(rate(payment_transactions_total{gateway=\"adyen\"}[5m])) * 100"

# 5. Rollback to Stripe when ready
cat > rollback-traffic.yml << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
  namespace: production
spec:
  hosts:
    - payment-service
  http:
    - route:
        - destination:
            host: payment-service-stripe
          weight: 10
        - destination:
            host: payment-service-adyen
          weight: 90
EOF

kubectl apply -f rollback-traffic.yml
```

## Root Cause

| Root Cause | Elimination |
|---|---|
| Single payment gateway dependency | Implement multi-gateway architecture |
| Backup gateway not integrated | Pre-integrate backup gateway, test regularly |
| No automated failover | Implement circuit breaker and auto-failover |
| No payment gateway monitoring | Add health checks and latency monitoring |
| No business continuity plan for payments | Create and test payment failover runbook |

## Immediate Mitigation

1. **Communicate with customers** — status page, social media
2. **Integrate Adyen** — rapid deployment
3. **Route traffic to Adyen** — feature flag or service mesh
4. **Monitor transaction success** — ensure no revenue loss

## Permanent Fix

1. Implement multi-gateway architecture from day one
2. Pre-integrate and regularly test backup gateways
3. Implement circuit breaker for payment gateways
4. Add automated failover based on health checks
5. Create payment failover runbook and test quarterly

## Monitoring

- **Payment success rate** — alert if < 99%
- **Payment gateway latency** — alert if > 2s
- **Payment gateway health** — continuous health checks
- **Revenue metrics** — alert on revenue drop > 5%
- **Transaction reconciliation** — daily reconciliation between gateways

## Security

- Payment gateway credentials in Secrets Manager
- PCI-DSS compliance for both gateways
- Transaction data encryption in transit and at rest
- Audit logging for all payment transactions
- Tokenization for card data (never store raw card numbers)

## Production Considerations

- **$2M daily revenue** — every minute of downtime costs ~$1,400
- Adyen integration may take 2-4 hours for rapid deployment
- Consider pre-built payment abstraction layer for future failovers
- Cost: Adyen transaction fees may differ from Stripe
- Compliance: PCI-DSS requires documented failover procedures

## Senior-Level Answer

"I'd handle this in parallel tracks: (1) Business — communicate with customers immediately, update status page, (2) Technical — rapid Adyen integration using feature flags, (3) Monitoring — track transaction success and revenue impact, (4) Rollback — keep Stripe ready for when it recovers. The key insight is that payment gateway failover should be pre-built, not built during an incident."

## Architect-Level Answer

"At the architectural level, I'd establish: (1) Payment Abstraction Layer — application code shouldn't depend on specific gateway, (2) Multi-Gateway Architecture — pre-integrate 2+ gateways with automatic failover, (3) Business Continuity — payment failover as a tested runbook, (4) Revenue Protection — circuit breakers and fallbacks for payment processing, (5) Compliance — PCI-DSS compliant failover procedures."

## Follow-Up Questions

1. "How do you implement a payment abstraction layer that supports multiple gateways?"
2. "What's the difference between active-active and active-passive payment gateway architectures?"
3. "How do you handle transaction reconciliation across multiple payment gateways?"
4. "How would you implement automated payment gateway failover based on health checks?"
5. "What's your approach to testing payment gateway failover without affecting real transactions?"

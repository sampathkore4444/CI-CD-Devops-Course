# 98. Man-in-the-Middle Attack on Internal Service

## Scenario

Network monitoring detected anomalous traffic patterns. An internal service (`payments-service`) is communicating with an unknown external IP (`203.0.113.45`). TLS certificates were recently changed (a maintenance task). Developers report that the API occasionally returns malformed responses. Some requests return certificate validation errors. Investigation shows the `payments-service` uses mTLS but the certificate trust store may have been modified during the certificate rotation. You suspect a Man-in-the-Middle (MITM) attack on the internal service.

## Interviewer Question

"Network monitoring detected anomalous traffic. An internal service is communicating with an unknown external IP. TLS certificates were recently changed. Investigate and contain a potential MITM attack on an internal service."

## What I Should Think About

- What is MITM: attacker intercepts TLS between services, decrypts/re-encrypts via own cert (if trust anchor compromised)
- Indicators: unexpected egress, cert validation errors, cert fingerprint mismatch, malformed responses
- Distinguish: real C2/exfil vs bad cert rotation mistake
- Certificate pinning vs trust store; PKI infra (internal CA vs public CA)
- Verify server identity: SAN/subjectAltName; hostname verification; mTLS client certs
- Forensic: egress flows (VPC flow logs / DNS), proxy/SNI logs, TLS handshakes
- Contain: block egress to suspicious IP; replace trust store; re-pin; rotate certs; rotate client secrets
- Trust anchors: detect if attacker added a rogue CA to the trust store

## Ideal Answer

**Phase 1 — Confirm/deny MITM:**

1. Check DNS — which hostname resolves to 203.0.113.45? Is it a known partner? (unlikely internal)
2. Check the TLS handshake — capture with tcpdump or use openssl s_client to see the presented cert:
   - Compare served leaf cert fingerprint against the known/internal CA-issued cert for that service
   - Look for mismatched issuer, self-signed, or unexpected CA
3. Check VPC flow logs / egress: `payments-service` adding systematic connections to that IP — C2 beaconing or MITM forwarder
4. Look at application cert validation errors over time vs certificate change timing

**Phase 2 — Contain:**

5. Network-kill: block IPv4/IPv6 egress to `203.0.113.45` at security group / NetworkPolicy / firewall → immediate containment of the channel
6. Confirm the trust store contents on affected hosts/services; remove the rogue/unknown CA if present
7. Rotate TLS certificates and any mTLS keys that were in play; rotate client-side secrets used in the channel

**Phase 3 — Root cause & harden:**

8. How did the rogue CA/trust entry get added? Where was the cert rotation done — a build step that imported a wrong bundle? Infra automation vuln?
9. Add certificate pinning for the critical service; enforce mTLS with strict hostname verification and pinned SPKI fingerprint
10. Enable best-practice monitoring: DNS/egress SPLUNK, TLS fingerprinting, cert expiry/rotate automation

## Architecture

```
  LEGITIMATE FLOW:
  payments-service ── mTLS (cert signed by internal CA) ──▶ payment-gateway
       │  trust store: [internal CA]                        cert check: SAN match ✓
       └────────────────────────────────────────────────────▶ metrics OK

  MITM ATTACK:
  payments-service ──mTLS?──▶ (attacker proxy, fake cert) ── decrypt──▶ payment-gateway (attacker impersonates)
       │  trust store: [internal CA + ROGUE CA ❗]             attacker transparently forwards
       │  presented cert signed by rogue → ACCEPTED (bad)     but can read/modify traffic
       │  └─ SAN mismatch? / issuer unknown?                   → malformed responses observed
       └── egress to 203.0.113.45 (attacker server) ◀──C2/fwd─

  DETECTION SIGNALS:
  1. DNS: api.payments.internal → 203.0.113.45 (unknown zone)
  2. TLS error rate spike right after "cert rotation" window
  3. cert.pem fingerprint differs from CA record
  4. VPC flow log: payments-service ↔ 203.0.113.45:443 constant handshakes
  5. metrics: connection success has 2 TLS handshakes (double termination)
```

## Investigation

**Step 1: Identify the endpoint and confirm resolution is anomalous**
```bash
# Which domain resolves to that IP?
nslookup 203.0.113.45
dig +short api.payments.internal @8.8.8.8
# If a partner domain? likely legit. If nothing / odd-zone → escalate

# Where did the payments service last resolve?
kubectl exec payments-0 -- getent hosts api.payments.internal
```

**Step 2: Capture TLS handshake cert**
```bash
# From the payments pod, check the presented cert:
kubectl exec payments-0 -- openssl s_client \
  -connect api.payments.internal:443 -servername api.payments.internal \
  -showcerts < /dev/null 2>/dev/null | openssl x509 -noout -subject -issuer \
    -fingerprint -sha256 -serial -dates -ext subjectAltName

# Compare to the internal CA's known value
# If subject/issuer/SAN mismatch the internal CA record → anomaly
```

**Step 3: Network forensics — egress flows**
```bash
# CloudWatch / VPC flow logs for that workload
aws logs filter-log-events --log-group-name /aws/vpc/flowlog \
  --filter-pattern '203.0.113.45' | jq .events | head -100

# check connections direction, ports (443 TLS), frequency (beacon → 1/min classic)
```

**Step 4: Inspect trust store on the host/service**
```bash
# Did a rogue CA sneak in during cert rotation?
kubectl exec payments-0 -- \
  grep -l "IA7P..." /etc/ssl/certs/*.pem | xargs -I{} openssl x509 -in {} -noout -subject -issuer

# Compare against the documented canonical CA list
# Any certificate that isn't from our internal CA = contamination
keytool -list -keystore /app/truststore.jks -storepass ***
```

**Step 5: Check client/app logs for cert validation errors before/after the cert change**
```bash
kubectl logs payments-0 --since=48h | grep -iE "cert.*(error|invalid|untrusted)|SSL|TLS" | tail -100
# correlate timestamp of first validation errors vs rotation window
```

**Step 6: Verify the legitimate service identity directly**
```bash
# directly reach the real service and compare fingerprints
openssl s_client -connect <real-gw>:443 -showcerts | openssl x509 -fingerprint -sha256
# mismatch between real fingerprint and what payments observes → MITM confirmed
```

## Commands

```bash
# IMMEDIATE containment: block egress to the IP
# AWS Security Group:
aws ec2 authorize-security-group-egress \
  --group-id sg-123 --ip-permissions \
  'IpProtocol=-1,IpRanges=[{CidrIp=203.0.113.45/32,Description=BLOCK-MITM}]'
# reverse: revoke any allow for it
aws ec2 revoke-security-group-egress --group-id sg-123 \
  --ip-permissions IpProtocol=tcp,FromPort=443,ToPort=443,IpRanges=[{CidrIp=203.0.113.45/32}]

# Kubernetes NetworkPolicy deny lockout:
kubectl apply -f - << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: block-external-ip}
spec:
  podSelector: {matchLabels: {app: payments}}
  policyTypes: [Egress]
  egress:
    - to:
        - ipBlock: {cidr: 0.0.0.0/0, except: ["203.0.113.45/32"]}
EOF

# Capture live traffic for evidence (pod-side):
kubectl exec payments-0 -- tcpdump -nn -i eth0 \
  host 203.0.113.45 -w /tmp/mitm.pcap

# Confirm cert mismatch (server-side inspection):
openssl s_client -connect 203.0.113.45:443 \
  -servername api.payments.internal -showcerts </dev/null \
  | openssl x509 -noout -subject -issuer -fingerprint -sha256
```

## Immediate Mitigation

```bash
# 1. Block the IP (see commands above) — cut the channel NOW
# 2. Hold cert rotation: revert to the last-known-good certs if rot broke trust
#    - redeploy the correct trust store + s/San updates
kubectl exec payments-0 -- /bin/sh -c 'rm -f /etc/ssl/certs/gibson.pem; update-ca-certificates'
# 3. Rotate whatever credentials passed through that channel (client keys, mTLS client certs)
# 4. Rotate payment-gateway secrets / tokens if any could have been decrypted
# 5. Freeze further deploys to that stack until unknown CA is removed + verified
```

## Permanent Fix

1. **Certificate pinning (SPKI/SAN pin)** for the payments service: pin the internal CA or the service's exact public key; validation fails against any unknown identity
2. **mTLS with strict hostname verification** — never accept a cert with a SAN/BasicConstraints mismatch
3. **Centralized PKI**: only the internal CA releases machine certs; automated rotation via cert-manager/Vault with vetting; remove any archaic import step
4. **Egress allowlist** (NetworkPolicies): internal workloads may only call known endpoints — anything else is blocked
5. **Runtime detection**: Falco/Datadog flag new external connections; EGRESS dashboard alert on unknown destinations
6. **Audit cert rotation**: any cert change is code-reviewed + they can't add CA entries silently (privileged path)
7. **DNS defense**: use internal DNS with dedicated records; alert on unknown external IP resolution

## Monitoring

```bash
# Detection-level monitoring:
# - VPC flow/network egress: any pod contacting unknown external IP → P1
# - DNS: suspicious resolution (CNAME/odd zone) → notify
# - TLS: cert validation failure rate, handshake count from SDK metrics
# - App: tripe handshake/timeouts, malformed response count
# - PKI: cert expiry, rotation events, added CA certs in trust store

# Alerts:
# - egress to non-allowlisted IP → warning/P1
# - cert validation error rate > 0.5% → warning
# - trust store change event → audit alert
# - unexpected DNS result for api.payments.internal → P1
```

## Security

- Never accept certs "from the rotation bundle" blindly — validate against the CA directly
- Park channel credentials: if MITM decrypted anything, rotate payment secrets, mTLS keys, database creds used within/through that service
- Keep forensic pcap + cert captures; interpretable evidence for incident response
- Ensure TLS is the ONLY encrypted path; terminate mTLS only at the service, not through proxies without verification
- Threat model: check if the attacker could have replayed requests (idempotency keys important)

## Production Considerations

- **HA**: while egress-blocked, payments calls will fail → plan a controlled switch to a fallback path (or isolate while offline)
- **Reliability**: cert rotation mistakes directly cause the malformed responses — test rotation in staging with integration tests
- **Cost**: public CA vs internal PKI — internal CA + cert-manager reduces cost and overhead, improves observability
- **Compliance**: financial channel integrity matters; incident response + evidence retention per PCI
- **Operational**: documented cert rotation runbook that includes external verification (from an independent vantage) before cutover
- **Governance**: minimize humans touching trust stores; use Vault/cert-manager with policy gates

## Senior-Level Answer

"First establish whether it's actually MITM or a failed cert rotation: check what cert the service presents, its fingerprint vs the internal CA record, and the DNS/egress path to 203.0.113.45. If the presented identity doesn't match our CA and the flow shows systematic egress to that IP, contain by blocking that IP at the NetworkPolicy/SG immediately, then remove any rogue CA from the trust store and rotate the certs + any secrets that traversed the channel. The permanent fix is certificate pinning for the critical service, internal DNS/egress allowlists, mTLS with strict SAN/hostname verification, and an auditable cert-rotation path via cert-manager/Vault — plus Falco-style alerts on unexpected external egress."

## Architect-Level Answer

"The failure is a trust-management problem: we handed machines a trust store whose contents were not verifiable. I'd centralize PKI: all services get certs from the internal CA via cert-manager/Vault; the trust store is declarative and versioned, so adding a CA requires review. Payments-service gets certificate pinning on top of normal validation for defense-in-depth. At the network layer, egress should be allowlisted by NetworkPolicy — no internal service may reach an arbitrary external IP, which turns this MITM channel into a hard failure instead of a silent leak. Add DNS + egress + cert-validation monitoring as SLO inputs, and an audited rotation path. This should be priced in: short-lived certs (24-48h) and automatic renewal via a CI pipeline, so the human 'cert rotation task' that let this in doesn't exist."

## Follow-Up Questions

1. "How do you distinguish a real MITM from an accidentally-installed export CA (`Windows`/corporate proxy)? What tests actually confirm interception?"
2. "Design the certificate-pinning strategy for a service that talks to multiple clients with potentially rotating certs."
3. "If the attacker had the decrypted traffic for 2 hours, how do you scope and rotate which secrets are affected?"
4. "What role do SNI logging and client-hello fingerprinting play in detecting MITM vs C2 exfil?"
5. "You find the rogue CA WAS legitimately pushed by automation. How do you trace the automation and fix the pipeline?"
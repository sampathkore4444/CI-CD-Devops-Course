# 51. Ingress Returning 404 Errors

## Scenario

An Nginx Ingress Controller is returning 404 for all routes in the `web` namespace. The ingress resource looks correct with proper host, path, and backend service references. The backend services (`frontend`, `api-gateway`) are running and respond correctly when accessed via `kubectl port-forward`. The TLS certificate is valid and correctly configured. The ingress controller is running healthy. Other namespaces with similar ingress configurations work fine. The issue started after deploying a new ingress resource for a `blog` service in the same namespace.

## Interviewer Question

"An Nginx Ingress Controller is returning 404 for all routes. The ingress resource looks correct. The backend services are running and respond correctly via port-forward. The TLS certificate is valid. How do you troubleshoot?"

## What I Should Think About

- 404 means the Ingress Controller is responding but can't match the request to a backend
- Backend services work via port-forward, so the issue is in the Ingress → Service path
- Check Ingress resource syntax: host, path, pathType, backend service name/port
- Nginx Ingress uses path-based routing; incorrect pathType can cause 404
- Check if the ingress class is correct (nginx vs traefik vs others)
- Check ingress controller logs for the exact routing decision
- The new blog ingress might have conflicting rules
- Check if the ingress controller has a default backend configured
- Verify the ingress controller's namespace and the backend services' namespace match
- The ingress controller might need to be reloaded/restarted

## Ideal Answer

"Start by checking the ingress controller logs — they'll show exactly why the route isn't matching:

```bash
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=100 | grep "404"
```

Then verify the Ingress resource is in the correct namespace and has the right ingress class:

```bash
kubectl get ingress -A
kubectl get ingress <ingress-name> -n web -o yaml
```

Common issues:
1. Wrong `ingressClassName` — the ingress might be using a different controller
2. Wrong `pathType` — `Prefix` vs `Exact` vs `ImplementationSpecific`
3. Wrong service name or port in the backend
4. Missing namespace — service is in a different namespace
5. Conflicting rules from the new blog ingress

Check if the ingress controller is actually watching this ingress:

```bash
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=100 | grep -i "web"
```

If the ingress resource looks correct, check the generated nginx.conf:

```bash
kubectl exec -it <ingress-controller-pod> -n ingress-nginx -- cat /etc/nginx/nginx.conf | grep -A 10 "server_name"
```

## Architecture

```
External Request Flow:

  User → DNS → Ingress Controller → Ingress Rule Match → Service → Pod

  Where 404 occurs:
  ┌──────────────────────────────────────────────────────┐
  │  Ingress Controller (Nginx)                          │
  │                                                      │
  │  nginx.conf generated from Ingress resources:        │
  │                                                      │
  │  server {                                            │
  │    server_name example.com;                          │
  │                                                      │
  │    location / {                                      │
  │      proxy_pass http://frontend-service:80;          │
  │    }                                                 │
  │                                                      │
  │    location /api {                                   │
  │      proxy_pass http://api-gateway-service:8080;     │
  │    }                                                 │
  │                                                      │
  │    # If no location matches → 404                    │
  │  }                                                   │
  └──────────────────────────────────────────────────────┘

  Possible 404 Causes:
  1. Ingress resource not loaded by controller
  2. Wrong pathType (Exact vs Prefix)
  3. Wrong service name in backend
  4. Service in different namespace
  5. Conflicting rules from other ingress
  6. Wrong ingressClassName
```

## Investigation

**Step 1: Check ingress controller logs**

```bash
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=100
```

**Step 2: Verify ingress resource**

```bash
kubectl get ingress -n web -o wide
kubectl describe ingress <ingress-name> -n web
```

**Step 3: Check generated nginx configuration**

```bash
kubectl exec -it <ingress-pod> -n ingress-nginx -- cat /etc/nginx/nginx.conf | grep -B 5 -A 15 "server_name"
```

**Step 4: Verify backend service exists and is correct**

```bash
kubectl get svc -n web
kubectl get endpoints -n web
```

**Step 5: Test with port-forward to confirm backend works**

```bash
kubectl port-forward svc/frontend 8080:80 -n web
curl http://localhost:8080
```

**Step 6: Check ingress class**

```bash
kubectl get ingressclass
kubectl get ingress <ingress-name> -n web -o jsonpath='{.spec.ingressClassName}'
```

**Step 7: Check for conflicting ingress rules**

```bash
kubectl get ingress -A -o wide | grep example.com
```

## Commands

```bash
# Get all ingress resources across namespaces
kubectl get ingress -A -o wide

# Describe the problematic ingress
kubectl describe ingress <ingress-name> -n web

# Check ingress controller logs for 404 errors
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=200 | grep "404"

# Check generated nginx.conf
kubectl exec -it <ingress-pod> -n ingress-nginx -- cat /etc/nginx/nginx.conf | grep -A 20 "server_name example.com"

# Verify backend service exists
kubectl get svc frontend api-gateway -n web

# Verify endpoints are populated
kubectl get endpoints frontend api-gateway -n web

# Test backend directly
kubectl exec -it <test-pod> -n web -- curl -s http://frontend:80/health
kubectl exec -it <test-pod> -n web -- curl -s http://api-gateway:8080/health

# Check ingress class
kubectl get ingressclass

# Check for annotations
kubectl get ingress <ingress-name> -n web -o jsonpath='{.metadata.annotations}'

# Check if ingress controller is watching the namespace
kubectl get ingress -n web -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.ingressClassName}{"\n"}{end}'

# Test external access
curl -v -H "Host: example.com" http://<ingress-controller-ip>/health

# Check TLS secret
kubectl get secret -n web tls-secret -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout

# Reload ingress controller (forces nginx config reload)
kubectl rollout restart deployment/ingress-nginx-controller -n ingress-nginx
```

## Root Cause

| Root Cause | Evidence | How to Eliminate |
|---|---|---|
| **Wrong ingressClassName** | Ingress class doesn't match controller | Set correct ingressClassName |
| **Wrong pathType** | Exact path doesn't match requests | Use Prefix pathType for catch-all |
| **Service in different namespace** | Backend service not found | Specify namespace in backend or use ExternalName |
| **Conflicting rules** | Multiple ingresses for same host | Consolidate rules or use host-based routing |
| **Controller not watching namespace** | Controller logs don't show the ingress | Add namespace filter to controller args |
| **Missing default backend** | No fallback for unmatched routes | Add defaultBackend to ingress |
| **TLS misconfiguration** | HTTPS returns 404, HTTP works | Fix TLS secret reference |

## Immediate Mitigation

```bash
# 1. Check if changing pathType fixes it
kubectl patch ingress <ingress-name> -n web --type='json' -p='[
  {"op": "replace", "path": "/spec/rules/0/http/paths/0/pathType", "value": "Prefix"}
]'

# 2. Add a catch-all path
kubectl patch ingress <ingress-name> -n web --type='json' -p='[
  {"op": "add", "path": "/spec/rules/0/http/paths/-", "value": {"path": "/","pathType":"Prefix","backend":{"service":{"name":"frontend","port":{"number":80}}}}}
]'

# 3. Force ingress controller reload
kubectl rollout restart deployment/ingress-nginx-controller -n ingress-nginx

# 4. If conflicting rules from blog ingress, delete it temporarily
kubectl delete ingress blog-ingress -n web
```

## Permanent Fix

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: web
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    secretName: tls-secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-gateway
            port:
              number: 8080
      - path: /blog
        pathType: Prefix
        backend:
          service:
            name: blog
            port:
              number: 80
```

## Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ingress-404
spec:
  groups:
  - name: ingress.rules
    rules:
    - alert: IngressHigh4xxRate
      expr: rate(nginx_ingress_controller_requests{status=~"4.."}[5m]) > 0.1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High 4xx rate on ingress {{ $labels.ingress }}"

    - alert: IngressNoTraffic
      expr: rate(nginx_ingress_controller_requests{namespace="web"}[5m]) == 0
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "No traffic to web namespace ingress"
```

## Security

- **TLS configuration**: Always use TLS for production ingress
- **Rate limiting**: Configure nginx annotations for rate limiting
- **WAF**: Consider ModSecurity or similar for web application firewall
- **CORS**: Configure CORS policies in ingress annotations

## Production Considerations

- **Ingress controller scaling**: Run multiple replicas for HA
- **Default backend**: Always configure a default backend for unmatched routes
- **IngressClass**: Use unique ingress classes for different controllers
- **Canary ingress**: Use nginx canary annotations for traffic splitting

## Senior-Level Answer

"I'd start by checking the ingress controller logs — they show exactly why the route doesn't match. The 404 means the controller is responding but can't find a matching backend. Since port-forward works, the backend services are healthy. The issue is in the Ingress → Service mapping. I'd check: (1) ingressClassName matches the running controller, (2) pathType is correct (Prefix for catch-all, Exact for specific paths), (3) backend service name and port are correct, (4) service is in the same namespace as the ingress. The new blog ingress might have conflicting rules for the same host. I'd verify by checking the generated nginx.conf inside the controller pod."

## Architect-Level Answer

"At the architecture level, I'd implement: (1) Ingress per team/namespace with clear ownership, (2) IngressClass per environment to prevent cross-environment routing, (3) Automated ingress validation in CI (check pathType, service existence, TLS config), (4) Service mesh for internal traffic routing instead of relying solely on ingress, and (5) GitOps-managed ingress resources with automated testing. The key principle is that ingress configuration should be validated and tested before deployment, not just applied and hoped for."

## Follow-Up Questions

1. "What's the difference between pathType: Exact, Prefix, and ImplementationSpecific?"
2. "How does Nginx Ingress Controller handle path rewriting with the rewrite-target annotation?"
3. "How would you implement canary deployments using Ingress annotations?"
4. "Explain the difference between host-based and path-based routing in Ingress."
5. "How does the Ingress Controller handle TLS termination and SNI?"

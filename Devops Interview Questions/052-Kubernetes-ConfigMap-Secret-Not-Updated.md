# 52. ConfigMap/Secret Change Not Reflected in Running Pod

## Scenario

You updated a ConfigMap named `app-config` in the `production` namespace with new database connection pool settings (increased `maxPoolSize` from 10 to 50). The `kubectl get configmap app-config -n production -o yaml` shows the new values. However, the running pods still use the old configuration. The application logs show the old `maxPoolSize=10`. You need the new configuration to take effect without disrupting service. The application reads configuration from environment variables AND mounted files.

## Interviewer Question

"You updated a ConfigMap with new configuration values. But the running pod still uses the old values. How does Kubernetes handle ConfigMap/Secret updates? What are the strategies to force a refresh without manual restart?"

## What I Should Think About

- Kubernetes mounts ConfigMaps/Secrets as files — the kubelet syncs file updates every 60 seconds by default
- Environment variables are set at pod creation time and NEVER update without restart
- If the app reads from mounted files, it might need to detect changes (inotify) or be restarted
- If the app reads from env vars, a restart is required
- Strategies: rolling restart, env var injection via deployment hash, Reloader operator, symlink swap
- The volume mount updates are eventual — not instant
- The `immutable` field on ConfigMap prevents accidental updates
- Need to understand HOW the app reads the config (file vs env var)

## Ideal Answer

"The behavior depends on how the application consumes the ConfigMap:

**If mounted as a volume (file):** The kubelet automatically syncs mounted ConfigMap files every 60 seconds (configurable via `--sync-frequency`). However, the application must detect the file change — typically via inotify or polling. Many applications read config at startup and never re-read.

**If injected as environment variables:** Environment variables are set at container creation time and NEVER update. The pod MUST be restarted.

To force a refresh without manual restart:

1. **Rolling restart:** `kubectl rollout restart deployment/<name>` — this recreates pods with new config
2. **Reloader operator:** Automatically restarts pods when referenced ConfigMaps/Secrets change
3. **ConfigMap hash annotation:** Add the ConfigMap hash to pod annotations so changes trigger a rollout
4. **Mutable ConfigMap with volume mount:** If the app supports inotify, the 60-second sync will update files

The recommended approach is either using the Reloader operator or adding a config hash to the deployment template."

## Architecture

```
ConfigMap Update Mechanism:

  ConfigMap Volume Mount (File):
  ┌────────────────────────────────────────────────┐
  │ Pod                                            │
  │                                                │
  │  /config/app.properties                        │
  │    ← kubelet syncs every 60s                   │
  │    ← file content updates                      │
  │    ← BUT app must re-read the file             │
  │                                                │
  │  Application behavior:                         │
  │    ├── Reads at startup → NEVER updates        │
  │    ├── Reads on each request → updates in 60s  │
  │    └── Uses inotify → updates in ~60s          │
  └────────────────────────────────────────────────┘

  Environment Variable:
  ┌────────────────────────────────────────────────┐
  │ Pod                                            │
  │                                                │
  │  env:                                          │
  │    MAX_POOL_SIZE=10  ← Set at creation         │
  │    NEVER updates without restart               │
  │                                                │
  │  Even after ConfigMap update:                  │
  │    MAX_POOL_SIZE=10  ← Still old value        │
  └────────────────────────────────────────────────┘

  Strategies to Force Update:
  ┌────────────────────────────────────────────────┐
  │ 1. Rolling Restart:                            │
  │    kubectl rollout restart deployment/app      │
  │    → Pods recreated with new config             │
  │                                                │
  │ 2. Reloader Operator:                          │
  │    Stakater Reloader watches ConfigMaps        │
  │    → Automatically triggers rolling restart    │
  │                                                │
  │ 3. Config Hash Annotation:                     │
  │    Add configmap-hash to pod template          │
  │    → Change in ConfigMap changes hash          │
  │    → Deployment detects template change        │
  │    → Triggers rolling update                   │
  │                                                │
  │ 4. Immutable ConfigMap + New ConfigMap:        │
  │    Create new ConfigMap, update deployment     │
  │    → Clean, auditable change                   │
  └────────────────────────────────────────────────┘
```

## Investigation

**Step 1: Verify ConfigMap has new values**

```bash
kubectl get configmap app-config -n production -o yaml
```

**Step 2: Check how the app consumes the config**

```bash
kubectl get deployment app -n production -o jsonpath='{.spec.template.spec.containers[*].env}'
kubectl get deployment app -n production -o jsonpath='{.spec.template.spec.containers[*].volumeMounts}'
```

**Step 3: Check the actual config inside the pod**

```bash
kubectl exec -it <pod-name> -n production -- cat /config/app.properties | grep maxPoolSize
kubectl exec -it <pod-name> -n production -- env | grep MAX_POOL_SIZE
```

**Step 4: Check if the volume mount is updating**

```bash
kubectl exec -it <pod-name> -n production -- ls -la /config/
kubectl exec -it <pod-name> -n production -- stat /config/app.properties
```

**Step 5: Check the last time the ConfigMap was updated**

```bash
kubectl get configmap app-config -n production -o jsonpath='{.metadata.resourceVersion}'
```

**Step 6: Verify pod spec references the ConfigMap**

```bash
kubectl get pod <pod-name> -n production -o jsonpath='{.spec.volumes[*].configMap}'
```

## Commands

```bash
# Check ConfigMap current values
kubectl get configmap app-config -n production -o yaml

# Check how app uses config (env vars vs volume mounts)
kubectl get deployment app -n production -o jsonpath='{.spec.template.spec.containers[*]}' | jq .

# Verify config inside pod (file mount)
kubectl exec -it <pod-name> -n production -- cat /config/app.properties

# Verify config inside pod (env var)
kubectl exec -it <pod-name> -n production -- env | grep -i pool

# Check volume mount timestamp
kubectl exec -it <pod-name> -n production -- ls -la /config/

# Force rolling restart
kubectl rollout restart deployment/app -n production

# Monitor rollout progress
kubectl rollout status deployment/app -n production

# Install Reloader operator (if not installed)
kubectl apply -f https://raw.githubusercontent.com/stakater/Reloader/master/deployments/kubernetes/reloader.yaml

# Annotate deployment to use Reloader
kubectl annotate deployment app -n production "reloader.stakater.com/auto: true"

# Add config hash to trigger rollout on ConfigMap change
kubectl get configmap app-config -n production -o jsonpath='{.data}' | md5sum

# Check if ConfigMap is immutable
kubectl get configmap app-config -n production -o jsonpath='{.metadata.labels}'

# Create new immutable ConfigMap
kubectl create configmap app-config-v2 -n production --from-literal=maxPoolSize=50 --dry-run=client -o yaml | kubectl label --local -f - app-config-version=v2 -o yaml | kubectl apply -f -
```

## Root Cause

| Root Cause | Evidence | How to Eliminate |
|---|---|---|
| **Env Vars Don't Update** | Config consumed via env vars | Use volume mount instead, or restart pod |
| **App Doesn't Re-read Config** | File updated but app uses old values | Implement config hot-reload in app, or restart |
| **Kubelet Sync Delay** | Config recently updated, <60s ago | Wait 60 seconds for kubelet to sync |
| **Volume Mount Not Updated** | File content still shows old values | Restart pod, check kubelet sync |
| **Immutable ConfigMap** | ConfigMap has `immutable: true` | Create new ConfigMap, update deployment |
| **Wrong ConfigMap Name** | Deployment references different ConfigMap | Fix ConfigMap reference in deployment |

## Immediate Mitigation

```bash
# 1. Rolling restart to force config reload
kubectl rollout restart deployment/app -n production

# 2. Monitor rollout
kubectl rollout status deployment/app -n production

# 3. Verify new config is loaded
kubectl exec -it <new-pod-name> -n production -- cat /config/app.properties | grep maxPoolSize

# 4. If env vars are the issue, patch to use volume mount
kubectl patch deployment app -n production --type='json' -p='[
  {"op": "add", "path": "/spec/template/spec/volumes/-", "value": {"name": "config","configMap": {"name": "app-config"}}},
  {"op": "add", "path": "/spec/template/spec/containers/0/volumeMounts/-", "value": {"name": "config","mountPath": "/config"}}
]'
```

## Permanent Fix

```yaml
# 1. Use Reloader operator for automatic updates
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: production
  annotations:
    reloader.stakater.com/auto: "true"
spec:
  template:
    spec:
      containers:
      - name: app
        env:
        - name: MAX_POOL_SIZE
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: maxPoolSize
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: app-config

---
# 2. Config hash annotation approach
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: production
spec:
  template:
    metadata:
      annotations:
        checksum/config: "a1b2c3d4e5f6"  # Update this on ConfigMap change
    spec:
      containers:
      - name: app
        env:
        - name: MAX_POOL_SIZE
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: maxPoolSize
```

## Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: configmap-drift
spec:
  groups:
  - name: configmap.rules
    rules:
    - alert: ConfigMapNotInSync
      expr: kube_configmap_info{namespace="production",configmap="app-config"} == 1
        and on() (time() - kube_configmap_created{namespace="production",configmap="app-config"}) > 3600
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "ConfigMap app-config may be out of sync with running pods"
```

## Security

- **Secrets in ConfigMaps**: Never put sensitive data in ConfigMaps — use Secrets instead
- **RBAC**: Restrict who can modify ConfigMaps in production
- **Audit logs**: Track ConfigMap changes for compliance
- **Immutable ConfigMaps**: Use immutability to prevent accidental changes

## Production Considerations

- **Config as Code**: Store ConfigMaps in Git and manage via ArgoCD/Flux
- **Secrets management**: Use External Secrets Operator or Vault for sensitive data
- **Rolling restart impact**: Ensure enough replicas to handle traffic during restart
- **Config validation**: Validate ConfigMap changes before applying

## Senior-Level Answer

"Kubernetes doesn't automatically restart pods when their ConfigMaps change. If the config is consumed as environment variables, a restart is required — env vars are immutable after pod creation. If mounted as files, the kubelet syncs updates every 60 seconds, but the application must re-read the file. Most applications don't, so a restart is usually needed. I'd use one of three approaches: (1) `kubectl rollout restart` for one-time changes, (2) Stakater Reloader operator for automatic restarts when ConfigMaps change, or (3) Add the ConfigMap's resource version hash to pod annotations so changes trigger a Deployment rollout. The best practice is to use Reloader for production workloads and ensure the application supports config hot-reload for file-mounted ConfigMaps."

## Architect-Level Answer

"At the architecture level, configuration management in Kubernetes should be: (1) GitOps-managed (ArgoCD/Flux), (2) Environment-specific (different ConfigMaps per environment), (3) Immutable where possible (create new ConfigMap, update deployment), and (4) Validated before deployment (OPA/Gatekeeper policies). For sensitive data, use External Secrets Operator or HashiCorp Vault instead of Kubernetes Secrets. The key architectural decision is whether to use file-mounted ConfigMaps (supports hot-reload if app supports it) or environment variables (requires restart). For production, I recommend file-mounted ConfigMaps with application-level hot-reload and Reloader as a safety net."

## Follow-Up Questions

1. "What's the difference between a Kubernetes ConfigMap and a Secret in terms of security and base64 encoding?"
2. "How does the Reloader operator detect ConfigMap changes and trigger restarts?"
3. "Explain the kubelet sync frequency for mounted ConfigMaps and how to configure it."
4. "How would you implement a zero-downtime config change for a stateful application?"
5. "What are the trade-offs between immutable and mutable ConfigMaps?"

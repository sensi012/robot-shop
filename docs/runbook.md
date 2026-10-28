# Runbook: Payment Service Outage

## Alert: `PaymentHighErrorRate`
**Severity**: Critical  
**Service**: `payment` (namespace: `robot-shop`)

---

## Quick Diagnosis (< 2 minutes)

### 1. Check Pod Status
```bash
kubectl get pods -n robot-shop -l app=payment
```

**Expected**: At least 1 pod in `Running` status with `1/1` Ready.

### 2. Check Pod Events
```bash
kubectl describe pod -n robot-shop -l app=payment
```

Look for: `OOMKilled`, `CrashLoopBackOff`, `ImagePullBackOff`, or `Pending` states.

### 3. Check Logs
```bash
kubectl logs -n robot-shop -l app=payment --tail=50
```

Or query in Grafana Loki:
```logql
{namespace="robot-shop", app="payment"} |= "error"
```

### 4. Check Upstream Dependencies
Payment depends on RabbitMQ and User services:
```bash
kubectl get pods -n robot-shop -l app=rabbitmq
kubectl get pods -n robot-shop -l app=user
```

---

## Mitigation Steps

### Scenario A: Pod Scaled to 0
```bash
kubectl scale deployment payment --replicas=1 -n robot-shop
kubectl rollout status deployment payment -n robot-shop
```

### Scenario B: CrashLoopBackOff
```bash
# Check recent logs for crash reason
kubectl logs -n robot-shop -l app=payment --previous --tail=100

# If config issue, restart the deployment
kubectl rollout restart deployment payment -n robot-shop
```

### Scenario C: OOMKilled
```bash
# Increase memory limits
kubectl patch deployment payment -n robot-shop --type=json \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/memory", "value": "256Mi"}]'
```

### Scenario D: Dependency Failure (RabbitMQ down)
```bash
kubectl get pods -n robot-shop -l app=rabbitmq
kubectl rollout restart statefulset rabbitmq -n robot-shop
```

---

## Verification

After mitigation, verify recovery:
```bash
# Pod is running
kubectl get pods -n robot-shop -l app=payment

# Load generator shows successful checkouts
kubectl logs -n robot-shop -l app=load --tail=20

# Alert resolves in Prometheus/Alertmanager
# Check: http://localhost:9090/alerts
```

---

## Escalation

If the issue persists after 15 minutes:
1. Escalate to the team lead
2. Check cluster-level issues (node health, resource pressure)
3. Review recent deployments/changes in the last 24 hours

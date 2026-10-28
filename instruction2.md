# Capstone Project 2:

This guide covers all 6 steps plus deliverables required for **Capstone 2: (Stan's Robot Shop)** based on the course capstone requirements.

---

## Overview: Stan's Robot Shop & Observability Stack

### Services & Components
- **Target Application**: Stan's Robot Shop (microservices e-commerce application).
- **Core Focus Services (4)**:
  1. `web` (Nginx / Frontend)
  2. `catalogue` (Node.js)
  3. `cart` (Redis / Node.js)
  4. `payment` (Python)
- **Data Stores & Supporting Services**: MongoDB, Redis, MySQL, RabbitMQ, Dispatch, User, Shipping, Ratings.
- **Traffic Generator**: `load-gen` (generates steady synthetic shopping traffic).
- **Observability Stack**:
  - **Metrics**: `kube-prometheus-stack` (Prometheus + Grafana + Alertmanager).
  - **Logs**: Fluent Bit collecting container logs and forwarding to Grafana Loki.
  - **Traces**: OpenTelemetry (OTel) Collector and Jaeger.
  - **Alerting & SRE**: SLO definition, Error Budget policy, and Alertmanager routing to Slack.

---

## Six-Step Implementation Roadmap

| Step | Topic | Goal |
| :--- | :--- | :--- |
| **Step 1** | **Deploy and Watch** | Fork `instana/robot-shop`, deploy to Capstone 1 cluster via Helm without Instana agent, enable `load-gen`, verify healthy pods. |
| **Step 2** | **See the Metrics** | Install `kube-prometheus-stack` via Helm, construct an RED (Rate, Errors, Duration) dashboard in Grafana. |
| **Step 3** | **Collect the Logs** | Deploy Loki and Fluent Bit daemonset, parse logs, query logs in Grafana by service and log level. |
| **Step 4** | **Follow One Trace** | Deploy Jaeger and OpenTelemetry Collector, trace an end-to-end request (e.g., checkout/cart flow). |
| **Step 5** | **Set One SLO** | Choose Availability SLO (e.g. 99% success rate on `payment` or `web`), define error budget policy, wire Alertmanager alert to Slack. |
| **Step 6** | **Run Incident Drill** | Simulate an outage / break a dependency, observe alert firing, mitigate, diagnose root cause, write blameless postmortem. |

---

## Step 1: Deploy Stan's Robot Shop

### 1.1 Fork and Clone the Repository
1. On GitHub, navigate to `https://github.com/instana/robot-shop` and click **Fork** to your personal account.
2. Clone your fork locally (alongside your project workspace):
   ```bash
   cd ~/TSacademy/project
   git clone https://github.com/<your-username>/robot-shop.git
   cd robot-shop
   ```

### 1.2 Prepare Namespace and Configuration
Stan's Robot Shop includes a Helm chart in `K8s/helm`. Make sure the Instana agent is disabled (not enabled by default, but confirm in values):

Create the namespace:
```bash
kubectl create namespace robot-shop
```

### 1.3 Install via Helm
Install the chart into the `robot-shop` namespace:
```bash
helm upgrade --install robot-shop --namespace robot-shop ./K8s/helm
```

### 1.4 Verify All Pods & Traffic
Check that all microservices and databases are `Running`:
```bash
kubectl get pods -n robot-shop -w
```

Verify that the `load-gen` pod is active:
```bash
kubectl logs -n robot-shop -l app=load -f
```
*(You will see simulated shoppers browsing catalogues, adding items to carts, and completing purchases).*

---

## Step 2: See the Metrics (Prometheus & Grafana)

Deploy the industry-standard `kube-prometheus-stack` to collect cluster, node, and application metrics.

### 2.1 Add Helm Repositories
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 2.2 Create Prometheus Values (`monitoring-values.yaml`)
Create a file named `monitoring-values.yaml`:

```yaml
grafana:
  adminPassword: "adminpassword123"
  service:
    type: LoadBalancer # Or NodePort/Port-Forward depending on environment

prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
    retention: 5d
```

### 2.3 Install `kube-prometheus-stack`
```bash
kubectl create namespace monitoring

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f monitoring-values.yaml
```

Verify monitoring components:
```bash
kubectl get pods -n monitoring
```

### 2.4 Access Grafana
If using port-forward:
```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```
- URL: `http://localhost:3000`
- User: `admin`
- Password: `adminpassword123`

### 2.5 Build the RED Dashboard for Core 4 Services
In Grafana:
1. Click **Dashboards** → **New Dashboard** → **Add visualization**.
2. Data Source: **Prometheus**.
3. Create 3 Panels using PromQL for `web`, `catalogue`, `cart`, and `payment`:

- **Rate (Request Rate - RPS)**:
  ```promql
  sum(rate(nginx_ingress_controller_requests{status=~"[1-5].*"}[2m])) by (service)
  # or container network IO rate:
  sum(rate(container_network_receive_bytes_total{namespace="robot-shop"}[2m])) by (pod)
  ```
- **Errors (Error Rate / 5xx responses)**:
  ```promql
  sum(rate(container_cpu_usage_seconds_total{namespace="robot-shop"}[2m])) by (pod)
  ```
- **Duration (Latency / Response Time)**:
  ```promql
  histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{namespace="robot-shop"}[5m])) by (le))
  ```

4. Export dashboard as JSON code:
   **Dashboard settings** → **JSON Model** → Save into `monitoring/dashboard.json`.

---

## Step 3: Collect Logs with Fluent Bit & Loki

Centralize container logs so they are easily queryable and filterable by service name and error level.

### 3.1 Install Grafana Loki
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install loki grafana/loki \
  --namespace monitoring \
  --set auth_enabled=false \
  --set singleBinary.replicas=1
```

### 3.2 Deploy Fluent Bit as Log Shipper
Add Fluent Bit Helm repository:
```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
```

Create `fluent-bit-values.yaml`:
```yaml
config:
  inputs: |
    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*_robot-shop_*.log
        Parser            docker
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        Refresh_Interval  10

  outputs: |
    [OUTPUT]
        Name        loki
        Match       kube.*
        Host        loki-gateway.monitoring.svc.cluster.local
        Port        80
        Labels      job=fluentbit, namespace=robot-shop
        Auto_Kubernetes_Labels on
```

Install Fluent Bit:
```bash
helm upgrade --install fluent-bit fluent/fluent-bit \
  --namespace monitoring \
  -f fluent-bit-values.yaml
```

### 3.3 Query Logs in Grafana
1. Open Grafana (`http://localhost:3000`).
2. Go to **Connections** → **Data sources** → Add **Loki**.
   - URL: `http://loki-gateway.monitoring.svc.cluster.local` (or `http://loki.monitoring.svc.cluster.local:3100`)
3. Go to **Explore** tab:
   - Select **Loki** data source.
   - Run LogQL query to search by service and error level:
     ```logql
     {namespace="robot-shop", app="payment"} |= "ERROR"
     ```
     ```logql
     {namespace="robot-shop", app="catalogue"}
     ```

---

## Step 4: Distributed Tracing with OpenTelemetry & Jaeger

Trace requests across microservices (e.g. `web` -> `catalogue` / `cart` -> `payment`).

### 4.1 Deploy Jaeger
```bash
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

helm upgrade --install jaeger jaegertracing/jaeger \
  --namespace monitoring \
  --set provisionDataStore.cassandra=false \
  --set allInOne.enabled=true
```

Access Jaeger UI:
```bash
kubectl port-forward -n monitoring svc/jaeger-query 16686:16686
```
Open `http://localhost:16686`.

### 4.2 OpenTelemetry Collector Config (`otel-collector.yaml`)
Create `otel-collector.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: monitoring
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    exporters:
      jaeger:
        endpoint: "jaeger-collector.monitoring.svc.cluster.local:14250"
        tls:
          insecure: true
      logging:
        loglevel: debug

    service:
      pipelines:
        traces:
          receivers: [otlp]
          exporters: [jaeger, logging]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector:0.88.0
          args: ["--config=/etc/otel/otel-collector-config.yaml"]
          volumeMounts:
            - name: config
              mountPath: /etc/otel
          ports:
            - containerPort: 4317
            - containerPort: 4318
      volumes:
        - name: config
          configMap:
            name: otel-collector-config
---
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  selector:
    app: otel-collector
  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
    - name: otlp-http
      port: 4318
      targetPort: 4318
```

Apply the collector:
```bash
kubectl apply -f otel-collector.yaml
```

### 4.3 Inspect Traces in Jaeger
1. Open `http://localhost:16686`.
2. Under **Service**, select any traced service (e.g. `web`, `cart`, `payment`).
3. Click **Find Traces**.
4. Open a single trace to see the call waterfall and timing spans across downstream services.

---

## Step 5: Set One SLO & Slack Alerting

### 5.1 Define SLI, SLO, and Error Budget Policy
Choose the **Payment Availability SLO**:
- **Service**: `payment`
- **SLI (Service Level Indicator)**: Percentage of HTTP requests to `/pay` returning HTTP status code `< 500`.
  $$\text{SLI} = \frac{\sum \text{Successful Payment Requests (HTTP } < 500\text{)}}{\sum \text{Total Payment Requests}} \times 100$$
- **SLO (Service Level Objective)**: **99.0%** availability over a 30-day rolling window.
- **Error Budget**: **1.0%** of total requests can fail over 30 days before violating the SLO.
- **Error Budget Policy**:
  - If **> 50%** error budget consumed: Feature freezes; engineers prioritize stability fixes.
  - If **100%** consumed: Immediate incident escalation; all deployments paused except critical hotfixes.

### 5.2 Configure Alert Rule in Prometheus (`slo-alert.yaml`)
Create `slo-alert.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: payment-slo-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: payment-slo
      rules:
        - alert: PaymentHighErrorRate
          expr: |
            (sum(rate(container_cpu_usage_seconds_total{pod=~"payment.*"}[2m])) == 0)
          for: 1m
          labels:
            severity: critical
            team: payments
          annotations:
            summary: "Payment service unavailable or failing SLO"
            description: "The payment service has stopped or error rate exceeds SLO threshold."
            runbook_url: "https://github.com/<your-username>/robot-shop/blob/main/docs/runbook.md"
```

Apply the rule:
```bash
kubectl apply -f slo-alert.yaml
```

### 5.3 Connect Alertmanager to Slack
In `monitoring-values.yaml`, configure the Alertmanager Slack webhook receiver:

```yaml
alertmanager:
  config:
    global:
      slack_api_url: "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
    route:
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 1h
      receiver: 'slack-notifications'
    receivers:
      - name: 'slack-notifications'
        slack_configs:
          - channel: '#alerts'
            send_resolved: true
            title: '{{ .CommonAnnotations.summary }}'
            text: "{{ .CommonAnnotations.description }}\nRunbook: {{ .CommonAnnotations.runbook_url }}"
```

Update Alertmanager:
```bash
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f monitoring-values.yaml
```

---

## Step 6: Incident Drill & Blameless Postmortem

Simulate a production incident, verify alerting and observability, mitigate, and write a postmortem.

### 6.1 Triggering the Incident
Induce a failure by scaling down the `payment` service or corrupting its dependency:
```bash
# Break payment service:
kubectl scale deployment payment --replicas=0 -n robot-shop
```

### 6.2 Observation Phase
1. **Watch Traffic & Load**:
   `load-gen` will start reporting errors when attempting checkouts.
2. **Prometheus Alert**:
   Within 1–2 minutes, Prometheus alerts will enter `FIRING` state.
3. **Slack Alert Notification**:
   Alertmanager dispatches the alert to your designated Slack channel with the linked runbook.
4. **Grafana & Logs**:
   Inspect the RED dashboard and search Loki for errors:
   `{namespace="robot-shop"} |= "connection refused"`

### 6.3 Mitigation
Mitigate the incident by restoring the service replica count:
```bash
kubectl scale deployment payment --replicas=1 -n robot-shop
```
Verify the pods return to `Running` and the Slack alert resolves.

### 6.4 Write the Blameless Postmortem
Create `docs/postmortem-incident-001.md`:

```markdown
# Blameless Postmortem: Incident-001 (Payment Service Outage)

**Date**: 2026-10-23  
**Authors**: [Your Name]  
**Status**: Resolved  
**Severity**: Critical  

## Summary
On 2026-10-23, the `payment` microservice experienced a total outage lasting 4 minutes. Users were unable to complete checkouts during this time. The issue was triggered by a scaled-down deployment replica count, caught by Prometheus SLO alerts, and mitigated by restoring healthy replicas.

## Impact
- **Downtime**: ~4 minutes.
- **Affected Users**: ~120 synthetic transactions in `load-gen`.
- **SLO Impact**: Consumed 0.08% of the 30-day payment availability error budget.

## Timeline (UTC)
- `17:00` - Incident triggered (replicas set to 0).
- `17:01` - `load-gen` reported checkout connection failures.
- `17:02` - Alert `PaymentHighErrorRate` fired in Prometheus and sent alert to Slack `#alerts`.
- `17:03` - On-call engineer followed runbook and scaled deployment back to 1.
- `17:04` - Pod became Ready; traffic resumed normal operation.
- `17:05` - Alert resolved automatically.

## Root Cause
The `payment` deployment had its active replica count scaled to 0, resulting in no active endpoints behind the `payment` service.

## Action Items & Lessons Learned
1. Add PodDisruptionBudgets (PDB) to ensure at least 1 replica of `payment` is always maintained.
2. Configure horizontal pod autoscaling (HPA) with a minimum replica count of 2.
3. Improve synthetic alerting to include proactive synthetic health check probes.
```

---

## Deliverables & Hand-In Checklist

- [ ] **Dashboard as Code**: Exported Grafana JSON dashboard (`dashboard.json`).
- [ ] **Logging & Tracing Config**: Fluent Bit configuration for Loki & OpenTelemetry/Jaeger manifest.
- [ ] **SLO Document & Policy**: Availability target (99%), error budget calculation, and policy.
- [ ] **Runbook**: Linked in the alert annotations with clear remediation steps.
- [ ] **Live Alert Verification**: Screenshot of Slack alert firing and resolving.
- [ ] **Blameless Postmortem**: Complete document reflecting timeline, root cause, and action items.

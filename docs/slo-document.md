# SLO Document: Payment Service Availability

## Service: `payment` (Stan's Robot Shop)

---

## SLI (Service Level Indicator)

**Definition**: The proportion of time the `payment` service is running and processing requests.

$$
\text{SLI} = \frac{\text{Minutes payment pod is Running and Ready}}{\text{Total Minutes in Window}} \times 100
$$

**Measurement Method**: Prometheus monitors `container_cpu_usage_seconds_total` for `payment.*` pods in the `robot-shop` namespace. A zero rate over 2 minutes indicates the service is down.

---

## SLO (Service Level Objective)

| Parameter         | Value                    |
|-------------------|--------------------------|
| **Target**        | 99.0% availability       |
| **Window**        | 30-day rolling           |
| **Measurement**   | Per-minute availability  |

### What This Means
Over a 30-day period (43,200 minutes):
- **Allowed downtime**: 432 minutes (~7.2 hours)
- **Error budget**: 1.0% of total minutes

---

## Error Budget Policy

| Budget Consumed | Action Required |
|-----------------|-----------------|
| **< 25%**       | Normal operations. Feature development proceeds as planned. |
| **25% – 50%**   | Increased monitoring. Team reviews recent deployments for risk. |
| **50% – 75%**   | Feature freeze for `payment` service. Engineers prioritize stability and reliability fixes. |
| **75% – 100%**  | Deployment freeze. Only critical hotfixes and rollbacks allowed. Incident review meeting triggered. |
| **100% (Exhausted)** | Immediate escalation. All deployments paused. Full incident postmortem required before resuming releases. |

---

## Alert Rules

### Critical: `PaymentHighErrorRate`
- **Condition**: Payment service CPU rate drops to 0 (service is down) for > 1 minute
- **Severity**: `critical`
- **Action**: Page on-call engineer, notify `#alerts` Slack channel
- **Runbook**: [runbook.md](https://github.com/your-username/robot-shop/blob/main/docs/runbook.md)

### Warning: `PaymentPodRestarts`
- **Condition**: Payment pod restarts > 2 times in 10 minutes
- **Severity**: `warning`
- **Action**: Notify `#alerts` Slack channel for awareness

---

## Prometheus Alert Configuration

Alert rules are defined in [`monitoring/slo-alert.yaml`](../monitoring/slo-alert.yaml) as a `PrometheusRule` CRD, automatically picked up by the `kube-prometheus-stack` Prometheus operator.

---

## Stakeholders

| Role | Responsibility |
|------|---------------|
| **On-Call Engineer** | Acknowledge alert, follow runbook, mitigate incident |
| **Team Lead** | Review error budget consumption weekly |
| **Product Owner** | Approve feature freeze when budget > 50% consumed |

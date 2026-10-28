# Blameless Postmortem: Incident-001 (Payment Service Outage)

**Date**: 2026-10-28
**Authors**: Adepegba Isaiah  
**Status**: Resolved  
**Severity**: Critical

## Summary

On 2026-10-28, the `payment` microservice experienced a total outage lasting 4 minutes. Users were unable to complete checkouts during this time. The issue was triggered by a scaled-down deployment replica count, caught by Prometheus SLO alerts, and mitigated by restoring healthy replicas.

## Impact

- **Downtime**: ~4 minutes.
- **Affected Users**: ~120 synthetic transactions in `load-gen`.
- **SLO Impact**: Consumed 0.08% of the 30-day payment availability error budget.

## Timeline (WAT)

- `04:59` - Incident triggered (replicas set to 0).
- `05:00` - `load-gen` reported checkout connection failures.
- `05:01` - Alert `PaymentHighErrorRate` fired in Prometheus.
- `05:02` - I followed runbook and scaled deployment back to 1.
- `05:03` - Pod became Ready; traffic resumed normal operation.
- `05:04` - Alert resolved automatically.

## Root Cause

The `payment` deployment had its active replica count scaled to 0, resulting in no active endpoints behind the `payment` service.


## What was done well?

- We successfully deployed a monitoring stack (Prometheus & Grafana) within the first two weeks.
- The SLO alert triggered automatically, reducing our detection time significantly.
- The runbook was clear and allowed for a swift manual mitigation.

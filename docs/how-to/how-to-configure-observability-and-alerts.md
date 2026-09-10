# How to Configure Observability, Server Metrics, and Fleet Alerts

This guide explains how to monitor Ejenix control plane health, export Prometheus metrics, track client-side status events, and set up production alerts.

---

## 1. Control Plane Observability

The Ejenix control plane exposes HTTP endpoints for health monitoring, readiness probes, and Prometheus metrics.

### Health & Readiness Endpoints

```bash
# General health probe (Liveness)
curl -fsS http://localhost:8080/v1/health

# Output:
# {"status":"ok","version":"0.1.0"}
```

### Prometheus Metrics Endpoint (`/v1/metrics`)

The control plane exposes metrics in Prometheus text format at `/v1/metrics`:

```bash
curl -fsS http://localhost:8080/v1/metrics
```

*Exposed Prometheus Metrics*:

| Metric Name | Type | Description |
|---|---|---|
| `ejenix_http_requests_total` | Counter | Total HTTP requests handled by method, route, and status code. |
| `ejenix_http_request_duration_seconds` | Histogram | Request latency distribution. |
| `ejenix_active_apps` | Gauge | Total registered applications on the control plane. |
| `ejenix_promotions_total` | Counter | Total patch promotions executed by app ID, channel, and environment. |
| `ejenix_rollbacks_total` | Counter | Total rollbacks executed by app ID, channel, and environment. |
| `ejenix_bundles_total` | Counter | Total uploaded bundles stored on the server. |

---

## 2. Client-Side Telemetry (`onStatus`)

Because Ejenix staged rollouts operate on-device with zero telemetry to protect user privacy, client applications should forward `onStatus` events to their existing observability platform (e.g., Sentry, Datadog, Firebase Crashlytics) where user consent is already managed.

### Wiring `onStatus` to Telemetry

```dart
EjenixPatchView(
  appId: 'com.acme.shop',
  channel: 'home',
  env: 'production',
  ...
  onStatus: (status) {
    // Forward status events to analytics/Sentry
    Sentry.addBreadcrumb(
      Breadcrumb(
        category: 'ejenix.ota',
        message: 'Ejenix status changed to: $status',
        level: status == PatchStatus.rolledBack || status == PatchStatus.rejected
            ? SentryLevel.warning
            : SentryLevel.info,
      ),
    );

    if (status == PatchStatus.rolledBack) {
      Sentry.captureMessage(
        'Ejenix patch auto-rolled back on device due to launch crashes',
        level: SentryLevel.error,
      );
    }
  },
)
```

---

## 3. Recommended Production Alerting Rules

### Rule 1: High Control Plane Error Rate (Server-Side Alert)

Alert if the control plane HTTP 5xx error rate exceeds 1% over a 5-minute window:

```yaml
# Prometheus AlertRule
- alert: EjenixControlPlaneHighErrorRate
  expr: sum(rate(ejenix_http_requests_total{status=~"5.."}[5m])) / sum(rate(ejenix_http_requests_total[5m])) * 100 > 1
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Ejenix control plane HTTP 5xx error rate > 1%"
```

### Rule 2: High Patch Rejection / Rollback Rate (Client-Side Alert)

Alert in Sentry/Datadog if `PatchStatus.rolledBack` or `PatchStatus.rejected` events spike following a patch promotion:

- **Trigger**: More than 10 `rolledBack` events within 10 minutes of promoting a new bundle.
- **Action**: Inspect client crash logs, execute `ejenix rollback`, and investigate patch source code.

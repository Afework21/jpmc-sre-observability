# Prometheus and Grafana Monitoring

## Overview

This project demonstrates application-level observability for a containerized Python Payment API running on Kubernetes.

The monitoring flow is:

```text
Payment API
    |
    | exposes /metrics
    v
Kubernetes Service
    |
    v
ServiceMonitor
    |
    v
Prometheus
    |
    v
Grafana
```

## Application Metrics

The Python Flask Payment API exposes Prometheus-compatible metrics through:

```text
/metrics
```

Prometheus periodically scrapes this endpoint and stores the resulting time-series metrics.

This enables monitoring of application behavior such as:

- HTTP request volume
- Request latency
- HTTP status codes
- Application errors
- Process metrics
- Service availability

## Kubernetes ServiceMonitor

The repository contains:

```text
k8s/monitoring/payment-api-monitor.yaml
```

The manifest uses the Prometheus Operator `ServiceMonitor` custom resource.

Configuration:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: payment-api
  labels:
    release: monitoring
spec:
  selector:
    matchLabels:
      app: payment-api
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

## How Service Discovery Works

The `ServiceMonitor` searches for a Kubernetes Service containing:

```yaml
app: payment-api
```

Prometheus then connects to the service's named:

```text
http
```

port and scrapes:

```text
/metrics
```

every:

```text
15 seconds
```

This avoids manually configuring individual pod IP addresses.

## Prometheus

Prometheus provides the metrics collection and time-series storage layer.

Example monitoring flow:

```text
Kubernetes Pod
     |
     v
Payment API
     |
     v
/metrics
     |
     v
Kubernetes Service
     |
     v
ServiceMonitor
     |
     v
Prometheus
```

Prometheus queries can then be used to investigate service health and application behavior.

## Grafana

Grafana provides the visualization layer for Prometheus metrics.

Typical dashboard panels for the Payment API can include:

- Request rate
- Error rate
- Request latency
- HTTP 5xx responses
- Pod availability
- CPU utilization
- Memory utilization
- Kubernetes pod restarts

This supports rapid identification of abnormal application behavior.

## Failure Simulation

The Payment API contains an `/error` endpoint that generates a simulated HTTP 500 response.

This allows observability testing by intentionally generating application failures.

Example:

```text
GET /error
        |
        v
HTTP 500
        |
        v
Prometheus Metrics
        |
        v
Grafana Visualization
```

This type of controlled failure is useful for validating monitoring and incident-detection capabilities.

## SRE Monitoring Model

The monitoring design focuses on core service reliability signals.

### Traffic

Monitor incoming request volume.

### Errors

Monitor HTTP failures, particularly HTTP 5xx responses.

### Latency

Monitor request response time and identify degradation.

### Saturation

Monitor resource utilization such as CPU and memory.

These signals help engineers determine whether a service is healthy and whether users are experiencing degradation.

## Example Reliability Indicators

Potential Service Level Indicators (SLIs) include:

```text
Availability
Successful requests / Total requests

Error Rate
HTTP 5xx requests / Total requests

Latency
Percentage of requests completed within the target response time
```

Example SLO:

```text
99.9% successful Payment API requests
within the defined measurement window
```

## Troubleshooting Workflow

When an alert or dashboard indicates a problem:

```text
Alert / Dashboard
       |
       v
Check Error Rate
       |
       v
Check Request Latency
       |
       v
Check Kubernetes Pods
       |
       v
Check Application Logs
       |
       v
Check CPU / Memory
       |
       v
Correlate Metrics and Traces
       |
       v
Identify Root Cause
```

This demonstrates a practical SRE approach to production troubleshooting.

## Observability Layers

The broader lab combines multiple observability signals:

```text
Metrics
  |
  +-- Prometheus
  +-- Grafana

Tracing
  |
  +-- OpenTelemetry

Application / Infrastructure Monitoring
  |
  +-- Dynatrace

Kubernetes
  |
  +-- Pod health
  +-- Deployment health
  +-- Resource utilization
```

Together, these technologies provide visibility across the application and Kubernetes stack.

## Security

No monitoring API tokens, Dynatrace credentials, AWS credentials, or private environment configuration are stored in this repository.

Environment-specific credentials should be managed using secure secret-management mechanisms rather than committed to Git.
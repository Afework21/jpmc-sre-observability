# SRE Observability: Prometheus, Grafana, OpenTelemetry & Dynatrace

A hands-on Site Reliability Engineering observability project demonstrating **metrics, dashboards, distributed tracing, Kubernetes monitoring, and production-style troubleshooting** for a containerized Python Payment API.

This repository is part of a broader SRE/DevOps lab covering infrastructure, CI/CD, Kubernetes, and observability.

## Observability Architecture

```text
                         Payment API
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
          /metrics      OpenTelemetry     Kubernetes
             |             Tracing          Health
             v                |                |
       ServiceMonitor         v                |
             |          OTLP Exporter           |
             v                                 |
        Prometheus                             |
             |                                 |
             v                                 |
          Grafana                              |
                                               |
                    Dynatrace Operator <-------+
                           |
                    +------+------+
                    |             |
                    v             v
                 OneAgent     ActiveGate
                    |
                    v
                 Dynatrace
```

## Project Goals

The lab demonstrates how an SRE can observe and troubleshoot a Kubernetes-hosted service using multiple telemetry signals.

Key areas include:

- Prometheus metrics
- Grafana dashboards
- Kubernetes ServiceMonitor
- HTTP error monitoring
- Request-rate monitoring
- OpenTelemetry tracing
- Dynatrace Kubernetes instrumentation
- Dynatrace Operator
- ActiveGate
- OneAgent injection
- Kubernetes troubleshooting
- CrashLoopBackOff investigation
- Root-cause analysis

## Repository Structure

```text
jpmc-sre-observability/
│
├── README.md
├── .gitignore
│
├── docs/
│   └── prometheus-grafana.md
│
├── dynatrace/
│   └── troubleshooting.md
│
├── grafana/
│   └── payment-api-dashboard.json
│
└── k8s/
    └── monitoring/
        └── payment-api-monitor.yaml
```

## Payment API Metrics

The Python Flask Payment API exposes Prometheus-compatible metrics at:

```text
/metrics
```

The application defines the metric:

```text
http_requests_total
```

with the label:

```text
status
```

This enables monitoring requests by HTTP response status.

Example PromQL:

```promql
sum(http_requests_total)
```

Request rate:

```promql
sum(rate(http_requests_total[5m]))
```

HTTP 500 failures:

```promql
sum(http_requests_total{status="500"})
```

HTTP 500 error rate:

```promql
sum(rate(http_requests_total{status="500"}[5m]))
```

## Prometheus ServiceMonitor

Kubernetes monitoring is configured using a Prometheus Operator `ServiceMonitor`.

File:

```text
k8s/monitoring/payment-api-monitor.yaml
```

The ServiceMonitor discovers the Payment API service using:

```yaml
matchLabels:
  app: payment-api
```

Prometheus scrapes:

```text
/metrics
```

every:

```text
15 seconds
```

This provides automatic Kubernetes service discovery instead of relying on static pod addresses.

## Grafana Dashboard

An importable Grafana dashboard is provided at:

```text
grafana/payment-api-dashboard.json
```

The dashboard includes panels for:

- Total HTTP requests
- Request rate
- HTTP 500 errors
- Requests grouped by HTTP status
- HTTP 500 error rate

Dashboard flow:

```text
Payment API
     |
     v
/metrics
     |
     v
Prometheus
     |
     v
PromQL
     |
     v
Grafana Dashboard
```

## Failure Simulation

The Payment API contains an `/error` endpoint that can intentionally generate an HTTP 500 response.

This provides a controlled way to test monitoring behavior.

```text
Request /error
      |
      v
HTTP 500 generated
      |
      v
http_requests_total{status="500"}
      |
      v
Prometheus
      |
      v
Grafana
```

This type of controlled failure testing helps validate whether monitoring detects application errors correctly.

## OpenTelemetry

The Payment API also includes OpenTelemetry instrumentation.

The tracing flow is designed around:

```text
Application
     |
     v
OpenTelemetry SDK
     |
     v
OTLP Exporter
     |
     v
Collector / Trace Backend
```

This demonstrates how distributed tracing can complement metrics during incident investigation.

Metrics answer questions such as:

> Is the service failing?

Tracing helps investigate:

> Where in the request path is the failure occurring?

## Dynatrace Kubernetes Monitoring

Dynatrace was also integrated into the Kubernetes lab.

Components tested included:

- Dynatrace Operator
- DynaKube
- Dynatrace Webhook
- CSI
- ActiveGate
- OpenTelemetry Collector
- OneAgent

The DynaKube configuration used the:

```text
cloudNativeFullStack
```

deployment model.

## Application Injection

Dynatrace application injection was successfully verified.

The Payment API pod reported injection indicators including:

```text
dynakube.dynatrace.com/injected: true
metadata-enrichment.dynatrace.com/injected: true
oneagent.dynatrace.com/injected: true
```

The application container also loaded the OneAgent process library.

This confirmed that the application-level injection mechanism was functioning.

## Dynatrace Troubleshooting Case

During the lab, the node-level OneAgent entered:

```text
CrashLoopBackOff
```

Instead of treating this as a generic installation failure, the problem was investigated layer by layer.

```text
Check Pods
    |
    v
Identify CrashLoopBackOff
    |
    v
Inspect Pod
    |
    v
Analyze Logs
    |
    v
Validate Operator
    |
    v
Validate ActiveGate
    |
    v
Validate Application Injection
    |
    v
Inspect Node Filesystem
    |
    v
Check Host Security Features
    |
    v
Identify Environment Limitation
```

The investigation identified a compatibility issue involving the node-level OneAgent and the local Docker Desktop / WSL2 Kubernetes filesystem environment.

Application injection succeeded, while full node-level OneAgent operation was not achieved in that local environment.

The troubleshooting details are documented in:

```text
dynatrace/troubleshooting.md
```

## Troubleshooting Methodology

The project demonstrates a layered SRE troubleshooting approach:

```text
User Request
     |
     v
Application
     |
     v
Metrics / Traces
     |
     v
Kubernetes Pod
     |
     v
Kubernetes Node
     |
     v
Observability Agent
     |
     v
Observability Backend
```

Each layer should be validated independently before assigning root cause.

Useful Kubernetes commands include:

```bash
kubectl get pods
```

```bash
kubectl describe pod <pod-name>
```

```bash
kubectl logs <pod-name>
```

```bash
kubectl get services
```

```bash
kubectl get servicemonitor
```

## SRE Golden Signals

The observability design aligns with common SRE service-health signals.

### Traffic

How much demand is reaching the application?

Example:

```promql
sum(rate(http_requests_total[5m]))
```

### Errors

How many requests are failing?

Example:

```promql
sum(rate(http_requests_total{status="500"}[5m]))
```

### Latency

How long are requests taking?

Latency instrumentation can be expanded using Prometheus histograms and distributed traces.

### Saturation

How close are application and infrastructure resources to their limits?

Typical signals include:

- CPU
- Memory
- Pod resource utilization
- Node resource utilization

## Example SLI and SLO

Example availability SLI:

```text
Successful Requests
-------------------
Total Requests
```

Example target:

```text
Payment API Availability SLO: 99.9%
```

The metrics collected by Prometheus can be used as the foundation for SLI/SLO calculations and alerting.

## Incident Investigation Example

A production-style investigation could follow:

```text
Grafana alert indicates increased HTTP 500s
                |
                v
Check Prometheus error-rate metric
                |
                v
Identify affected Kubernetes workload
                |
                v
Check pod status and restarts
                |
                v
Inspect application logs
                |
                v
Review OpenTelemetry traces
                |
                v
Check infrastructure monitoring
                |
                v
Correlate telemetry
                |
                v
Identify root cause
                |
                v
Remediate and verify recovery
```

This demonstrates the relationship between monitoring and production incident response.

## Security

This public repository intentionally excludes:

- Dynatrace API tokens
- Dynatrace data-ingest tokens
- ActiveGate authentication tokens
- Tenant-specific Dynatrace URLs
- AWS access keys
- AWS secret keys
- Kubernetes credentials
- Private keys
- Environment files

Secrets should be stored using appropriate credential or secret-management systems rather than committed to Git.

## Key SRE Skills Demonstrated

This project demonstrates hands-on experience with:

- Observability architecture
- Prometheus
- PromQL
- Grafana
- Kubernetes monitoring
- Prometheus Operator
- ServiceMonitor
- OpenTelemetry
- Dynatrace
- Dynatrace Operator
- ActiveGate
- OneAgent
- Kubernetes debugging
- CrashLoopBackOff troubleshooting
- Log analysis
- Root-cause analysis
- SLI/SLO concepts
- Production incident investigation

## Related Portfolio Projects

### Terraform / AWS EKS

```text
jpmc-sre-terraform-eks
```

Demonstrates Infrastructure as Code for AWS networking and Amazon EKS.

### CI/CD Pipeline

```text
jpmc-sre-cicd-pipeline
```

Demonstrates:

```text
GitHub
   |
   v
Jenkins
   |
   v
Docker
   |
   v
Amazon ECR
   |
   v
Amazon EKS
   |
   v
Kubernetes
```

### Observability

This repository completes the workflow:

```text
Infrastructure
     |
     v
CI/CD
     |
     v
Application
     |
     v
Kubernetes
     |
     v
Observability
     |
     v
Incident Detection & Troubleshooting
```

## Portfolio Architecture

Together, the three projects demonstrate an end-to-end SRE/DevOps workflow:

```text
Terraform
    |
    v
AWS VPC + EKS
    |
    v
GitHub
    |
    v
Jenkins CI/CD
    |
    v
Docker
    |
    v
Amazon ECR
    |
    v
Kubernetes Deployment
    |
    +--------------------+
    |                    |
    v                    v
Prometheus          OpenTelemetry
    |                    |
    v                    v
Grafana             Distributed Tracing
    |
    +--------------------+
             |
             v
         Dynatrace
             |
             v
Monitoring + Troubleshooting
```

## Disclaimer

This repository is a personal hands-on SRE/DevOps lab and portfolio project.

It does not contain JPMorgan Chase proprietary source code, production configuration, customer information, internal documentation, credentials, or confidential infrastructure details.
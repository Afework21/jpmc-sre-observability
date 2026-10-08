# Dynatrace Kubernetes Troubleshooting

## Overview

This document describes the Dynatrace Kubernetes observability implementation and troubleshooting performed in the SRE lab.

The environment included:

- Docker Desktop
- WSL2
- Kubernetes
- Dynatrace Operator
- DynaKube
- OneAgent
- ActiveGate
- OpenTelemetry
- Python Flask Payment API

The objective was to instrument the Payment API and investigate Kubernetes infrastructure and application observability.

## Architecture

```text
Payment API
     |
     v
Kubernetes Pod
     |
     +--------------------+
     |                    |
     v                    v
OneAgent Injection   OpenTelemetry
     |                    |
     v                    v
Dynatrace Operator   Trace Export
     |
     v
ActiveGate
     |
     v
Dynatrace
```

## Dynatrace Operator

The Dynatrace Operator was deployed to Kubernetes using Helm.

The operator manages Dynatrace components inside the cluster and coordinates observability configuration.

Components observed during the lab included:

```text
Dynatrace Operator
Dynatrace Webhook
CSI Driver
ActiveGate
OpenTelemetry Collector
OneAgent
```

The DynaKube deployment used the:

```text
cloudNativeFullStack
```

configuration model.

## Application Injection

The Payment API pod was successfully instrumented by the Dynatrace admission process.

The following injection indicators were verified:

```text
dynakube.dynatrace.com/injected: true
metadata-enrichment.dynatrace.com/injected: true
oneagent.dynatrace.com/injected: true
```

The application container also contained the OneAgent preload configuration:

```text
LD_PRELOAD=/opt/dynatrace/oneagent-paas/agent/lib64/liboneagentproc.so
```

The OneAgent library was verified as loaded inside the application process.

This confirmed that application-level injection was functioning.

## ActiveGate

ActiveGate was successfully deployed inside the Kubernetes environment.

The ActiveGate instance established upstream connectivity to the Dynatrace environment.

During troubleshooting, ActiveGate was healthy, but the environment reported:

```text
Active agents: 0
```

This indicated that the problem was not simply ActiveGate availability.

Investigation therefore continued at the OneAgent/node level.

## OneAgent Problem

The node-level OneAgent entered:

```text
CrashLoopBackOff
```

The failure was investigated through Kubernetes pod status and OneAgent logs.

The important error was related to determining the underlying host volume path.

Conceptually, the failure occurred because OneAgent expected a host filesystem layout that did not match the Docker Desktop / WSL2 Kubernetes environment.

The node filesystem included an overlay-based container filesystem and Docker Desktop-specific mounts.

As a result, OneAgent could not correctly determine the expected host root-volume mapping.

## Root Cause

The issue was identified as an environment compatibility limitation involving:

```text
Docker Desktop
        +
WSL2
        +
Docker Desktop Kubernetes
        +
Dynatrace node-level OneAgent
```

The application injection mechanism worked, but the node-level OneAgent could not initialize correctly against the Docker Desktop host filesystem layout.

This distinction was important:

```text
Application Injection     SUCCESS
ActiveGate                RUNNING
Dynatrace Operator        RUNNING
CSI                       RUNNING
Node OneAgent             FAILED
```

The failure therefore did not indicate that the entire Dynatrace deployment was incorrectly configured.

## AppArmor Observation

During troubleshooting, AppArmor availability was also checked.

The environment reported AppArmor as disabled.

This affected node-level configuration collection and represented another difference between the local Docker Desktop environment and a typical production Linux Kubernetes node.

## Troubleshooting Process

The investigation followed a structured SRE workflow:

```text
Check Kubernetes Pods
        |
        v
Identify CrashLoopBackOff
        |
        v
Inspect Pod Description
        |
        v
Inspect Container Logs
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
Inspect Host Filesystem
        |
        v
Check AppArmor
        |
        v
Identify Environment Limitation
```

## Useful Kubernetes Commands

Check Dynatrace components:

```bash
kubectl get pods -n dynatrace
```

Inspect a failing pod:

```bash
kubectl describe pod <pod-name> -n dynatrace
```

Review container logs:

```bash
kubectl logs <pod-name> -n dynatrace
```

Check application injection:

```bash
kubectl describe pod <payment-api-pod>
```

Inspect environment variables:

```bash
kubectl exec <payment-api-pod> -- env
```

These commands help determine whether a failure is occurring at the application, agent, operator, networking, or Kubernetes node layer.

## Key SRE Lesson

A monitoring component reporting failure does not automatically mean the application or entire observability platform is unavailable.

The troubleshooting process separated the system into layers:

```text
Application
     |
Instrumentation
     |
Kubernetes
     |
Dynatrace Operator
     |
OneAgent
     |
ActiveGate
     |
Dynatrace Platform
```

Each layer was validated independently.

This allowed the problem to be narrowed to the node-level OneAgent interaction with the local Docker Desktop / WSL2 environment.

## Production Consideration

Docker Desktop Kubernetes is useful for development and hands-on labs, but its node filesystem and operating-system integration differ from standard Linux Kubernetes worker nodes.

A production deployment should validate Dynatrace against the actual Kubernetes platform and supported node operating system.

Examples could include:

```text
Amazon EKS
Azure Kubernetes Service
Google Kubernetes Engine
Standard Linux Kubernetes nodes
```

Compatibility should always be verified against the Dynatrace support requirements for the specific platform and version.

## OpenTelemetry

The Payment API also contains OpenTelemetry instrumentation.

This provides an additional observability path for distributed tracing.

Conceptually:

```text
Payment API
     |
     v
OpenTelemetry SDK
     |
     v
OTLP Exporter
     |
     v
Trace Backend
```

In the lab application, the tracing configuration was designed around an OTLP-compatible collector/backend.

The OpenTelemetry instrumentation is separate from the Dynatrace node-level OneAgent filesystem issue.

## Security

No Dynatrace API tokens, data-ingest tokens, ActiveGate authentication tokens, tenant-specific URLs, AWS credentials, or other secrets are stored in this repository.

Credentials used during observability testing should be stored securely and rotated if exposed.

## Outcome

The lab successfully demonstrated:

- Dynatrace Operator deployment
- DynaKube configuration
- Kubernetes admission/injection
- Application OneAgent library injection
- ActiveGate deployment and connectivity
- OpenTelemetry instrumentation
- Kubernetes troubleshooting
- CrashLoopBackOff investigation
- Log-based root-cause analysis
- Host filesystem investigation
- Identification of a Docker Desktop / WSL2 compatibility limitation

Full node-level OneAgent operation was **not achieved** in the local Docker Desktop Kubernetes environment.

This limitation is documented intentionally rather than representing the lab as fully operational.

That distinction demonstrates an important SRE practice: document observed system behavior accurately and separate successful components from unresolved or environment-specific limitations.
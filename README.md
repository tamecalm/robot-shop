# Stan's Robot Shop: EKS Observability & SRE Platform

This repository contains the containerized deployment of Stan's Robot Shop microservices on Amazon EKS, equipped with a full observability stack (Prometheus, Grafana, Loki, Jaeger) and Site Reliability Engineering (SRE) operational policies.

## What Was Built

1. **Microservices Deployment:** Deployed the 10+ polyglot microservices via customized Helm charts, removing proprietary Instana agent dependencies and embedding a synthetic `load` generator to maintain continuous traffic.
2. **RED Metrics Dashboard:** Installed `kube-prometheus-stack` and exported an automated Grafana dashboard monitoring Rate, Errors, and Duration across all frontend and backend services.
3. **Centralized Logging:** Deployed Grafana Loki with a cluster-wide Fluent Bit DaemonSet parsing containerd CRI logs for real-time error inspection across all pods.
4. **Distributed Tracing:** Configured OpenTelemetry Collector and Jaeger all-in-one to trace end-to-end user transactions across the checkout flow (`web` -> `catalogue` -> `cart` -> `payment`).
5. **SLO Alerting & Incident Response:** Implemented Prometheus alerting rules for Payment and Web availability SLO breaches, documented standard operating procedures in `RUNBOOK.md`, and executed an intentional outage drill documented in `POSTMORTEM.md`.

## SRE & Operational Documentation

- **SLO Policy:** [SLO_AND_ERROR_BUDGET_POLICY.md](SLO_AND_ERROR_BUDGET_POLICY.md) defines SLIs, SLO targets (99.0% Web, 99.5% Payment), and burn rate escalation tiers.
- **Operational Runbook:** [RUNBOOK.md](RUNBOOK.md) provides triage steps and recovery commands for on-call engineers.
- **Incident Postmortem:** [POSTMORTEM.md](POSTMORTEM.md) documents a simulated payment outage drill, root cause analysis (5 Whys), and error budget consumption math.

## Deployment & Verification

### 1. Deploy Robot Shop
```bash
helm install robot-shop ./K8s/helm -n robot-shop --create-namespace
```

### 2. Deploy Observability Stack
```bash
# RED Metrics Dashboard
kubectl apply -f monitoring/dashboards/dashboard-configmap.yaml

# Centralized Logging (Loki & Fluent Bit)
kubectl apply -f monitoring/logging/loki.yaml
kubectl apply -f monitoring/logging/loki-datasource.yaml
kubectl apply -f monitoring/logging/fluent-bit.yaml

# Distributed Tracing (Jaeger & OpenTelemetry)
kubectl apply -f monitoring/tracing/jaeger.yaml
kubectl apply -f monitoring/tracing/otel-collector.yaml
kubectl apply -f monitoring/tracing/trace-job.yaml

# SLO Alert Rules
kubectl apply -f monitoring/alerts/robot-shop-slo-rules.yaml
```

### 3. Verify Health
```bash
kubectl get pods -n robot-shop
kubectl get pods -n monitoring
```

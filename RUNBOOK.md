# Operational Runbook: Robot Shop Alert Response

This runbook defines standard operating procedures for triaging, mitigating, and resolving alerts fired by Prometheus and Alertmanager for the Stan's Robot Shop platform.

---

## Quick Reference Table

| Alert Name | Severity | SLO Impact | Target Service | Runbook Section |
| :--- | :--- | :--- | :--- | :--- |
| `RobotShopPaymentAvailabilitySLOBreached` | **Critical (P1)** | 99.5% Availability | `payment` | [Section 1](#robotshoppaymentavailabilityslobreached) |
| `RobotShopWebAvailabilitySLOBreached` | **Critical (P1)** | 99.0% Availability | `web` | [Section 2](#robotshopwebavailabilityslobreached) |
| `RobotShopHighCrashLoopRate` | **Warning (P2)** | Pod Stability | Any microservice | [Section 3](#robotshophighcrashlooprate) |

---

<a id="robotshoppaymentavailabilityslobreached"></a>
## 1. Alert: RobotShopPaymentAvailabilitySLOBreached

### 1.1 Overview & Severity
- **Severity:** `Critical (P1)`
- **Impact:** Customers cannot complete checkout or process payments. Severe revenue loss and immediate error budget consumption.
- **Trigger Condition:** Payment pod readiness is `0` or absent for > 30 seconds.

### 1.2 Step 1: Initial Triage (0 - 3 minutes)
1. Check the status of payment pods:
   ```bash
   kubectl get pods -n robot-shop -l service=payment -o wide
   ```
2. Inspect pod events and readiness probe failure reasons:
   ```bash
   kubectl describe pod -n robot-shop -l service=payment
   ```
3. Open Grafana RED Metrics Dashboard (`http://<SERVER_IP>:30300`) and navigate to the **Payment Service** panels to check Error Rate (%) and Request Volume.

### 1.3 Step 2: Diagnostic Inspection (3 - 7 minutes)
1. Query centralized logs in Loki via CLI or Grafana Explore:
   ```bash
   # From EC2 or Loki pod:
   kubectl exec -n monitoring deploy/loki -- wget -qO- 'http://localhost:3100/loki/api/v1/query_range?query=%7Bservice%3D%22payment%22%7D+%7C%3D+%22error%22&limit=20'
   ```
2. Verify downstream dependencies:
   - Check if `rabbitmq` is running and accepting AMQP connections:
     ```bash
     kubectl get pods -n robot-shop -l service=rabbitmq
     kubectl logs -n robot-shop -l service=rabbitmq --tail=30
     ```
3. Check Jaeger Distributed Traces:
   - Identify whether traces for `POST /pay/:partner` are timing out or returning 500 status codes.

### 1.4 Step 3: Mitigation & Immediate Containment (7 - 12 minutes)
- **Scenario A: Transient pod lockup or memory leak**
  Restart the payment deployment to restore service immediately:
  ```bash
  kubectl rollout restart deployment payment -n robot-shop
  kubectl rollout status deployment payment -n robot-shop
  ```
- **Scenario B: RabbitMQ connection lost**
  Restart RabbitMQ dependency:
  ```bash
  kubectl rollout restart deployment rabbitmq -n robot-shop
  ```
- **Scenario C: Faulty release or configuration drift**
  Roll back the Helm release to the last known good revision:
  ```bash
  helm history robot-shop -n robot-shop
  helm rollback robot-shop <PREVIOUS_REVISION> -n robot-shop
  ```

### 1.5 Step 4: Verification & Recovery Confirmation
1. Verify pod is `1/1 Running`:
   ```bash
   kubectl get pods -n robot-shop -l service=payment
   ```
2. Send synthetic payment transaction to verify end-to-end processing:
   ```bash
   kubectl exec -n robot-shop deploy/load -- curl -s -X POST http://payment:8080/pay/partner-57 -H "Content-Type: application/json" -d '{"cartId":"test"}'
   ```
3. Verify the alert in Prometheus / Alertmanager has transitioned from `FIRING` to `RESOLVED`:
   ```bash
   kubectl exec -n monitoring deploy/loki -- wget -qO- 'http://prometheus-operated.monitoring.svc.cluster.local:9090/api/v1/alerts' | grep -i payment
   ```

---

<a id="robotshopwebavailabilityslobreached"></a>
## 2. Alert: RobotShopWebAvailabilitySLOBreached

### 2.1 Overview & Severity
- **Severity:** `Critical (P1)`
- **Impact:** Total frontend outage. Users receive connection refused or 502/504 Bad Gateway errors.
- **Trigger Condition:** Web ingress pod is unready or missing for > 30 seconds.

### 2.2 Step 1: Immediate Triage & Diagnostics
1. Check Web pods state:
   ```bash
   kubectl get pods -n robot-shop -l service=web -o wide
   ```
2. Query NGINX access and error logs in Loki:
   ```bash
   kubectl exec -n monitoring deploy/loki -- wget -qO- 'http://localhost:3100/loki/api/v1/query_range?query=%7Bservice%3D%22web%22%7D+%7C%3D+%22error%22&limit=20'
   ```
3. Check upstream backend services (`catalogue`, `cart`, `user`):
   ```bash
   kubectl get pods -n robot-shop
   ```

### 2.3 Step 2: Mitigation
1. Restart Web pods:
   ```bash
   kubectl rollout restart deployment web -n robot-shop
   ```
2. If node resource starvation is detected, verify node capacity:
   ```bash
   kubectl top nodes
   kubectl describe nodes | grep -A 5 "Allocated resources"
   ```

---

<a id="robotshophighcrashlooprate"></a>
## 3. Alert: RobotShopHighCrashLoopRate

### 3.1 Overview & Severity
- **Severity:** `Warning (P2)`
- **Impact:** Microservice instability, flapping pods, degraded user experience.
- **Trigger Condition:** Pod restarts exceed 1 restart per minute over a 5-minute rolling window.

### 3.2 Step 1: Diagnostics
1. Identify the restarting container:
   ```bash
   kubectl get pods -n robot-shop | grep -v "Running"
   ```
2. Inspect previous container exit code and logs:
   ```bash
   kubectl logs -n robot-shop <POD_NAME> --previous --tail=50
   kubectl describe pod -n robot-shop <POD_NAME> | grep -A 8 "Last State"
   ```
3. Common Exit Codes:
   - `Exit Code 137`: Out Of Memory (OOMKilled). Increase memory limits in deployment manifest.
   - `Exit Code 1`: Application syntax / uncaught exception. Inspect stack trace.
   - `Exit Code 143`: Graceful termination SIGTERM.

### 3.3 Step 2: Mitigation
- Adjust resource requests/limits in Helm chart `values.yaml` and run `helm upgrade robot-shop ./K8s/helm -n robot-shop`.
- Verify database connectivity (`mongodb`, `mysql`, `redis`).

---

## 4. Escalation Contacts & Communications

| Role | Contact Channel | Response SLA |
| :--- | :--- | :--- |
| **Primary On-Call SRE** | PagerDuty Schedule / #robot-shop-ops | 5 minutes (P1) / 30 minutes (P2) |
| **Tech Lead / Engineering Manager** | #robot-shop-alerts | 15 minutes for P1 incidents |
| **Incident Commander** | Video bridge | Spun up for incidents exceeding 15 minutes |

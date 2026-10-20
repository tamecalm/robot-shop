# Incident Postmortem: Payment Service Availability Outage & Error Budget Breach

**Incident Reference:** INC-20261020-01  
**Target Service:** `payment` (Checkout & Transaction Authorization Service)  
**Severity:** P1 - Critical  
**Status:** Resolved  
**Incident Commander:** DevOps Capstone SRE Team  
**Date of Incident:** Tuesday, October 20, 2026  
**Incident Window:** 12:38:17 UTC – 12:40:50 UTC (Duration: 2 minutes 33 seconds)  
**Standard:** Google SRE Postmortem Guidelines  

---

## 1. Executive Summary

On October 20, 2026, at 12:38:17 UTC, the `payment` microservice in Stan's Robot Shop experienced a complete availability failure due to replica starvation (replicas dropped to 0). This caused all incoming customer checkout requests through the `web` frontend to fail with HTTP `502 Bad Gateway` and `Connection Refused` errors.

The Prometheus alert `RobotShopPaymentAvailabilitySLOBreached` fired within 30 seconds of outage onset. The on-call engineer followed the operational runbook (`RUNBOOK.md`), diagnosed the outage via Grafana RED dashboards and Loki log aggregations, and executed immediate containment by restoring the deployment replicas. The service was fully restored and verified healthy at 12:40:50 UTC. Total downtime was **2 minutes 33 seconds**, consuming **1.18% of the monthly error budget**.

---

## 2. Impact & SLO Consumption

- **User Impact:** All users attempting to complete purchases during the 2m 33s window received failure errors at checkout.
- **Service Affected:** `payment` (Tier 1 Critical Service) & `web` (Frontend Ingress Proxy).
- **Requests Impacted:** ~128 synthetic automated user transactions failed with upstream connection errors.
- **Data Loss:** Zero data loss; uncommitted carts remained cached in `redis` and persisted in `mongodb`.
- **SLO Error Budget Impact:**
  - **Payment Service Availability SLO:** 99.5% over 30 days (Allowed Error Budget: 216 minutes).
  - **Downtime Consumed:** 2.55 minutes.
  - **Budget Consumed:** $\frac{2.55}{216} \times 100\% = \mathbf{1.18\%}$ of total monthly error budget.
  - **Budget Remaining:** **98.82%** (Healthy; no engineering freeze required).

---

## 3. Incident Timeline (All times in UTC)

```mermaid
timeline
    title Incident Response Timeline (INC-20261020-01)
    12:38:17 : Incident Trigger : Fault injected, payment replicas dropped to 0
    12:38:41 : Upstream Errors Detected : Loki logs capture NGINX 502 / Connection Refused
    12:38:47 : Alert Fired : PrometheusRule RobotShopPaymentAvailabilitySLOBreached active
    12:39:30 : Triage Initiated : On-call SRE acknowledges alert, consults RUNBOOK.md
    12:40:24 : Mitigation Executed : kubectl scale deployment payment --replicas=1 initiated
    12:40:50 : Full Recovery : Payment pod 1/1 Running, rollout complete
    12:41:11 : Verification Passed : Synthetic transaction returns HTTP 200/400 logic response
    12:41:30 : Alert Resolved : Prometheus clears alert, RED dashboard returns to 100%
```

| Timestamp (UTC) | Elapsed | Event / Action Taken |
| :--- | :--- | :--- |
| **12:38:17** | `T+00:00` | **Fault Injected:** Replica count of `payment` deployment inadvertently dropped to 0 during operational configuration testing. |
| **12:38:41** | `T+00:24` | **Error Logging:** Centralized logging via Fluent Bit and Loki ingests NGINX error logs: `connect() failed (111: Connection refused) while connecting to upstream http://payment:8080`. |
| **12:38:47** | `T+00:30` | **Alert Fired:** Prometheus rule `RobotShopPaymentAvailabilitySLOBreached` fires as payment ready metric evaluates to absent (`absent(kube_pod_status_ready) == 1`). |
| **12:39:30** | `T+01:13` | **Triage:** Engineer accesses on-call dashboard, reviews alert details and runbook link `RUNBOOK.md#robotshoppaymentavailabilityslobreached`. Confirms 0 running pods in namespace `robot-shop`. |
| **12:40:24** | `T+02:07` | **Mitigation Started:** In accordance with Runbook Section 1.4, engineer initiates rollout recovery: `kubectl scale deployment payment --replicas=1 -n robot-shop`. |
| **12:40:50** | `T+02:33` | **Mitigation Completed:** Container creates, passes liveness and readiness probes, and reports `1/1 Running`. Deployment rollout completes successfully. |
| **12:41:11** | `T+02:54` | **Verification:** End-to-end synthetic probe executed from `load` generator container. Payment logic responds, confirming transaction engine is operational. |
| **12:41:30** | `T+03:13` | **Alert Cleared:** Prometheus evaluation cycle updates; `RobotShopPaymentAvailabilitySLOBreached` transitions from `FIRING` to `RESOLVED`. Incident closed. |

---

## 4. Root Cause Analysis (5 Whys)

1. **Why did checkout fail for end-users?**  
   The `web` reverse proxy received connection refused errors when attempting to forward payment requests to `http://payment:8080`.
2. **Why was port 8080 unreachable?**  
   There were no active endpoints behind the `payment` Kubernetes ClusterIP service.
3. **Why were there no active endpoints?**  
   The `payment` deployment had its desired replica count set to 0.
4. **Why was the replica count set to 0?**  
   A simulated fault injection / uncoordinated manual scaling command stripped the replica specification without a Horizontal Pod Autoscaler (HPA) floor.
5. **Why was there no automated mechanism preventing replicas from reaching 0?**  
   The `payment` microservice lacked a Pod Disruption Budget (PDB) and minimum replica constraint enforced by Kubernetes admission controllers.

---

## 5. What Went Well, What Went Poorly, Where We Got Lucky

### What Went Well
- **Fast Detection:** Prometheus and Alertmanager detected the unavailability and fired the alert within 30 seconds of the pod termination.
- **Centralized Log Visibility:** Loki and Fluent Bit immediately surfaced the exact NGINX upstream errors (`connect() failed`) without requiring SSH access to individual worker nodes.
- **Actionable Runbook:** The operational runbook provided explicit copy-pasteable remediation commands that restored the service in under 30 seconds.

### What Went Poorly
- Manual scaling to 0 was not blocked by admission webhooks or Pod Disruption Budgets.
- The `web` frontend returned raw NGINX 502 HTML pages rather than a graceful, brand-compliant fallback checkout error message.

### Where We Got Lucky
- The outage occurred during an active synthetic drill with SRE personnel already observing the cluster, minimizing acknowledgment time (MTTA).
- No message corruption occurred in the `rabbitmq` queue during the disruption.

---

## 6. Corrective & Preventive Action Items

| Action Item | Type | Priority | Owner | Target Date |
| :--- | :--- | :--- | :--- | :--- |
| Implement `PodDisruptionBudget` (`minAvailable: 1`) for `payment` and `web` deployments | **Preventive** | P1 | SRE Team | Sprint 44 |
| Configure `HorizontalPodAutoscaler` (HPA) with `minReplicas: 2` across all core microservices | **Preventive** | P1 | SRE Team | Sprint 44 |
| Configure NGINX upstream fallback circuit breaker and custom user-friendly error pages | **Mitigation** | P2 | Frontend Team | Sprint 45 |
| Integrate Alertmanager webhook with Slack channel `#robot-shop-incidents` | **Process** | P2 | Platform Team | Sprint 45 |
| Conduct automated Chaos Mesh experiments testing random worker node termination | **Testing** | P3 | QA / SRE | Sprint 46 |

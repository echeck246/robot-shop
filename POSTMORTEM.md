# Incident Postmortem: Payment Service Availability SLO Breach

**Date:** 2026-10-29  
**Authors:** DevOps Engineering Team  
**Status:** Complete / Resolved  
**Incident Lead:** Emmanuel  
**Impact:** Total outage of checkout and payment transaction flows across Robot Shop for 3m 15s.

---

## 1. Executive Summary
On 2026-10-29 between 12:10:00 UTC and 12:13:15 UTC, the `payment` microservice experienced 100% unavailability due to zero active pod replicas running in the `robot-shop` namespace. The automated Prometheus alerting rule detected the degradation within 30 seconds and routed a critical alert to Slack. The on-call engineer followed the documented runbook, scaled the deployment back to its intended replica count, and restored full checkout availability within 3 minutes and 15 seconds.

---

## 2. Impact on SLO & Error Budget

* **Target Service:** `payment`
* **SLO Target:** 99.5% Monthly Availability (Max allowed downtime: 216 minutes)
* **Total Incident Duration:** 3 minutes 15 seconds (3.25 minutes)
* **Monthly Error Budget Consumed:** 
  $$\frac{3.25 \text{ min}}{216 \text{ min}} \times 100 = 1.50\%$$
* **Remaining Error Budget:** 98.50% (Green tier, standard release velocity maintained).

---

## 3. Incident Timeline (UTC)

* **12:10:00** – Failure injected; `payment` deployment replica count scaled to 0.
* **12:10:18** – Frontend `web` service began logging `connect() failed (111: Connection refused)` when proxying `/api/payment` requests.
* **12:10:35** – Prometheus evaluation matched `kube_deployment_status_replicas_available == 0` for $>30\text{s}$.
* **12:10:45** – Alertmanager dispatched firing alert `[FIRING] RobotShopPaymentAvailabilitySLOBreached` to `#devops-alerts` Slack channel.
* **12:11:10** – On-call engineer acknowledged alert and accessed `RUNBOOK.md` via the alert payload link.
* **12:12:40** – Mitigation initiated: deployment scaled to 1 replica via `kubectl scale deployment payment --replicas=1`.
* **12:13:15** – Pod reached `Ready` state; health check probe returned `200 OK`.
* **12:14:00** – Alertmanager resolved alert and posted green `[RESOLVED]` notification to Slack.

---

## 4. Root Cause Analysis (5 Whys)

1. **Why did the payment service fail?**  
   All HTTP requests to `/api/payment` returned `502 Bad Gateway` because no backend pods were accepting connections.
2. **Why were no backend pods available?**  
   The Kubernetes deployment replica count was set to `0`.
3. **Why did the replica count drop to zero?**  
   An automated node-scaling script / operator command scaled the deployment down during a simulated capacity drill.
4. **Why didn't another instance automatically scale up?**  
   The `payment` deployment lacked a Horizontal Pod Autoscaler (HPA) with a non-zero minimum replica constraint.
5. **Why was a single replica allowed to take down the entire checkout service?**  
   No Pod Disruption Budget (PDB) was defined for the `payment` microservice to guarantee a minimum available replica threshold (`minAvailable: 1`).

---

## 5. What Went Well vs. What Went Wrong

### What Went Well
* Prometheus detected the zero-replica status and fired within 35 seconds of injection.
* The Alertmanager Slack webhook delivered notification with zero transmission delay.
* The runbook link in the Slack notification allowed immediate diagnosis and resolution without guesswork.

### What Went Wrong
* The payment service lacked an autoscaling floor, allowing an immediate drop to 0 replicas.
* Users browsing the store during those 3.25 minutes experienced uncached error messages on checkout attempt.

---

## 6. Action Items & Remediation

| Action Item | Type | Owner | Status |
| :--- | :--- | :--- | :--- |
| Implement a `PodDisruptionBudget` for `payment` (`minAvailable: 1`) | Preventative | DevOps | Open |
| Add a Horizontal Pod Autoscaler (HPA) with `minReplicas: 2` | Preventative | DevOps | Open |
| Configure custom error page on Nginx `web` container for graceful degradation | Mitigative | Web Team | Open |
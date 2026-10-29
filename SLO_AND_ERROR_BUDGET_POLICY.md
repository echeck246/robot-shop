# Robot Shop: Service Level Objective (SLO) & Error Budget Policy

## 1. Service Scope
* **Target Focus Services:** `web`, `catalogue`, `cart`, `payment`
* **Critical Service Selected:** `payment`

---

## 2. Service Level Indicator (SLI)
* **Definition:** The proportion of valid HTTP checkout/health requests handled successfully by the payment service.
* **Formula:**
  $$\text{Availability SLI} = \frac{\text{Total Available Pod Minutes}}{\text{Total Scheduled Minutes}} \times 100$$
* **PromQL Implementation:**
  `sum(kube_deployment_status_replicas_available{namespace="robot-shop", deployment="payment"}) / count(kube_deployment_status_replicas_available{namespace="robot-shop", deployment="payment"})`

---

## 3. Service Level Objective (SLO)
* **Target:** **99.5% availability** over a rolling 30-day window.
* **Rationale:** A 0.5% margin accommodates routine node rebalancing, container restarts, and scheduled maintenance while ensuring uninterrupted checkout experiences.

---

## 4. Error Budget Calculations (30-Day Window)
* **Total Minutes in 30 Days:** $30 \times 24 \times 60 = 43{,}200 \text{ minutes}$
* **Allowed Downtime (0.5%):** 
  $$43{,}200 \times 0.005 = 216 \text{ minutes (3 hours, 36 minutes)}$$

---

## 5. Error Budget Policy & Escalation Rules

| Budget Consumed | Severity Tier | Action Required |
| :--- | :--- | :--- |
| **0% – 50%** | Green (Healthy) | Normal deployment speed; standard CI/CD pipeline cadence. |
| **50% – 80%** | Yellow (Warning) | Incident postmortem required for any downtime; engineering reviews non-critical changes. |
| **80% – 100%** | Orange (Critical) | Feature freeze initiated; priority shifts to reliability and automated testing. |
| **> 100%** | Red (Exhausted) | Complete release freeze. No new product features are shipped until error budget resets or availability recovers. |

---

## 6. Alerting Strategy
* **Alert Trigger:** `RobotShopPaymentAvailabilitySLOBreached` fires to `#devops-alerts` on Slack when available replicas equal `0` for $> 30$ seconds.
* **Notification Routing:** Alerts link directly to `RUNBOOK.md` for rapid triage.
# Robot Shop Operational Runbooks

## Runbook: Payment Service Outage (`RobotShopPaymentAvailabilitySLOBreached`)

### Alert Summary
* **Service:** `payment`
* **Severity:** Critical
* **SLO Target:** 99.5% Monthly Availability
* **Impact:** Users are unable to complete checkouts and cart transactions.

---

### Triage & Investigation Steps

1. **Verify Pod Status:**
   ```bash
   kubectl get pods -l service=payment -n robot-shop
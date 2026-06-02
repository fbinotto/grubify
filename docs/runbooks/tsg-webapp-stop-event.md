# TSG: Web App Stop Event Remediation

**Trigger:** `StopWeb` Activity Log alert fires on an Azure App Service web app.
**Alert Type:** Activity Log — Administrative
**Severity:** Sev4
**Scope:** Azure App Service (microsoft.web/sites)

---

## Overview

This runbook handles incidents where an Azure App Service web app is stopped — either intentionally by an administrator or accidentally — causing downtime. The `StopWeb` alert fires when an `Microsoft.Web/sites/stop/action` operation is detected in the Activity Log.

## Prerequisites

- Azure CLI access with `Contributor` or `Website Contributor` role on the target resource group
- Access to Azure Activity Logs for the subscription

---

## Step 1: Confirm Web App State

Check the current state of the web app:

```bash
az webapp show \
  --name <APP_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --subscription <SUBSCRIPTION_ID> \
  --query "{name:name, state:state, enabled:enabled, availabilityState:availabilityState}" \
  -o json
```

**Expected finding:** `"state": "Stopped"`

If the state is `Running`, the app may have already been restarted. Proceed to Step 3 to review who stopped it and why.

## Step 2: Restart the Web App

If the web app is confirmed stopped, restart it immediately:

```bash
az webapp start \
  --name <APP_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --subscription <SUBSCRIPTION_ID>
```

Verify the app is running after the restart:

```bash
az webapp show \
  --name <APP_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --subscription <SUBSCRIPTION_ID> \
  --query "{name:name, state:state}" \
  -o json
```

**Expected result:** `"state": "Running"`

Optionally, verify the app is responding to HTTP requests:

```bash
curl -s -o /dev/null -w "%{http_code}" https://<APP_NAME>.azurewebsites.net/
```

**Expected result:** `200` (or the app's normal response code)

## Step 3: Investigate the Stop Action

Review the Activity Log to identify who stopped the web app and when:

```bash
az monitor activity-log list \
  --resource-group <RESOURCE_GROUP> \
  --subscription <SUBSCRIPTION_ID> \
  --offset 6h \
  --query "[?contains(operationName.value, 'Microsoft.Web/sites/stop')].{operation:operationName.value, status:status.value, caller:caller, time:eventTimestamp, level:level}" \
  -o json
```

Key fields to examine:
- **caller**: The identity that performed the stop action (email or service principal)
- **time**: When the stop occurred
- **status**: Whether the action succeeded

### Common Callers

| Caller Type | Interpretation |
|---|---|
| `user@domain.com` | Manual stop by a human administrator |
| Service Principal / Managed Identity | Automated stop (e.g., CI/CD pipeline, Azure Automation, scheduled task) |
| `Microsoft.Web` (internal) | Platform-initiated stop (rare, usually during maintenance) |

## Step 4: Assess Impact

Determine the downtime window:
- **Start:** Timestamp of the `stop/action` event
- **End:** Timestamp when the app was restarted (from `start/action` or this remediation)
- **Duration:** End - Start

Check if any dependent services or users were affected during the downtime window.

## Step 5: Create Incident Report

Create a GitHub issue in the repository following the incident report template with:
- Summary of what happened
- Impact assessment
- Timeline of events (UTC)
- Evidence (Activity Log entries, web app state before/after)
- Root cause (who stopped the app and why)
- Remediation actions taken
- Follow-up action items

## Step 6: Follow-Up Actions

Based on the root cause, consider these preventive measures:

### If Accidental Stop
- [ ] Add a `CanNotDelete` or `ReadOnly` resource lock on the web app
- [ ] Review RBAC permissions — restrict `Microsoft.Web/sites/stop/action` to specific roles
- [ ] Enable "Always On" setting if not already enabled

### If Unauthorized Stop
- [ ] Review RBAC assignments on the resource group
- [ ] Enable Azure AD Conditional Access for sensitive operations
- [ ] Review Azure AD sign-in logs for the caller identity

### If Automated Stop
- [ ] Review the automation pipeline/runbook that triggered the stop
- [ ] Ensure automation targets the correct environment (dev vs. prod)
- [ ] Add environment guards to automation scripts

### General Improvements
- [ ] Reduce alert detection latency (current: ~4.5 minutes from stop to alert)
- [ ] Consider adding automated restart as an action group response
- [ ] Add health endpoint monitoring for faster downtime detection

---

## Appendix

### Related Alert Rule

- **Alert Name:** `StopWeb`
- **Type:** Activity Log — Administrative
- **Condition:** `Microsoft.Web/sites/stop/action` operation detected
- **Severity:** Sev4

### Incident History

| Date | App | Caller | Duration | Issue |
|---|---|---|---|---|
| 2026-06-02 | ultimategivingchallenge | admin@MngEnvMCAP325336.onmicrosoft.com | ~7 min | [#16](https://github.com/fbinotto/grubify/issues/16) |

---
name: webapp-password-rotation-azure-kv-cyberark
description: >
  Use this skill for troubleshooting, validating, and executing password rotation
  for web applications using Azure Key Vault integrated with CyberArk.
  This skill covers end-to-end workflow including ServiceNow Change Request
  validation, CyberArk secret rotation, Azure Key Vault synchronization,
  application configuration verification, and post-rotation validation.
  Trigger whenever a ticket involves password rotation, secret expiry,
  CyberArk-managed credentials, Azure Key Vault secrets,
  or application authentication failures related to rotated credentials.
---

# Web Application Password Rotation Skill

**Azure Key Vault + CyberArk + ServiceNow Change**

This skill provides a **standardized operational workflow** for rotating
application passwords securely using **CyberArk** and **Azure Key Vault (AKV)**,
with all changes governed by a **ServiceNow Change Request (CR)**.

***

## Step 1 — Read & Validate the Ticket

Before performing **any action**, extract and validate the following details:

| Field                     | What to Validate                          |
| ------------------------- | ----------------------------------------- |
| **Application name**      | Web application / service name            |
| **Environment**           | Dev / Test / UAT / Prod                   |
| **Credential type**       | DB password, API account, service account |
| **Authentication target** | DB / API / LDAP / external system         |
| **Azure subscription**    | Subscription ID and name                  |
| **Key Vault name**        | AKV storing the secret                    |
| **CyberArk safe**         | Safe name, object name                    |
| **ServiceNow CR**         | CR number, status (Approved)              |
| **Maintenance window**    | Start & end time                          |
| **Rollback plan**         | Available or documented                   |

**Do not proceed** if:

*   CR is not approved
*   Application owner is not listed
*   Production without change window

***

## Step 2 — Validate Change Request (ServiceNow)

Ensure the **CR is approved and active**.

### Mandatory Checks

*   CR status = **Approved**
*   Correct environment (Prod vs Non-Prod)
*   Application owner approval present
*   Backout/rollback plan documented
*   CyberArk + AKV mentioned in implementation steps

 **Example CR validation note**:

> Change CRQ00012345 is approved with maintenance window from 01:00–02:00 IST.  
> Scope includes CyberArk password rotation and AKV secret sync.

***

## Step 3 — Identify the Credential Flow

Understand **how the password flows** end-to-end:

CyberArk (Password Vault)
   ↓
Azure Key Vault (Secret Sync)
   ↓
Web Application (Config / Managed Identity)
   ↓
Target System (DB / API / External Service)


Confirm:

*   Password is **owned and rotated by CyberArk**
*   AKV secret is **auto-synced or pulled**
*   App reads secret **at runtime or restart**

***

## Step 4 — Pre-Rotation Health Checks

### Azure Key Vault Checks

# Verify secret exists
az keyvault secret show \
  --vault-name <kv-name> \
  --name <secret-name>

# Check secret expiry
az keyvault secret show \
  --vault-name <kv-name> \
  --name <secret-name> \
  --query attributes.expires

### Application Connectivity (Pre-check)

*   Application login working
*   No authentication errors
*   Baseline monitoring green (App Insights / Dynatrace)

***

## Step 5 — CyberArk Validation (Before Rotation)

Confirm CyberArk object is healthy.

### Checks to Perform

*   Safe access available
*   Password status = **In Sync**
*   Last change timestamp visible
*   Target system connectivity green

Typical CyberArk indicators:

*   CyberArk Central Policy Manager (CPM) running successfully
*   No “Access denied” or “Failed to reconcile” alerts

***

## Step 6 — Execute Password Rotation (CyberArk)

 **Perform rotation only during approved window**

### High-Level Actions

1.  Trigger CyberArk password change (manual or scheduled)
2.  Wait for CPM to complete successfully
3.  Verify password updated on target system
4.  Confirm no CyberArk alerts

 Rotation is **always initiated from CyberArk**, not AKV.

***

## Step 7 — Azure Key Vault Synchronization

After CyberArk rotation:

Confirm AKV secret is updated


# Fetch latest secret version
az keyvault secret show \
  --vault-name <kv-name> \
  --name <secret-name> \
  --query id


Things to verify:

*   New secret version created
*   Secret timestamps updated
*   No sync delay warnings

If sync fails:

*   Check CyberArk → AKV integration logs
*   Validate managed identity / access policy

***

## Step 8 — Application Validation (Post-Rotation)

### Mandatory Validation Steps

*   Restart application **if required**
*   Validate successful authentication
*   Run application smoke test
*   Check logs for auth failures

### Common Log Indicators

 Success:

*   Login successful
*   Connection established

 Failure:

*   Invalid credentials
*   Authentication failed
*   Timeout during secret retrieval

***

## Step 9 — Rollback (If Required)

Rollback is **CyberArk-driven**, not manual password reset.

### Rollback Conditions

*   Application fails post-rotation
*   Sync delay exceeds SLA
*   Unexpected auth errors

### Rollback Steps

1.  Use CyberArk to restore previous password
2.  Confirm target system access restored
3.  Validate AKV re-sync
4.  Restart application if needed

 Rollback must be documented in the CR.

***

## Step 10 — Monitoring & Alerts

Post-rotation monitoring (minimum 30–60 min):

*   App availability
*   Authentication metrics
*   Error rates
*   Azure Key Vault access logs
*   CyberArk CPM status

***

## Step 11 — Close the Change & Ticket

Before closure:

*   [ ] Application owner confirmation received
*   [ ] No authentication errors observed
*   [ ] AKV secret updated and in use
*   [ ] CyberArk status healthy
*   [ ] CR updated with execution notes

 **Closure Note Example**:

> Password rotation completed successfully via CyberArk.  
> Azure Key Vault secret synced and application validated.  
> No issues observed post-change.

***

## Common Issues & Fixes

###  AKV Secret Not Updated

**Cause:** Sync delay or permission issue  
**Fix:** Verify access policy / managed identity

***

###  App Still Using Old Password

**Cause:** Secret cached  
**Fix:** Restart app / pod / IIS

***

###  CyberArk Rotation Failed

**Cause:** Target system access issue  
**Fix:** Verify account permissions, reconcile password

***

## Ownership Matrix

| Component            | Owner                 |
| -------------------- | --------------------- |
| CyberArk Vault / CPM | Security Team         |
| Azure Key Vault      | Cloud / Platform Team |
| Application          | App Owner             |
| Change Approval      | CAB                   |
| Rotation Execution   | DevOps / Platform     |

***

## References

*   Azure Key Vault Secrets  
    <https://learn.microsoft.com/azure/key-vault/secrets/>

*   CyberArk Password Vault  
    <https://docs.cyberark.com/>

*   Azure Managed Identity  
    <https://learn.microsoft.com/azure/active-directory/managed-identities-azure-resources/>


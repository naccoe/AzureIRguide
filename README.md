# Azure Fraud Advisory — Investigation Guide for Arrow Partners



## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [How You Receive Alerts](#how-you-receive-alerts)
4. [Investigation Process](#investigation-process)
   - [Step 1: Validate the Alert](#step-1-validate-the-alert)
   - [Step 2: Access the Security Alerts Dashboard](#step-2-access-the-security-alerts-dashboard)
   - [Step 3: Review the Affected Subscription](#step-3-review-the-affected-subscription)
   - [Step 4: Check Azure Activity Log](#step-4-check-azure-activity-log)
   - [Step 5: Review Cost & Usage Anomalies](#step-5-review-cost--usage-anomalies)
   - [Step 6: Examine Entra ID Sign-In Logs](#step-6-examine-entra-id-sign-in-logs)
   - [Step 7: Classify the Alert](#step-7-classify-the-alert)
   - [Step 8: Update the Alert Status](#step-8-update-the-alert-status)
5. [Remediation — If Fraud Is Confirmed](#remediation--if-fraud-is-confirmed)
6. [Advanced: SOC Analyst Investigation](#advanced-soc-analyst-investigation)
   - [Sentinel Workspace Setup](#sentinel-workspace-setup)
   - [KQL Queries for Fraud Indicators](#kql-queries-for-fraud-indicators)
   - [Investigation Workflow for SOC Analysts](#investigation-workflow-for-soc-analysts)
   - [Generic SIEM Guidance](#generic-siem-guidance)
7. [Preventive Measures](#preventive-measures)
8. [Useful Links](#useful-links)
9. [FAQ](#faq)

---

## Overview

As an Arrow Electronics CSP partner, you are financially responsible for your customers' Azure consumption. When Microsoft detects suspicious activity — such as crypto mining, anomalous compute usage, or account takeover — they issue an **Azure fraud advisory** through Partner Center.

**Failing to investigate and resolve these alerts can result in:**

- Significant unauthorized Azure charges billed to your organization
- Compromised customer environments and data
- Compliance violations
- Erosion of customer trust

Microsoft requires all active security alerts to be resolved, including historical alerts dating back to December 2021.

> ⚠️ **Important:** Microsoft's security alerts do not detect *all* types of fraud. You must use additional monitoring (spending budgets, Azure Monitor, Defender for Cloud) as supplementary detection.

---

## Prerequisites

Before you can investigate alerts, ensure you have:

| Requirement | Details |
|---|---|
| **Partner Center Role** | Admin Agent role is required to access security alerts |
| **Partner Center Access** | [partner.microsoft.com/dashboard](https://partner.microsoft.com/dashboard/home) |
| **Azure Portal Access** | GDAP and appropriate RBAC role in the customer subscriptions |
| **MFA Enabled** | All accounts with customer management permissions must have MFA enforced |
| **Security Contact Email** | Ensure your [Partner Admin Agent preferred email](https://learn.microsoft.com/en-us/partner-center/security/security-contact#update-security-contact-information-for-your-csp-partner-tenant) is up-to-date |
| **Notification Preferences** | Subscribe to customer security alerts in Partner Center (see [How You Receive Alerts](#how-you-receive-alerts)) |

---

## How You Receive Alerts

Microsoft delivers Azure fraud advisories through four channels:

### 1. Email Notifications

| Alert Type | Frequency |
|---|---|
| CryptoMining-related | Max once every 24 hours per subscription |
| All other security alerts | Max once per hour per subscription |
| Service health security advisories | Max once per hour per subscription |
| Daily summary (all active/under-investigation) | Once every 24 hours |

Emails are sent from: **`azure-noreply@microsoft.com`**

**To subscribe:**
1. Sign in to [Partner Center](https://partner.microsoft.com/dashboard/home) → click the **bell** icon (Notifications)
2. Go to [My Preferences](https://partner.microsoft.com/dashboard/actioncenter/overview) 
3. Set your preferred email and language
4. Click **Edit** next to Email Notification Preferences
5. Check all boxes under **Customers** in the Workspace column
6. Click **Save**

### 2. Partner Center Security Alerts Dashboard

Access directly at:  
🔗 [partner.microsoft.com/dashboard/commerce2/insights/security/alerts](https://partner.microsoft.com/dashboard/commerce2/insights/security/alerts)

### 3. Webhook

Register for the `azure-fraud-event-detected` webhook event. See [Partner Center Webhook Events](https://learn.microsoft.com/en-us/partner-center/developer/partner-center-webhooks).

### 4. API (Microsoft Graph Security Alerts)

Use the [Microsoft Graph Security Alerts API (Beta)](https://learn.microsoft.com/en-us/graph/api/resources/partner-security-partnersecurityalert-api-overview?view=graph-rest-beta) for programmatic access.

> **Note:** The legacy FraudEvents API is deprecated as of Q4 2024. Migrate to the Microsoft Graph Security Alerts API.

---

## Investigation Process

### Step 1: Validate the Alert

**Goal:** Confirm the alert is genuine and not a phishing attempt.

- [x] Verify the email sender is **`azure-noreply@microsoft.com`**
- [x] Cross-reference the alert in the [Partner Center Security Alerts Dashboard](https://partner.microsoft.com/dashboard/commerce2/insights/security/alerts)
- [x] Check the [Partner Center Action Center](https://partner.microsoft.com/dashboard/actioncenter/overview) (bell icon) for the same alert
- [x] Confirm the Customer Tenant ID and Subscription ID match known customers

> 🛑 **If the email appears suspicious or cannot be verified in Partner Center, do not click any links. Report it as phishing.**

---

### Step 2: Access the Security Alerts Dashboard

1. Sign in to [Partner Center](https://partner.microsoft.com/dashboard/home)
2. Navigate to **Insights** → **Security Alerts** or go directly to:  
   🔗 [Security Alerts Dashboard](https://partner.microsoft.com/dashboard/commerce2/insights/security/alerts)
3. Locate the alert matching the notification you received
4. Note the following details:
   - **Alert ID**
   - **Alert Type** (Crypto Mining, Anomalous Compute, Azure ML Usage, Service Health Advisory)
   - **Customer Name / Tenant ID**
   - **Subscription ID(s)**
   - **Detection timestamp**

---

### Step 3: Review the Affected Subscription

1. Log in to the [Azure Portal](https://portal.azure.com)
2. Navigate to the affected subscription (use the Subscription ID from the alert)
3. Review:
   - **Resource Groups** — Look for any unfamiliar or recently created resource groups
   - **Resources** — Identify unknown VMs, container instances, ML workspaces, or storage accounts
   - **Deployments** — Check recent deployments for unauthorized activity
   - **Access Control (IAM)** — Verify no unknown role assignments were added

**Key questions to answer:**
- Are there resources you or the customer did not create?
- Are there VMs in unusual regions or with unusually high SKUs?
- Are there resources with names that look auto-generated or suspicious?

---

### Step 4: Check Azure Activity Log

1. In the Azure Portal, go to **Monitor** → **Activity Log**
2. Filter by:
   - **Subscription:** The affected subscription
   - **Timespan:** Starting from the alert detection time (extend backward by 48-72 hours)
   - **Event severity:** All levels
3. Look for:
   - Resource creation events you don't recognize
   - Role assignment changes (especially new Owner or Contributor assignments)
   - Subscription-level configuration changes
   - API calls from unfamiliar IP addresses or user agents

> 💡 **Tip:** Export the activity log to CSV for offline analysis: **Monitor → Activity Log → Download as CSV**

---

### Step 5: Review Cost & Usage Anomalies

1. Navigate to **Cost Management + Billing** in the Azure Portal
2. Open **Cost Analysis** for the affected subscription
3. Look for:
   - Sudden spikes in daily spend
   - Charges from unfamiliar services (especially Compute, GPU, Machine Learning)
   - Resource consumption in regions where the customer does not normally operate
4. Compare against the customer's [spending budget](https://learn.microsoft.com/en-us/partner-center/billing/set-an-azure-spending-budget-for-your-customers) if one is configured

> 💡 **Tip:** Check [unbilled consumption line items](https://learn.microsoft.com/en-us/partner-center/developer/get-invoice-unbilled-consumption-lineitems) via the Partner Center API for the most current charges.

---

### Step 6: Examine Entra ID Sign-In Logs

1. Go to [Entra ID → Sign-in Logs](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/SignIns) on the customer tenant
2. Review for:
   - Sign-ins from unusual locations or IP addresses
   - Sign-ins from unfamiliar devices or user agents
   - Failed sign-in attempts followed by successes (credential stuffing)
   - Sign-ins at unusual times
3. Check [Users at Risk](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/UsersAtRisk):
   - Review Identity Protection risk reports
   - Note any users flagged as high risk
4. Review the [Entra ID Audit Log](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/AuditLog) for:
   - New user creation
   - Password resets
   - MFA configuration changes
   - Application registrations or consent grants

---

### Step 7: Classify the Alert

Based on your investigation, classify the alert as one of:

| Classification | When to Use | Action |
|---|---|---|
| **Legitimate** | The activity was expected, authorized, or is a false positive | Mark as Legitimate; document your reasoning |
| **Fraud** | The activity is unauthorized — resources were created by a malicious actor | Proceed to [Remediation](#remediation--if-fraud-is-confirmed) immediately |
| **Ignore** | The alert is old/stale and no longer actionable | Mark as Ignore (see [FAQ](#why-am-i-receiving-old-alerts)) |

---

### Step 8: Update the Alert Status

You **must** update the alert status in Partner Center to close the investigation loop.

**Via the Dashboard:**
1. Open the alert in the [Security Alerts Dashboard](https://partner.microsoft.com/dashboard/commerce2/insights/security/alerts)
2. Select the appropriate classification (Legitimate / Fraud / Ignore)
3. Add investigation notes
4. Submit

**Via the API:**
Use the [Update partnerSecurityAlert](https://learn.microsoft.com/en-us/graph/api/partner-security-partnersecurityalert-update?view=graph-rest-beta) endpoint in the Microsoft Graph Security Alerts API.

> ⚠️ **All alerts must be resolved.** Unresolved alerts continue to appear in daily summary emails and may affect your partner standing.

---

## Remediation — If Fraud Is Confirmed

If your investigation confirms unauthorized activity, take these actions **immediately**:

### 🔐 1. Secure Compromised Identities

- **Rotate credentials** for all tenant admins and RBAC owners on the affected subscription
- Follow [Microsoft password policy recommendations](https://learn.microsoft.com/en-us/microsoft-365/admin/misc/password-policy-recommendations)
- **Verify and enforce MFA** on all admin accounts
- Review and update admin password recovery emails and phone numbers in Entra ID
- Revoke active sessions for compromised accounts

### 🛑 2. Contain the Threat

- **Suspend affected Azure resources** immediately:
  - Deallocate unauthorized VMs and container instances
  - Delete unauthorized resource groups
  - [Cancel the Azure subscription](https://learn.microsoft.com/en-us/partner-center/customers/azure-plan-manage#cancel-an-azure-subscription) if the compromise is severe
- Remove any unauthorized role assignments (IAM)
- Revoke any suspicious application registrations or consent grants

### 🧹 3. Clean Up

- Delete all resources created by the malicious actor
- [Find and delete unattached disks](https://learn.microsoft.com/en-us/azure/virtual-machines/windows/find-unattached-disks)
- Review storage accounts for exfiltrated or planted data
- Check for any persistent access mechanisms (e.g., automation runbooks, scheduled tasks, logic apps)

### 📞 4. Report and Escalate

- **Contact Azure Support** immediately: [Azure Support Portal](https://portal.azure.com/#blade/Microsoft_Azure_Support/HelpAndSupportBlade/overview)
- **Submit a Partner Center support request**: [Partner Support](https://partner.microsoft.com/support/?stage=2&topicid=3641ccf1-d75b-d53d-2436-301c11466107)
- Provide:
  - Partner Tenant ID
  - Customer Tenant ID
  - Subscription ID
  - Resource ID(s) involved
  - Impact start and end dates

### 📉 5. Reduce Exposure

- [Reduce unused Azure quota](https://learn.microsoft.com/en-us/azure/quotas/quotas-overview) on the subscription to prevent future abuse at scale
- Review and tighten the customer's [Azure spending budget](https://learn.microsoft.com/en-us/partner-center/billing/set-an-azure-spending-budget-for-your-customers)

---

## Advanced: SOC Analyst Investigation

This section is for MSPs with a Security Operations Center (SOC) or dedicated security analysts who have Azure telemetry flowing into a SIEM. It covers how to investigate Azure fraud advisories using **Microsoft Sentinel** and provides guidance adaptable to other SIEM platforms.

### Sentinel Workspace Setup

To investigate Azure fraud alerts in Sentinel, ensure the following **data connectors** are enabled on customer tenants:

| Data Connector | Log Table | Why It Matters |
|---|---|---|
| **Azure Activity** | `AzureActivity` | Resource creation, RBAC changes, subscription-level operations |
| **Microsoft Entra ID** | `SigninLogs`, `AuditLogs` | Sign-in anomalies, credential stuffing, admin activity |
| **Microsoft Entra ID Protection** | `SecurityAlert` | Risky users, risky sign-ins, identity compromise signals |
| **Microsoft Defender for Cloud** | `SecurityAlert`, `SecurityRecommendation` | Threat detections, misconfigurations, anomalous resource behavior |
| **Azure Key Vault** (if applicable) | `AzureDiagnostics` | Unauthorized secret/key access |

> 💡 **Tip:** If you manage multiple customer tenants, use [Azure Lighthouse](https://learn.microsoft.com/en-us/azure/lighthouse/overview) to centralize Sentinel monitoring across tenants in a single workspace.

### KQL Queries for Fraud Indicators

Run these queries in **Sentinel → Logs** or **Log Analytics** against the affected customer workspace.

#### 1. Detect Unusual Resource Deployments

Look for resource creation events that may indicate crypto mining or unauthorized compute:

```kql
AzureActivity
| where TimeGenerated > ago(7d)
| where OperationNameValue has_any ("Microsoft.Compute/virtualMachines/write", 
    "Microsoft.ContainerInstance/containerGroups/write",
    "Microsoft.MachineLearningServices/workspaces/computes/write")
| where ActivityStatusValue == "Success"
| project TimeGenerated, Caller, CallerIpAddress, OperationNameValue, 
    ResourceGroup, _ResourceId, Properties
| sort by TimeGenerated desc
```

#### 2. Identify VM Deployments in Unusual Regions

Flag compute resources spun up in regions the customer doesn't normally use:

```kql
AzureActivity
| where TimeGenerated > ago(7d)
| where OperationNameValue == "Microsoft.Compute/virtualMachines/write"
| where ActivityStatusValue == "Success"
| extend Region = tostring(parse_json(Properties).resource_location)
| summarize Count = count(), Callers = make_set(Caller) by Region
| sort by Count desc
```

#### 3. Detect Suspicious RBAC Role Assignments

Catch unauthorized privilege escalation:

```kql
AzureActivity
| where TimeGenerated > ago(7d)
| where OperationNameValue == "Microsoft.Authorization/roleAssignments/write"
| where ActivityStatusValue == "Success"
| extend RoleDefinitionId = tostring(parse_json(Properties).requestbody)
| project TimeGenerated, Caller, CallerIpAddress, ResourceGroup, 
    RoleDefinitionId, _ResourceId
| sort by TimeGenerated desc
```

#### 4. Anomalous Sign-Ins (Entra ID)

Identify sign-ins from unfamiliar locations or flagged as risky:

```kql
SigninLogs
| where TimeGenerated > ago(7d)
| where ResultType == 0  // Successful sign-ins
| where RiskLevelDuringSignIn in ("high", "medium") 
    or RiskLevelAggregated in ("high", "medium")
| project TimeGenerated, UserPrincipalName, IPAddress, Location, 
    AppDisplayName, RiskLevelDuringSignIn, RiskLevelAggregated, 
    DeviceDetail, ConditionalAccessStatus
| sort by TimeGenerated desc
```

#### 5. Detect Credential Stuffing Patterns

Look for bursts of failed sign-ins followed by a success:

```kql
let FailedAttempts = SigninLogs
| where TimeGenerated > ago(7d)
| where ResultType != 0
| summarize FailCount = count(), 
    FailedIPs = make_set(IPAddress) by UserPrincipalName, bin(TimeGenerated, 1h);
let SuccessAfterFail = SigninLogs
| where TimeGenerated > ago(7d)
| where ResultType == 0;
FailedAttempts
| where FailCount > 10
| join kind=inner (SuccessAfterFail) on UserPrincipalName
| where SuccessAfterFail.TimeGenerated between (FailedAttempts.TimeGenerated .. (FailedAttempts.TimeGenerated + 2h))
| project UserPrincipalName, FailCount, FailedIPs, 
    SuccessTime = SuccessAfterFail.TimeGenerated, 
    SuccessIP = SuccessAfterFail.IPAddress
```

#### 6. Unauthorized Application Consent Grants

Detect OAuth consent grants that may indicate a consent phishing attack:

```kql
AuditLogs
| where TimeGenerated > ago(7d)
| where OperationName == "Consent to application"
| extend ConsentActor = tostring(InitiatedBy.user.userPrincipalName)
| extend TargetApp = tostring(TargetResources[0].displayName)
| project TimeGenerated, ConsentActor, TargetApp, 
    AdditionalDetails, CorrelationId
| sort by TimeGenerated desc
```

#### 7. Cost Spike Detection (if cost data is ingested)

If you export Azure cost data to Log Analytics:

```kql
AzureActivity
| where TimeGenerated > ago(30d)
| where OperationNameValue has "Microsoft.Compute"
| where ActivityStatusValue == "Success"
| summarize DailyOperations = count() by bin(TimeGenerated, 1d)
| render timechart
```

### Investigation Workflow for SOC Analysts

When a fraud advisory alert lands in your SOC queue, follow this triage workflow:

```
┌─────────────────────────────────────────────────┐
│  1. ALERT INTAKE                                │
│  • Validate alert in Partner Center Dashboard   │
│  • Create incident ticket in your ITSM          │
│  • Assign severity: HIGH                        │
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│  2. INITIAL TRIAGE (Target: < 30 min)           │
│  • Run KQL #1 & #2 — check for rogue resources │
│  • Run KQL #3 — check for RBAC changes         │
│  • Run KQL #4 — check for risky sign-ins       │
│  • Check Azure Activity Log for the sub         │
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│  3. DEEP INVESTIGATION                          │
│  • Run KQL #5 — credential stuffing patterns    │
│  • Run KQL #6 — consent grant abuse             │
│  • Correlate IPs across SigninLogs & Activity    │
│  • Review resource configs for persistence      │
│    (automation accounts, logic apps, webhooks)  │
│  • Check for data exfiltration (storage logs)   │
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│  4. CLASSIFICATION & RESPONSE                   │
│  • Classify: Legitimate / Fraud / Ignore        │
│  • If Fraud → execute Remediation playbook      │
│  • Update alert in Partner Center               │
│  • Document findings in incident ticket         │
│  • Notify stakeholders                          │
└─────────────────────────────────────────────────┘
```

### Generic SIEM Guidance

If you're using a SIEM other than Sentinel (e.g., Splunk, QRadar, Elastic, Chronicle):

| What to Ingest | Source | Purpose |
|---|---|---|
| Azure Activity Logs | [Azure Event Hub](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/activity-log?tabs=powershell) or Diagnostic Settings export | Resource-level operations, RBAC changes |
| Entra ID Sign-In & Audit Logs | [Microsoft Graph API](https://learn.microsoft.com/en-us/graph/api/signin-list) or Event Hub streaming | Identity-based threat detection |
| Defender for Cloud Alerts | [Continuous export](https://learn.microsoft.com/en-us/azure/defender-for-cloud/continuous-export) to Event Hub | Security threat detections |
| Azure Cost Data | [Cost Management export](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-export-acm-data) | Spending anomaly detection |

**Key detection rules to build in any SIEM:**

1. **New VM in unusual region** — Alert when compute resources are created outside known customer regions
2. **RBAC Owner/Contributor added** — Alert on any new privileged role assignment
3. **High-risk sign-in followed by resource creation** — Correlate risky identity signals with Azure resource operations within a short time window
4. **Burst resource creation** — Alert when more than N resources are created within a short period (e.g., 10+ VMs in 1 hour)
5. **Cost threshold breach** — Alert when daily spend exceeds a defined baseline by a configurable percentage

> 📖 **For Sentinel-specific automation:** Consider building [Sentinel Playbooks (Logic Apps)](https://learn.microsoft.com/en-us/azure/sentinel/automate-responses-with-playbooks) to auto-enrich fraud advisory alerts with KQL query results and notify your SOC team via Teams/email.

---

## Preventive Measures

Adopt these practices across all customer tenants to reduce fraud risk:

| Measure | Details |
|---|---|
| **Enforce MFA** | All accounts with Azure management permissions must use MFA. See [CSP Security Best Practices](https://learn.microsoft.com/en-us/partner-center/security/csp-security-best-practices). |
| **Set Spending Budgets** | Configure [Azure spending budgets](https://learn.microsoft.com/en-us/partner-center/billing/set-an-azure-spending-budget-for-your-customers) for every customer. |
| **Enable Defender for Cloud** | Turn on [Microsoft Defender for Cloud](https://learn.microsoft.com/en-us/azure/defender-for-cloud/concept-cloud-security-posture-management) (free tier available). |
| **Monitor RBAC Changes** | Set up alerts for [Azure RBAC permission changes](https://learn.microsoft.com/en-us/partner-center/customers/azure-plan-manage#confirm-that-you-have-admin-access). |
| **Use Azure Monitor Alerts** | Configure [usage and cost alerts](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-create-new-alert-rule) for anomaly detection. |
| **Reduce Unused Quota** | Minimize the [compute quota](https://learn.microsoft.com/en-us/azure/quotas/quotas-overview) on subscriptions to limit blast radius. |
| **Customer Education** | Work with customers to implement [Customer Security Best Practices](https://learn.microsoft.com/en-us/partner-center/security/customer-security-best-practices). |
| **Set Up Service Health Alerts** | Have customers configure [Service Health Alerts](https://learn.microsoft.com/en-us/azure/service-health/service-health-overview) for security notifications. |
| **Review Operational Security** | Follow [Azure operational security best practices](https://learn.microsoft.com/en-us/azure/security/fundamentals/operational-best-practices). |
| **Identity Protection** | Implement [risk policies and alerting](https://learn.microsoft.com/en-us/azure/active-directory/identity-protection/overview-identity-protection) for high-risk users and sign-ins. |

---

## Useful Links

### Partner Center

| Resource | URL |
|---|---|
| Partner Center Home | [partner.microsoft.com/dashboard](https://partner.microsoft.com/dashboard/home) |
| Security Alerts Dashboard | [Security Alerts](https://partner.microsoft.com/dashboard/commerce2/insights/security/alerts) |
| Action Center | [Action Center](https://partner.microsoft.com/dashboard/actioncenter/overview) |
| Partner Support (Fraud) | [Submit Request](https://partner.microsoft.com/support/?stage=2&topicid=3641ccf1-d75b-d53d-2436-301c11466107) |

### Azure Portal

| Resource | URL |
|---|---|
| Azure Portal | [portal.azure.com](https://portal.azure.com) |
| Users at Risk | [Entra ID — Users at Risk](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/UsersAtRisk) |
| Azure Support | [Help + Support](https://portal.azure.com/#blade/Microsoft_Azure_Support/HelpAndSupportBlade/overview) |

### Microsoft Documentation

| Resource | URL |
|---|---|
| Detect and Respond to Security Alerts | [learn.microsoft.com/...azure-fraud-notification](https://learn.microsoft.com/en-us/partner-center/security/azure-fraud-notification) |
| CSP Security Best Practices | [learn.microsoft.com/...csp-security-best-practices](https://learn.microsoft.com/en-us/partner-center/security/csp-security-best-practices) |
| Customer Security Best Practices | [learn.microsoft.com/...customer-security-best-practices](https://learn.microsoft.com/en-us/partner-center/security/customer-security-best-practices) |
| Azure Quotas Overview | [learn.microsoft.com/...quotas-overview](https://learn.microsoft.com/en-us/azure/quotas/quotas-overview) |
| Microsoft Graph Security Alerts API | [learn.microsoft.com/...graph API](https://learn.microsoft.com/en-us/graph/api/resources/partner-security-partnersecurityalert-api-overview?view=graph-rest-beta) |
| Managing Nonpayment, Fraud, or Misuse | [learn.microsoft.com/...nonpayment-fraud-misuse](https://learn.microsoft.com/en-us/partner-center/security/nonpayment-fraud-misuse) |

---

## FAQ

### Why am I receiving old alerts?

Microsoft has been sending Azure fraud alerts since December 2021. Previously, alerts were opt-in only. Microsoft changed this — all partners now receive notifications for any unresolved alerts, including historical ones. You must resolve all active alerts to stop receiving daily summary emails about them. Use the **Ignore** classification for old, stale alerts that are no longer actionable.

### Why am I not seeing all alerts?

Microsoft's security alerts detect *patterns* of anomalous activity but do not catch all types of fraud. If you suspect unauthorized usage that wasn't flagged, contact [Partner Support](https://partner.microsoft.com/support/?stage=2) with:
- Partner Tenant ID
- Customer Tenant ID
- Subscription ID
- Resource ID
- Impact start and end dates

### How often are alerts sent?

- **CryptoMining alerts:** Max once every 24 hours per subscription
- **Other security alerts:** Max once per hour per subscription  
- **Daily summary of all active/under-investigation alerts:** Once every 24 hours

### What if the alert is a false positive?

Classify it as **Legitimate** in the Security Alerts Dashboard and document your reasoning. This closes the alert and stops further notifications for that specific detection.

### Can I automate alert handling?

Yes. Use the [Microsoft Graph Security Alerts API](https://learn.microsoft.com/en-us/graph/api/resources/partner-security-partnersecurityalert-api-overview?view=graph-rest-beta) to programmatically list, read, and update alerts. You can also register a [webhook](https://learn.microsoft.com/en-us/partner-center/developer/partner-center-webhooks) for the `azure-fraud-event-detected` event to trigger automated workflows.

### Who at Microsoft should I contact for support?

- **Azure issues:** [Azure Support Portal](https://portal.azure.com/#blade/Microsoft_Azure_Support/HelpAndSupportBlade/overview)
- **Partner Center issues:** [Partner Center Support](https://partner.microsoft.com/support/?stage=2)
- **Fraud-specific support:** [Partner Support — Fraud Topic](https://partner.microsoft.com/support/?stage=2&topicid=3641ccf1-d75b-d53d-2436-301c11466107)

---

*This guide is maintained by Arrow Electronics NA CCoE. Last updated: March 2026.*  
*Based on [Microsoft Partner Center documentation](https://learn.microsoft.com/en-us/partner-center/security/azure-fraud-notification).*

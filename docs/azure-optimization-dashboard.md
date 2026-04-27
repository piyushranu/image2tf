# Azure Environment Review Dashboard (Resource Scan + Recommendations)

This dashboard design helps you continuously **scan Azure resources**, identify risky or costly configurations, and provide actionable recommendations to improve **security, reliability, performance, and cost**.

> **Requirement:** Recommendations should be ingested directly from **Azure Advisor** and presented as the primary recommendation feed in the dashboard.

## 1) Goal

Build a single view that answers:
- What resources are misconfigured?
- Which recommendations are highest priority?
- What should be fixed first to reduce risk and cost?
- Which Azure Advisor recommendations are new, in progress, or completed?

## 2) Recommended Architecture

1. **Data collection (Advisor-first)**
   - **Azure Advisor recommendations (required primary source)**
   - Azure Resource Graph (inventory + metadata)
   - Microsoft Defender for Cloud (security posture)
   - Azure Policy compliance results
   - Azure Monitor + Log Analytics (runtime signals)

2. **Advisor ingestion implementation**
   - Pull recommendations via Advisor API on a schedule (for example every 6-24 hours)
   - Capture fields such as category, impact, impacted resource, short description, and remediation guidance
   - Store snapshots to track status over time (Active, Snoozed, Dismissed, Resolved)

3. **Processing / scoring**
   - Normalize findings into one schema
   - Keep Advisor recommendation ID as a stable key
   - Priority score = `Business Impact x Security Severity x Cost Impact x Effort`

4. **Visualization**
   - Power BI or Azure Workbook dashboard
   - Executive summary + technical drill-down

5. **Action orchestration**
   - Ticket creation (Jira/Azure DevOps)
   - Optional auto-remediation (Logic App / Function App / Automation Runbook)

## 3) Core Dashboard Pages

### A) Executive Overview
- Total subscriptions / resource groups / resources scanned
- Count of **active Azure Advisor recommendations**
- Compliance score (%)
- Monthly potential savings (USD)
- Critical security findings count
- Trend chart (7/30/90 days)

### B) Advisor Recommendations (Primary View)
- Recommendations grouped by Advisor category:
  - Cost
  - Security
  - Reliability
  - Operational Excellence
  - Performance
- Filters: subscription, resource group, tag, severity, category, status
- Columns: recommendation name, impacted resource, impact level, estimated savings, owner, due date, status

### C) Security Hardening
- Publicly exposed resources
- Missing endpoint protection / threat protection
- Unencrypted disks / storage
- NSG overly permissive rules (Any/Any)
- Missing MFA/Conditional Access for privileged identities
- Advisor Security recommendations not yet remediated

### D) Cost Optimization
- Underutilized VM/AKS/Database resources
- Idle public IPs / unattached disks / orphan resources
- Rightsizing recommendations
- Reserved Instance / Savings Plan opportunities
- Storage tier optimization (Hot/Cool/Archive)
- Advisor Cost recommendations with estimated monthly savings

### E) Reliability & Operations
- No backup configured
- Missing availability zones / sets
- Missing health alerts / diagnostics settings
- Expiring certificates / secrets
- Single points of failure
- Advisor Reliability/Operational Excellence recommendations backlog

### F) Governance & Compliance
- Tagging compliance (owner, environment, costCenter, dataClassification)
- Resource lock coverage
- Policy compliance by initiative
- Region restrictions and naming standard violations

## 4) Suggested "Better Settings" Baseline

Use this as default policy targets:

### Identity & Access
- Enforce MFA for all privileged roles
- Use PIM for just-in-time admin access
- Disable standing Global Admin where possible
- Managed Identity over client secrets

### Network
- Deny public access by default for PaaS
- Private Endpoints for data services
- NSG least-privilege rules (no broad inbound)
- DDoS Standard for internet-facing critical workloads

### Data Protection
- Encryption at rest + customer-managed keys for sensitive workloads
- Soft delete and purge protection for Key Vault and backup-enabled services
- TLS 1.2+ only

### Compute & Platform
- Enable automatic patching where applicable
- Minimum SKU standards for production reliability
- Autoscale on workloads with variable demand
- Diagnostics settings enabled to central Log Analytics workspace

### Monitoring & Response
- Alert rules for CPU/memory/storage/availability anomalies
- Activity log alerts for high-risk actions (role assignment, NSG changes, policy changes)
- Retention policy aligned to compliance requirements

### Cost
- Mandatory tagging for chargeback/showback
- Stop/deallocate non-prod workloads out of hours
- Rightsizing cadence monthly
- Review Advisor + Cost Management weekly

## 5) Priority Scoring Model

Example formula:

`Priority Score = (SecuritySeverity * 4) + (CostImpact * 3) + (BusinessCriticality * 2) - (RemediationEffort * 1)`

Score bands:
- **P1 (>= 24):** Fix within 7 days
- **P2 (16-23):** Fix within 30 days
- **P3 (< 16):** Backlog / planned sprint

## 6) Minimum Data Model (for recommendations table)

- `advisorRecommendationId`
- `subscriptionId`
- `resourceGroup`
- `resourceId`
- `resourceType`
- `environmentTag`
- `findingCategory` (Security/Cost/Reliability/Governance/Performance/OperationalExcellence)
- `findingTitle`
- `currentSetting`
- `recommendedSetting`
- `estimatedMonthlySavings`
- `impact` (High/Medium/Low)
- `riskSeverity`
- `priorityScore`
- `owner`
- `remediationAction`
- `dueDate`
- `advisorStatus` (Active/Snoozed/Dismissed/Resolved)
- `status`
- `lastSeenAt`

## 7) Starter Query Ideas

### Azure Resource Graph: find resources missing required tags
```kusto
Resources
| where type !startswith 'microsoft.resources/subscriptions'
| extend owner = tostring(tags.owner), env = tostring(tags.environment), costCenter = tostring(tags.costCenter)
| where isempty(owner) or isempty(env) or isempty(costCenter)
| project subscriptionId, resourceGroup, name, type, location, owner, env, costCenter
```

### Azure Resource Graph: identify public IP resources
```kusto
Resources
| where type =~ 'microsoft.network/publicipaddresses'
| project subscriptionId, resourceGroup, name, location, sku=tostring(sku.name), ip=tostring(properties.ipAddress)
```

### Azure Advisor (CLI): export Advisor recommendations
```bash
az advisor recommendation list \
  --query "[].{id:id,name:shortDescription.problem,category:category,impact:impact,resourceId:resourceMetadata.resourceId}" \
  -o table
```

### Azure Advisor (REST): list recommendations by subscription
```http
GET https://management.azure.com/subscriptions/{subscriptionId}/providers/Microsoft.Advisor/recommendations?api-version=2023-01-01
Authorization: Bearer <token>
```

## 8) Rollout Plan

1. **Week 1:** Connect Azure Advisor feed + inventory + compliance widgets
2. **Week 2:** Add Security + Cost recommendations and scoring
3. **Week 3:** Add workflow integration (tickets/owners/SLA)
4. **Week 4:** Enable automation for low-risk remediations

## 9) Success Metrics

- Active Advisor recommendations reduced (#)
- Compliance score increase (%) month over month
- Critical findings reduced (#)
- Mean time to remediate (MTTR)
- Realized monthly savings vs forecast
- % resources with required tags and diagnostics enabled

---

If you want, the next step is to generate a **ready-to-import Azure Workbook JSON template** and a matching **Power BI semantic model** that uses Azure Advisor as the core recommendations table.


## 10) Dashboard Preview (Wireframe)

Below is a practical preview of how the dashboard can be arranged in an Azure Workbook or Power BI page.

### Page 1: Executive Overview

```text
+-----------------------------------------------------------------------------------+
| Azure Optimization Dashboard (Advisor-first)                                      |
| Last refresh: 2026-04-27 12:00 UTC | Scope: 8 subscriptions                       |
+-----------------------------------------------------------------------------------+
| Active Advisor Recs | Potential Monthly Savings | Compliance Score | Critical Risk |
|        142          |        $38,420            |       81%        |      19       |
+-----------------------------------------------------------------------------------+
| Recommendations Trend (30 days)                | Category Split                    |
| - Active: 190 -> 142                            | Cost: 46                          |
| - Resolved: 75                                  | Security: 39                      |
|                                                 | Reliability: 27                   |
|                                                 | Perf/OpsEx: 30                    |
+-----------------------------------------------------------------------------------+
| Top 10 High-Impact Recommendations (table)                                        |
| [Category] [Title] [Resource] [Impact] [Savings] [Owner] [Due Date] [Status]     |
+-----------------------------------------------------------------------------------+
```

### Page 2: Advisor Recommendations (Primary Grid)

```text
Filters: [Subscription] [Resource Group] [Environment Tag] [Category] [Impact] [Status]

| Recommendation                         | Category   | Impact | Resource              | Savings | Owner   | Status |
|----------------------------------------|------------|--------|-----------------------|---------|---------|--------|
| Right-size or shutdown underutilized VM| Cost       | High   | /subs/.../vm-prod-01 | $1,240  | team-a  | Active |
| Enable soft delete on storage account  | Security   | Medium | /subs/.../stgcore01  | -       | team-b  | Active |
| Configure zone redundancy              | Reliability| Medium | /subs/.../sql-prod01 | -       | team-c  | Planned|
```

### Page 3: Cost + Security Deep Dive

```text
Cost View:
- Monthly savings by subscription (bar chart)
- Top waste patterns (idle disks, idle public IPs, oversized compute)
- Advisor Cost backlog aging (0-7, 8-30, 31+ days)

Security View:
- Advisor Security findings by severity
- Public exposure map (public IP + NSG Any/Any)
- Unencrypted resources list
```

### Recommended Visual Components

- KPI cards: Active recommendations, critical count, compliance score, savings.
- Stacked column: recommendation categories by status over time.
- Heatmap: subscriptions vs categories (count + severity).
- Detailed data grid: sortable/exportable recommendation list.
- Drill-through panel: recommendation details + remediation steps + ticket link.

### Suggested Workbook Tabs

1. `Overview`
2. `Advisor Recommendations`
3. `Cost Optimization`
4. `Security Hardening`
5. `Reliability & Operations`
6. `Governance & Compliance`


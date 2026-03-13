# Azure Fundamentals L100 — Session 2
**Controlling & Governing Azure**

---

## High-Level Overview

This 90-minute session focuses on how to control access, enforce organizational standards, and govern Azure at scale. You'll learn about Azure RBAC for identity and access management, Azure Policy for compliance enforcement, resource tagging strategies, cost management, and how the Cloud Adoption Framework ties everything together. We'll demonstrate these concepts through live portal demos.

**Duration:** 1.5 hours (90 minutes)

---

## Learning Objectives

By the end of this session, you will be able to:
- Implement identity and access management using Azure RBAC
- Apply governance through Azure Policy and resource tagging
- Understand cost management and optimization strategies
- Apply Cloud Adoption Framework principles to enterprise architecture
- Use Azure's monitoring and management tools effectively

---

## Core Learning Outcomes

By the end of Session 2, participants will understand:

### 1. Identity & Access Management (25 min)
How Azure RBAC controls who can do what, and where
- Entra ID (formerly Azure AD) as the identity foundation
- RBAC's three components: Security Principal + Role + Scope
- How permissions inherit down the hierarchy
- AWS IAM → Azure RBAC translation
- RBAC baseline best practices

### 2. Governance & Compliance (20 min)
How Azure Policy and tagging enforce organizational standards
- RBAC vs Policy (access vs compliance)
- Policy effects: Deny, Audit, Append, DeployIfNotExists
- Tagging strategy for cost allocation, ownership, and automation
- Enterprise governance approach and best practices

### 3. Cost Management (8 min)
How to track, allocate, and optimize Azure spending
- Cost hierarchy and subscription billing boundaries
- Cost Analysis, Budgets, and Advisor recommendations
- Pricing models and optimization strategies

### 4. Cloud Adoption Framework Integration (10 min)
How fundamentals fit into enterprise architecture
- CAF's 6 methodology phases (Strategy → Plan → Ready → Adopt → Govern → Manage)
- 8 design areas that map to Azure services
- How enterprise organizations implement CAF principles

### 5. Monitoring & Management (10 min)
Tools for operational visibility
- Azure Monitor, Log Analytics, Service Health
- Management tools comparison (Portal, CLI, PowerShell, Bicep)

### 6. Hands-On Reinforcement (10 min)
Live demos tying concepts together
- Assign RBAC roles, configure policies, apply tags, analyze costs

---

## Detailed Session Agenda

### 1:00 – 1:05 | Welcome Back & Session 1 Recap (5 min)

**Quick Knowledge Check:**
- What are the 5 levels of the Azure organizational hierarchy?
  - Tenant → Management Groups → Subscriptions → Resource Groups → Resources
- What Azure service is equivalent to AWS S3?
  - Azure Blob Storage
- What does CAF stand for?
  - Cloud Adoption Framework

**Today's Focus:** How to control access, enforce standards, and govern Azure at scale.

---

### 1:05 – 1:30 | Identity & Access Control with Azure RBAC (25 min)

#### Authentication vs. Authorization (3 min)
- **Authentication:** Who are you? (Identity verification)
  - Handled by **Microsoft Entra ID** (formerly Azure AD)
  - Users, groups, service principals, managed identities
- **Authorization:** What can you do? (Permission verification)
  - Handled by **Azure Role-Based Access Control (RBAC)**

#### Microsoft Entra ID Fundamentals (5 min)
- **What is it?** Cloud-based identity and access management service
- **Security Principals:**
  - **Users** — Individual accounts
  - **Groups** — Collections of users
  - **Service Principals** — Identity for applications/services
  - **Managed Identities** — Auto-managed identities for Azure resources (no credential management)

**AWS Equivalent:** IAM (but Entra ID is broader — includes B2B, B2C, MFA, Conditional Access)

#### Azure RBAC Deep Dive (12 min)

**Core Concept:** RBAC answers "**Who** can do **What** on **Which** resource"

**Three Components:**
1. **Security Principal** (Who) — User, group, service principal, managed identity
2. **Role Definition** (What) — Collection of permissions
3. **Scope** (Which) — Where the role applies

**Built-in Roles (most common):**

| Role | Description | Use Case |
|------|-------------|----------|
| **Owner** | Full access including permissions management | Subscription admins |
| **Contributor** | Full access EXCEPT permission management | Developers, operators |
| **Reader** | View only, no modifications | Auditors, viewers |
| **User Access Administrator** | Manage access only, not resources | Security team |

**Custom Roles:** Define your own permissions (JSON-based)

**Scope Hierarchy & Inheritance:**
```
Management Group (policies apply to all below)
↓
Subscription (role assigned here = access to all resource groups)
↓
Resource Group (role assigned here = access to all resources in group)
↓
Resource (role assigned here = access to this resource only)
```

**Key Principle:** Permissions inherit downward; use **least privilege** approach

**Reference:**
- [Azure RBAC overview](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview)
- [Azure built-in roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles)
- [Best practices for RBAC](https://learn.microsoft.com/en-us/azure/role-based-access-control/best-practices)

---

### 1:30 – 1:50 | Governance: Azure Policy & Resource Tagging (20 min)

#### Azure Policy vs. RBAC (3 min)

**Confusion Buster:**
- **RBAC** = What actions can a user **perform**? (e.g., Can Alice create a VM?)
- **Azure Policy** = What standards must resources **comply with**? (e.g., All VMs must be in Australia East)

**Use Together:** RBAC grants access; Policy enforces guardrails

#### Azure Policy Deep Dive (10 min)

**What is Azure Policy?**
- Service to create, assign, and manage policies that enforce rules on resources
- Evaluate compliance continuously
- Can prevent non-compliant resources or audit existing ones

**Policy Effects:**

| Effect | What It Does |
|--------|-------------|
| **Deny** | Block non-compliant resource creation |
| **Audit** | Log non-compliance (doesn't block) |
| **Append** | Add properties (e.g., force a tag) |
| **DeployIfNotExists** | Deploy resources if they don't exist |
| **Modify** | Add/update/remove tags |

**Policy Initiatives (Policy Sets):**
- Group multiple policies together
- Example: "CIS Microsoft Azure Foundations Benchmark" initiative

**Policy Assignment:**
- Apply policies at: Management Group, Subscription, or Resource Group scope
- Inheritance applies downward
- Use **exclusions** for exceptions

**Common Use Cases:**
- Enforce allowed regions (e.g., Australia only)
- Require tags on all resources
- Deny public IP addresses on VMs
- Enforce encryption at rest
- Require Network Security Groups on subnets

**Reference:**
- [Azure Policy overview](https://learn.microsoft.com/en-us/azure/governance/policy/overview)
- [Azure Policy built-in definitions](https://learn.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies)

#### Resource Tagging Strategy (7 min)

**Why Tag?**
1. **Cost allocation** — Chargeback to teams/projects
2. **Ownership tracking** — Who owns this resource?
3. **Automation** — Trigger scripts based on tags
4. **Compliance** — Track compliance requirements
5. **Resource organization** — Filter and search

**Tag Anatomy:**
- Key-value pairs (e.g., `Environment: Production`)
- Case-sensitive
- Limit: 50 tags per resource

**Common Tag Examples:**

| Tag Key | Purpose | Example Values |
|---------|---------|----------------|
| `Environment` | Workload stage | Dev, Test, Production |
| `CostCenter` | Financial tracking | CC-1234, CC-5678 |
| `Owner` | Team responsible | TeamName or Email |
| `BusinessUnit` | Organizational unit | Finance, Engineering, Sales |
| `DataClassification` | Sensitivity level | Public, Internal, Confidential |

**Note:** Actual tag requirements will vary by organization.

**Enforcement via Policy:**
- Use "Require tag" policy (Deny effect)
- Use "Inherit tag from resource group" policy
- Apply tagging standards at subscription creation

**Best Practices:**
- Define tagging standard early
- Use Azure Policy to enforce
- Include tags in cost reports
- Automate tag application via IaC (Bicep)

**Reference:**
- [Resource naming and tagging](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming-and-tagging-decision-guide)
- [Define your tagging strategy](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-tagging)

---

### 1:50 – 1:58 | Cost Management (8 min)

#### Azure Cost Management Basics (8 min)

**Cost Hierarchy:**
- Costs roll up: Resource → Resource Group → Subscription → Management Group
- **Subscriptions are billing boundaries** (each gets its own invoice)

**Cost Management Tools:**
1. **Cost Analysis** — View spending by service, resource group, tag, location
2. **Budgets** — Set spending limits, get alerts at thresholds (50%, 80%, 100%)
3. **Advisor Recommendations** — Cost optimization suggestions
4. **Cost Alerts** — Get notified when thresholds are hit

**Pricing Models:**
- **Pay-as-you-go** — Per-second billing (most VMs, storage)
- **Reservations** — Commit 1-3 years, save up to 72%
- **Azure Hybrid Benefit** — Use existing Windows/SQL licenses
- **Spot VMs** — Use spare capacity at deep discounts (can be evicted)

**Cost Optimization Strategies:**
- Right-size VMs (use Advisor)
- Use auto-shutdown for dev/test VMs
- Delete unused resources
- Use tags for chargeback
- Monitor spending with budgets

**Reference:**
- [Azure Cost Management overview](https://learn.microsoft.com/en-us/azure/cost-management-billing/cost-management-billing-overview)
- [Plan to manage costs](https://learn.microsoft.com/en-us/azure/cost-management-billing/understand/plan-manage-costs)

---

### 1:58 – 2:08 | Cloud Adoption Framework — Bringing It All Together (10 min)

**Recap: What is CAF?**
- Microsoft's comprehensive guidance for cloud adoption
- **Not just Azure Fundamentals** — covers strategy, planning, governance, operations
- Industry best practices distilled

**CAF Methodology Phases:**
1. **Strategy** — Define business justification (Why cloud?)
2. **Plan** — Align organization, skills, cloud adoption plan
3. **Ready** — Prepare the environment (landing zones)
4. **Adopt** — Migrate and innovate (move workloads)
5. **Govern** — Implement controls (policies, cost, security)
6. **Manage** — Operations management (monitoring, patching, backup)

**CAF Design Areas (Landing Zone Architecture):**

| Design Area | What It Covers | Key Topics |
|-------------|----------------|------------|
| Azure billing and AD tenant | Tenant setup, subscription model | Multi-tenant vs single-tenant, subscription strategy |
| Identity and access | Entra ID, RBAC | Authentication, authorization, role assignments |
| Resource organization | Management groups, subscriptions, resource groups | Hierarchy, governance inheritance |
| Network topology | VNets, connectivity, segmentation | Hub-spoke, network security, hybrid connectivity |
| Security | Network security, encryption, monitoring | Zero trust, defense in depth |
| Management | Monitoring, backup, update management | Operational Excellence |
| Governance | Policies, tagging, compliance | Standards enforcement, compliance |
| Platform automation | IaC, CI/CD pipelines | Infrastructure as Code, deployment automation |

**The Big Picture:**
- Everything we learned in Sessions 1 & 2 fits into CAF design areas:
  - **Session 1:** Resource organization, network topology, Azure billing
  - **Session 2:** Identity, governance, management
- CAF provides the framework; organizations document implementation decisions

**Visual:** Show CAF methodology diagram + design areas
- Display: The 6 CAF methodology phases
- Show: How the 8 design areas map to Azure services
- Highlight: How enterprise organizations reference these design areas

**Key Takeaway:** Azure Fundamentals gives you the building blocks. CAF shows you how to assemble them into a secure, scalable, compliant cloud environment.

**Reference:**
- [Cloud Adoption Framework overview](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/)
- [Azure landing zone design areas](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-areas)
- [CAF Ready methodology](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/)

---

### 2:08 – 2:18 | Monitoring & Management Tools (Quick Overview) (10 min)

#### Azure Monitor
- **What it does:** Collect, analyze, act on telemetry from Azure and on-prem
- Metrics (performance counters) + Logs (diagnostic data)
- Create alerts based on thresholds

#### Log Analytics
- Query engine for logs (uses KQL — Kusto Query Language)
- Centralized log repository
- Used by Azure Monitor, Sentinel, Application Insights

#### Azure Service Health
- **Service Health** — Azure platform status (outages, maintenance)
- **Resource Health** — Health of your specific resources
- **Health Alerts** — Get notified of issues

#### Management Tools Summary

| Tool | Purpose | AWS Equivalent |
|------|---------|----------------|
| **Azure Portal** | Web-based GUI | AWS Console |
| **Azure CLI** | Command-line interface | AWS CLI |
| **Azure PowerShell** | PowerShell-based scripting | AWS Tools for PowerShell |
| **Cloud Shell** | Browser-based shell (Bash/PowerShell) | AWS CloudShell |
| **Azure Mobile App** | Manage resources on mobile | AWS Mobile App |
| **Bicep / ARM Templates** | Infrastructure as Code | CloudFormation |
| **Azure Arc** | Manage non-Azure resources | (No direct equivalent) |

**Reference:**
- [Azure Monitor overview](https://learn.microsoft.com/en-us/azure/azure-monitor/overview)
- [Azure Service Health](https://learn.microsoft.com/en-us/azure/service-health/overview)
- [Describe features and tools for managing and deploying Azure resources](https://learn.microsoft.com/en-us/training/modules/describe-features-tools-manage-deploy-azure-resources/)

---

### 2:18 – 2:28 | Hands-On Demos (10 min)

#### Demo 1: Azure RBAC in Action
1. Portal → Access Control (IAM) on a resource group
2. Assign "Contributor" role to a user
3. View "Role assignments" across scope
4. Show "My permissions" to see effective permissions

#### Demo 2: Azure Policy Configuration
1. Azure Portal → Policy
2. Browse built-in policies (e.g., "Allowed locations")
3. Assign policy to a resource group
4. Check compliance dashboard

#### Demo 3: Resource Tagging
1. Add tags to a resource group
2. Show how tags appear in cost analysis
3. Show policy that requires tags

#### Demo 4: Cost Management
1. Azure Portal → Cost Management + Billing
2. Cost Analysis filtered by tag (`Environment: Production`)
3. Create a budget with alert rule

---

### 2:28 – 2:30 | Final Wrap-up & Next Steps (2 min)

**Session 2 Key Takeaways:**
1. **RBAC** controls who can do what (start with least privilege)
2. **Azure Policy** enforces compliance and standards (use with tagging)
3. **Tagging** enables cost allocation and resource organization
4. **CAF** provides the framework; organizations document decisions
5. **Monitoring** ensures visibility and operational health

**Complete Picture (Sessions 1 + 2):**
- **Organize:** Hierarchy (Management Groups → Subscriptions → Resource Groups)
- **Deploy:** Services (Compute, Storage, Networking)
- **Control:** RBAC (who gets access)
- **Govern:** Policies + Tags (enforce standards)
- **Optimize:** Cost Management (track and reduce spend)
- **Monitor:** Azure Monitor (visibility and alerts)

**Your Journey with Azure:**
- You now have **foundational knowledge** (L100 complete ✅)
- **Next:** L200/L300 Platform Bootcamp — hands-on deep dive
- Optional: Ask the Expert sessions

**Certification Path (Optional):**
- [AZ-900: Azure Fundamentals](https://learn.microsoft.com/en-us/certifications/exams/az-900) — validates what you learned today
- Study resources: Microsoft Learn (free), practice exams

**Next Actions:**
- Review organizational cloud adoption documentation
- Explore the Azure Portal
- Attend Ask the Expert session (if available) — bring your questions

**Final Q&A:** 
- Any questions before we wrap?
- Feedback on the training?

**Thank You!**

---

## Additional Resources

**Official Microsoft Learn:**
- [Azure Fundamentals learning path](https://learn.microsoft.com/en-us/training/paths/azure-fundamentals-describe-azure-architecture-services/)
- [Describe Azure management and governance](https://learn.microsoft.com/en-us/training/paths/describe-azure-management-governance/)
- [Cloud Adoption Framework](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/)

**AWS to Azure Guides:**
- [Azure for AWS professionals](https://learn.microsoft.com/en-us/azure/architecture/aws-professional/)
- [Service comparison guide](https://learn.microsoft.com/en-us/azure/architecture/aws-professional/services)

---

*End of Session 2 — Azure Fundamentals L100 Complete*

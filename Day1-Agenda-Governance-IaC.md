# Advanced Azure Bootcamp — Day 1 Agenda
## Governance & Infrastructure as Code
**Duration:** 1.5 Hours

---

## Schedule Overview

| Time | Activity | Type |
|------|----------|------|
| 0:00 – 0:05 | Welcome & Session Objectives | Intro |
| 0:05 – 0:15 | Governance Context & Guardrails *(builds on previous bootcamp)* | Slides |
| 0:15 – 0:35 | Azure Resource Manager & IaC | Slides |
| 0:35 – 1:00 | Cloud Shell & Deployment Lab | Lab |
| 1:00 – 1:20 | Deployment Best Practices | Slides |
| 1:20 – 1:30 | Recap & Q&A | Discussion |

---

## Detailed Agenda

### 1. Welcome & Session Objectives (0:00 – 0:05)
- Introductions and housekeeping
- Overview of Day 1 learning outcomes
- Prerequisites check (Azure subscription access, Cloud Shell)

---

### 2. Governance Context & Guardrails (0:05 – 0:15) — *Slides*

> **Note:** Management group hierarchy, Azure Policy, and tagging strategy were covered in the previous bootcamp and are not repeated here.

**Topics:**
- Azure subscription model and purpose
- Defining environments (dev, test, prod) within Azure
- Guardrails: what they are and why they matter
- Ownership and accountability in cloud governance
- Environment separation strategies: subscriptions vs. resource groups
- Naming conventions and RBAC least-privilege access patterns

**Key Slides:**
- Shared responsibility model for governance
- Comparison table: environment separation approaches
- Example naming convention schema

---

### 3. Azure Resource Manager & Infrastructure as Code (0:15 – 0:35) — *Slides*

**Topics:**
- ARM overview: control plane, resource providers, and the ARM API
- ARM templates: structure (schema, parameters, variables, resources, outputs)
- Introduction to Bicep: syntax, benefits over raw JSON, transpilation
- IaC principles: idempotency, declarative configuration, version control

**Key Slides:**
- ARM request lifecycle diagram
- Side-by-side: ARM JSON template vs. Bicep equivalent
- IaC workflow: author → validate → deploy → drift detection

---

### 4. Cloud Shell & Deployment Lab (0:35 – 1:00) — *Hands-On Lab*

**Objective:** Deploy a resource using both an ARM template and a Bicep file via Azure Cloud Shell.

**Lab Steps:**

1. **Open Azure Cloud Shell** (Bash or PowerShell)
   - Verify CLI version: `az --version`
   - Set default subscription: `az account set --subscription <id>`

2. **Deploy an ARM template**
   ```bash
   az deployment group create \
     --resource-group rg-bootcamp-dev \
     --template-file azuredeploy-storage.json \
     --parameters storageAccountName=stbootcamp001
   ```

3. **Convert ARM template to Bicep**
   ```bash
   az bicep decompile --file azuredeploy-storage.json
   ```

4. **Deploy the Bicep file**
   ```bash
   az deployment group create \
     --resource-group rg-bootcamp-dev \
     --template-file azuredeploy-storage.bicep
   ```

5. **Validate idempotency** — re-run the deployment and observe no changes

**Expected Outcome:** A storage account deployed successfully via both ARM and Bicep, demonstrating IaC principles.

---

### 5. Deployment Best Practices (1:00 – 1:20) — *Slides*

**Topics:**
- What-if deployments: validate before you apply
- Scoping deployments: resource group vs. subscription vs. management group
- Storing templates in source control (Git)
- Parameterisation and environment-specific parameter files
- CI/CD pipeline integration overview (Azure DevOps / GitHub Actions)
- Common pitfalls: hardcoded values, missing locks, unmanaged drift

**Key Slides:**
- `az deployment group what-if` demo screenshot
- Git-based IaC workflow diagram
- Checklist: pre-deployment validation steps

---

### 6. Recap & Q&A (1:20 – 1:30)

- Summary of key takeaways:
  - Governance guardrails and ownership underpin reliable cloud environments
  - ARM and Bicep are the native IaC tools for Azure
  - Deployments should be repeatable, version-controlled, and validated
- Open Q&A
- Preview of Day 2: Networking (VNets, DNS, Load Balancing)

---

## Resources

- [Azure Governance documentation](https://learn.microsoft.com/en-us/azure/governance/)
- [ARM template reference](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/)
- [Bicep documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [Azure Policy built-in definitions](https://learn.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies)
- Lab file: `azuredeploy-storage.json` (provided in session materials)

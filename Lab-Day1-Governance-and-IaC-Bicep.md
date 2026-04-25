---
lab:
  title: 'Lab Day 1: Governance & Infrastructure as Code (Bicep)'
  module: 'Advanced Azure Bootcamp – Day 1'
---

# Lab Day 1 – Governance & Infrastructure as Code (Bicep)

## Lab Introduction

In this lab you build a governance foundation in Azure and then deploy infrastructure
using Bicep as the primary Infrastructure as Code language. You will work through the following tasks:
- Configure RBAC with least-privilege role assignment
- Enforce standards with Azure Policy, tags, and resource locks
- Understand Bicep language constructs: parameters, variables, resources, and outputs
- Author a Bicep file from scratch using decorators, conditions, and string interpolation
- Deploy Bicep files via Cloud Shell using environment-specific parameter files
- Apply deployment best practices — what-if, `.bicepparam` files, and deployment history

## Pre-requisites
- An Azure subscription
- A resource group named **RG-Lab1**
- Owner rights at the resource group level
- Bicep CLI v0.18 or later (pre-installed in Azure Cloud Shell)

## Estimated Timing: 90 minutes

## Lab Scenario

Your organisation is formalising its Azure landing zone ahead of a production workload migration. Before any infrastructure can be deployed, governance guardrails must be in place. You have been tasked with:

- Providing users with a Virtual Machine Contributor role to allow self-service provisioning of compute resources
- Enforcing mandatory `Environment`, `CostCenter`, and `Department` tags using Azure Policy
- Protecting shared resources against accidental deletion using resource locks
- Understanding Bicep's language constructs and how they map to Azure Resource Manager concepts
- Authoring a Bicep file from scratch that uses parameters with decorators, variables, conditions, resources, and outputs
- Deploying a baseline storage account using Bicep via Cloud Shell, with separate `.bicepparam` files for dev and prod
- Running a what-if check to validate changes before applying them

## Architecture Overview

```
Tenant Root Group
└── Root Management Group
    └── [Your Subscription]
        ├── RG-Lab1 (Resource Group)
        │   ├── Azure Policy Assignments (3 tag policies)
        │   ├── Resource Lock (delete lock)
        │   └── Storage Account (deployed via Bicep)
```

## Job Skills

- Task 1: Configure RBAC with least-privilege role assignment
- Task 2: Enforce governance with Azure Policy and resource tags
- Task 3: Protect resources with a resource lock
- Task 4: Understand Bicep language constructs
- Task 5: Author a Bicep file from scratch
- Task 6: Deploy Bicep via Cloud Shell with environment parameter files
- Task 7: Apply deployment best practices — what-if, `.bicepparam`, and history

---

## Task 1: Configure RBAC with Least-Privilege Role Assignment

Azure RBAC controls who can do what at which scope. In this task you assign a built-in role to an individual at the resource group scope.

1. Navigate to **RG-Lab1** and select **Access control (IAM)**.

2. Select the **Check access** tab and click **View my access** to see your current permissions on the resource group.

3. Select **Check access** button and in the **Check access** dialog search for any of your colleagues' user account. Notice that they have no access to the resource group.

4. Select the **Roles** tab and browse the list. Locate the **Virtual Machine Contributor** role and select **View under Details** to review the permissions it grants.

5. Visit Microsoft Learn: https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles and find the **Virtual Machine Contributor** role. Review the list of permissions it grants.

6. Return to the portal and select **+ Add** → **Add role assignment**.

7. Search for and select **Virtual Machine Contributor**. Select **Next**.

8. On the **Members** tab, select **+ Select members**. Search for your colleague's user account. Select **Select**.

9. Select **Review + assign** twice.

10. Go to the **Role assignments** tab. Confirm the recent assignment appears on the **Role assignments** tab.

Best practices for RBAC:
- Assign roles to groups, not individuals — this simplifies management and ensures access is automatically updated as users join or leave groups.
- Use the narrowest scope that satisfies the requirement — assign at the resource group level instead of subscription if access is only needed for that resource group.
- Use built-in roles where possible, but create custom roles to remove permissions that built-in roles include but your scenario does not require (e.g. if Virtual Machine Contributor includes permissions you do not want to grant, create a custom role that excludes those permissions).

---

## Task 2: Enforce Governance with Azure Policy and Resource Tags

Azure Policy lets you enforce standards automatically. In this task you assign a built-in policy that requires a tag on all resources, then test that it blocks non-compliant deployments.

1. In the Azure portal, search for and select **Policy**.

2. In the **Authoring** blade, select **Definitions**.

3. Search for `Require a tag on resources`. Select the policy and review its definition — note the `deny` effect, which blocks non-compliant resources at deployment time.

4. Select **Assign policy**.

5. Set the **Scope** by selecting the ellipsis (…):

   | Setting | Value |
   | --- | --- |
   | Resource Group | **RG-Lab1** |

   Select **Select**.

6. Select **Next** and set **Parameters**:

   | Setting | Value |
   | --- | --- |
   | Tag Name | `Environment` |

7. On the **Non-compliance messages** tab, enter "Require Environment Tag on Resource".

8. Select **Review + Create**, then **Create**.

9. Repeat the **Assign policy** steps (Steps 4-8) two more times to enforce additional required tags:

   **Second assignment – Cost Center:**
   | Setting | Value |
   | --- | --- |
   | Tag Name | `CostCenter` |
   | Non-compliance message | `Require CostCenter Tag on Resource` |

   **Third assignment – Department:**
   | Setting | Value |
   | --- | --- |
   | Tag Name | `Department` |
   | Non-compliance message | `Require Department Tag on Resource` |

**Note:**
- The built-in policy `Require a tag on resources` enforces one tag per assignment. Assigning it once per required tag is the standard approach. Alternatively, use a Policy Initiative (policy set) to group all tag requirements into a single assignment.
- Policy enforcement typically takes 5–10 minutes to activate.

### Test the policy enforcement

1. Search for and select **Storage accounts**. Select **+ Create**.

2. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab1** |
   | Storage account name | `stlabyourname` |
   | Region | **Australia East** |
   | Preferred storage type | **Azure Blob Storage or Azure Data Lake Storage Gen 2** |
   | Redundancy | **LRS** |

3. **Do not add any tags.** Select **Review + Create**.

4. You should receive a **Validation failed** error. Select the Tags tab and confirm the error messages reference the missing tags.

**What just happened:** The ARM control plane evaluated the deployment request against all assigned policies before provisioning any resource. The `deny` effect caused the deployment to be rejected before anything was created.

5. In the **Tags** tab add all three required tags:

   | Name | Value |
   | --- | --- |
   | Environment | `dev` |
   | CostCenter | `CC1234` |
   | Department | `IT` |

6. Select **Review + Create**. This time validation passes.

7. Select **Create** and wait for the deployment to succeed.

8. Go to the Resource Group, expand the **Settings** blade and select **Policies**. You can see the compliance state of the Resource Group.

---

## Task 3: Protect Resources with a Resource Lock

Resource locks prevent accidental deletion or modification, independent of RBAC permissions. An Owner can be blocked from deleting a resource if a lock is present.

1. Navigate to **RG-Lab1**.

2. In the **Settings** blade, select **Locks**.

3. Select **+ Add** and configure:

   | Setting | Value |
   | --- | --- |
   | Lock name | `resourcegroup-nodelete` |
   | Lock type | **Delete** |
   | Notes | `Protects lab resource group from accidental deletion` |

4. Select **OK**.

5. Return to the **Overview** blade. Select **Delete resource group**.

6. Copy your resource group name and paste it in the confirmation field and select **Delete**.

7. You should receive a deletion failure notification. Confirm the error references the lock.

**Key point:** Resource locks operate at the Azure control plane layer and apply regardless of user permissions. A delete lock prevents removal; a read-only lock prevents both modifications and deletions. Locks must be explicitly removed before a resource can be deleted — this adds a deliberate friction step.

---

## Task 4: Understand Bicep Language Constructs

Bicep is the recommended Infrastructure as Code language for Azure. It is a domain-specific language that compiles to ARM JSON — everything you can express in ARM JSON, you can express in Bicep with cleaner syntax, type safety, and better tooling support.

### Bicep vs ARM JSON

| Concept | ARM JSON | Bicep |
| --- | --- | --- |
| Schema declaration | `"$schema": "https://..."` | Not required |
| Parameter declaration | `"parameters": { "name": { "type": "string" } }` | `param name string` |
| Variable declaration | `"variables": { "sku": "Standard_LRS" }` | `var sku = 'Standard_LRS'` |
| Resource declaration | Nested JSON object with `"type"` and `"apiVersion"` fields | `resource sa 'Microsoft.Storage/storageAccounts@2023-01-01' = { ... }` |
| String interpolation | `"[concat('prefix', parameters('name'))]"` | `'prefix${name}'` |
| Conditional expression | `"[if(equals(parameters('env'), 'prod'), 'GRS', 'LRS')]"` | `env == 'prod' ? 'GRS' : 'LRS'` |
| Output declaration | `"outputs": { "id": { "type": "string", "value": "..." } }` | `output id string = sa.id` |

### Bicep language constructs

**Parameters** accept values at deployment time. Decorators (prefixed with `@`) add metadata and validation:

```bicep
@description('Deployment environment.')
@allowed(['dev', 'test', 'prod'])
param environment string = 'dev'

@description('Base name for the storage account.')
@minLength(3)
@maxLength(11)
param storageAccountName string
```

**Variables** compute values from parameters. Bicep supports ternary expressions and string interpolation:

```bicep
// Ternary: choose SKU based on environment
var storageSku = environment == 'prod' ? 'Standard_GRS' : 'Standard_LRS'

// String interpolation: combine strings without concat()
var fullName = '${storageAccountName}${environment}'
```

**Resources** use a symbolic name, resource type, API version, and a properties block. You reference other resources by their symbolic name:

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: fullName
  location: resourceGroup().location
  sku: {
    name: storageSku
  }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
  }
}
```

**Outputs** return values after the deployment completes. Bicep lets you reference resource properties directly via the symbolic name — no `resourceId()` or `reference()` functions needed:

```bicep
output storageAccountName string = storageAccount.name
output storageAccountId string = storageAccount.id
output blobEndpoint string = storageAccount.properties.primaryEndpoints.blob
```

### How the constructs connect

```
param storageAccountName = 'mystore'       // input
param environment        = 'dev'           // input
          ↓
var fullName = 'mystoredev'                // computed
var storageSku = 'Standard_LRS'           // computed (env ≠ 'prod')
          ↓
resource storageAccount  name='mystoredev' // deployed
          ↓
output storageAccountId  = storageAccount.id  // returned
```

---

## Task 5: Author a Bicep File from Scratch

In this task you write a complete Bicep file that deploys a tagged storage account, using all the constructs from Task 4.

### Open Cloud Shell

1. Select the **Cloud Shell** icon in the top-right of the Azure portal.

2. When prompted, select **Bash**.

3. On the **Getting started** screen, select **No storage account required**, and select **Apply**. It may take a minute for the shell to provision.

4. Verify your Bicep version:

   ```bash
   az bicep version
   ```
   If not found, install Bicep

   ```bash
   az bicep install
   ```

5. Confirm you are using the correct subscription:

   ```bash
   az account show --query "{name:name, id:id}" -o table
   ```

   If not, set it:

   ```bash
   az account set --subscription "<your-subscription-id>"
   ```

### Create the Bicep file

1. Open the Cloud Shell editor:

   ```bash
   code main.bicep
   ```

2. Paste in the following Bicep file:

```bicep
// ── Parameters ────────────────────────────────────────────────────────────────

@description('Base name for the storage account (3–11 characters, lowercase).')
@minLength(3)
@maxLength(11)
param storageAccountName string

@description('Deployment environment.')
@allowed(['dev', 'test', 'prod'])
param environment string = 'dev'

@description('Cost center code for billing.')
param costCenter string

@description('Department that owns this resource.')
param department string

@description('Azure region for all resources.')
param location string = resourceGroup().location

// ── Variables ─────────────────────────────────────────────────────────────────

// Use GRS replication for production; LRS is sufficient for non-prod
var storageSku = environment == 'prod' ? 'Standard_GRS' : 'Standard_LRS'

// Append a deterministic hash so the name is globally unique, then cap at 24 characters
// (storageAccountName ≤11 + environment ≤4 + uniqueString 13 = up to 28; take() enforces the limit)
var fullStorageName = take('${storageAccountName}${environment}${uniqueString(resourceGroup().id)}', 24)

// Centralise tags so every resource shares the same values
var commonTags = {
  Environment: environment
  CostCenter: costCenter
  Department: department
}

// ── Resources ─────────────────────────────────────────────────────────────────

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: fullStorageName
  location: location
  sku: {
    name: storageSku
  }
  kind: 'StorageV2'
  tags: commonTags
  properties: {
    accessTier: 'Hot'
    minimumTlsVersion: 'TLS1_2'
    supportsHttpsTrafficOnly: true
  }
}

// ── Outputs ───────────────────────────────────────────────────────────────────

output storageAccountName string = storageAccount.name
output storageAccountId string = storageAccount.id
output blobEndpoint string = storageAccount.properties.primaryEndpoints.blob
```

3. Use **Ctrl+S** to save, **Ctrl+Q** to close the editor.

### Understand what each section does

| Section | Purpose in this file |
| --- | --- |
| **Parameters** | Accept `storageAccountName`, `environment`, `costCenter`, `department`, and `location` at deploy time. Decorators add validation and documentation. |
| **Variables** | Compute the SKU (GRS for prod, LRS otherwise), build the globally-unique storage name, and assemble the shared tag object. |
| **Resources** | Deploy a `StorageV2` storage account using the computed values. Symbolic name `storageAccount` is used in outputs. |
| **Outputs** | Return the final storage account name, resource ID, and blob endpoint after deployment. Bicep resolves these from the symbolic name — no `reference()` function needed. |

**Key Bicep advantages demonstrated:**
- `@description`, `@allowed`, `@minLength`, `@maxLength` decorators replace verbose JSON metadata blocks
- String interpolation `'${var}'` replaces `concat()` ARM functions
- Ternary expression `x ? a : b` replaces the ARM `if()` function
- Symbolic name `storageAccount.id` replaces `resourceId(...)` in outputs
- No `$schema`, `contentVersion`, or `apiVersion` boilerplate at the file level

---

## Task 6: Deploy Bicep via Cloud Shell with Environment Parameter Files

In this task you deploy the Bicep file authored in Task 5 using environment-specific parameter files, then verify idempotency.

### Create environment parameter files

Bicep supports native `.bicepparam` files (Bicep 0.18+), which are strongly typed and can be validated by the Bicep compiler. This is preferred over JSON parameter files.

1. Open the Cloud Shell editor to create the dev parameter file:

   ```bash
   code parameters.dev.bicepparam
   ```

2. Paste the following, replacing `yourname` with your name to avoid naming conflicts:

   ```bicep
   using 'main.bicep'

   param storageAccountName = 'stlabdevyourname'
   param environment = 'dev'
   param costCenter = 'CC1234'
   param department = 'IT'
   ```

   Use **Ctrl+S** to save, **Ctrl+Q** to close.

3. Create the prod parameter file:

   ```bash
   code parameters.prod.bicepparam
   ```

4. Paste the following, again replacing `yourname`:

   ```bicep
   using 'main.bicep'

   param storageAccountName = 'stlabprodyourname'
   param environment = 'prod'
   param costCenter = 'CC1234'
   param department = 'IT'
   ```

   Use **Ctrl+S** to save, **Ctrl+Q** to close.

**Why `.bicepparam` over `.json`?**
- The `using 'main.bicep'` reference lets the Bicep compiler validate that all required parameters are provided and that values match allowed types and decorator constraints.
- You get IntelliSense and type checking in VS Code with the Bicep extension.
- Parameter names are plain identifiers — no JSON boilerplate.

### Deploy using the dev parameter file

1. Run the deployment:

   ```bash
   az deployment group create \
     --resource-group RG-Lab1 \
     --template-file main.bicep \
     --parameters parameters.dev.bicepparam
   ```

2. Wait for the output to show `"provisioningState": "Succeeded"`.

3. Inspect the outputs returned by the deployment:

   ```bash
   az deployment group show \
     --resource-group RG-Lab1 \
     --name main \
     --query properties.outputs
   ```

   Confirm `storageAccountName`, `storageAccountId`, and `blobEndpoint` are all present.

4. List storage accounts to confirm the new resource:

   ```bash
   az storage account list \
     --resource-group RG-Lab1 \
     --query "[].{Name:name, SKU:sku.name, Location:location}" \
     -o table
   ```

### Verify idempotency

1. Run the exact same deployment command again:

   ```bash
   az deployment group create \
     --resource-group RG-Lab1 \
     --template-file main.bicep \
     --parameters parameters.dev.bicepparam
   ```

2. Confirm the output again shows `"provisioningState": "Succeeded"` with no errors and no new resource created.

**Idempotency** means running the same deployment twice produces the same end state without duplicating or erroring on existing resources. This is a core IaC principle — your Bicep deployments should be safe to re-run at any time.

### Deploy using the prod parameter file

1. Run the prod deployment:

   ```bash
   az deployment group create \
     --resource-group RG-Lab1 \
     --template-file main.bicep \
     --parameters parameters.prod.bicepparam
   ```

2. Confirm success, then list storage accounts and verify the prod account has `Standard_GRS` SKU while the dev account has `Standard_LRS`.

   ```bash
   az storage account list \
     --resource-group RG-Lab1 \
     --query "[].{Name:name, SKU:sku.name}" \
     -o table
   ```

   This confirms the ternary variable `storageSku` correctly selected a different SKU for prod.

---

## Task 7: Apply Deployment Best Practices — What-If, Parameters, History

### What-if: validate before you apply

The `--what-if` flag shows exactly what a deployment *would* change without making any changes. Use this before every production deployment.

1. In Cloud Shell, open your dev parameter file to make a change:

   ```bash
   code parameters.dev.bicepparam
   ```

2. Change `accessTier` is controlled by the Bicep file, but you can test what-if by updating the `storageAccountName` value to `stlabwhatifyourname` (replacing `yourname` with your name):

   ```bicep
   using 'main.bicep'

   param storageAccountName = 'stlabwhatifyourname'
   param environment = 'dev'
   param costCenter = 'CC1234'
   param department = 'IT'
   ```

   Use **Ctrl+S** to save, **Ctrl+Q** to close.

3. Run a what-if check:

   ```bash
   az deployment group what-if \
     --resource-group RG-Lab1 \
     --template-file main.bicep \
     --parameters parameters.dev.bicepparam
   ```

4. Review the output. Notice the colour-coded change indicators:
   - **Create** (green `+`) — new resource will be added
   - **Modify** (yellow `~`) — existing resource will be changed
   - **Delete** (red `-`) — resource will be removed
   - **No change** (grey `=`) — resource is already in the desired state

5. Only proceed with `az deployment group create` once you are satisfied with the what-if output:

   ```bash
   az deployment group create \
     --resource-group RG-Lab1 \
     --template-file main.bicep \
     --parameters parameters.dev.bicepparam
   ```

### Linting with the Bicep CLI

Bicep includes a built-in linter that catches common authoring mistakes before deployment.

1. Run the linter against your Bicep file:

   ```bash
   az bicep lint --file main.bicep
   ```

2. Review any warnings or errors. The linter checks for issues such as:
   - Parameters declared but never used
   - Hardcoded locations (should use `resourceGroup().location`)
   - Missing `@description` decorators on public parameters
   - Use of deprecated API versions

3. Fix any reported issues, then re-run the linter to confirm a clean result.

### Review deployment history

1. In the Azure portal, navigate to **RG-Lab1**.

2. In the **Settings** blade, select **Deployments**.

3. Review the list of all deployments made during this lab — each entry records the template, all parameter values, start time, duration, and outcome.

4. Select one deployment and open the **Template** blade. Confirm the ARM JSON that Bicep compiled to is preserved for audit purposes.

5. Select the **Inputs** blade. Confirm that all parameter values passed at deploy time are recorded.

**Common pitfalls to avoid:**
- Hardcoded resource names in Bicep files — use `param` instead
- Missing `@allowed` decorators on environment parameters — allows invalid values to slip through
- Skipping `--what-if` before production deployments
- Not committing `.bicepparam` files to source control — they are part of your IaC and should be versioned
- Deploying directly from the Azure portal without a Bicep file — creates unmanaged drift

---

## Cleanup

**Note:** Remove the resource lock before deleting the resource group, otherwise the deletion will be blocked.

1. Go to your Resource Group.

2. In the **Settings** → **Locks**, select the lock and click **Delete**.

3. In the **Overview**, select **Delete resource group**.

4. Copy and paste the name to confirm deletion.

5. Select **Delete**.

---

## Key Takeaways

- **RBAC least privilege**: assign roles to groups, not individuals; use the narrowest scope that satisfies the requirement; use built-in roles as much as possible.

- **Azure Policy** enforces governance standards at the control plane before resources are created. The `deny` effect prevents non-compliant deployments; the `modify` effect with a remediation task corrects existing resources.

- **Resource locks** protect critical resources from accidental deletion or modification regardless of RBAC permissions — they add deliberate friction.

- **Bicep** is the recommended authoring language for Azure IaC. It compiles to ARM JSON, so every Bicep deployment goes through Azure Resource Manager. Bicep offers cleaner syntax, type safety, built-in linting, and better tooling support than raw ARM JSON.

- **Parameters with decorators** (`@description`, `@allowed`, `@minLength`) make Bicep files self-documenting and self-validating without extra tooling.

- **Ternary expressions and string interpolation** replace verbose ARM functions like `if()`, `concat()`, and `reference()`, making template logic easier to read and maintain.

- **Symbolic names** allow Bicep to resolve resource properties (e.g. `storageAccount.id`) without `resourceId()` calls, and enable the compiler to infer implicit `dependsOn` relationships.

- **`.bicepparam` files** (Bicep native parameter files) are strongly typed, compiler-validated, and preferred over JSON parameter files. The `using` directive links the parameter file to its template, enabling full IntelliSense and validation.

- **Idempotency** means the same deployment can be safely re-run at any time and will converge to the same result without duplicating resources.

- **What-if** (`--what-if`) is a mandatory step before production deployments — it shows exactly what will change before anything is applied.

---

## Resources

- [Bicep documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [Bicep language reference](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/bicep-functions)
- [Bicep decorators](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/parameters#decorators)
- [Bicep parameter files (.bicepparam)](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/parameter-files)
- [Bicep linter rules](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/linter)
- [Azure Governance documentation](https://learn.microsoft.com/en-us/azure/governance/)
- [Azure RBAC built-in roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles)
- [Azure Policy built-in definitions](https://learn.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies)

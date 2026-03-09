---
lab:
  title: 'Lab Day 1: Governance & Infrastructure as Code'
  module: 'Advanced Azure Bootcamp – Day 1'
---

# Lab Day 1 – Governance & Infrastructure as Code

## Lab Introduction

In this lab you build a governance foundation in Azure and then deploy infrastructure
using both ARM templates and Bicep. You will work through the full delivery chain:
organise subscriptions into management groups, enforce standards with Azure Policy and
resource locks, author and deploy ARM templates, convert them to Bicep, and validate
deployments using what-if before applying changes.

This lab requires an Azure subscription and Owner or Contributor rights at the
subscription level. Steps are written using **Australia East** as the region but you
may change this to suit your environment.

## Estimated Timing: 75 minutes

## Lab Scenario

Your organisation is formalising its Azure landing zone ahead of a production
workload migration. Before any infrastructure can be deployed, governance guardrails
must be in place. You have been tasked with:

- Creating a management group hierarchy to organise subscriptions by environment
- Enforcing a mandatory `Environment` tag using Azure Policy
- Protecting shared resources against accidental deletion using resource locks
- Deploying a baseline storage account using an ARM template and a Bicep file via
  Cloud Shell, using separate parameter files for dev and prod
- Running a what-if check to validate changes before applying them

## Architecture Overview

```
Tenant Root Group
└── mg-myob-prod (Management Group)
    └── [Your Subscription]
        ├── rg-governance-lab (Resource Group)
        │   ├── Azure Policy Assignment (Require Environment tag)
        │   ├── Resource Lock (delete lock)
        │   └── Storage Account (deployed via ARM + Bicep)
```

## Job Skills

- Task 1: Create a management group and review subscription scope
- Task 2: Configure RBAC with least-privilege role assignment
- Task 3: Enforce governance with Azure Policy and resource tags
- Task 4: Protect resources with a resource lock
- Task 5: Export and understand an ARM template
- Task 6: Deploy ARM and Bicep templates via Cloud Shell
- Task 7: Apply deployment best practices — parameters, what-if, and history

---

## Task 1: Create a Management Group and Review Subscription Scope (0:00 – 0:10)

Management groups let you organise subscriptions and apply policy and RBAC at scale.
In this task you create a management group representing your production environment
boundary, then associate your subscription with it.

1. Sign in to the **Azure portal** – `https://portal.azure.com`.

1. Search for and select **Management groups**.

1. Select **+ Create**.

1. Provide the following values and select **Submit**:

   | Setting | Value |
   | --- | --- |
   | Management group ID | `mg-myob-prod` |
   | Management group display name | `mg-myob-prod` |

1. **Refresh** the page and confirm the management group appears under the
   **Tenant Root Group**.

   > **Why this matters:** Every subscription that belongs to `mg-myob-prod` will
   > inherit any Azure Policy assignments and RBAC role assignments made at this
   > level. This is how guardrails are applied consistently across multiple
   > subscriptions without repeating configuration.

1. Select **mg-myob-prod** and then select **Subscriptions**.

1. Select **+ Add** and add your current subscription to the management group.
   Select **Save**.

   > **Environment separation pattern:** In a production landing zone you would
   > create separate management groups per environment tier (e.g. `mg-myob-dev`,
   > `mg-myob-test`, `mg-myob-prod`), each containing the relevant subscriptions.
   > This allows different policies to apply to dev vs prod without manual
   > per-subscription configuration. Within a single subscription, resource groups
   > can separate environments, but subscription-level separation provides stronger
   > isolation for billing, blast radius, and policy inheritance.

---

## Task 2: Configure RBAC with Least-Privilege Role Assignment (0:10 – 0:20)

Azure RBAC controls who can do what at which scope. In this task you assign a
built-in role to a group at the management group scope, then create the resource
group that later tasks will use.

### Create the resource group

1. Search for and select **Resource groups**.

1. Select **+ Create** and provide the following:

   | Setting | Value |
   | --- | --- |
   | Subscription | *your subscription* |
   | Resource group name | `rg-governance-lab` |
   | Region | **Australia East** |

1. Select the **Tags** tab. Add a tag:

   | Name | Value |
   | --- | --- |
   | Environment | dev |

1. Select **Review + Create**, then **Create**.

### Review and assign a built-in role

1. Navigate to **mg-myob-prod** and select **Access control (IAM)**.

1. Select the **Roles** tab and browse the list. Locate and open the
   **Contributor** role. Review the **Permissions** and **JSON** tabs.

   > **Note:** The `Actions`, `NotActions`, and `AssignableScopes` fields in the
   > JSON define exactly what the role can and cannot do. This is the same format
   > you edit when creating a custom role.

1. Select **+ Add** → **Add role assignment**.

1. Search for and select **Reader**. Select **Next**.

1. On the **Members** tab, select **+ Select members**. Search for your own
   account or a test user/group. Select **Select**.

1. Select **Review + assign** twice.

   > **Best practice:** Always assign roles to groups, not individual users.
   > Assigning at management group scope means the role applies to all
   > subscriptions beneath it — use the narrowest scope that satisfies the
   > requirement.

1. Confirm the assignment appears on the **Role assignments** tab.

---

## Task 3: Enforce Governance with Azure Policy and Resource Tags (0:20 – 0:35)

Azure Policy lets you enforce standards automatically. In this task you assign a
built-in policy that requires an `Environment` tag on all resources, then test that
it blocks non-compliant deployments.

### Assign the policy

1. In the Azure portal, search for and select **Policy**.

1. In the **Authoring** blade, select **Definitions**.

1. Search for `Require a tag and its value on resources`. Select the policy and
   review its definition — note the `deny` effect, which blocks non-compliant
   resources at deployment time.

1. Select **Assign policy**.

1. Set the **Scope** by selecting the ellipsis (…):

   | Setting | Value |
   | --- | --- |
   | Subscription | *your subscription* |
   | Resource Group | **rg-governance-lab** |

   Select **Select**.

1. Configure **Basics**:

   | Setting | Value |
   | --- | --- |
   | Assignment name | `Require Environment tag on all resources` |
   | Description | `Blocks deployment of resources missing the Environment tag` |
   | Policy enforcement | **Enabled** |

1. Select **Next** and set **Parameters**:

   | Setting | Value |
   | --- | --- |
   | Tag Name | `Environment` |
   | Tag Value | `dev` |

1. Select **Review + Create**, then **Create**.

   > **Note:** Policy enforcement typically takes 5–10 minutes to activate.

### Test the policy enforcement

1. Search for and select **Storage accounts**. Select **+ Create**.

1. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **rg-governance-lab** |
   | Storage account name | `stlabpolicytest001` |
   | Region | **Australia East** |
   | Redundancy | **LRS** |

1. **Do not add any tags.** Select **Review**, then **Create**.

1. You should receive a **Validation failed** error. Select the error to view
   details — confirm it references your policy assignment
   `Require Environment tag on all resources`.

   > **What just happened:** The ARM control plane evaluated the deployment
   > request against all assigned policies before provisioning any resource.
   > The `deny` effect caused the deployment to be rejected before anything was
   > created.

1. Return to **Basics**, then select the **Tags** tab and add:

   | Name | Value |
   | --- | --- |
   | Environment | dev |

1. Select **Review**, then **Create**. This time validation passes.

1. Wait for the deployment to succeed, then select **Go to resource**.

   > **Optional – remediation:** In production you would also assign the
   > `Inherit a tag from the resource group if missing` policy with a remediation
   > task so existing non-compliant resources are automatically updated. This
   > mirrors the approach in LAB_02b.

---

## Task 4: Protect Resources with a Resource Lock (0:35 – 0:40)

Resource locks prevent accidental deletion or modification, independent of RBAC
permissions. An Owner can be blocked from deleting a resource if a lock is present.

1. Navigate to **rg-governance-lab**.

1. In the **Settings** blade, select **Locks**.

1. Select **+ Add** and configure:

   | Setting | Value |
   | --- | --- |
   | Lock name | `rg-governance-lab-nodelete` |
   | Lock type | **Delete** |
   | Notes | `Protects lab resource group from accidental deletion` |

1. Select **OK**.

1. Return to the **Overview** blade. Select **Delete resource group**.

1. Enter `rg-governance-lab` in the confirmation field and select **Delete**.

1. You should receive a deletion failure notification. Confirm the error
   references the lock.

   > **Key point:** Resource locks operate at the Azure control plane layer and
   > apply regardless of user permissions. A delete lock prevents removal;
   > a read-only lock prevents both modifications and deletions. Locks must be
   > explicitly removed before a resource can be deleted — this adds a deliberate
   > friction step.

---

## Task 5: Export and Understand an ARM Template (0:40 – 0:50)

Before writing templates, it helps to read one. In this task you export the ARM
template for the storage account already deployed, examine its structure, and
edit it to understand parameterisation.

### Export the template

1. Navigate to the storage account created in Task 3 (`stlabpolicytest001`).

1. In the left blade, under **Automation**, select **Export template**.

1. Take 2 minutes to examine the two files:

   **Template file** (`template.json`) — notice these sections:
   - `$schema` — declares the deployment schema and API version
   - `contentVersion` — version string for your own change tracking
   - `parameters` — values injected at deployment time (no hardcoding)
   - `variables` — computed values derived from parameters
   - `resources` — the actual Azure resources to create or update
   - `outputs` — values returned after deployment (e.g. resource IDs, endpoints)

   **Parameters file** (`parameters.json`) — a separate file that supplies values
   for each parameter, allowing the same template to be re-used across environments
   by swapping the parameter file.

1. Select **Download** to save both files locally.

   > **Why separate parameters?** A template describes *what* to deploy.
   > A parameters file describes *how* to deploy it for a specific environment.
   > Storing `parameters.dev.json` and `parameters.prod.json` in source control
   > alongside your template means environment differences are explicit, auditable,
   > and reproducible.

### Edit the template for reuse

1. Open `template.json` in a text editor.

1. In the `parameters` section, find the storage account name parameter.
   Change its default value to `stlabdeploy001`.

1. Save the file.

---

## Task 6: Deploy ARM and Bicep Templates via Cloud Shell (0:50 – 1:10)

In this task you use Azure Cloud Shell to deploy the storage account using both the
ARM template and a Bicep equivalent. You will also verify idempotency by re-running
a deployment with no changes.

### Open Cloud Shell

1. Select the **Cloud Shell** icon in the top-right of the Azure portal
   (or navigate to `https://shell.azure.com`).

1. When prompted, select **Bash**.

1. On the **Getting started** screen, select **Mount storage account**, select
   your subscription, and select **Apply**.

1. Select **I want to create a storage account**, complete the form, and
   select **Next**, then **Create**.

   > It may take a minute for the shell to provision.

1. Verify your CLI version:

   ```bash
   az --version
   ```

1. Confirm you are using the correct subscription:

   ```bash
   az account show --query "{name:name, id:id}" -o table
   ```

   If not, set it:

   ```bash
   az account set --subscription "<your-subscription-id>"
   ```

### Upload your template and parameter files

1. Select the **Manage files** (upload/download) icon in the Cloud Shell toolbar
   and select **Upload**.

1. Upload both `template.json` and `parameters.json` downloaded in Task 5.

1. Confirm they are present:

   ```bash
   ls
   ```

### Deploy using the ARM template

```bash
az deployment group create \
  --resource-group rg-governance-lab \
  --template-file template.json \
  --parameters parameters.json \
  --parameters storageAccountName=stlabarm001
```

1. Wait for the output to show `"provisioningState": "Succeeded"`.

1. List the storage accounts in the resource group to confirm:

   ```bash
   az storage account list \
     --resource-group rg-governance-lab \
     --query "[].{Name:name, Location:location, Sku:sku.name}" \
     -o table
   ```

### Verify idempotency

1. Run the exact same deployment command again:

   ```bash
   az deployment group create \
     --resource-group rg-governance-lab \
     --template-file template.json \
     --parameters parameters.json \
     --parameters storageAccountName=stlabarm001
   ```

1. Confirm the output again shows `"provisioningState": "Succeeded"` with no
   errors and no new resource created.

   > **Idempotency** means running the same deployment twice produces the same
   > end state without duplicating or erroring on existing resources. This is a
   > core IaC principle — your deployments should be safe to re-run at any time.

### Convert the ARM template to Bicep

1. In Cloud Shell, run:

   ```bash
   az bicep decompile --file template.json
   ```

1. Open the generated `.bicep` file in the Cloud Shell editor:

   ```bash
   code template.bicep
   ```

1. Compare the Bicep syntax to the ARM JSON. Notice:
   - No `$schema` or `contentVersion` boilerplate
   - Parameters declared with `param` keyword and types (`string`, `int`, `bool`)
   - Resources use a clean `resource` block with symbolic names
   - String interpolation uses `'${paramName}'` syntax instead of `[parameters('paramName')]`

1. Change the storage account name in the Bicep file to `stlabbicep001`.
   Use **Ctrl+S** to save, **Ctrl+Q** to close the editor.

### Deploy using the Bicep file

```bash
az deployment group create \
  --resource-group rg-governance-lab \
  --template-file template.bicep
```

1. Confirm `"provisioningState": "Succeeded"`.

1. List storage accounts again to confirm both `stlabarm001` and `stlabbicep001`
   exist:

   ```bash
   az storage account list \
     --resource-group rg-governance-lab \
     --query "[].{Name:name}" \
     -o table
   ```

   > **ARM vs Bicep:** ARM JSON is the wire format — all Bicep files transpile to
   > ARM JSON before being sent to Azure Resource Manager. Bicep is the
   > recommended authoring experience for new templates. Use ARM JSON when
   > integrating with tooling that does not yet support Bicep.

---

## Task 7: Deployment Best Practices – What-If, Parameters, History (1:10 – 1:20)

### What-if: validate before you apply

The `--what-if` flag shows exactly what a deployment *would* change without making
any changes. Use this before every production deployment.

1. In Cloud Shell, edit `template.bicep` and change the storage account name to
   `stlabwhatif001` (or change the SKU from `Standard_LRS` to `Standard_GRS`).
   Save and close.

1. Run a what-if check:

   ```bash
   az deployment group what-if \
     --resource-group rg-governance-lab \
     --template-file template.bicep
   ```

1. Review the output. Notice the colour-coded change indicators:
   - **Create** (green `+`) — new resource will be added
   - **Modify** (yellow `~`) — existing resource will be changed
   - **Delete** (red `-`) — resource will be removed
   - **No change** (grey `=`) — resource is already in the desired state

1. Only proceed with `az deployment group create` once you are satisfied with
   the what-if output.

### Environment-specific parameter files

In practice, do not rely only on inline `--parameters` overrides. Create separate
parameter files per environment:

1. In Cloud Shell, create a dev parameter file:

   ```bash
   cat > parameters.dev.json << 'EOF'
   {
     "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
     "contentVersion": "1.0.0.0",
     "parameters": {
       "storageAccountName": {
         "value": "stlabdev001"
       },
       "location": {
         "value": "australiaeast"
       }
     }
   }
   EOF
   ```

1. Create a prod parameter file:

   ```bash
   cat > parameters.prod.json << 'EOF'
   {
     "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
     "contentVersion": "1.0.0.0",
     "parameters": {
       "storageAccountName": {
         "value": "stlabprod001"
       },
       "location": {
         "value": "australiaeast"
       }
     }
   }
   EOF
   ```

1. Deploy using the dev parameter file:

   ```bash
   az deployment group create \
     --resource-group rg-governance-lab \
     --template-file template.bicep \
     --parameters @parameters.dev.json
   ```

   > By committing `template.bicep`, `parameters.dev.json`, and
   > `parameters.prod.json` to a Git repository, you get a full audit trail of
   > who changed what and when. CI/CD pipelines (Azure DevOps or GitHub Actions)
   > then run `az deployment group what-if` on pull requests and
   > `az deployment group create` on merge — no manual portal deployments.

### Review deployment history

1. In the Azure portal, navigate to **rg-governance-lab**.

1. In the **Settings** blade, select **Deployments**.

1. Review the list of all deployments made during this lab — each entry records
   the template, all parameter values, start time, duration, and outcome.

1. Select one deployment and open the **Template** blade. Confirm the exact
   template used is preserved for audit purposes.

   > **Common pitfalls to avoid:**
   > - Hardcoded resource names in templates (use parameters instead)
   > - Missing resource locks on shared/production resources
   > - Deploying directly from portal without template — creates unmanaged drift
   > - Not using what-if before production deployments
   > - Storing parameter files with secrets in source control (use Key Vault
   >   references in parameter files instead)

---

## Cleanup

If you are using your own subscription, remove lab resources to avoid ongoing costs.

> **Note:** Remove the resource lock before deleting the resource group,
> otherwise the deletion will be blocked.

1. Remove the lock first:

   ```bash
   az lock delete \
     --name rg-governance-lab-nodelete \
     --resource-group rg-governance-lab
   ```

1. Delete the resource group:

   ```bash
   az group delete --name rg-governance-lab --yes --no-wait
   ```

1. Remove the management group:

   ```bash
   az account management-group delete --name mg-myob-prod
   ```

---

## Key Takeaways

- **Management groups** organise subscriptions into a hierarchy that enables policy
  and RBAC inheritance at scale. Environment separation at the subscription level
  (dev/test/prod) provides stronger isolation than resource groups alone.

- **RBAC least privilege**: assign roles to groups, not individuals; use the
  narrowest scope that satisfies the requirement; use custom roles to remove
  permissions that built-in roles include but your scenario does not require.

- **Azure Policy** enforces governance standards at the control plane before
  resources are created. The `deny` effect prevents non-compliant deployments;
  the `modify` effect with a remediation task corrects existing resources.

- **Resource locks** protect critical resources from accidental deletion or
  modification regardless of RBAC permissions — they add deliberate friction.

- **ARM templates** are the native Azure IaC format: declarative JSON files
  describing the desired state of your infrastructure. All deployments — portal,
  CLI, PowerShell, pipelines — go through Azure Resource Manager.

- **Bicep** is the recommended authoring language for ARM templates. It compiles
  to ARM JSON and offers cleaner syntax, type safety, and better tooling support.

- **Idempotency** means the same deployment can be safely re-run at any time and
  will converge to the same result without duplicating resources.

- **What-if** (`--what-if`) is a mandatory step before production deployments —
  it shows exactly what will change before anything is applied.

- **Separate parameter files** (`parameters.dev.json`, `parameters.prod.json`)
  allow a single template to be re-deployed across environments. Committed to
  source control, they provide a full audit trail and enable CI/CD automation.

---

## Resources

- [Azure Governance documentation](https://learn.microsoft.com/en-us/azure/governance/)
- [Management groups overview](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview)
- [Azure RBAC built-in roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles)
- [Azure Policy built-in definitions](https://learn.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies)
- [ARM template reference](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/)
- [Bicep documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [ARM template what-if](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/deploy-what-if)
- [Key Vault references in parameter files](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/key-vault-parameter)
- Lab file reference: `azuredeploy-storage.json` (provided in session materials)

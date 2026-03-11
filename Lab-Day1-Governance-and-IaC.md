---
lab:
  title: 'Lab Day 1: Governance & Infrastructure as Code'
  module: 'Advanced Azure Bootcamp – Day 1'
---

# Lab Day 1 – Governance & Infrastructure as Code

## Lab Introduction

In this lab you build a governance foundation in Azure and then deploy infrastructure
using both ARM templates and Bicep. You will work through the following tasks:
- Configure RBAC with least-privilege role assignment
- Enforce standards with Azure Policy, tags, and resource locks
- Export, understand, and author ARM templates
- Deploy ARM and Bicep templates via Cloud Shell
- Apply deployment best practices — what-if, parameter files, and history

## Pre-requisites
- an Azure subscription 
- a resource group named RG-Lab1 
- Owner rights at the resource group level. 

## Estimated Timing: 90 minutes

## Lab Scenario

Your organisation is formalising its Azure landing zone ahead of a production workload migration. Before any infrastructure can be deployed, governance guardrails must be in place. You have been tasked with:

- Providing users with a Virtual Machine Contributor role to allow self-service provisioning of compute resources
- Enforcing mandatory `Environment`, `CostCenter`, and `Department` tags using Azure Policy
- Protecting shared resources against accidental deletion using resource locks
- Exporting and examining an ARM template to understand its structure and sections
- Authoring an ARM template from scratch using all five sections: parameters, variables, functions, resources, and outputs
- Deploying a baseline storage account using an ARM template and a Bicep file via
  Cloud Shell, using separate parameter files for dev and prod
- Running a what-if check to validate changes before applying them

## Architecture Overview

```
Tenant Root Group
└── Root Management Group
    └── [Your Subscription]
        ├── RG-Lab1 (Resource Group)
        │   ├── Azure Policy Assignments (3 tag policies)
        │   ├── Resource Lock (delete lock)
        │   └── Storage Account (deployed via ARM + Bicep)
```

## Job Skills

- Task 1: Configure RBAC with least-privilege role assignment
- Task 2: Enforce governance with Azure Policy and resource tags
- Task 3: Protect resources with a resource lock
- Task 4: Export and understand an ARM template
- Task 5: Author an ARM template with all five sections
- Task 6: Deploy ARM and Bicep templates via Cloud Shell
- Task 7: Apply deployment best practices — parameters, what-if, and history

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

Azure Policy lets you enforce standards automatically. In this task you assign a built-in policy that requires tag on all resources, then test that it blocks non-compliant deployments.

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

7. Select Create and Wait for the deployment to succeed.

8. Go to the Resource Group, expand the Settings blade and select **Policies**. You can see the compliance state of the Resource Group.

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

## Task 4: Export and Understand an ARM Template

Before writing templates, it helps to read one. In this task you export the ARM
template for the storage account already deployed, examine its structure, and
edit it to understand parameterisation.

### Export the template

1. Navigate to the storage account created in Task 3.

2. In the left blade, under **Automation**, select **Export template**.

3. Take 2 minutes to examine the two files:

   **Template file** (`template.json`) — notice these sections:
   - `$schema` — declares the deployment schema and API version
   - `contentVersion` — version string for your own change tracking
   - `parameters` — values injected at deployment time (no hardcoding)
   - `variables` — computed values derived from parameters
   - `resources` — the actual Azure resources to create or update


   **Parameters file** (`parameters.json`) — a separate file that supplies values
   for each parameter, allowing the same template to be re-used across environments
   by swapping the parameter file.

4. Select **Download** for the Template and Parameters files.

**Note:** The exported template will typically include `parameters`, `variables`, and `resources` sections but will not contain `functions` or `outputs` — those are less common in portal-exported templates. Task 5 demonstrates all five sections together in a template authored from scratch.

**Why separate parameters?** A template describes *what* to deploy. A parameters file describes *how* to deploy it for a specific environment. Storing `parameters.dev.json` and `parameters.prod.json` in source control alongside your template means environment differences are explicit, auditable, and reproducible.

### Edit the template for reuse

1. Open `template.json` in a text editor.

2. In the `parameters` section, find the storage account name parameter.
   Change its default value to `stlabdeployyourname`.

3. Save the file.

---

## Task 5: Deploy ARM and Bicep Templates via Cloud Shell

In this task you use Azure Cloud Shell to deploy the storage account using both the ARM template and a Bicep equivalent. You will also verify idempotency by re-running a deployment with no changes.

### Open Cloud Shell

1. Select the **Cloud Shell** icon in the top-right of the Azure portal
   
2. When prompted, select **Bash**.

3. On the **Getting started** screen, select **No storage account required**, and select **Apply**. It may take a minute for the shell to provision.

4. Verify your CLI version:

   ```bash
   az --version
   ```

5. Confirm you are using the correct subscription:

   ```bash
   az account show --query "{name:name, id:id}" -o table
   ```

   If not, set it:

   ```bash
   az account set --subscription "<your-subscription-id>"
   ```

### Upload your template and parameter files

1. Select the **Manage files** icon in the Cloud Shell toolbar and select **Upload**.

2. Select both `template.json` and `parameters.json` downloaded in previous task and click Open.

3. Confirm they are present:

   ```bash
   ls
   ```

### Deploy using the ARM template

1. Run the deployment command below:

   > **Note:** The parameter name `storageAccounts_stlab***` is generated from your specific storage account name during export. Open your downloaded `parameters.json` and use the parameter name that appears there instead.

```bash
az deployment group create \
  --resource-group RG-Lab1 \
  --template-file template.json \
  --parameters parameters.json \
  --parameters storageAccounts_stlabyourname=stlabdeployyourname
```
Note: The parameters section in the ARM template may have a different parameter name based on the storage account name you used during export. Make sure to match the parameter name in the command with the one in your `parameters.json` file. In addition ensure stlabdeployyourname is replaced with your name to avoid naming conflicts with other students.


2. Wait for the output to show `"provisioningState": "Succeeded"`.

3. List the storage accounts in the resource group to confirm:

```bash
   az storage account list \
     --resource-group RG-Lab1 \
     --query "[].{Name:name, Location:location, Sku:sku.name}" \
     -o table
```

### Verify idempotency

1. Run the exact same deployment command again you used previously.

2. Confirm the output again shows `"provisioningState": "Succeeded"` with no
   errors and no new resource created.

**Idempotency** means running the same deployment twice produces the same end state without duplicating or erroring on existing resources. This is a core IaC principle — your deployments should be safe to re-run at any time.

3. Go the Azure Portal and visit the Resource Group. Confirm that there are now two storage accounts present — the one created in Task 2 via the portal, and the one just deployed via the ARM template.

### Convert the ARM template to Bicep

1. In Cloud Shell, select Settings->Go to Classic version. 

Then run:

```bash
   az bicep decompile --file template.json
```

2. Open the generated `.bicep` file in the Cloud Shell editor:

```bash
   code template.bicep
```

3. Compare the Bicep syntax to the ARM JSON. Notice:
   - No `$schema` or `contentVersion` boilerplate
   - Parameters declared with `param` keyword and types (`string`, `int`, `bool`)
   - Resources use a clean `resource` block with symbolic names

4. Change the storage account name in the Bicep file to `stlabbicepyourname`.
   Use **Ctrl+S** to save, **Ctrl+Q** to close the editor.

Note: you must modify stlabbicepyourname to include your name to avoid naming conflicts with other students.

### Deploy using the Bicep file

1. Run the deployment this time referencing the bicep file.
```bash
az deployment group create \
  --resource-group RG-Lab1 \
  --template-file template.bicep
```

2. Confirm `"provisioningState": "Succeeded"`.

3. List storage accounts again to confirm that 3 storage accounts exist:

```bash
   az storage account list \
     --resource-group RG-Lab1 \
     --query "[].{Name:name}" \
     -o table
```
---
## Task 6: Author an ARM Template with All Five Sections

In this task you author an ARM template from scratch that uses all five sections: parameters, variables, user-defined functions, resources, and outputs.

### Create the template

1. Open a text editor and create a new file called `sample-all-elements.json`.

2. Paste in the following template:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",

    "parameters": {
        "storageAccountName": {
            "type": "string",
            "metadata": {
                "description": "Base name for the storage account."
            }
        },
        "environment": {
            "type": "string",
            "defaultValue": "dev",
            "allowedValues": [ "dev", "test", "prod" ],
            "metadata": {
                "description": "Deployment environment."
            }
        },
        "costCenter": {
            "type": "string",
            "metadata": {
                "description": "Cost center for billing."
            }
        },
        "department": {
            "type": "string",
            "metadata": {
                "description": "Department that owns this resource."
            }
        }
    },

    "variables": {
        "storageSku": "[if(equals(parameters('environment'), 'prod'), 'Standard_GRS', 'Standard_LRS')]",
        "fullStorageName": "[concat(parameters('storageAccountName'), parameters('environment'))]"
    },

    "functions": [
        {
            "namespace": "myOrg",
            "members": {
                "uniqueStorageName": {
                    "parameters": [
                        {
                            "name": "baseName",
                            "type": "string"
                        }
                    ],
                    "output": {
                        "type": "string",
                        "value": "[toLower(concat(parameters('baseName'), uniqueString(resourceGroup().id)))]"
                    }
                }
            }
        }
    ],

    "resources": [
        {
            "type": "Microsoft.Storage/storageAccounts",
            "apiVersion": "2023-01-01",
            "name": "[myOrg.uniqueStorageName(variables('fullStorageName'))]",
            "location": "[resourceGroup().location]",
            "sku": {
                "name": "[variables('storageSku')]"
            },
            "kind": "StorageV2",
            "tags": {
                "Environment": "[parameters('environment')]",
                "CostCenter": "[parameters('costCenter')]",
                "Department": "[parameters('department')]"
            },
            "properties": {
                "accessTier": "Hot"
            }
        }
    ],

    "outputs": {
        "storageAccountName": {
            "type": "string",
            "value": "[myOrg.uniqueStorageName(variables('fullStorageName'))]"
        },
        "storageAccountId": {
            "type": "string",
            "value": "[resourceId('Microsoft.Storage/storageAccounts', myOrg.uniqueStorageName(variables('fullStorageName')))]"
        }
    }
}

```

### Understand what each section does

| Section | Purpose in this template |
| --- | --- |
| **Parameters** | Accept `storageAccountName`, `environment`, `costCenter`, and `department` at deploy time |
| **Variables** | Compute the SKU (GRS for prod, LRS otherwise) and combine name + environment into one string |
| **Functions** | `myOrg.uniqueStorageName()` appends a deterministic hash to guarantee a globally unique storage account name |
| **Resources** | Deploys a `StorageV2` storage account using the computed values |
| **Outputs** | Returns the final storage account name and resource ID after deployment |

**How the sections connect:**
```
Parameter: storageAccountName = "mystore"
Parameter: environment        = "dev"
Parameter: costCenter         = "CC1234"
Parameter: department         = "IT"
                ↓
Variable:  fullStorageName    = "mystoredev"
                ↓
Function:  uniqueStorageName("mystoredev") = "mystoredevabc123xyz7890"
                ↓
Resource:  name = "mystoredevabc123xyz7890"
                ↓
Output:    storageAccountName = "mystoredevabc123xyz7890"
```
3. Upload the file into the Cloud Shell using the **Manage files** pane.

### Run a what-if check

1. Before deploying, preview the changes by running a what-if deployment:

```bash
az deployment group what-if \
  --resource-group RG-Lab1 \
  --template-file sample-all-elements.json \
  --parameters storageAccountName=mystore environment=dev costCenter=CC1234 department=IT
```

2. Review the output and confirm the storage account shows as **Create**.

### Deploy the template

1. Run the command below:

```bash
az deployment group create \
  --resource-group RG-Lab1 \
  --template-file sample-all-elements.json \
  --parameters storageAccountName=mystore environment=dev costCenter=CC1234 department=IT
```

2. Confirm `"provisioningState": "Succeeded"` in the output.

3. Verify the resource group that the new storage account has been created with the expected name, SKU, and tags.

---

## Task 7: Deployment Best Practices – What-If, Parameters, History

### What-if: validate before you apply

The `--what-if` flag shows exactly what a deployment *would* change without making
any changes. Use this before every production deployment.

1. In Cloud Shell, edit `template.bicep`:

   ```bash
   code template.bicep
   ```

2. Change the storage account name to `stlabwhatifyourname`:

   ```bicep
   param storageAccounts_stlabtestuser1_name string = 'stlabwhatifyourname'
   ```

   Use **Ctrl+S** to save, **Ctrl+Q** to close the editor.

   Note: you must modify stlabwhatifyourname to include your name to avoid naming conflicts with other students.

3. Run a what-if check:

```bash
   az deployment group what-if \
     --resource-group RG-Lab1 \
     --template-file template.bicep
```

4. Review the output. Notice the colour-coded change indicators:
   - **Create** (green `+`) — new resource will be added
   - **Modify** (yellow `~`) — existing resource will be changed
   - **Delete** (red `-`) — resource will be removed
   - **No change** (grey `=`) — resource is already in the desired state

5. Only proceed with `az deployment group create` once you are satisfied with
   the what-if output.

```bash
   az deployment group create \
     --resource-group RG-Lab1 \
     --template-file template.bicep
```

### Environment-specific parameter files

In practice, do not rely only on inline `--parameters` overrides. Create separate
parameter files per environment:

1. In a text editor create the following files. 

Note: Make sure to replace the storage account parameter to match the parameter name in your bicep template, and modify the storage account names to include your name to avoid naming conflicts with other students.

   **parameters.dev.json**
   ```json
   {
     "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
     "contentVersion": "1.0.0.0",
     "parameters": {
       "storageAccounts_stlabyourname": {
         "value": "stlabdevyourname"
       }
     }
   }
   ```

   **parameters.prod.json**
   ```json
   {
     "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
     "contentVersion": "1.0.0.0",
     "parameters": {
       "storageAccounts_stlabyourname": {
         "value": "stlabprodyourname"
       }
     }
   }
   ```

2. Upload them in the Cloud Shell using the **Manage files** pane.

3. Deploy using the dev parameter file:

```bash
   az deployment group create \
     --resource-group RG-Lab1 \
     --template-file template.bicep \
     --parameters @parameters.dev.json
```

### Review deployment history

1. In the Azure portal, navigate to **RG-Lab1**.

2. In the **Settings** blade, select **Deployments**.

3. Review the list of all deployments made during this lab — each entry records
   the template, all parameter values, start time, duration, and outcome.

4. Select one deployment and open the **Template** blade. Confirm the exact
   template used is preserved for audit purposes.

**Common pitfalls to avoid:**
- Hardcoded resource names in templates (use parameters instead)
- Missing resource locks on shared/production resources
- Deploying directly from portal without template — creates unmanaged drift
- Not using what-if before production deployments

---

## Cleanup

**Note:** Remove the resource lock before deleting the resource group, otherwise the deletion will be blocked.

1. Go to your Resource Group.

2. In the Settings -> Locks. **Delete the lock**.

3. In the Overview, select **Delete resource group**.

4. Copy and paste the name to confirm deletion.

5. Select **Delete**.

---

## Key Takeaways

- **RBAC least privilege**: assign roles to groups, not individuals; use the
  narrowest scope that satisfies the requirement; use built-in roles as much as possible.

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
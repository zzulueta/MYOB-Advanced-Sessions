# Azure Fundamentals L100 — Session 2: Hands-On Portal Demo Script

**Segment:** 2:18 – 2:28 (10 minutes)
**Purpose:** Bring together the governance and control concepts covered in Session 2 — RBAC, Azure Policy, Resource Tagging, and Cost Management — through live Azure Portal demonstrations.

---

## Trainer Preparation (Before the Session)

- [ ] Log into the Azure Portal at [https://portal.azure.com](https://portal.azure.com) using the demo/sandbox account
- [ ] Ensure the account has **Owner** or **User Access Administrator** role on at least one Resource Group so RBAC assignments can be demonstrated
- [ ] Have a pre-existing Resource Group ready (e.g., `rg-l100-demo`) — this will be used across all four demos
- [ ] Have a second demo user account ready (e.g., `demo-reader@yourdomain.com`) to assign roles to — do **not** use a real user account
- [ ] Verify **Azure Policy** is accessible (search "Policy" in the portal)
- [ ] Verify **Cost Management + Billing** is accessible and shows some spend data (even minimal)
- [ ] Set browser zoom to ~90% so participants can see more of the portal
- [ ] Close unrelated browser tabs

---

## Demo 1: Azure RBAC in Action (3 min)

> "Let's start with access control. RBAC answers the question: who can do what, and where? Let me show you how this works in practice."

### Step 1 — Navigate to Access Control (IAM)

1. In the Azure Portal search bar, type **Resource groups** and click the result.
2. Click into your demo Resource Group (Session2-Demo).
3. In the left-hand navigation blade, click **Access control (IAM)**.
   > "This is where RBAC lives — every resource, resource group, and subscription has this blade. It's the gate that controls who gets in and what they can do."

---

### Step 2 — Review Existing Role Assignments

1. Click the **Role assignments** tab.
2. Scroll through the list of current assignments.
   > "Each row here is a role assignment: a security principal — a person, group, or service identity — paired with a role and a scope. Notice the 'Scope' column — some roles are inherited from the subscription level above."
3. Point to an **inherited** assignment if visible.
   > "This is role inheritance in action. A role assigned at the subscription level flows down to every resource group and resource below it. This is why we use the least-privilege principle — assign roles as low in the hierarchy as needed."

---

### Step 3 — Assign a Role

1. Click **+ Add** → **Add role assignment**.
2. On the **Role** tab, search for and select **Storage Blob Data Reader**.
   > "allows for read access to blob data in storage accounts."
3. Click **Next** to go to the **Members** tab.
4. Click **+ Select members** and search for your demo user account.
5. Select it and click **Select**.
   > "We're assigning the Storage Blob DataReader role to this user on this specific resource group. They'll be able to read blob data inside it, but not change or delete anything."
6. Click **Review + assign** — review the summary, then click **Review + assign** again.
   > "Role assignments take effect within a few minutes. Azure RBAC is additive — a user gets the union of all roles assigned to them across all scopes."

---

### Step 4 — Check Effective Permissions

1. Go back to the **Access control (IAM)** blade.
2. Click the **Check access** tab.
3. Search for the demo user you just assigned.
4. Click on their name to view their **effective permissions** on this resource group.
   > "This is a great troubleshooting tool. If someone says 'I can't do X', you come here and check what they actually have. It shows you every role they hold and what actions those roles allow."

### Step 5 — Give Test User Storage Blob Data Contributor Role at the Storage Account Level.
1. Navigate to a storage account within the same resource group.
2. Click on the storage account and go to **Access control (IAM)**.
3. Click **+ Add** → **Add role assignment**.
4. On the **Role** tab, search for and select **Storage Blob Data Contributor**.
5. Click **Next** to go to the **Members** tab.
6. Click **+ Select members** and search for your demo user account.
7. Select it and click **Select**.
8. Click **Review + assign** — review the summary, then click **Review + assign** again.
   > "Now this user has Storage Blob Data Contributor at the storage account level, which allows them to read and write blob data in that storage account. This demonstrates how permissions can be layered and how scope affects access."

---

## Demo 2: Azure Policy Configuration (3 min)

> "RBAC controls what people can do. Azure Policy controls what resources must comply with. Let me show you how to apply guardrails."

### Step 1 — Open Azure Policy

1. In the top search bar, type **Policy** and click the result.
2. The **Overview** dashboard loads.
   > "This is the Policy dashboard. The compliance score shows you how many resources in scope are meeting your policies right now. A score below 100% means something is out of compliance — either something you need to fix, or something you're auditing intentionally."

---

### Step 2 — Browse Built-in Policy Definitions

1. In the left nav, click **Definitions**.
2. In the search bar, type **Allowed locations**.
3. Click on the **Allowed locations** policy definition to open it.
   > "This is a built-in policy. Microsoft ships hundreds of these. This one restricts which Azure regions resources can be deployed into — useful for data sovereignty, for example, enforcing that MYOB workloads only land in Australian regions."
4. Point out the **Effect** field in the policy rule JSON.
   > "The effect here is 'Deny' — meaning if someone tries to deploy a resource in a disallowed region, Azure blocks it before it's created. Other effects like 'Audit' just log the violation without blocking."

---

### Step 3 — Assign a Policy

1. Click **Assign policy** which opens the assignment form.
2. Walk through the assignment form:
- Scope: Select your Subscription.
- Exclusions: Select some Resource Group.
3. Click **Next** to reach **Parameters**.
4. In the **Allowed locations** drop-down, select **Australia East**.
   > "This is the guardrail — only Australia East is allowed. Any resource creation attempt in another region will be denied."
5. Click Next to reach **Remediation**.
How Remediation works:
    You create a policy.
    Azure evaluates all resources (old and new).
    Old resources that don’t match the policy become non-compliant.
    Azure does not automatically fix them.
    If the policy uses deployIfNotExists or modify, you can run a remediation task to bring old resources into compliance.
    If not, the policy only reports the issue.
6. Go to **Non-compliance messages** and enter: "Only Australia East is allowed for this subscription"
7. Click **Review + create** then click Create.
8. Try to create a Resource Gruop in a disallowed region (e.g., Australia West) to show the policy in action.
   > "When I try to create a resource group in Australia West, I get an error — the policy is doing its job and preventing non-compliant resources from being created."
9. Go to Authoring->Assignments and click on the Assignment you just created. Click on the **View Compliance** tab to show the compliance state of existing resources.
   > "Here we can see the compliance state of all resources in scope. Any resource that isn't in Australia East will show up as non-compliant, along with the message we defined."
10. Click Delete Assignment to clean up the policy assignment. 

---

## Demo 3: Resource Tagging (2 min)

> "Tags are how we attach metadata to resources — for cost allocation, ownership tracking, and automation. Let me show you tagging in action."

### Step 1 — Add Tags to a Resource Group

1. Navigate back to your demo Resource Group (`Session2-Demo`).
2. In the left nav, click **Tags**.
3. Add the following tags one by one:

   | Name (Key) | Value |
   |------------|-------|
   | `Environment` | `Dev` |
   | `CostCenter` | `CC-1234` |
   | `Owner` | `platform-team` |
   | `BusinessUnit` | `Engineering` |

4. Click **Apply**.
   > "Tags are just key-value pairs. We've added four here. Notice they're on the resource group itself — but we can also tag individual resources inside it. The important thing is consistency: a tagging standard only works if everyone uses the same keys."

---

### Step 2 — Show Tags in Cost Analysis

1. Go to the Subscription.
2. In the left nav, click **Cost Management + Billing** → **Cost analysis**.
3. In the Cost analysis view, click Add filter → Tag → select `CostCenter` → select `CC-1234`.
   > "Now we're filtering the cost view to only show resources tagged with CostCenter = CC-1234. This is how you can break down costs by project, team, or any other dimension you choose to tag with. Tags are critical for cost visibility and chargeback/showback models."
---

## Demo 4: Cost Management — Budgets & Alerts (2 min)

> "Knowing your spend is useful. Getting alerted before you overspend is better. Let me show you how budgets work."

### Step 1 — Create a Budget

1. In **Cost Management**, click **Budgets** in the left nav.
2. Click **+ Add**.
3. Walk through the budget creation form:

   | Field | Value | Talking Point |
   |-------|-------|---------------|
   | **Scope** | Subscription | "Budgets can be scoped to a subscription or resource group." |
   | **Name** | `Demo-Monthly-Budget` | "Name it clearly so your team knows what it's for." |
   | **Reset period** | Monthly | "Budget resets each month — aligns with billing cycles." |
   | **Amount** | `$500` | "Set this to whatever monthly spend threshold matters to you." |

4. Click **Next** to the **Alert conditions** step.
5. Show the threshold settings:
   - 50% → Email notification
   - 80% → Email notification
   - 100% → Email notification
   > "These thresholds are percentage-based. At 50% of your budget, you get an early warning. At 80%, time to investigate. At 100%, you've hit the limit — the budget doesn't stop resources, but your team is notified to take action."
6. Add an email address to the **Alert recipients** field.
   > "Alerts go to whoever you nominate here — the platform team, finance, the workload owner. You can add multiple recipients."
7. Click **Review + create** — review, then stop short of creating or proceed as appropriate.
   > "In production, you'd set this up for every subscription so you always have visibility before bills arrive."

---

### Demo 5 — Show Advisor Cost Recommendations (Optional — if time allows)

1. In the top search bar, type **Advisor** and click the result.
2. Click the **Cost** tab.
   > "Azure Advisor scans your environment continuously and surfaces recommendations. The Cost tab shows you specific actions — like right-sizing an over-provisioned VM or buying a reservation — each with an estimated monthly saving. This is an easy win for cost optimisation."

---

## Wrap-Up Talking Points

> "Let's tie all four demos together:
> - **RBAC** controls who has access and at what scope — least privilege is the default mindset
> - **Azure Policy** enforces standards before problems occur — Deny blocks, Audit logs, DeployIfNotExists remediates
> - **Tags** make cost allocation and resource ownership visible — they're only useful if enforced consistently
> - **Cost Management** gives you the financial lens — budgets and alerts keep you informed before spend gets out of hand
> 
> None of these work in isolation. RBAC prevents unauthorised changes. Policy prevents non-compliant ones. Tags make costs attributable. Budgets keep stakeholders informed. Together, they're your governance foundation."

---

### Suggested Questions to Prompt Audience

- "What's the difference between RBAC and Azure Policy — when would you use each?"
  *(Answer: RBAC = who can act; Policy = what the result must comply with. Use both together.)*
- "If you wanted to enforce that all Azure resources must have a CostCenter tag, what policy effect would you use?"
  *(Answer: Deny prevents creation without the tag; Modify can add it automatically; Append can append it.)*
- "If a user has Contributor at the subscription level and Reader at the resource group level, what's their effective access to the resource group?"
  *(Answer: Contributor — RBAC is additive; the higher permission wins.)*
- "How would you set up cost visibility for each business unit in Azure?"
  *(Answer: Tag resources with BusinessUnit, then filter Cost Analysis by that tag.)*

---

## Trainer Notes

- **If RBAC role assignment is blocked:** The demo account may not have Owner or User Access Administrator permissions at the needed scope. Have a screenshot ready as backup. Focus on showing the read-only view of existing assignments.
- **If Policy assignment requires approval or is locked:** Walk through the forms without submitting — participants can follow the logic without a live policy being applied to the demo tenant.
- **If Cost Analysis shows no data (new/empty subscription):** Explain that data populates once resources start generating cost. Show the filtering and grouping UI anyway and describe what a populated view looks like.
- **If participants ask about Policy vs Defender for Cloud:** Note that Defender for Cloud builds on top of Policy — it uses policy to enforce security standards and surfaces findings. It's a Layer 2 governance topic beyond today's L100 scope.
- **If participants ask about automating tag enforcement via Bicep:** Acknowledge it — this is exactly what IaC covers in L200/L300. For now, Policy's Modify/Append effects provide the enforcement layer.
- **If Advisor shows no recommendations:** A new or lightly used demo environment won't have data. Briefly explain what Advisor does and move on — don't spend time on an empty screen.
- **Keep it moving:** Prioritise Demos 1 and 2 (RBAC and Policy) as they cover the most conceptually dense material. Demos 3 and 4 can move faster — participants can explore tagging and cost management independently.

---

*End of Demo Script — Session 2*

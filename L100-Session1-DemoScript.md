# Azure Fundamentals L100 — Session 1: Hands-On Portal Demo Script

**Segment:** 2:15 – 2:20 (5 minutes) — *extend to 7–8 min if time allows to include Copilot for Azure*
**Purpose:** Bring together everything covered in Session 1 by navigating the live Azure Portal, and briefly showcase Copilot for Azure as a practical productivity tool.

---

## Trainer Preparation (Before the Session)

- [ ] Log into the Azure Portal at [https://portal.azure.com](https://portal.azure.com) using the demo/sandbox account
- [ ] Ensure the account has at least one Management Group and 2–3 Subscriptions visible
- [ ] Have a pre-existing Resource Group ready (e.g., `rg-l100-demo`) to reference if creation is skipped
- [ ] Set browser zoom to ~90% so participants can see more of the portal
- [ ] Close unrelated browser tabs
- [ ] Verify **Copilot for Azure** is enabled on the demo tenant (portal top-right sparkle icon is visible); if not, prepare a screenshot/recording as backup

---

## Part 1: Navigate the Hierarchy (2 min)

### Step 1 — Show the Entra ID Tenant

1. In the Azure Portal top-right corner, click the **account icon** (your name/email).
2. Point to the **Directory** shown — this is the **Entra ID Tenant**.
   > "This is the top of the hierarchy — our identity and directory boundary. Everything in Azure lives under this tenant."
3. If your account has access to multiple directories, briefly show the **Switch directory** option.
   > "An organisation typically has one tenant. Some enterprises have more than one, but that adds management complexity."

---

### Step 2 — Show Management Groups

1. In the search bar at the top, type **Management groups** and click the result.
2. Show the management group tree.
   > "Management Groups sit above subscriptions. Think of them as folders that let you apply policies and access controls across multiple subscriptions at once."
3. Point to a group and explain its purpose (e.g., by business unit or environment type).
   > "A common pattern is to organise these by business unit — for example, Platform, Workloads, and Sandbox — then apply different governance policies at each level."

---

### Step 3 — Show Subscriptions

1. In the search bar, type **Subscriptions** and click the result.
2. Show the list of subscriptions.
   > "Each subscription is a billing boundary and an access control boundary. Notice we have separate subscriptions here — that's the pattern we recommend: Production, Non-Production, Sandbox, each isolated from each other."
3. Click into one subscription to briefly show the **Overview** blade.
   > "You can see the subscription ID, which management group it belongs to, and current spend. This is the level where cost tracking and quotas are managed independently."

---

### Step 4 — Show Resource Groups

1. In the left nav or search bar, navigate to **Resource groups**.
2. Show the list of resource groups inside the selected subscription.
   > "Resource Groups are logical containers. Everything in Azure belongs to exactly one resource group — the VM, the database, the network — all grouped by their lifecycle."
3. Click into one resource group to show its contents.
   > "Notice the resources here share a common lifecycle — if we delete this resource group, all these resources go with it. That's intentional design."
4. Click the **Tags** blade on the left.
   > "Tags here apply to the resource group itself, but they're also a way to track cost allocation. We'll talk more about tagging governance in Session 2."

---

## Part 2: Create a Resource (2 min)

### Step 5 — Create a Resource Group

1. Click **+ Create** (or navigate to Resource Groups → + Create).
2. Walk through the creation form, pausing on each field:

   | Field | Value to Enter | Talking Point |
   |-------|----------------|---------------|
   | **Subscription** | Select demo subscription | "This is where it fits in the hierarchy." |
   | **Resource group name** | `rg-l100-live-demo` | "Best practice: use a consistent naming convention — rg for resource group, then a short descriptor." |
   | **Region** | e.g., `Australia East` | "Azure deploys metadata about this resource group to this region. Choose a region close to your workloads or aligned to your data residency requirements." |

3. Click the **Tags** tab.
   > "Tags are key-value pairs. Here we could tag by environment, cost centre, or project. We'll go deeper on governance in Session 2."

4. Click **Review + create**, then **Create**.
   > "That's it — the resource group is created in seconds. Now we have a container ready to hold our resources."

---

### Step 6 — Briefly Show Creating a Storage Account (Optional — skip if short on time)

1. Navigate to **Storage accounts** → **+ Create**.
2. Point out the key fields without completing creation:

   | Field | Talking Point |
   |-------|---------------|
   | **Subscription & Resource group** | "Notice it immediately asks where in the hierarchy this resource lives." |
   | **Region** | "This is where the actual data will be stored — relevant for latency and data residency compliance." |
   | **Redundancy** | "Here's our LRS/ZRS/GRS choice from the storage section — this is where it shows up in practice." |

3. Do **not** click Create — navigate away.
   > "We won't create this one today, but you can see how every concept we covered — hierarchy, regions, redundancy — surfaces right here in the creation form."

---

## Part 3: Quick Observation & Q&A (1 min)

### Wrap-Up Talking Points

> "The portal isn't just a UI — it's a reflection of everything we've covered today:
> - The **hierarchy**: tenant → management groups → subscriptions → resource groups → resources
> - The **region**: every resource is placed in a geographic location
> - **Tags and naming**: governance starts at resource creation
> - **Service choice**: PaaS, IaaS, serverless — they all live side by side here"

### Suggested Questions to Prompt Audience

- "What's the difference between a subscription boundary and a resource group boundary?"
  *(Answer: Subscription = billing + access control at scale; Resource Group = lifecycle management for related resources)*
- "If you were setting up Azure for MYOB, how many subscriptions would you start with?"
  *(Answer: Encourage thinking about Production, Non-Production, Sandbox as a minimum)*
- "Where would you apply a policy to enforce that all resources must be tagged?"
  *(Answer: Management Group or Subscription level, so it inherits downward)*

---

## Part 4: Copilot for Azure (2–3 min, if time allows)

> "Before we finish, let me show you one more thing. Azure has a built-in AI assistant — Copilot for Azure — available right in the portal. It can explain resources, help troubleshoot, and guide you when you're not sure what to do next."

### Step 7 — Open Copilot for Azure

1. In the Azure Portal top-right toolbar, click the **Copilot** icon (sparkle/star icon).
   - If not visible, search **Microsoft Copilot in Azure** in the portal search bar.
   > "This is Microsoft Copilot for Azure — it's context-aware, meaning it can see what you're currently looking at in the portal."

---

### Step 8 — Ask a Hierarchy Question

1. Type (or paste) the following prompt into the Copilot chat:
   ```
   What's the difference between a Management Group and a Subscription in Azure?
   ```
2. Show the response to participants.
   > "Notice it gives a clear, concise answer — exactly the kind of thing you might look up in documentation. This is useful when you're onboarding to Azure and need quick explanations without leaving the portal."

---

### Step 9 — Ask a Contextual Resource Question

1. Navigate to the resource group you created earlier (`rg-l100-live-demo`).
2. Open Copilot and ask:
   ```
   What resources should I put in this resource group for a typical 3-tier web application?
   ```
3. Show the response.
   > "Because Copilot is context-aware, it can tailor suggestions to what you have selected. This is handy for exploring services or getting started with a new architecture pattern."

---

### Step 10 — Ask a Region/Service Question (Optional)

1. Ask Copilot:
   ```
   What Azure regions support Availability Zones in Australia?
   ```
   > "You can use Copilot to quickly look up region capabilities, service availability, or best practices — things that would otherwise require documentation searches."

---

### Copilot for Azure — Key Points to Emphasise

- It is **grounded in Azure documentation and your own Azure environment** — not a generic chatbot
- Useful for **exploration and learning**, not just experienced engineers
- Can help with: explaining services, writing Bicep/CLI snippets, diagnosing issues, understanding costs
- Available in the portal, Azure CLI, and via Microsoft 365 Copilot integrations
- **Not a replacement for proper governance** — it assists decisions, it doesn't make them

> "For a team moving from AWS to Azure, Copilot is genuinely useful for bridging that knowledge gap day-to-day — asking it 'what's the Azure equivalent of an S3 bucket lifecycle policy?' is a lot faster than searching docs."

---

## Trainer Notes

- **If the portal is slow:** Have screenshots ready as a backup — navigate from the pre-prepared slide deck.
- **If participants ask about costs:** Redirect to Session 2 where Cost Management is covered in detail.
- **If someone asks about Bicep/ARM:** Acknowledge it — note it's flagged as an L200/L300 topic in the reference table and beyond today's scope.
- **If Copilot for Azure is not enabled on the demo tenant:** Show a screenshot or recording instead — it requires the Copilot for Azure preview/feature to be enabled at the tenant level.
- **If participants want to explore Copilot further:** Note that GitHub Copilot and Copilot for Azure are separate products; Copilot for Azure is specifically for portal and infrastructure tasks.
- **Keep it moving:** The goal is to make concepts tangible, not to do a full portal walkthrough. Prioritise Parts 1–3; Part 4 is a bonus if time allows.

---

*End of Demo Script — Session 1*

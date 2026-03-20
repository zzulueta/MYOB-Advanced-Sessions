# MYOB Azure Bootcamp — Knowledge Check Quiz
## Days 1–3: Governance & IaC · Networking · Compute

> **30 Multiple Choice Questions**  
> Source: Microsoft Learn + lab exercises  
> Instructions: Select the single best answer for each question.

---

## Part 1 — Day 1: Governance and Infrastructure as Code
*Topics: Azure RBAC, Azure Policy, Resource Locks, ARM Templates, Bicep, What-If Deployments*

---

**Question 1**

Which of the following correctly lists the Azure RBAC scope hierarchy from broadest to narrowest?

A) Subscription → Management Group → Resource Group → Resource  
B) Management Group → Subscription → Resource Group → Resource  
C) Resource Group → Subscription → Management Group → Resource  
D) Management Group → Resource Group → Subscription → Resource  

> ✅ **Correct Answer: B**  
> *Azure RBAC defines four levels of scope in parent-child order: Management Group → Subscription → Resource Group → Resource. Permissions assigned at a broader (parent) scope are inherited by all child scopes beneath it.*  
> *Source: [Understand scope for Azure RBAC](https://learn.microsoft.com/azure/role-based-access-control/scope-overview)*

---

**Question 2**

A developer is assigned the **Reader** role at the **Management Group** scope. What access does this grant?

A) Read access to resources only within the management group itself, not child subscriptions  
B) Read access to all resources in all subscriptions within that management group  
C) Read and write access to all resources in all subscriptions within that management group  
D) No access — Reader cannot be assigned at Management Group scope  

> ✅ **Correct Answer: B**  
> *Role assignments at a parent scope are inherited by child scopes. Assigning Reader at a Management Group scope grants read access to everything in all subscriptions within that management group.*  
> *Source: [Steps to assign an Azure role](https://learn.microsoft.com/azure/role-based-access-control/role-assignments-steps)*

---

**Question 3**

Which Azure built-in role grants a user the ability to **create and manage virtual machines** without granting access to the virtual network or storage account they connect to?

A) Owner  
B) Contributor  
C) Virtual Machine Contributor  
D) User Access Administrator  

> ✅ **Correct Answer: C**  
> *The Virtual Machine Contributor role allows a user to create and manage virtual machines but does not grant access to related resources such as virtual networks or storage accounts.*  
> *Source: [Azure built-in roles — Virtual Machine Contributor](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles)*

---

**Question 4**

The **Contributor** built-in role differs from **Owner** in which key way?

A) Contributor cannot create resources; Owner can  
B) Owner cannot delete resources; Contributor can  
C) Contributor cannot assign roles in Azure RBAC; Owner can  
D) Contributor has read-only access; Owner has full access  

> ✅ **Correct Answer: C**  
> *The Contributor role grants full access to manage all resources but does not allow assigning roles in Azure RBAC. Owner includes that additional ability.*  
> *Source: [Azure built-in roles](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles)*

---

**Question 5**

When Azure Policy evaluates a resource creation request, in what order are the effects processed?

A) Deny → Audit → Modify → Append → Disabled  
B) Disabled → Append/Modify → Deny → Audit → AuditIfNotExists  
C) Audit → Deny → Append → Disabled → AuditIfNotExists  
D) AuditIfNotExists → Deny → Modify → Audit → Disabled  

> ✅ **Correct Answer: B**  
> *Azure Policy evaluates effects in this order: Disabled is checked first; Append and Modify are next (they may alter the request); Deny is evaluated before Audit to prevent double-logging; Audit follows; then AuditIfNotExists.*  
> *Source: [Azure Policy — Order of evaluation](https://learn.microsoft.com/azure/governance/policy/concepts/effect-basics#order-of-evaluation)*

---

**Question 6**

A team deploys an Azure Policy with the **deny** effect targeting resource groups that are missing a required `CostCenter` tag. What happens when a user tries to create a resource group without this tag?

A) The resource group is created but flagged as non-compliant  
B) The request is blocked and the user receives an HTTP 403 Forbidden error  
C) The tag is automatically added with a default value  
D) An email alert is sent to the subscription owner  

> ✅ **Correct Answer: B**  
> *The deny effect prevents the request before it reaches the Resource Provider, returning an HTTP 403 (Forbidden) response. The resource is not created.*  
> *Source: [Azure Policy definitions deny effect](https://learn.microsoft.com/azure/governance/policy/concepts/effect-deny)*

---

**Question 7**

Which Azure Policy effect would you use to **automatically add a missing tag** to a resource by inheriting it from the parent resource group — without blocking the resource creation?

A) Deny  
B) Audit  
C) Modify  
D) DeployIfNotExists  

> ✅ **Correct Answer: C**  
> *The Modify effect can add, replace, or remove tags on resources during creation or update, without blocking the operation. It can be paired with remediation tasks to fix existing non-compliant resources.*  
> *Source: [Azure Policy pattern: the value operator](https://learn.microsoft.com/azure/governance/policy/samples/pattern-value-operator)*

---

**Question 8**

A resource **delete lock** is applied to a storage account. Which of the following actions is still permitted?

A) Deleting the storage account  
B) Deleting the resource group that contains the storage account  
C) Reading data from the storage account  
D) Renaming the storage account  

> ✅ **Correct Answer: C**  
> *A Delete lock (CanNotDelete) prevents deletion but allows read and write operations. Users can still read and modify the resource. A Read-only lock would prevent both writes and deletes.*  
> *Source: [Lock resources to prevent unexpected changes](https://learn.microsoft.com/azure/azure-resource-manager/management/lock-resources)*

---

**Question 9**

An ARM template is deployed twice with the **identical parameter values**. What is the expected result?

A) The second deployment fails because the resources already exist  
B) Azure creates duplicate resources alongside the existing ones  
C) The same desired state is enforced; no unwanted changes occur  
D) The template must include a version stamp for repeat deployments to succeed  

> ✅ **Correct Answer: C**  
> *ARM templates are idempotent — deploying the same template with the same parameters repeatedly produces the same result. If the resource already exists and matches the declared state, no changes are made.*  
> *Source: [ARM template overview](https://learn.microsoft.com/azure/azure-resource-manager/templates/overview)*

---

**Question 10**

Which Azure CLI command allows you to **preview the changes** an ARM or Bicep deployment will make — without actually applying them?

A) `az deployment group validate`  
B) `az deployment group what-if`  
C) `az deployment group confirm`  
D) `az deployment group dry-run`  

> ✅ **Correct Answer: B**  
> *The `az deployment group what-if` command shows a preview of what will be created, modified, or deleted by a deployment without executing it. This helps catch unintended changes before committing.*  
> *Source: [ARM template deployment what-if operation](https://learn.microsoft.com/azure/azure-resource-manager/templates/deploy-what-if)*

---

## Part 2 — Day 2: Networking
*Topics: Virtual Networks, NSGs, DNS Zones, VNet Peering, Azure Load Balancer, Application Gateway*

---

**Question 11**

When two NSG rules have conflicting settings, how does Azure decide which rule to apply?

A) The rule with the highest priority number wins  
B) The rule with the lowest priority number wins  
C) All matching rules are applied simultaneously  
D) The most recently created rule wins  

> ✅ **Correct Answer: B**  
> *NSG rules are processed in priority order, and the rule with the lowest priority number is evaluated first (i.e., has the highest precedence). Priority values must be unique within each direction (inbound/outbound).*  
> *Source: [Network security groups overview](https://learn.microsoft.com/azure/virtual-network/network-security-groups-overview)*

---

**Question 12**

A network administrator tries to delete the default "DenyAllInbound" rule in an NSG. What happens?

A) The rule is deleted immediately  
B) A warning is shown, but deletion is allowed  
C) The deletion fails — default rules cannot be deleted  
D) The rule is replaced with an "AllowAllInbound" rule automatically  

> ✅ **Correct Answer: C**  
> *All NSGs contain a set of default rules that cannot be deleted. Because these default rules are assigned the lowest priority (highest number), custom rules you create always take precedence over them.*  
> *Source: [Virtual networks and virtual machines in Azure](https://learn.microsoft.com/azure/virtual-network/network-overview#network-security-groups)*

---

**Question 13**

At which two levels can a Network Security Group (NSG) be associated?

A) Resource group and subscription  
B) Subnet and network interface card (NIC)  
C) Virtual network and virtual machine  
D) Load balancer and application gateway  

> ✅ **Correct Answer: B**  
> *An NSG can be associated with a subnet (applying rules to all VMs in that subnet) or directly with an individual NIC on a VM. Both levels can be used simultaneously, and both sets of rules are evaluated.*  
> *Source: [Azure for network engineers](https://learn.microsoft.com/azure/networking/azure-for-network-engineers)*

---

**Question 14**

Your organization wants DNS names registered in Azure to resolve **only within your virtual network** — not from the internet. Which Azure DNS feature should you use?

A) A public DNS zone with private access restrictions  
B) Azure Traffic Manager  
C) A private DNS zone linked to the virtual network  
D) Azure DNS Resolver with internet filtering  

> ✅ **Correct Answer: C**  
> *Azure Private DNS zones provide name resolution within one or more virtual networks. You link the private zone to a VNet, and optionally enable auto-registration to automatically create A records for VMs.*  
> *Source: [Azure Private DNS documentation](https://learn.microsoft.com/azure/dns/private-dns-overview)*

---

**Question 15**

A company purchases the domain `contoso.com` and wants to host its DNS records in Azure. Which resource should they create?

A) An Azure Private DNS zone for `contoso.com`  
B) An Azure Public DNS zone for `contoso.com`  
C) An Azure Traffic Manager profile for `contoso.com`  
D) An Azure Front Door routing rule for `contoso.com`  

> ✅ **Correct Answer: B**  
> *Azure Public DNS zones host DNS records for domains that are publicly resolvable on the internet. After creating a public DNS zone in Azure, you delegate the domain to Azure name servers through your domain registrar.*  
> *Source: [Azure DNS overview](https://learn.microsoft.com/azure/dns/dns-overview)*

---

**Question 16**

VNet A is peered with VNet B, and VNet B is peered with VNet C. Can a VM in VNet A communicate with a VM in VNet C through this configuration?

A) Yes — VNet peering is transitive by default  
B) Yes — peering automatically chains across connected VNets  
C) No — VNet peering is non-transitive; A direct peering between A and C is required  
D) No — VNet peering only works within the same Azure region  

> ✅ **Correct Answer: C**  
> *Azure VNet peering is non-transitive. Even though A↔B and B↔C are peered, traffic cannot flow from A to C through B. A direct A↔C peering must be created for direct connectivity.*  
> *Source: [Virtual network peering](https://learn.microsoft.com/azure/virtual-network/virtual-network-peering-overview)*

---

**Question 17**

Azure Load Balancer operates at which layer of the OSI model?

A) Layer 3 — Network (IP)  
B) Layer 4 — Transport (TCP/UDP)  
C) Layer 7 — Application (HTTP/HTTPS)  
D) Layer 2 — Data Link  

> ✅ **Correct Answer: B**  
> *Azure Load Balancer operates at Layer 4 (Transport layer), distributing TCP and UDP traffic based on IP address and port combinations. It does not inspect HTTP headers or URLs.*  
> *Source: [Azure Load Balancer overview](https://learn.microsoft.com/azure/load-balancer/load-balancer-overview)*

---

**Question 18**

Which Azure networking service should you choose when you need to route traffic to different backend pools **based on the URL path** (e.g., `/images/*` to one pool and `/api/*` to another)?

A) Azure Standard Load Balancer  
B) Azure Traffic Manager  
C) Azure Application Gateway  
D) Azure VPN Gateway  

> ✅ **Correct Answer: C**  
> *Azure Application Gateway is a Layer 7 (application-layer) load balancer that supports URL path-based routing. It can inspect HTTP/HTTPS content and route requests to different backend pools based on URL patterns.*  
> *Source: [Application Gateway request routing rules](https://learn.microsoft.com/azure/application-gateway/configuration-request-routing-rules)*

---

**Question 19**

You deploy a Standard Azure Load Balancer in front of two VMs. Clients cannot connect to the VMs through the load balancer. What is the most likely cause?

A) Standard Load Balancers do not support TCP connections  
B) The backend VMs need public IP addresses of their own  
C) No NSG rule exists to permit inbound traffic on the load balancer port  
D) The health probe interval is set too low  

> ✅ **Correct Answer: C**  
> *Standard Load Balancers (and standard public IP addresses) are **secure by default** — inbound connections are blocked unless you explicitly add NSG rules to allow the traffic on the required ports.*  
> *Source: [Troubleshoot Azure Load Balancer](https://learn.microsoft.com/azure/load-balancer/load-balancer-troubleshoot)*

---

**Question 20**

When deploying an Azure Application Gateway, which special inbound NSG rule is required on the Application Gateway subnet?

A) Allow inbound from the internet on port 443 only  
B) Allow inbound from the **GatewayManager** service tag on ports 65200–65535  
C) Allow inbound from the load balancer on port 80  
D) No special NSG rules are required for Application Gateway subnets  

> ✅ **Correct Answer: B**  
> *The Application Gateway subnet requires an NSG rule allowing inbound traffic from the **GatewayManager** service tag on ports 65200–65535 (v2 SKU). These ports are used by Azure infrastructure for health communication and management.*  
> *Source: [Application Gateway infrastructure configuration — NSGs](https://learn.microsoft.com/azure/application-gateway/configuration-infrastructure#network-security-groups)*

---

## Part 3 — Day 3: Compute
*Topics: Virtual Machines, Azure Functions, App Service, Azure Container Apps*

---

**Question 21**

A VM in Azure is **stopped from inside the guest OS** (e.g., `shutdown /s`) but not deallocated through the Azure portal or CLI. What is the billing impact?

A) No charges — the VM is powered off  
B) Compute charges continue because the VM is still allocated in Azure infrastructure  
C) Storage charges stop but compute charges continue at half rate  
D) All charges stop except for the public IP address  

> ✅ **Correct Answer: B**  
> *When a VM is stopped from within the OS, it moves to a "Stopped" state but remains allocated in Azure infrastructure. Compute charges continue. To stop billing for compute, you must **deallocate** the VM through the Azure portal, CLI, or PowerShell.*  
> *Source: [States and billing status of Azure Virtual Machines](https://learn.microsoft.com/azure/virtual-machines/states-billing)*

---

**Question 22**

Your company runs Windows Server workloads both on-premises and in Azure. Which Azure pricing benefit lets you **apply existing on-premises Windows Server licenses** to Azure VMs — reducing licensing costs?

A) Azure Reserved Instances  
B) Azure Spot VMs  
C) Azure Hybrid Benefit  
D) Dev/Test pricing  

> ✅ **Correct Answer: C**  
> *Azure Hybrid Benefit allows customers with Software Assurance-eligible Windows Server and SQL Server licenses to use those licenses on Azure VMs, significantly reducing the per-hour cost.*  
> *Source: [Azure Hybrid Benefit for Windows Server](https://learn.microsoft.com/azure/virtual-machines/windows/hybrid-use-benefit-licensing)*

---

**Question 23**

A team runs production web servers 24/7 and wants to **maximize cost savings** on compute. They are willing to commit to 3 years. Which pricing option provides the deepest discount?

A) Pay-as-you-go with auto-shutdown  
B) Azure Spot VMs  
C) Azure Reserved VM Instances (1-year)  
D) Azure Reserved VM Instances (3-year)  

> ✅ **Correct Answer: D**  
> *A 3-year Reserved VM Instance commitment provides the maximum discount — up to 72% compared to pay-as-you-go pricing. Reserved Instances are best for predictable, steady-state workloads.*  
> *Source: [Azure Reserved Virtual Machine Instances](https://learn.microsoft.com/azure/virtual-machines/prepay-reserved-vm-instances)*

---

**Question 24**

Which type of Azure VM uses **spare, unused Azure capacity** at a significantly reduced price but can be **evicted with short notice** when Azure needs the capacity back?

A) Reserved VM Instances  
B) Spot VMs  
C) Burstable (B-series) VMs  
D) Dedicated Host VMs  

> ✅ **Correct Answer: B**  
> *Azure Spot VMs leverage spare datacenter capacity and offer deep discounts (up to 90%). They can be evicted by Azure at any time when that capacity is needed. They are suitable for fault-tolerant workloads like batch jobs — not production web apps.*  
> *Source: [Azure Spot Virtual Machines](https://learn.microsoft.com/azure/virtual-machines/spot-vms)*

---

**Question 25**

Your team enables **auto-shutdown** on a development VM, scheduled for 7:00 PM daily. What happens at 7:00 PM?

A) The VM restarts automatically  
B) The VM is deallocated, stopping compute billing  
C) The VM is deleted from the resource group  
D) The VM is hibernated but compute charges continue at a reduced rate  

> ✅ **Correct Answer: B**  
> *Azure VM auto-shutdown deallocates the VM at the scheduled time, which stops compute billing. This is a common cost-saving practice for dev/test environments that are not needed outside business hours.*  
> *Source: [Set up auto-shutdown for Azure VMs](https://learn.microsoft.com/azure/virtual-machines/auto-shutdown-vm)*

---

**Question 26**

Which Azure Functions hosting plan is the **current recommended option** for new serverless function apps and supports scale-to-zero, always-ready instances, and virtual network integration?

A) Consumption plan (legacy)  
B) Premium plan  
C) Flex Consumption plan  
D) Dedicated (App Service) plan  

> ✅ **Correct Answer: C**  
> *The Flex Consumption plan is Microsoft's recommended plan for new serverless function apps. It supports scale-to-zero, always-ready instances (to reduce cold starts), pay-per-execution billing, and virtual network integration — improvements over the legacy Consumption plan.*  
> *Source: [Azure Functions hosting options](https://learn.microsoft.com/azure/azure-functions/functions-scale)*

---

**Question 27**

You need to run Azure Functions that have specific requirements: they must run inside a **custom container image** and support **GPU workloads**. Which hosting option should you choose?

A) Flex Consumption plan  
B) Premium plan  
C) Functions on Azure Container Apps (Dedicated plan with GPU workload profile)  
D) Consumption plan (legacy)  

> ✅ **Correct Answer: C**  
> *Functions on Azure Container Apps supports custom container images (bring-your-own container) and GPU-enabled workload profiles in the Dedicated plan. The Flex Consumption plan does not support custom containers or GPU compute.*  
> *Source: [Azure Functions on Azure Container Apps overview](https://learn.microsoft.com/azure/container-apps/functions-overview)*

---

**Question 28**

A developer wants to use **deployment slots** in Azure App Service to test a new version of an application before swapping it to production. Which App Service pricing tier is the **minimum** required for deployment slots?

A) Free (F1)  
B) Basic (B1)  
C) Standard (S1)  
D) Premium (P1v3)  

> ✅ **Correct Answer: C**  
> *Deployment slots are available from the **Standard (S1)** tier and above. The Free and Basic tiers do not support deployment slots. Each slot runs the same app in an isolated environment, allowing staging and blue/green deployments.*  
> *Source: [Set up staging environments in Azure App Service](https://learn.microsoft.com/azure/app-service/deploy-staging-slots)*

---

**Question 29**

In Azure Container Apps, what is a **revision**?

A) An audit log entry created each time the app is accessed  
B) An immutable snapshot of a container app version, created when container image or configuration changes  
C) A tag applied to a container for version tracking purposes  
D) A billing period boundary set by Azure Container Apps  

> ✅ **Correct Answer: B**  
> *A revision in Azure Container Apps is an immutable snapshot of your app at a point in time. New revisions are created when the container image or app configuration changes. Multiple active revisions can run simultaneously, enabling traffic splitting for canary or blue/green deployments.*  
> *Source: [Revisions in Azure Container Apps](https://learn.microsoft.com/azure/container-apps/revisions)*

---

**Question 30**

Azure Container Apps uses **KEDA** for scaling. What does KEDA stand for, and what is its primary function?

A) Kubernetes External Deployment Automation — it deploys new container images automatically  
B) Kubernetes Event-Driven Autoscaling — it scales container replicas based on event sources such as queue depth or HTTP requests  
C) Kubernetes Elastic Distribution Algorithm — it redistributes traffic across replicas  
D) Kubernetes Encrypted Data Architecture — it manages secrets and certificates for containers  

> ✅ **Correct Answer: B**  
> *KEDA (Kubernetes Event-Driven Autoscaling) is the open-source scaler that powers Container Apps scaling. It can scale replicas — including scaling to zero — based on event sources like Service Bus queue length, HTTP traffic, Kafka topics, and more.*  
> *Source: [Azure Container Apps scaling](https://learn.microsoft.com/azure/container-apps/scale-app)*

---

## Answer Key Summary

| # | Topic | Answer |
|---|-------|--------|
| 1 | RBAC scope hierarchy | B |
| 2 | RBAC scope inheritance | B |
| 3 | Virtual Machine Contributor role | C |
| 4 | Contributor vs Owner | C |
| 5 | Azure Policy evaluation order | B |
| 6 | Policy deny effect | B |
| 7 | Policy modify effect | C |
| 8 | Resource delete lock | C |
| 9 | ARM template idempotency | C |
| 10 | What-if deployment command | B |
| 11 | NSG priority rules | B |
| 12 | NSG default rules | C |
| 13 | NSG association levels | B |
| 14 | Private DNS zone | C |
| 15 | Public DNS zone | B |
| 16 | VNet peering transitivity | C |
| 17 | Load Balancer OSI layer | B |
| 18 | Path-based routing | C |
| 19 | Standard LB NSG requirement | C |
| 20 | App Gateway NSG rule | B |
| 21 | VM stopped vs deallocated billing | B |
| 22 | Azure Hybrid Benefit | C |
| 23 | Reserved Instances 3-year | D |
| 24 | Spot VMs | B |
| 25 | VM auto-shutdown behavior | B |
| 26 | Functions Flex Consumption plan | C |
| 27 | Functions with custom containers + GPU | C |
| 28 | App Service deployment slots tier | C |
| 29 | Container Apps revision | B |
| 30 | KEDA in Container Apps | B |

---

*Generated using Microsoft Learn documentation. Questions are aligned with hands-on labs from MYOB Azure Bootcamp Days 1–3.*

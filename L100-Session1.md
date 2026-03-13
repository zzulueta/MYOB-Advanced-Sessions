# Azure Fundamentals L100 — Session 1
**How Azure is Organized**

---

## High-Level Overview

This 90-minute session provides foundational knowledge of Azure's organizational structure and core services. You'll learn how Azure organizes resources from the global infrastructure level down to individual resources, understand the Azure hierarchy, and map key Azure services to familiar AWS equivalents.

**Duration:** 1.5 hours (90 minutes)

---

## Learning Objectives

By the end of this session, you will be able to:
- Explain Azure's global infrastructure and how it supports scale and resilience
- Navigate the Azure organizational hierarchy from tenant to resources
- Understand Azure's core compute, storage, and networking services
- Identify appropriate Azure services for common cloud workloads
- Apply Azure fundamentals to real-world cloud adoption scenarios

---

## Core Learning Outcomes

By the end of Session 1, participants will understand:

### 1. Azure Global Infrastructure (15 min)
How Azure's physical architecture supports global scale and resilience
- Geographies as regulatory and compliance boundaries
- Regions as physical datacenter locations
- Availability Zones for high availability within regions
- Region Pairs for disaster recovery and business continuity
- Best practices for region selection

### 2. Azure Organizational Hierarchy (25 min)
How Azure structures resources from top to bottom
- Entra ID Tenant as the identity and directory boundary
- Management Groups for organizing subscriptions at scale
- Subscriptions as billing and access control boundaries
- Resource Groups as logical containers for related resources
- Resources as the actual Azure services
- How RBAC and policies inherit downward through the hierarchy
- Enterprise tenant and subscription strategy best practices

### 3. Azure Core Services (25 min)
Understanding Azure's service portfolio
- Compute: App Service, Functions, AKS, VMs
- Storage: Blob Storage, Managed Disks, Azure Files, archive tiers
- Networking: VNet, Load Balancer, Application Gateway, Azure DNS, ExpressRoute
- Database & other services: Azure SQL family, Cosmos DB, Bicep/ARM, Azure Monitor, Entra ID + RBAC
- Understanding Azure's core services and their common use cases

### 4. Hands-On Portal Navigation (5 min)
Practical demonstration of concepts
- Navigate the Azure hierarchy in the portal
- Create a resource and observe region, resource group, and tagging options
- See how hierarchy, regions, and services connect in practice

---

## Detailed Session Agenda

### 1:00 – 1:10 | Welcome & Setting Context (10 min)

**Topics:**
- Introductions and trainer background
- Overview of the 2-session format
- Housekeeping: Q&A approach, pre-reading materials, Ask the Expert session

**Key Message:** This training connects Microsoft Learn concepts directly to real-world cloud transformation scenarios and enterprise best practices.

**Note:** We'll do one hands-on portal demo today where we'll navigate the Azure hierarchy and create a resource together.

---

### 1:10 – 1:25 | Azure Global Infrastructure (15 min)

**Core Concepts:**
- **Geographies** → Regulatory and compliance boundaries
- **Regions** → Physical locations with multiple datacenters
- **Availability Zones** → Isolated datacenters within a region
- **Region Pairs** → Disaster recovery and business continuity

**Key Concepts:**
- Data residency and compliance considerations
- Service availability varies by region
- Latency and proximity to users

**Reference:**
- [Azure geographies and regions](https://learn.microsoft.com/en-us/azure/reliability/availability-zones-overview)

---

### 1:25 – 1:50 | Understanding the Azure Organizational Hierarchy (25 min)

**The Structure (Top to Bottom):**

#### 1. Entra ID Tenant (formerly Azure AD)
- Your identity and directory boundary
- One tenant per organization (typically)
- Houses all users, groups, applications
- Single tenant vs. multi-tenant strategy

#### 2. Management Groups
- Organize subscriptions at scale
- Apply policies and RBAC inheritance across multiple subscriptions
- Nest up to 6 levels deep
- Enterprise-scale architecture pattern
- Common patterns: organize by business unit, team, or workload type

#### 3. Subscriptions
- **Billing boundary** — Each subscription gets its own invoice
- **Access control boundary** — RBAC applied at subscription level
- **Environment separation** — Best practice: separate subscriptions for Prod, Dev, Test, Sandbox
- Subscription limits and quotas (independent per subscription)
- What "subscription vending" means (automated provisioning)
- How teams request and receive subscriptions
- Governance applied at subscription creation

**Why separate subscriptions per environment?**
- Cost tracking per environment
- Blast radius containment (issues in Dev don't affect Prod)
- Different access controls and policies per environment

#### 4. Resource Groups
- **Logical container** for related resources
- Lifecycle management (delete group = delete all resources)
- RBAC and policies can be applied here
- Best practice: group by lifecycle, not by resource type

#### 5. Resources
- The actual Azure services (VMs, storage accounts, databases, etc.)
- Belong to exactly one resource group
- Can be moved between resource groups

**Key Concept:** Inheritance flows downward for RBAC and policies

**Reference:**
- [Azure management groups overview](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview)
- [Organize your resources](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-setup-guide/organize-resources)

---

### 1:50 – 2:15 | Azure Core Services (25 min)

**Approach:** Understanding Azure's service portfolio and choosing the right service for your workload.

#### Compute Services (8 min)

| Service | Description | When to Use |
|---------|-------------|-------------|
| **App Service** | Fully managed PaaS for web apps and APIs; built-in auto-scaling, deployment slots, CI/CD integration; supports .NET, Java, Node, Python, PHP | Web applications, REST APIs, mobile backends that need minimal infrastructure management |
| **Azure Functions** | Event-driven serverless compute; automatically scales based on demand; consumption and premium hosting plans; integrates with Event Grid, Service Bus, Storage | Event processing, scheduled tasks, microservices, API backends, automation workflows |
| **Azure Kubernetes Service (AKS)** | Managed Kubernetes platform; automated updates and patching; integrates with Azure Monitor, Key Vault, Container Registry, Azure networking | Container orchestration, microservices at scale, hybrid cloud applications |
| **Virtual Machines (VMs)** | Full IaaS control; Windows and Linux support; multiple VM sizes optimized for compute, memory, storage, or GPU; per-second billing | Legacy applications, applications requiring specific OS configurations, lift-and-shift migrations |

**Key Decision:** Start with PaaS (App Service/Functions) for new applications; use AKS for container orchestration; use VMs only when IaaS control is necessary.

#### Storage Services (7 min)

| Service | Description | When to Use |
|---------|-------------|-------------|
| **Blob Storage** | Massively scalable object storage; tiered storage (Hot, Cool, Archive); lifecycle management policies; supports Data Lake Storage Gen2 for analytics | Unstructured data, backups, media files, logs, data lakes, static website hosting |
| **Managed Disks** | Persistent block storage for VMs; SSD and HDD options; automatically replicated for durability; snapshot and backup support | VM operating system disks, application data disks, database storage for VMs |
| **Azure Files** | Fully managed cloud file shares; SMB and NFS protocols; mountable from cloud or on-premises; Azure File Sync for hybrid scenarios | Shared file storage, lift-and-shift of applications requiring file shares, development environments |
| **Queue Storage** | Simple message queuing service; durable message storage; supports millions of messages | Application decoupling, asynchronous processing, work distribution |

**Key Concept:** Choose redundancy levels (LRS, ZRS, GRS, GZRS) based on durability requirements and budget.

**Redundancy Options:**
- **LRS** (Locally Redundant) — 3 copies in one datacenter
- **ZRS** (Zone Redundant) — 3 copies across availability zones
- **GRS** (Geo Redundant) — 6 copies across two regions
- **GZRS** (Geo-Zone Redundant) — Combines ZRS and GRS

#### Networking Services (7 min)

| Service | Description | When to Use |
|---------|-------------|-------------|
| **Virtual Network (VNet)** | Isolated network in Azure; define your own IP address space; create subnets; control routing and filtering | Network isolation, hosting applications, connecting to on-premises networks |
| **Network Security Groups (NSGs)** | Firewall rules to control inbound and outbound traffic; applied at subnet or NIC level; priority-based rule evaluation | Network-level access control, micro-segmentation, security filtering |
| **Load Balancer** | Layer 4 (TCP/UDP) load balancing; supports inbound and outbound scenarios; zone-redundant for high availability | Distributing traffic across VMs, high availability for applications, outbound connectivity |
| **Application Gateway** | Layer 7 (HTTP/HTTPS) load balancer; Web Application Firewall (WAF); URL-based routing; SSL termination; session affinity | Web application load balancing, URL-based routing, SSL offload, web application protection |
| **Azure DNS** | Host DNS domains using Azure infrastructure; public and private DNS zones; anycast networking for performance | Domain name hosting, internal name resolution for VNets |
| **VPN Gateway** | Secure cross-premises connectivity; site-to-site, point-to-site, VNet-to-VNet connections | Connecting on-premises networks to Azure, remote user access |
| **ExpressRoute** | Private dedicated connection to Azure; bypasses public internet; higher reliability and speed; consistent latencies | Enterprise hybrid cloud, large data transfers, compliance requirements for private connectivity |

**Visual:** Show VNet architecture diagram
- VNets spanning regions
- Subnets for application tiers (web, app, data)
- NSGs controlling traffic flow
- Hub-and-spoke topology for enterprise connectivity

#### Database & Additional Services (3 min)

| Service | Description | When to Use |
|---------|-------------|-------------|
| **Azure SQL Database** | Fully managed SQL Server database; built-in high availability; automatic backups and patching | Relational data, applications requiring SQL Server compatibility |
| **Azure Database for PostgreSQL/MySQL** | Fully managed open-source databases; built-in high availability | Applications using PostgreSQL or MySQL |
| **Cosmos DB** | Globally distributed NoSQL database; multiple APIs (SQL, MongoDB, Cassandra, Gremlin); tunable consistency levels | Globally distributed applications, low-latency requirements, flexible schema data |
| **Azure Monitor** | Comprehensive monitoring solution; metrics, logs, alerts, Application Insights | Operational visibility, performance monitoring [covered in Session 2] |
| **Microsoft Entra ID** | Cloud-based identity and access management; SSO, MFA, conditional access | Authentication and authorization [covered in Session 2] |
| **Bicep / ARM Templates** | Infrastructure as Code for Azure; declarative resource deployment | Automated, repeatable infrastructure deployments [L200/L300 topic] |

**Key Takeaway:** Azure offers a comprehensive portfolio of services. Choose based on your workload requirements, not by mapping from other platforms.

**Reference:**
- [Azure for AWS professionals](https://learn.microsoft.com/en-us/azure/architecture/aws-professional/)

---

### 2:15 – 2:20 | Hands-On Portal Demo (5 min)

**Live Demo:** Bringing it all together in the Azure Portal

1. **Navigate the hierarchy** (2 min)
   - Show: Tenant → Management Groups → Subscriptions → Resource Groups
   - Demonstrate typical enterprise structure

2. **Create a resource** (2 min)
   - Create a Resource Group or Storage Account
   - Observe: Region selection, resource group assignment, tagging options
   - Note: How hierarchy, regions, and services concepts connect

3. **Quick Q&A** (1 min)
   - Questions on what we just saw?

**Key Observation:** The portal reflects everything we've learned — hierarchy, regions, naming conventions, and service types.

---

### 2:20 – 2:25 | Quick Reference: Azure to AWS Service Mapping (5 min)

**Note:** This is for reference only. Azure and AWS services are not 1:1 equivalents; each platform has unique capabilities and approaches.

| Category | Azure Service | AWS Service (Approximate) |
|----------|---------------|---------------------------|
| **Compute** | App Service | Elastic Beanstalk |
| | Azure Functions | Lambda |
| | Azure Kubernetes Service (AKS) | EKS / ECS |
| | Virtual Machines | EC2 |
| **Storage** | Blob Storage | S3 |
| | Managed Disks | EBS |
| | Azure Files | EFS |
| | Queue Storage | SQS |
| **Networking** | Virtual Network (VNet) | VPC |
| | Network Security Groups | Security Groups |
| | Load Balancer | ELB (NLB/CLB) |
| | Application Gateway | ALB |
| | Azure DNS | Route 53 |
| | ExpressRoute | Direct Connect |
| **Database** | Azure SQL Database | RDS for SQL Server |
| | Azure Database for PostgreSQL/MySQL | RDS for PostgreSQL/MySQL |
| | Cosmos DB | DynamoDB |
| **Management** | Azure Monitor | CloudWatch |
| | Bicep / ARM Templates | CloudFormation |
| | Microsoft Entra ID | IAM |

**Important:** Use this as a rough guide only. Focus on understanding what each Azure service does, not on one-to-one translations.

---

### 2:25 – 2:30 | Wrap-up & Homework (5 min)

**Key Takeaways:**
1. Azure is organised hierarchically: Tenant → Management Groups → Subscriptions → Resource Groups → Resources
2. Azure regions and availability zones provide global scale and resilience
3. Choose Azure services based on your workload needs: PaaS first, then containers, then IaaS
4. Everything we've learned fits into the Cloud Adoption Framework approach (we'll connect the dots in Session 2)

**Homework for Session 2:**
Review these Microsoft Learn modules (focus on the concepts, not labs):
- [Describe Azure identity, access, and security](https://learn.microsoft.com/en-us/training/modules/describe-azure-identity-access-security/)
- [Describe features and tools for governance and compliance](https://learn.microsoft.com/en-us/training/modules/describe-features-tools-azure-for-governance-compliance/)

**Coming Up in Session 2:**
- Azure RBAC and identity management
- Governance with policies and tagging
- Cost management
- Cloud Adoption Framework — bringing it all together

**Q&A:** Any burning questions before we finish?

**Reminder:** Optional "Ask the Expert" session details will be shared after Session 2.

---

## Additional Resources

**Official Microsoft Learn:**
- [Azure Fundamentals learning path](https://learn.microsoft.com/en-us/training/paths/azure-fundamentals-describe-azure-architecture-services/)
- [Cloud Adoption Framework](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/)
- [Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/)

**Optional — For AWS Professionals:**
- [Azure for AWS professionals](https://learn.microsoft.com/en-us/azure/architecture/aws-professional/)

---

*End of Session 1*

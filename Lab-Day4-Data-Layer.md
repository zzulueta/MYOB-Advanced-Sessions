---
lab:
  title: 'Lab Day 4: Data Layer – Storage & Databases'
  module: 'Advanced Azure Bootcamp – Day 4'
---

# Lab Day 4 – Data Layer: Storage & Databases

## Lab Introduction

In this lab you explore the Azure data layer by working hands-on with the four core
storage service categories — object, file, disk, and database — and then practice
selecting the right service for a given workload. You will work through the following tasks:
- Create a storage account and explore its service types
- Upload blobs and configure access tiers and lifecycle management
- Create an Azure File Share and mount it in Cloud Shell
- Review managed disk types and select the right SKU for a workload
- Deploy and query an Azure SQL Database (relational)
- Deploy Azure Cosmos DB and work with a NoSQL data model
- Apply storage and database design considerations to a real-world scenario

## Pre-requisites
- An Azure subscription
- A resource group named **RG-Lab4**.
Note: You will be assigned your own unique resource group name in the sandbox environment to avoid conflicts with other participants. Create a resource group in the same region you will deploy resources to (e.g., Australia East). Name the resource group RG-Lab4-yourname. Note the resource group name for use in all lab tasks.
- Owner rights at the resource group level

## Estimated Timing: 120 minutes

## Lab Scenario

Your organisation is migrating a multi-tier retail application to Azure. The application
has several distinct data requirements that cannot be served by a single storage
technology:

- Product images and order receipts need cost-effective, durable object storage with
  automatic tiering to reduce cost over time
- A legacy ERP system requires a shared network file system accessible from multiple
  virtual machines simultaneously
- Virtual machines running the application tier need high-performance managed disks
  with predictable IOPS
- Transactional order data must be stored relationally with ACID guarantees and
  queryable via SQL
- A product catalogue with variable and frequently changing attributes is a better
  fit for a flexible NoSQL document model

You have been tasked with provisioning and validating each of these services and then
documenting the trade-offs that led to each technology choice.

## Architecture Overview

```
RG-Lab4 (Resource Group)
├── Storage Account (stlab4yourname)
│   ├── Blob Container  — object storage (product images, receipts)
│   │   ├── Hot tier    — recently uploaded objects
│   │   └── Cool tier   — objects older than 30 days (lifecycle rule)
│   └── File Share (erp-share)  — SMB share mapped as Z: inside the VM
│
├── Virtual Network (vnet-lab4)
│   └── Subnet: default (10.0.0.0/24)
│       └── Windows VM (vm-lab4-erp)
│           ├── Public IP  — RDP access (port 3389)
│           └── Z: drive   — mapped to erp-share via SMB 3.x
│
├── Azure SQL Database  — relational (orders, transactions)
│   └── SQL Server (sqlserver-lab4-yourname)
│       └── Database: orders-db
│
└── Azure Cosmos DB     — NoSQL document (product catalogue)
    └── Account: cosmos-lab4-yourname
        └── Database: catalogue  →  Container: products
```

## Job Skills

- Task 1: Create a storage account and explore storage service types
- Task 2: Configure Blob Storage — upload objects and set access tiers
- Task 3: Apply a lifecycle management policy to Blob Storage
- Task 4: Deploy a Windows VM, connect via RDP, and map an Azure File Share
- Task 5: Deploy and query an Azure SQL Database
- Task 6: Deploy Azure Cosmos DB and work with a NoSQL document model
- Task 7: Apply storage and database design best practices

---

## Task 1: Create a Storage Account and Explore Storage Service Types

A single Azure Storage account exposes four storage services under one namespace:
Blob (object), Files, Queues, and Tables. In this task you create a storage account
and review what each service offers.

1. In the Azure portal, search for and select **Storage accounts**.

2. Select **+ Create** and configure the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab4** |
   | Storage account name | `stlab4yourname` (replace *yourname* to keep it globally unique) |
   | Region | **Australia East** |
   | Primary service | **Azure Blob Storage or Azure Data Lake Storage Gen 2** |
   | Performance | **Standard** |
   | Redundancy | **Locally-redundant storage (LRS)** |

3. Select the **Advanced** tab and review the following options (leave defaults):
   - **Require secure transfer** — enforces HTTPS-only access
   - **Allow Blob public access** — controls whether anonymous access is possible
   - **Minimum TLS version** — set to TLS 1.2

4. Select **Review + Create**, then **Create**. Wait for deployment to succeed.

5. Navigate to the new storage account. In the left menu, expand the **Data storage**
   section. Notice the four services listed:

   | Service | Use case |
   | --- | --- |
   | **Containers** (Blob) | Unstructured object storage — images, documents, backups, logs |
   | **File shares** | Managed SMB/NFS file shares mountable by VMs and on-premises clients |
   | **Queues** | Message queuing for decoupling application components |
   | **Tables** | Lightweight NoSQL key-value/entity store for structured, non-relational data |

   > **Note:** Queue and Table storage are covered in the Event-Driven Architecture
   > session (Day 6). This lab focuses on Blob and Files.

---

## Task 2: Configure Blob Storage — Upload Objects and Set Access Tiers

Azure Blob Storage is organised into containers (similar to directories). Each blob
can be individually assigned to an access tier: **Hot**, **Cool**, **Cold**, or **Archive**.
The tier determines the storage cost versus retrieval cost trade-off.

### Create a container and upload a blob

1. In your storage account, select **Containers** under **Data storage**.

2. Select **+ Container** and set:

   | Setting | Value |
   | --- | --- |
   | Name | `product-images` |
   | Anonymous access level | **Private (no anonymous access)** |

   Select **Create**.

3. Open the `product-images` container. Select **Upload**.

4. Select any small file from your local machine (a JPG or PDF works well).
   Expand **Advanced** and review the **Access tier** dropdown — notice the options:
   Hot, Cool, and Cold are available at upload time. Leave it as **Hot**.

5. Select **Upload**. Once complete, select the file to open its properties blade.

6. Review the **Access tier** field. Confirm it shows **Hot (inferred)** — the blob
   inherits the storage account's default tier.

### Change the tier on an individual blob

1. With the blob properties blade open, select **Change tier**.

2. Change the tier to **Cool** and select **Save**.

3. Return to the blob list and confirm the **Access tier** column now shows **Cool**.

**Access tier comparison:**

| Tier | Storage cost | Retrieval cost | Minimum storage duration | Suitable for |
| --- | --- | --- | --- | --- |
| Hot | Highest | Lowest | None | Frequently accessed data |
| Cool | Medium | Medium | 30 days | Infrequently accessed, read within 30 days |
| Cold | Low | Higher | 90 days | Rarely accessed, read within 90 days |
| Archive | Lowest | Highest + rehydration delay | 180 days | Long-term retention, rare access |

> **Key point:** Archive tier blobs cannot be read directly — they must be *rehydrated*
> to Hot or Cool first, which can take up to 15 hours. For compliance archiving where
> retrieval is rare (e.g. audit logs), Archive is extremely cost-effective.

---

## Task 3: Apply a Lifecycle Management Policy to Blob Storage

Lifecycle management policies automate tier transitions and deletions based on blob
age. This removes the need to manually manage tiers as data ages.

1. In your storage account, select **Lifecycle management** under **Data management**.

2. Select **+ Add a rule** and configure the **Details** tab:

   | Setting | Value |
   | --- | --- |
   | Rule name | `tier-to-cool-then-archive` |
   | Rule scope | **Apply rule to all blobs in your storage account** |
   | Blob type | **Block blobs** |
   | Blob subtype | **Base blobs** |

3. Select **Next** to open the **Base blobs** tab. Configure the following transitions:

   | Condition | Days after last modification | Action |
   | --- | --- | --- |
   | Move to Cool storage | `30` | ✔ |
   | Move to Cold storage | `90` | ✔ |
   | Move to Archive storage | `180` | ✔ |
   | Delete the blob | `365` | ✔ |

4. Select **Add** to save the rule.

5. Return to the **Lifecycle management** blade and confirm the rule appears in the list.

**Why this matters:** For the retail workload in this lab scenario, product images
uploaded more than 30 days ago are unlikely to be accessed frequently. Automatically
moving them to Cool and eventually Archive reduces storage cost without any
application code changes.

> **Note:** Lifecycle policies are evaluated once per day. Changes do not take effect
> immediately — they are applied during the next evaluation cycle.

---

## Task 4: Deploy a Windows VM, Connect via RDP, and Map an Azure File Share

Azure Files provides fully managed SMB and NFS file shares hosted inside a standard
Azure Storage account. Unlike Blob Storage — which exposes objects via HTTP/HTTPS
with no concept of directories, permissions, or file locking — Azure Files exposes a
**true file system hierarchy** that Windows and Linux machines can mount as a
network drive. This makes it the correct choice for lifting and shifting workloads
that depend on a shared UNC path, including ERP systems and departmental file servers.

**Azure Files tiers:**

| Tier | Storage type | Use case |
| --- | --- | --- |
| **Transaction optimized** | Standard (HDD) | General-purpose shares, mixed read/write workloads |
| **Hot** | Standard (HDD) | Frequently accessed team drives and active project files |
| **Cool** | Standard (HDD) | Infrequently accessed shares, compliance archives |
| **Premium** | SSD | Latency-sensitive workloads, high IOPS requirements |

In this task you will:
1. Create the `erp-share` Azure File Share
2. Deploy a small Windows VM in the same resource group
3. RDP into the VM and map the share as a persistent network drive
4. Write a file from inside the VM and confirm it is visible in the portal

### Step 1: Create the Azure File Share

1. In your storage account (`stlab4yourname`), select **File shares** under **Data storage**.

2. Select **+ File share** and configure:

   | Setting | Value |
   | --- | --- |
   | Name | `erp-share` |
   | Tier | **Transaction optimized** |
   | Quota | `100` GiB |

   > **Quota:** Caps maximum share size. You are billed only for data actually stored —
   > the quota does not pre-allocate space.

3. Select **Create**.

4. Open the `erp-share` file share. Select **+ Add directory** and create two folders:
   - `invoices`
   - `reports`

5. Select the `invoices` directory, then **Upload** and upload any small file from
   your local machine as a test fixture.

### Step 2: Deploy a Windows Virtual Machine

1. In the Azure portal, search for and select **Virtual machines**.

2. Select **+ Create** → **Azure virtual machine** and configure the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab4** |
   | Virtual machine name | `vm-lab4-erp` |
   | Region | **Australia East** |
   | Availability options | **No infrastructure redundancy required** |
   | Image | **Windows Server 2022 Datacenter – x64 Gen2** |
   | Size | **Standard_B2s** (2 vCPUs, 4 GiB RAM) |
   | Administrator username | `azureuser` |
   | Password | A strong password (note this down) |
   | Public inbound ports | **Allow selected ports** |
   | Select inbound ports | **RDP (3389)** |

   > **Cost note:** Standard_B2s is the smallest practical Windows size for this lab.
   > Stop (deallocate) and delete the VM as soon as the task is complete to avoid
   > ongoing charges.

3. Select the **Networking** tab. Confirm a new virtual network (`vnet-lab4`) and
   subnet are being created automatically. Leave all networking defaults.

4. Select **Review + Create**, then **Create**. Wait for the deployment to complete
   (typically 2–3 minutes).

### Step 3: Connect to the VM via RDP

1. Navigate to the `vm-lab4-erp` virtual machine resource.

2. Select **Connect** → **Connect** from the top toolbar.

3. Under the **Native RDP** section, select **Download RDP file**.

4. Open the downloaded `.rdp` file. When prompted, select **Connect**.

5. Enter the credentials:
   - **Username:** `azureuser`
   - **Password:** the password you set during VM creation

6. Accept the certificate warning and select **Yes**. The Windows desktop should
   appear inside the RDP session.

   > If RDP is blocked by your corporate firewall, use the **Bastion** option in the
   > Connect blade instead — Azure Bastion provides browser-based RDP over HTTPS
   > on port 443 with no public IP required on the VM.

### Step 4: Retrieve the Azure Files Mount Script

1. **Keep the RDP session open.** Switch back to the Azure portal on your local machine.

2. Navigate to your storage account → **File shares** → **erp-share**.

3. Select **Connect** from the top toolbar.

4. Select **Windows**. Read through the generated PowerShell script before using it.
   Key elements to notice:
   - **`net use Z:`** — maps the share to drive letter `Z:`
   - **`\\storageaccountname.file.core.windows.net\erp-share`** — the UNC path of
     the share; this is identical in format to a traditional Windows file server path
   - **`/user:Azure\storageaccountname`** and the storage account key — authenticate
     using the account key
   - **`-Persist`** — ensures the mapping survives reboots

   > **Security note:** The storage account key grants full access to the entire
   > storage account. For production workloads, use **Azure AD Kerberos
   > authentication** or **on-premises AD DS integration** so individual user
   > identities are used instead of a shared key. This enables per-user NTFS
   > permissions and removes the need to distribute the account key.

5. Select the **copy** icon to copy the full PowerShell script to your clipboard.

### Step 5: Map the Share as Drive Z: Inside the VM

1. Switch back to your RDP session.

2. Open **PowerShell** as Administrator:
   - Right-click the **Start** button → **Windows PowerShell (Admin)**

3. Paste and run the copied mount script.

4. Confirm the drive is mapped:

   ```powershell
   Get-PSDrive Z
   ```

   The output should show drive `Z:` with a `Root` of
   `\\storageaccountname.file.core.windows.net\erp-share`.

5. Open **File Explorer** inside the RDP session. Confirm drive `Z:` appears under
   **This PC** with the label **erp-share**. Open it — you should see the `invoices`
   and `reports` directories you created in Step 1.

6. Navigate to `Z:\invoices`. Confirm the test file you uploaded from the portal
   earlier is present.

### Step 6: Write a File from the VM and Verify in the Portal

1. In the RDP session PowerShell window, create a new file on the share:

   ```powershell
   "ERP data written from vm-lab4-erp" | Out-File Z:\invoices\from-vm.txt
   Get-Content Z:\invoices\from-vm.txt
   ```

2. Create a second file in the `reports` folder:

   ```powershell
   "Monthly report - March 2026" | Out-File Z:\reports\report-2026-03.txt
   ```

3. Switch back to the Azure portal on your local machine. Navigate to your storage
   account → **File shares** → **erp-share** → **invoices**.

4. Confirm `from-vm.txt` appears in the portal. Open it — the content written from
   inside the VM is visible immediately.

   This demonstrates that Azure Files behaves like a real network share: any
   client with the share mounted sees writes made by any other client in real time.

### Step 7: Clean Up the VM

1. Close the RDP session.

2. In the Azure portal, navigate to **RG-Lab4** and select the `vm-lab4-erp` VM.

3. Select **Stop** to deallocate the VM and halt compute billing. You will delete
   the entire resource group at the end of the lab.

> **What is not cleaned up yet:** The `vnet-lab4` virtual network, public IP, NIC,
> OS disk, and NSG were all created automatically alongside the VM. All of these
> will be removed when the resource group is deleted at the end of the lab.

**SMB vs NFS:**

| Protocol | OS support | Use case |
| --- | --- | --- |
| **SMB 3.x** | Windows and Linux | Line-of-business apps, Active Directory integration, home drives |
| **NFS 4.1** | Linux only | POSIX permission requirements, Linux-native workloads, high-throughput pipelines |

> NFS shares require a **Premium** tier file share. Standard storage accounts only
> support SMB.

### Azure File Sync (conceptual)

For organisations that need to keep on-premises Windows file servers in sync with
Azure Files, **Azure File Sync** transforms a local Windows Server into a smart
cache. Frequently accessed files are kept on-premises for low-latency access;
infrequently accessed files are *cloud-tiered* — their content is moved to Azure
Files and replaced locally with a transparent placeholder stub. When a user opens a
stub, the content is downloaded on demand.

This enables gradual migration: the file server continues to work normally while
Azure Files becomes the system of record. The on-premises server can be
decommissioned at any time once users are comfortable accessing the share directly.

---

## Task 5: Deploy and Query an Azure SQL Database

Azure SQL Database is a fully managed relational database service (PaaS) built on
SQL Server. It handles patching, backups, and high availability automatically. In
this task you create a server and database, then run queries via the portal's
built-in Query Editor.

### Create an Azure SQL Server and Database

1. In the Azure portal, search for and select **SQL databases**.

2. Select **+ Create** and configure the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab4** |
   | Database name | `orders-db` |
   | Server | Select **Create new** (see below) |
   | Want to use SQL elastic pool? | **No** |
   | Workload environment | **Development** |
   | Compute + storage | Select **Configure database** → Choose **Basic** tier (5 DTUs, 2 GB) |
   | Backup storage redundancy | **Locally-redundant backup storage** |

3. In the **Create SQL Server** dialog, configure:

   | Setting | Value |
   | --- | --- |
   | Server name | `sqlserver-lab4-yourname` (must be globally unique) |
   | Location | **Australia East** |
   | Authentication method | **Use SQL authentication** |
   | Server admin login | `sqladmin` |
   | Password | A strong password (note this down) |

   Select **OK**.

4. Select **Next: Networking**. Under **Firewall rules**, set:

   | Setting | Value |
   | --- | --- |
   | Add current client IPv4 address | **Yes** |

5. Select **Review + Create**, then **Create**. Wait for deployment to complete.

### Create the orders table and insert data

1. Navigate to the `orders-db` database resource.

2. In the left menu, select **Query editor (preview)**.

3. Log in using `sqladmin` and the password you set.

4. In the query window, create the orders table:

   ```sql
   CREATE TABLE orders (
       order_id     INT IDENTITY(1,1) PRIMARY KEY,
       customer_id  INT            NOT NULL,
       product_code NVARCHAR(50)   NOT NULL,
       quantity     INT            NOT NULL,
       order_date   DATE           NOT NULL DEFAULT GETDATE(),
       total_amount DECIMAL(10,2)  NOT NULL
   );
   ```

   Select **Run**.

5. Insert sample rows:

   ```sql
   INSERT INTO orders (customer_id, product_code, quantity, order_date, total_amount)
   VALUES
       (1001, 'SKU-SHOES-42', 2, '2026-03-01', 159.90),
       (1002, 'SKU-SHIRT-M',  1, '2026-03-02',  49.95),
       (1001, 'SKU-HAT-L',    3, '2026-03-05',  89.85),
       (1003, 'SKU-SHOES-38', 1, '2026-03-10',  79.95);
   ```

   Select **Run**.

6. Query the data:

   ```sql
   SELECT
       order_id,
       customer_id,
       product_code,
       quantity,
       order_date,
       total_amount
   FROM orders
   ORDER BY order_date;
   ```

7. Run an aggregation to demonstrate relational query capabilities:

   ```sql
   SELECT
       customer_id,
       COUNT(*)          AS total_orders,
       SUM(total_amount) AS lifetime_value
   FROM orders
   GROUP BY customer_id
   ORDER BY lifetime_value DESC;
   ```

### Review high-availability and backup configuration

1. In the left menu, select **Overview**. Note the **Server name** FQDN format:
   `sqlserver-lab4-yourname.database.windows.net`

2. Select **Backups** under **Data management**. Observe:
   - **Full backups** are taken weekly
   - **Differential backups** are taken every 12–24 hours
   - **Transaction log backups** are taken every 5–10 minutes
   - The **Retention period** defaults to 7 days on the Basic tier

3. Select **Compute + storage** under **Settings** and review the DTU-based pricing
   model. Note the option to scale up to higher service tiers without re-deploying
   the database.

**Key relational database concepts demonstrated:**
- **Schema enforcement** — the `CREATE TABLE` statement defines column names, types,
  and constraints; SQL Server rejects any insert that violates them
- **ACID transactions** — inserts and updates are atomic and durable by default
- **Aggregation and joins** — SQL natively expresses cross-row and cross-table
  calculations without application-layer iteration

---

## Task 6: Deploy Azure Cosmos DB and Work with a NoSQL Document Model

Azure Cosmos DB is a fully managed, globally distributed NoSQL database. Unlike
relational databases, Cosmos DB stores documents (JSON objects) with flexible schemas
— no up-front table definition is required, and each document in a container can have
a different shape.

In this task you deploy a Cosmos DB account, create a container for the product
catalogue, and compare how the same data is modelled differently from the relational
orders table.

### Create a Cosmos DB account

1. In the Azure portal, search for and select **Azure Cosmos DB**.

2. Select **+ Create**. On the **Select API option** page, choose **Azure Cosmos DB
   for NoSQL**. Select **Create**.

3. Configure the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab4** |
   | Account name | `cosmos-lab4-yourname` (globally unique) |
   | Location | **Australia East** |
   | Capacity mode | **Serverless** |
   | Apply Free Tier discount | **Apply** (if available on your subscription) |

4. Select **Review + Create**, then **Create**. Deployment takes 3–5 minutes.

### Create a database and container

1. Navigate to the Cosmos DB account. In the left menu, select **Data Explorer**.

2. Select **New Container** and configure:

   | Setting | Value |
   | --- | --- |
   | Database id | **Create new** → `catalogue` |
   | Container id | `products` |
   | Partition key | `/category` |

   Select **OK**.

3. Expand **catalogue** → **products** in the left pane. Select **Items**.

### Insert product documents

1. Select **New Item** and paste the following JSON document:

   ```json
   {
     "id": "prod-001",
     "category": "footwear",
     "name": "Running Shoe X200",
     "brand": "SpeedFit",
     "sizes": ["36", "37", "38", "39", "40", "41", "42", "43", "44", "45"],
     "price": 79.95,
     "stock": 243,
     "attributes": {
       "colour": "black/white",
       "material": "mesh upper, rubber sole",
       "waterproof": false
     },
     "tags": ["running", "sport", "lightweight"]
   }
   ```

   Select **Save**.

2. Select **New Item** again and paste a second document with a different structure:

   ```json
   {
     "id": "prod-002",
     "category": "apparel",
     "name": "Merino Pullover",
     "brand": "WoolCo",
     "sizes": ["XS", "S", "M", "L", "XL"],
     "price": 119.00,
     "stock": 89,
     "attributes": {
       "colour": "navy",
       "material": "100% merino wool",
       "care": "hand wash cold",
       "sustainablySourced": true
     },
     "tags": ["casual", "warm", "sustainable"]
   }
   ```

   Select **Save**.

   > **Notice:** The two documents share common fields (`id`, `category`, `price`,
   > `stock`) but their `attributes` objects are entirely different. In a relational
   > model you would need to add nullable columns or a separate EAV table. In Cosmos
   > DB, the flexible schema requires no schema change.

### Query the container using the NoSQL API

1. Select **New SQL Query** (Cosmos DB for NoSQL uses a SQL-like query syntax).

2. Run the following query to retrieve all products:

   ```sql
   SELECT * FROM products p
   ```

3. Filter by category:

   ```sql
   SELECT p.id, p.name, p.price, p.stock
   FROM products p
   WHERE p.category = "footwear"
   ```

4. Query a nested attribute:

   ```sql
   SELECT p.name, p.attributes.colour
   FROM products p
   WHERE p.price < 100
   ```

5. Select **Query Stats** after running a query. Note the **Request Units (RU)** consumed.
   RUs are the Cosmos DB billing and throughput unit — one RU equals the cost of
   reading a 1 KB document.

### Review global distribution (read-only)

1. In the left menu, select **Replicate data globally**.

2. Observe the world map. The primary write region is **Australia East** (as configured).

3. Review the **Add region** option — in a production account you would add read
   regions here to serve geographically distributed users with low-latency reads.
   **Do not add regions** during this lab (additional cost).

---

## Task 7: Apply Storage and Database Design Best Practices

In this task you review the design decisions made across the lab and work through
a structured decision framework for common data layer scenarios.

### Storage service selection framework

Use this table when selecting between Azure storage services:

| Requirement | Recommended service | Key reason |
| --- | --- | --- |
| Store millions of images, videos, or documents | **Blob Storage** | Cheapest per-GB; HTTP access; lifecycle tiering |
| Share files between multiple VMs (SMB/NFS) | **Azure Files** | Native file system semantics; mountable from many OSes |
| OS and data disks for Virtual Machines | **Managed Disks** | Block storage; SLA-backed IOPS; snapshot support |
| Key-value lookups, simple entity storage | **Table Storage** | Schemaless; low cost; no relational overhead |
| Decouple application components asynchronously | **Queue Storage** | Simple FIFO queue; at-least-once delivery |

### Database service selection framework

| Requirement | Recommended service | Key reason |
| --- | --- | --- |
| Transactions, joins, reporting (SQL) | **Azure SQL Database** | ACID, full SQL surface area, built-in HA |
| Large-scale analytics, data warehouse | **Azure Synapse Analytics** | Massively parallel processing; columnar storage |
| Flexible schema, global distribution, high throughput | **Azure Cosmos DB** | Multi-model NoSQL; single-digit ms latency; multi-region writes |
| Open-source relational (PostgreSQL) | **Azure Database for PostgreSQL** | Managed PG with extensions; familiar tooling |
| Open-source relational (MySQL) | **Azure Database for MySQL** | Managed MySQL; LAMP stack compatibility |
| In-memory caching, session state | **Azure Cache for Redis** | Sub-millisecond reads; supports cache-aside pattern |

### Scenario analysis: choose the right services

Review the following workload requirements and consider which Azure data services
best fit each scenario before revealing the answer.

**Scenario A — Medical imaging archive**
> 50 TB of DICOM scan images, accessed frequently in the first 30 days post-scan
> and rarely thereafter. Regulatory requirement to retain images for 7 years.

- **Answer:** Blob Storage with a lifecycle policy moving blobs from Hot → Cool at
  30 days → Archive at 90 days. Archive provides the lowest storage cost for the
  long tail. Use immutability policies to satisfy regulatory retention requirements.

**Scenario B — Multi-region e-commerce product catalogue**
> 5 million product documents with variable attributes per category. Users in
> Australia, Europe, and North America; maximum 10 ms read latency per region.

- **Answer:** Azure Cosmos DB (NoSQL API) with read replicas in Australia East,
  West Europe, and East US. The flexible document schema handles per-category
  attribute variation. Cosmos DB's multi-region reads satisfy the latency SLA;
  no relational join path is required for catalogue reads.

**Scenario C — Financial transactions ledger**
> Core banking transactions requiring ACID guarantees, double-entry accounting
> (every debit has a matching credit), and ad-hoc SQL reporting by the finance team.

- **Answer:** Azure SQL Database (Business Critical tier for high IOPS and read
  replicas). The relational model with foreign key constraints, transaction isolation,
  and full SQL aggregate support (SUM, JOIN, GROUP BY) directly satisfies all three
  requirements. Cosmos DB's eventual consistency model does not satisfy ACID
  requirements for financial ledgers without additional application complexity.

**Scenario D — Shared configuration files for 20 VMs running a legacy app**
> A .NET application reads INI and XML config files from `\\fileserver\config` at
> startup. The app cannot be changed to use a different access pattern.

- **Answer:** Azure Files (SMB share). Blob Storage does not support SMB; the
  application requires the UNC path syntax and file system semantics that only
  Azure Files provides. Mount the share on each VM using the storage account key.

### Cost optimisation principles

1. **Match the tier to the access pattern.** Storing infrequently accessed data in Hot
   tier wastes money; storing frequently accessed data in Archive causes expensive
   and slow retrievals.

2. **Right-size databases.** Azure SQL Database DTUs and Cosmos DB RU/s can be scaled
   independently of storage — scale compute down during off-peak hours.

3. **Use LRS for non-critical data.** LRS is sufficient for dev/test and easily
   reproducible data. Reserve ZRS and GRS for production data where cross-zone or
   cross-region recovery is required.

4. **Avoid premium disks for dev/test VMs.** Standard SSD provides adequate performance
   for development workloads at a significantly lower cost than Premium SSD.

5. **Enable soft delete and versioning on Blob Storage** for production containers —
   this provides a recovery window for accidental deletes without the cost of
   maintaining separate backup storage.

---

## Cleanup

**Note:** Azure SQL Database, Cosmos DB, and managed disks incur ongoing charges.
Delete resources promptly after the lab.

1. In the portal, navigate to **RG-Lab4**.

2. Select **Delete resource group**.

3. Copy and paste the resource group name to confirm deletion.

4. Select **Delete**.

   This removes all resources in the group including the storage account, SQL server
   and database, Cosmos DB account, and managed disk.

---

## Key Takeaways

- **Azure Storage** is a single account that provides four services: Blob (object),
  Files (SMB/NFS), Queues (messaging), and Tables (key-value). Choosing the right
  service depends on the access pattern, not on familiarity.

- **Blob access tiers** (Hot, Cool, Cold, Archive) let you match storage cost to
  access frequency. Lifecycle management policies automate tier transitions and
  deletions based on blob age, removing the need for manual maintenance.

- **Azure Files** provides managed file shares mountable via SMB or NFS. It is the
  correct choice for lifting and shifting workloads that depend on shared network
  drives — Blob Storage does not support the file system semantics these workloads
  require.

- **Managed Disks** provide block storage for Virtual Machine OS and data disks.
  The SKU (Standard HDD → Ultra Disk) determines IOPS and throughput; Premium SSD
  v2 and Ultra Disk decouple IOPS from disk capacity.

- **Azure SQL Database** provides a fully managed relational database with ACID
  transactions, full SQL support, automatic backups, and built-in high availability.
  Choose it when your workload requires structured schemas, joins, or aggregate
  reporting.

- **Azure Cosmos DB** provides a fully managed NoSQL database with a flexible
  document schema, single-digit millisecond latency, and native multi-region
  distribution. Choose it when schema flexibility, global reach, or extreme write
  throughput is more important than relational consistency.

- **Relational vs non-relational** is not a quality judgement — it is a fit
  judgement. Financial transactions belong in SQL; globally distributed product
  catalogues with variable attributes belong in Cosmos DB. Many real applications
  use both.

- **Storage design decisions are hard to reverse.** Migrating from Blob to Files or
  from SQL to Cosmos DB mid-project is expensive. Invest time up front in
  understanding access patterns, consistency requirements, latency targets, and
  cost constraints before choosing a service.

---

## Resources

- [Azure Storage documentation](https://learn.microsoft.com/en-us/azure/storage/)
- [Blob Storage access tiers](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview)
- [Azure Files documentation](https://learn.microsoft.com/en-us/azure/storage/files/)
- [Azure Managed Disks overview](https://learn.microsoft.com/en-us/azure/virtual-machines/managed-disks-overview)
- [Azure SQL Database documentation](https://learn.microsoft.com/en-us/azure/azure-sql/database/)
- [Azure Cosmos DB documentation](https://learn.microsoft.com/en-us/azure/cosmos-db/)
- [Choose an Azure data store](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/data-store-overview)
- [Azure Storage redundancy options](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)
- [Cosmos DB request units](https://learn.microsoft.com/en-us/azure/cosmos-db/request-units)

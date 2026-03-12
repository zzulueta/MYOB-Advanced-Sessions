---
lab:
  title: 'Lab Day 2: Networking'
  module: 'Advanced Azure Bootcamp – Day 2'
---

# Lab Day 2 – Networking

## Lab Introduction

In this lab you build a segmented network foundation in Azure and then progressively
connect and protect it. You will work through the following tasks:

- Design and create virtual networks with subnets
- Protect subnets with Network Security Groups and Application Security Groups
- Provide name resolution with public and private Azure DNS zones
- Connect segmented networks using virtual network peering
- Distribute traffic at Layer 4 using Azure Load Balancer
- Route traffic at Layer 7 using Azure Application Gateway with path-based rules

## Pre-requisites

- An Azure subscription
- A resource group named **RG-Lab2**
- Owner rights at the resource group level

## Estimated Timing: 90 minutes

## Lab Scenario

Your organisation is deploying a segmented Azure network environment. Core IT
services (DNS, security tooling, databases) are isolated from the manufacturing
department's operational systems, but controlled connectivity between the two
segments is required for inter-system communication.

At the same time, a public-facing web application must be highly available.
Incoming HTTP requests must be distributed across multiple virtual machines using a
standard Load Balancer, and image and video content must be routed to dedicated
backend pools using the Application Gateway's path-based routing capability.

You have been tasked with:

- Creating two virtual networks with properly sized subnets for each segment
- Enforcing traffic controls between subnets using NSG and ASG rules
- Providing private and public DNS resolution
- Enabling cross-segment connectivity via virtual network peering
- Deploying and configuring an Azure Load Balancer for Layer 4 traffic distribution
- Deploying and configuring an Azure Application Gateway for Layer 7 path-based routing

## Architecture Overview

```
Subscription
└── RG-Lab2
    ├── CoreServicesVnet (10.20.0.0/16)
    │   ├── SharedServicesSubnet (10.20.10.0/24)  ← NSG applied
    │   └── DatabaseSubnet (10.20.20.0/24)
    │
    ├── ManufacturingVnet (10.30.0.0/16)
    │   ├── SensorSubnet1 (10.30.20.0/24)
    │   └── SensorSubnet2 (10.30.21.0/24)
    │
    ├── VNet Peering: CoreServicesVnet ↔ ManufacturingVnet
    │
    ├── AppVnet (10.60.0.0/16)
    │   ├── BackendSubnet1 (10.60.1.0/24)  ← vm0
    │   ├── BackendSubnet2 (10.60.2.0/24)  ← vm1, vm2
    │   └── AppGwSubnet (10.60.3.224/27)   ← Application Gateway
    │
    ├── Application Security Group: asg-web
    ├── Network Security Group: myNSGSecure (→ SharedServicesSubnet)
    ├── Public DNS Zone: contoso.com
    ├── Private DNS Zone: private.contoso.com (→ CoreServicesVnet)
    ├── Azure Load Balancer (Standard, Public) → vm0, vm1
    └── Application Gateway (Standard V2, path-based) → /image/* vm1, /video/* vm2
```

## Job Skills

- Task 1: Create virtual networks and subnets
- Task 2: Secure networks with NSG and ASG
- Task 3: Configure public and private Azure DNS zones
- Task 4: Connect networks with virtual network peering
- Task 5: Configure an Azure Load Balancer
- Task 6: Configure an Azure Application Gateway with path-based routing

---

## Task 1: Create Virtual Networks and Subnets

Virtual networks are the foundational building block for private networking in Azure.
In this task you create two virtual networks — one for core services and one for
manufacturing — each with appropriately sized subnets to accommodate current resources
and planned growth.

> **Design note:** It is a good practice to avoid overlapping IP address ranges across all virtual networks and any connected on-premises networks. Overlapping ranges block peering and increase troubleshooting complexity. Plan your IP addressing scheme across the entire environment before deploying.

### Create the CoreServicesVnet

1. Sign in to the [Azure portal](https://portal.azure.com).

2. Search for and select **Virtual Networks**, then select **+ Create**.

3. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab2** |
   | Name | `CoreServicesVnet` |
   | Region | **Australia East** (or your preferred region — keep consistent throughout the lab) |

4. Select the **IP Addresses** tab. Replace the default IPv4 address space with `10.20.0.0/16`.

5. Delete the default subnet, then select **+ Add a subnet** for each of the following. Select **Add** after each:

   | Subnet | Subnet name | Starting address | Size |
   | --- | --- | --- | --- |
   | First | `SharedServicesSubnet` | `10.20.10.0` | `/24` |
   | Second | `DatabaseSubnet` | `10.20.20.0` | `/24` |

   > **Note:** Every virtual network must have at least one subnet. Azure reserves five IP addresses within each subnet (the network address, broadcast address, and three addresses reserved by Azure), so a /24 gives you 251 usable addresses.

6. Select **Review + create**, then **Create**. Wait for the deployment to succeed.

### Create the ManufacturingVnet

7. Select **+ Create** again to create a second virtual network.

8. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab2** |
   | Name | `ManufacturingVnet` |
   | Region | **Australia East** |

9. On the **IP Addresses** tab, replace the address space with `10.30.0.0/16`.

10. Delete the default subnet and add the following two subnets:

    | Subnet | Subnet name | Starting address | Size |
    | --- | --- | --- | --- |
    | First | `SensorSubnet1` | `10.30.20.0` | `/24` |
    | Second | `SensorSubnet2` | `10.30.21.0` | `/24` |

11. Select **Review + create**, then **Create**.

12. After both virtual networks are deployed, navigate to **CoreServicesVnet**. On the **Overview** blade, verify the address space (`10.20.0.0/16`) and confirm both subnets appear under the **Subnets** section.

---

## Task 2: Secure Networks with NSG and ASG

Network Security Groups (NSGs) filter inbound and outbound traffic using rules based
on source, destination, protocol, and port. Application Security Groups (ASGs) let
you group virtual machines logically — such as "all web servers" — and use that group
as the source or destination in NSG rules, rather than managing individual IP addresses.

### Create the Application Security Group

1. Search for and select **Application security groups**, then select **+ Create**.

2. Configure the following:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab2** |
   | Name | `asg-web` |
   | Region | **Australia East** |

3. Select **Review + create**, then **Create**.

   > **Note:** In production you would associate this ASG with virtual machine NICs. Any NSG rule targeting `asg-web` will automatically apply to all VMs associated with it — no IP address management required.

### Create the Network Security Group

4. Search for and select **Network security groups**, then select **+ Create**.

5. Configure:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab2** |
   | Name | `myNSGSecure` |
   | Region | **Australia East** |

6. Select **Review + create**, then **Create**, then **Go to resource**.

### Associate the NSG with a subnet

7. In the **Settings** blade of `myNSGSecure`, select **Subnets**, then **+ Associate**.

   | Setting | Value |
   | --- | --- |
   | Virtual network | **CoreServicesVnet (RG-Lab2)** |
   | Subnet | **SharedServicesSubnet** |

8. Select **OK** to save the association.

### Add an inbound rule to allow web traffic from the ASG

9. In the **Settings** blade, select **Inbound security rules**, then **+ Add**.

10. Configure the inbound rule:

    | Setting | Value |
    | --- | --- |
    | Source | **Application security group** |
    | Source application security groups | **asg-web** |
    | Source port ranges | `*` |
    | Destination | **Any** |
    | Service | **Custom** |
    | Destination port ranges | `80,443` |
    | Protocol | **TCP** |
    | Action | **Allow** |
    | Priority | `100` |
    | Name | `AllowASG` |

11. Select **Add**.

### Add an outbound rule to deny internet access

12. Select **Outbound security rules**, then **+ Add**.

13. Configure the outbound rule:

    | Setting | Value |
    | --- | --- |
    | Source | **Any** |
    | Source port ranges | `*` |
    | Destination | **Service tag** |
    | Destination service tag | **Internet** |
    | Service | **Custom** |
    | Destination port ranges | `*` |
    | Protocol | **Any** |
    | Action | **Deny** |
    | Priority | `4096` |
    | Name | `DenyInternetOutbound` |

14. Select **Add**.

**Key point:** NSG rules are evaluated in priority order — lowest number is processed first. The default `AllowInternetOutBound` rule has priority 65001, so the `DenyInternetOutbound` rule at 4096 overrides it. The default inbound and outbound rules cannot be deleted; you override them with higher-priority rules.

---

## Task 3: Configure Public and Private Azure DNS Zones

Azure DNS hosts DNS zones and provides name resolution using Azure's global
infrastructure. Public DNS zones serve internet-facing name resolution. Private DNS
zones provide name resolution exclusively within virtual networks — no public exposure.

### Configure a public DNS zone

1. Search for and select **DNS zones**, then select **+ Create**.

2. Configure:

   | Property | Value |
   | --- | --- |
   | Resource group | **RG-Lab2** |
   | Name | `adventuretravel.com` |
   | Region | **Australia East** |

3. Select **Review + create**, then **Create**. Select **Go to resource**.

4. On the **Overview** blade, note the four Azure DNS name server addresses assigned to the zone — these would be registered with your domain registrar to delegate resolution to Azure.

5. In the **DNS Management** blade, select **Recordsets**, then **+ Add**.

   | Property | Value |
   | --- | --- |
   | Name | `www` |
   | Type | **A** |
   | TTL | `1` |
   | IP address | `10.20.10.4` |

   > **Note:** In a real-world scenario this would be the public IP of your web server or load balancer.

6. Select **Add**. Confirm the `www` A record appears in the recordset list.

7. Open a local command prompt (Windows) or terminal (macOS/Linux) and run the following, replacing `<name server>` with one of the name server addresses from step 4:

   ```sh
   nslookup www.adventuretravel.com <name server>
   ```

   Verify the response resolves to `10.20.10.4`. This confirms the public DNS zone is working correctly.

### Configure a private DNS zone

8. Search for and select **Private DNS zones**, then select **+ Create**.

9. Configure:

   | Property | Value |
   | --- | --- |
   | Resource group | **RG-Lab2** |
   | Name | `private.contoso.com` |

10. Select **Review + create**, then **Create**. Select **Go to resource**.

11. Notice the **Overview** blade shows no name server records — private zones are not registered in public DNS.

12. In the **DNS Management** blade, select **Virtual network links**, then **+ Add**.

    | Property | Value |
    | --- | --- |
    | Link name | `coreservices-link` |
    | Virtual network | **CoreServicesVnet** |
    | Enable auto registration | Leave unchecked |

13. Select **OK** and wait for the link status to show **Completed**.

14. Select **Recordsets**, then **+ Add**:

    | Property | Value |
    | --- | --- |
    | Name | `db01` |
    | Type | **A** |
    | TTL | `1` |
    | IP address | `10.20.20.4` |

    > **Note:** In production, this record would point to an actual database server private IP in the DatabaseSubnet.

**Why private DNS?** Any VM connected to `CoreServicesVnet` can now resolve `db01.private.contoso.com` to `10.20.20.4` without the record being visible on the internet. This eliminates hardcoded IP addresses in application configuration files.

---

## Task 4: Connect Networks with Virtual Network Peering

By default, resources in different virtual networks cannot communicate — even within
the same subscription and region. Virtual network peering creates a direct,
low-latency connection that uses the Microsoft backbone, not the public internet.
Peered networks appear as one for connectivity purposes.

### Configure bidirectional peering

1. In the Azure portal, navigate to **CoreServicesVnet**.

2. Under **Settings**, select **Peerings**, then **+ Add**.

3. Fill in both sides of the peering simultaneously — Azure creates the return peering automatically:

   | Parameter | Value |
   | --- | --- |
   | **Remote virtual network summary** | |
   | Peering link name | `ManufacturingVnet-to-CoreServicesVnet` |
   | Virtual network | **ManufacturingVnet (RG-Lab2)** |
   | **Remote virtual network peering settings** | |
   | Allow 'ManufacturingVnet' to access 'CoreServicesVnet' | Selected (default) |
   | Allow 'ManufacturingVnet' to receive forwarded traffic from 'CoreServicesVnet' | Selected |
   | **Local virtual network summary** | |
   | Peering link name | `CoreServicesVnet-to-ManufacturingVnet` |
   | **Local virtual network peering settings** | |
   | Allow 'CoreServicesVnet' to access 'ManufacturingVnet' | Selected (default) |
   | Allow 'CoreServicesVnet' to receive forwarded traffic from 'ManufacturingVnet' | Selected |

4. Select **Add** and wait a few seconds.

5. In the **Peerings** blade of `CoreServicesVnet`, confirm the peering **Peering status** shows **Connected**. Refresh if needed.

6. Navigate to **ManufacturingVnet → Peerings** and verify the reverse peering also shows **Connected**.

### Verify connectivity with Network Watcher

7. Search for and select **Network Watcher**.

8. In the **Network diagnostic tools** section, select **Connection troubleshoot**.

9. If you have virtual machines deployed in the two VNets, test connectivity between them. If not, observe the available fields:

   | Field | Value |
   | --- | --- |
   | Source type | **Virtual machine** |
   | Destination type | **Select a virtual machine** |
   | Protocol | **TCP** |
   | Destination port | `3389` |

   Before peering: the test would show **Unreachable**.  
   After peering: the test shows **Reachable** — confirming the peering is active.

**Key point:** Virtual network peering is non-transitive by default. If VNet A peers with VNet B, and VNet B peers with VNet C, VNet A cannot reach VNet C through VNet B. To achieve transitive routing you need either a hub VNet with a Network Virtual Appliance, or Azure Virtual WAN.

---

## Task 5: Configure an Azure Load Balancer

Azure Load Balancer operates at Layer 4 (TCP/UDP) and distributes inbound traffic
across backend virtual machines. The Standard SKU provides zone redundancy, SLA
guarantees, and health probes — use it for all production workloads.

In this task you also create the `AppVnet` and provision two backend VMs required
for Tasks 5 and 6. To keep within the lab timing, deploy the VMs using Cloud Shell.

### Provision the lab infrastructure

1. Open **Cloud Shell** (Bash) from the top-right of the portal.

2. Run the following commands to create the `AppVnet` and two backend VMs. Replace `<password>` with a complex password of your choice.

   ```bash
   # Create AppVnet with two backend subnets
   az network vnet create \
     --resource-group RG-Lab2 \
     --name AppVnet \
     --address-prefixes 10.60.0.0/16 \
     --subnet-name BackendSubnet1 \
     --subnet-prefixes 10.60.1.0/24

   az network vnet subnet create \
     --resource-group RG-Lab2 \
     --vnet-name AppVnet \
     --name BackendSubnet2 \
     --address-prefixes 10.60.2.0/24

   # Create an NSG to allow HTTP traffic to the backend VMs
   az network nsg create --resource-group RG-Lab2 --name app-nsg
   az network nsg rule create \
     --resource-group RG-Lab2 --nsg-name app-nsg \
     --name AllowHTTP --priority 100 \
     --protocol Tcp --destination-port-ranges 80 --access Allow

   # Deploy vm0 into BackendSubnet1
   az vm create \
     --resource-group RG-Lab2 \
     --name az104-06-vm0 \
     --image Ubuntu2204 \
     --vnet-name AppVnet \
     --subnet BackendSubnet1 \
     --nsg app-nsg \
     --admin-username azureuser \
     --admin-password <password> \
     --custom-data '#!/bin/bash
   apt-get update && apt-get install -y nginx
   echo "<h1>Hello World from az104-06-vm0</h1>" > /var/www/html/index.html
   mkdir -p /var/www/html/image /var/www/html/video
   echo "<h1>Image server - vm0</h1>" > /var/www/html/image/index.html
   echo "<h1>Video server - vm0</h1>" > /var/www/html/video/index.html
   systemctl enable nginx && systemctl start nginx' \
     --no-wait

   # Deploy vm1 into BackendSubnet2
   az vm create \
     --resource-group RG-Lab2 \
     --name az104-06-vm1 \
     --image Ubuntu2204 \
     --vnet-name AppVnet \
     --subnet BackendSubnet2 \
     --nsg app-nsg \
     --admin-username azureuser \
     --admin-password <password> \
     --custom-data '#!/bin/bash
   apt-get update && apt-get install -y nginx
   echo "<h1>Hello World from az104-06-vm1</h1>" > /var/www/html/index.html
   mkdir -p /var/www/html/image /var/www/html/video
   echo "<h1>Image server - vm1</h1>" > /var/www/html/image/index.html
   echo "<h1>Video server - vm1</h1>" > /var/www/html/video/index.html
   systemctl enable nginx && systemctl start nginx' \
     --no-wait
   ```

3. While the VMs are deploying (approximately 5 minutes), proceed to create the Load Balancer.

### Create the Load Balancer

4. In the Azure portal, search for and select **Load balancers**, then **+ Create**.

5. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab2** |
   | Name | `az104-lb` |
   | Region | **Australia East** |
   | SKU | **Standard** |
   | Type | **Public** |
   | Tier | **Regional** |

6. Select **Next: Frontend IP configuration**, then **+ Add a frontend IP configuration**:

   | Setting | Value |
   | --- | --- |
   | Name | `az104-fe` |
   | IP type | **IP address** |
   | Public IP address | Select **Create new** |
   | New public IP name | `az104-lbpip` |
   | SKU | **Standard** |
   | Assignment | **Static** |
   | Routing preference | **Microsoft network** |

   Select **Save**, then **Next: Backend pools**.

7. Select **+ Add a backend pool**:

   | Setting | Value |
   | --- | --- |
   | Name | `az104-be` |
   | Virtual network | **AppVnet (RG-Lab2)** |
   | Backend pool configuration | **NIC** |

   Select **+ Add** and add both `az104-06-vm0` and `az104-06-vm1`. Select **Add**, then **Save**.

8. Leave **Inbound rules** and remaining tabs at defaults. Select **Review + create**, then **Create**. Wait for deployment.

### Add a load balancing rule

9. Navigate to the deployed `az104-lb` resource. In the **Settings** blade, select **Load balancing rules**, then **+ Add**:

   | Setting | Value |
   | --- | --- |
   | Name | `az104-lbrule` |
   | IP Version | **IPv4** |
   | Frontend IP address | **az104-fe** |
   | Backend pool | **az104-be** |
   | Protocol | **TCP** |
   | Port | `80` |
   | Backend port | `80` |
   | Health probe | **Create new** → name `az104-hp`, Protocol `TCP`, Port `80`, Interval `5` |
   | Session persistence | **None** |
   | Idle timeout (minutes) | `4` |
   | Enable TCP reset | **Disabled** |
   | Enable Floating IP | **Disabled** |

   Select **Save**.

### Test the Load Balancer

10. On the **Frontend IP configuration** blade of `az104-lb`, copy the public IP address.

11. Open a browser and navigate to `http://<frontend-ip-address>`. You should see **Hello World from az104-06-vm0** or **az104-06-vm1**.

12. Refresh the page several times (or use an InPrivate window). Confirm that responses alternate between both virtual machines — this demonstrates round-robin load distribution.

**Key point:** The Standard Load Balancer is zone-redundant and SLA-backed. The Basic SKU is for dev/test only. Health probes monitor backend instance health — if a probe fails, the Load Balancer stops sending traffic to that instance until it recovers. Session persistence (sticky sessions) can be enabled if your application requires a client to always reach the same backend.

---

## Task 6: Configure an Application Gateway with Path-Based Routing

Azure Application Gateway is a Layer 7 (HTTP/HTTPS) load balancer. Unlike the
standard Load Balancer which routes based on IP and port, Application Gateway can
route based on the URL path — directing `/image/*` traffic to an image-serving backend
pool and `/video/*` traffic to a video-serving backend pool.

### Add the Application Gateway subnet

The Application Gateway requires a dedicated subnet of /27 or larger.

1. Navigate to **AppVnet → Subnets**, then **+ Subnet**.

   | Setting | Value |
   | --- | --- |
   | Name | `AppGwSubnet` |
   | Starting address | `10.60.3.224` |
   | Size | `/27` |

2. Select **Add**.

### Create the Application Gateway

3. Search for and select **Application gateways**, then **+ Create**.

4. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab2** |
   | Application gateway name | `az104-appgw` |
   | Region | **Australia East** |
   | Tier | **Standard V2** |
   | Enable autoscaling | **No** |
   | Instance count | `2` |
   | HTTP2 | **Disabled** |
   | Virtual network | **AppVnet** |
   | Subnet | **AppGwSubnet (10.60.3.224/27)** |

5. Select **Next: Frontends**:

   | Setting | Value |
   | --- | --- |
   | Frontend IP address type | **Public** |
   | Public IP address | **Add new** → name `az104-gwpip` |

6. Select **Next: Backends**. Add three backend pools:

   **Default pool (both VMs):**

   | Setting | Value |
   | --- | --- |
   | Name | `az104-appgwbe` |
   | Add backend pool without targets | **No** |
   | Virtual machine | **az104-06-vm0 NIC** |
   | Virtual machine | **az104-06-vm1 NIC** |

   Select **Add**, then **+ Add a backend pool** again.

   **Images pool (vm0 only):**

   | Setting | Value |
   | --- | --- |
   | Name | `az104-imagebe` |
   | Virtual machine | **az104-06-vm0 NIC** |

   Select **Add**, then **+ Add a backend pool** again.

   **Videos pool (vm1 only):**

   | Setting | Value |
   | --- | --- |
   | Name | `az104-videobe` |
   | Virtual machine | **az104-06-vm1 NIC** |

   Select **Add**.

7. Select **Next: Configuration**, then **+ Add a routing rule**:

   | Setting | Value |
   | --- | --- |
   | Rule name | `az104-gwrule` |
   | Priority | `10` |
   | **Listener tab** | |
   | Listener name | `az104-listener` |
   | Frontend IP | **Public IPv4** |
   | Protocol | **HTTP** |
   | Port | `80` |
   | Listener type | **Basic** |

8. On the **Backend targets** tab:

   | Setting | Value |
   | --- | --- |
   | Target type | **Backend pool** |
   | Backend target | `az104-appgwbe` |
   | Backend settings | **Add new** → name `az104-http`, Protocol `HTTP`, Port `80` → **Add** |

9. In the **Path-based routing** section, select **Add multiple targets to create a path-based rule** and add two path rules:

   **Image routing rule:**

   | Setting | Value |
   | --- | --- |
   | Path | `/image/*` |
   | Target name | `images` |
   | Backend settings | **az104-http** |
   | Backend target | `az104-imagebe` |

   Select **Add**, then add the second:

   **Video routing rule:**

   | Setting | Value |
   | --- | --- |
   | Path | `/video/*` |
   | Target name | `videos` |
   | Backend settings | **az104-http** |
   | Backend target | `az104-videobe` |

   Select **Add**.

10. Select **Next: Tags**, then **Next: Review + create**, then **Create**.

    > **Note:** The Application Gateway takes 5–10 minutes to deploy.

### Test path-based routing

11. Once deployed, navigate to **az104-appgw → Overview** and copy the **Frontend public IP address**.

12. Open a browser and navigate to `http://<frontend-ip>/image/`. Confirm you see the image server response from **vm0**.

13. Open a new browser window and navigate to `http://<frontend-ip>/video/`. Confirm you see the video server response from **vm1**.

14. Navigate to `http://<frontend-ip>/` (no path). Confirm the default pool responds — either vm0 or vm1.

**What just happened:** The Application Gateway inspected the URL path of each request and forwarded it to the appropriate backend pool — `/image/*` went to vm0, `/video/*` went to vm1, and unmatched paths went to the default backend pool. This is Layer 7 routing: the gateway understands HTTP, not just TCP ports.

**Key point:** Application Gateway (Standard V2) supports SSL/TLS termination, WAF (upgraded to WAF V2 tier), cookie-based session affinity, connection draining, and custom health probes. Use Application Gateway when you need URL-based or host-header-based routing. Use Load Balancer when you only need TCP/UDP distribution.

---

## Cleanup

**Remove all lab resources to avoid ongoing charges.**

1. In the Azure portal, navigate to **RG-Lab2**.

2. Select **Delete resource group**.

3. Copy and paste the resource group name `RG-Lab2` to confirm, then select **Delete**.

   > Alternatively, from Cloud Shell:
   > ```bash
   > az group delete --name RG-Lab2 --yes --no-wait
   > ```

---

## Key Takeaways

- **Virtual networks and subnets** provide the foundation for all Azure networking.
  Avoid overlapping IP address ranges across VNets and on-premises networks from the
  outset — fixing this later requires tearing down and re-creating resources.

- **NSGs** filter traffic at the subnet or NIC level using allow/deny rules evaluated
  in priority order. **ASGs** group VMs logically so NSG rules can reference a set of
  machines by name instead of by IP address, reducing management overhead as the
  environment scales.

- **Azure DNS** provides both public and private DNS hosting. Public zones are
  internet-accessible; private zones are scoped to specific virtual networks. Use
  private DNS zones to give Azure resources human-readable names without exposing
  them publicly.

- **Virtual network peering** connects two VNets directly over the Microsoft backbone
  with low latency. Peering is non-transitive — hub-and-spoke topologies require
  either a Network Virtual Appliance or Azure Virtual WAN for transitive routing.

- **Azure Load Balancer (Standard)** distributes TCP/UDP traffic at Layer 4 using
  health probes and configurable rules. Use it for VM-level traffic distribution
  where the application is not HTTP/HTTPS.

- **Azure Application Gateway** routes HTTP/HTTPS traffic at Layer 7 using URL path
  and host-header rules. Use it when requests must be directed to different backends
  based on content type, API version, tenant, or any other URL attribute.

---

## Resources

- [Azure Virtual Networks documentation](https://learn.microsoft.com/en-us/azure/virtual-network/)
- [Network Security Groups overview](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)
- [Application Security Groups](https://learn.microsoft.com/en-us/azure/virtual-network/application-security-groups)
- [Azure DNS documentation](https://learn.microsoft.com/en-us/azure/dns/)
- [Virtual network peering](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview)
- [Azure Load Balancer documentation](https://learn.microsoft.com/en-us/azure/load-balancer/)
- [Azure Application Gateway documentation](https://learn.microsoft.com/en-us/azure/application-gateway/)
- [Azure Network Watcher](https://learn.microsoft.com/en-us/azure/network-watcher/)

---
lab:
  title: 'Lab Day 5: Kubernetes – Azure Kubernetes Service (AKS)'
  module: 'Advanced Azure Bootcamp – Day 5'
---

# Lab Day 5 – Kubernetes: Azure Kubernetes Service (AKS)

## Lab Introduction

In this lab you deploy and operate a workload on Azure Kubernetes Service (AKS). You
will work through the core operational tasks a platform engineer performs when running
containerised applications on a managed Kubernetes cluster. You will work through the
following tasks:

- Provision an AKS cluster and connect to it with `kubectl`
- Explore cluster structure: nodes, namespaces, and system workloads
- Deploy an application using a Kubernetes Deployment and inspect Pod lifecycle
- Expose the application externally using a Kubernetes Service of type LoadBalancer
- Perform a rolling update and observe zero-downtime deployment behaviour
- Attach persistent storage using a PersistentVolumeClaim backed by Azure Disks
- Apply operational patterns: resource quotas, Horizontal Pod Autoscaler, and namespace isolation

## Pre-requisites

- An Azure subscription
- A resource group named **RG-Lab5**.
  Note: You will be assigned your own unique resource group name in the sandbox environment to avoid conflicts with other participants. Create a resource group in the same region you will deploy resources to (e.g., Australia East). Name the resource group RG-Lab5-yourname. Note the resource group name for use in all lab tasks.
- Owner rights at the resource group level
- Azure Cloud Shell (browser-based) — no local tooling installation required

## Estimated Timing: 90 minutes

## Lab Scenario

Your organisation is modernising a retail order management platform. The development
team has already containerised two microservices:

- A **front-end web application** (`nginx`-based) that serves the order entry UI
- A **stateful order processor** that writes order data to a persistent volume

Your task is to deploy both workloads onto a shared AKS cluster, expose the front-end
to the internet, demonstrate a zero-downtime update, and attach durable storage to the
order processor. You will then apply operational guardrails — resource quotas, autoscaling,
and namespace separation — that reflect production-ready cluster practices.

## Architecture Overview

```
RG-Lab5 (Resource Group)
└── AKS Cluster (aks-lab5-yourname)
    ├── System Node Pool (Standard_D2s_v3 × 1)  — kube-system workloads
    └── User Node Pool (Standard_D2s_v3 × 2)    — application workloads
        │
        ├── Namespace: frontend
        │   ├── Deployment: web-frontend  (nginx)
        │   │   └── ReplicaSet → 2 × Pod
        │   └── Service: web-svc  (type: LoadBalancer → Azure Public IP)
        │
        ├── Namespace: backend
        │   ├── Deployment: order-processor
        │   │   └── Pod → /data mount
        │   ├── PersistentVolumeClaim: orders-pvc
        │   │   └── PersistentVolume — Azure Managed Disk (Standard_LRS)
        │   └── Service: order-svc  (type: ClusterIP — internal only)
        │
        └── Namespace: kube-system  (AKS-managed)
            ├── CoreDNS
            ├── metrics-server
            └── cloud-controller-manager
```

## Job Skills

- Task 1: Provision an AKS cluster and connect with kubectl
- Task 2: Explore cluster structure — nodes, namespaces, and system workloads
- Task 3: Deploy an application using a Kubernetes Deployment
- Task 4: Expose the application with a LoadBalancer Service
- Task 5: Perform a rolling update and observe zero-downtime behaviour
- Task 6: Attach persistent storage with a PersistentVolumeClaim
- Task 7: Apply operational patterns — resource limits, autoscaling, and namespace isolation

---

## Task 1: Provision an AKS Cluster and Connect with kubectl

Azure Kubernetes Service (AKS) is a managed Kubernetes offering. Azure provisions
and manages the Kubernetes control plane (API server, etcd, scheduler) at no additional
cost; you pay only for the agent nodes that run your workloads.

### Create the AKS cluster

1. In the Azure portal, search for and select **Kubernetes services**.

2. Select **+ Create** → **Create a Kubernetes cluster** and configure the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab5** |
   | Kubernetes cluster name | `aks-lab5-yourname` (replace *yourname* to keep it unique) |
   | Region | **Australia East** |
   | Availability zones | **None** (cost saving for lab) |
   | AKS Pricing tier | **Free** |
   | Kubernetes version | Accept the default (latest stable) |
   | Automatic upgrade | **Enabled with patch (recommended)** — leave default |
   | Authentication and Authorization | **Local accounts with Kubernetes RBAC** |

3. Select the **Node pools** tab. You will see two node pools: **agentpool** (System) and **userpool** (User).

   Select **agentpool** to edit it and configure:

   | Setting | Value |
   | --- | --- |
   | Node size | **Standard_D2s_v3** (select **Choose a size**, search for `D2s_v3`) |
   | Scale method | **Manual** |
   | Node count | `1` |

   Select **Update** to save.

   Then select **userpool** and configure:

   | Setting | Value |
   | --- | --- |
   | Node size | **Standard_D2s_v3** |
   | Scale method | **Manual** |
   | Node count | `2` |

   Select **Update** to save the node pool changes.

   > **Why these counts?** The **agentpool** (System) runs only AKS system components
   > such as CoreDNS, Cilium, and the CSI drivers — one node is sufficient for a lab.
   > The **userpool** (User) is where your application Pods land. Two nodes are needed
   > so the Kubernetes scheduler can spread the two `web-frontend` replicas across
   > separate nodes, demonstrating fault tolerance: if one user node goes down, the
   > other replica on the remaining node keeps the application available.

4. Select the **Networking** tab and review:

   | Setting | Value |
   | --- | --- |
   | Network configuration | **Azure CNI Overlay** (default — leave as-is) |
   | DNS name prefix | Accept the default |
   | Enable Cilium dataplane and network policy engine | Leave as default |

   > **Azure CNI Overlay vs Azure CNI Node Subnet:** With Azure CNI Overlay, Pods
   > receive IPs from a private overlay address space separate from the VNet — this
   > is more scalable and is now the default. Azure CNI Node Subnet (previously just
   > called "Azure CNI") assigns each Pod a VNet IP directly, which is required when
   > Pods must be reachable by other VNet resources by IP. Azure CNI Overlay is
   > sufficient for this lab.

5. Select the **Monitoring** tab and **disable all monitoring options** to keep the
   lab resource count low and avoid extra charges:

   | Setting | Value |
   | --- | --- |
   | Enable Container Logs | **Off** (disables Container Insights) |
   | Enable Prometheus metrics | **Off** (otherwise Azure creates a Monitor workspace + Grafana instance) |
   | Enable Managed Grafana | **Off** |
   | Enable recommended alert rules | **Off** (uncheck the checkbox) |

   > **Why this matters:** Leaving these options enabled causes Azure to automatically
   > provision extra resources — an Azure Monitor workspace, Azure Managed Grafana,
   > Prometheus rule groups, data collection rules, metric alert rules, and email
   > alert notifications. For this lab you only need the Kubernetes service itself.

6. Select **Review + Create**, then **Create**. Provisioning takes 4–6 minutes.

### Connect to the cluster with kubectl

1. Once deployment completes, navigate to the `aks-lab5-yourname` resource.

2. Select **Connect** from the top toolbar. The portal displays the commands needed
   to configure `kubectl` credentials.

3. Open **Azure Cloud Shell** (the `>_` icon in the portal toolbar). Select **Bash**.
   If prompted, create a storage account to persist Cloud Shell files.

4. In Cloud Shell, run the following commands (substitute your resource group and
   cluster name):

   ```bash
   az aks get-credentials \
     --resource-group RG-Lab5 \
     --name aks-lab5-yourname \
     --overwrite-existing
   ```

   This writes the cluster's `kubeconfig` to `~/.kube/config` in your Cloud Shell
   session, configuring `kubectl` to talk to this cluster.

5. Verify the connection:

   ```bash
   kubectl get nodes
   ```

   Expected output (three nodes in the **Ready** state):

   ```
   NAME                                STATUS   ROLES    AGE     VERSION
   aks-agentpool-30245557-vmss000000   Ready    <none>   5m10s   v1.33.7
   aks-userpool-30245557-vmss000000    Ready    <none>   5m6s    v1.33.7
   aks-userpool-30245557-vmss000001    Ready    <none>   5m24s   v1.33.7
   ```

   > If you see **NotReady**, wait 60 seconds and re-run the command. Nodes may still
   > be completing their bootstrap sequence.

---

## Task 2: Explore Cluster Structure — Nodes, Namespaces, and System Workloads

Before deploying your own workloads, it is important to understand what AKS has
already provisioned inside the cluster.

### Inspect nodes

1. Retrieve detailed information about one node:

   ```bash
   kubectl describe node $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
   ```

   Scroll through the output and note:
   - **Allocatable** resources — the CPU and memory available to schedule Pods on
     this node (less than Capacity because the OS and AKS system daemons consume some)
   - **Conditions** — the standard Kubernetes conditions (Ready, MemoryPressure,
     DiskPressure, PIDPressure) plus several AKS-specific conditions added by the
     node-problem-detector (e.g. KernelDeadlock, FrequentContainerdRestart,
     NetworkUnavailable). This is normal — all should show `False` or `True` for Ready.
   - **System Info** — kernel version, container runtime (containerd), kubelet version.
     Note: **Kube-Proxy Version** will be blank on clusters using the Cilium dataplane,
     because Cilium replaces kube-proxy for service routing.

### Explore namespaces

2. List all namespaces in the cluster:

   ```bash
   kubectl get namespaces
   ```

   AKS provisions several namespaces by default:

   | Namespace | Purpose |
   | --- | --- |
   | `default` | Where resources are created when no namespace is specified |
   | `kube-system` | Kubernetes and AKS system components |
   | `kube-public` | Metadata readable by unauthenticated clients |
   | `kube-node-lease` | Node heartbeat lease objects (internal use) |

### Review system Pods

3. List the Pods running in the `kube-system` namespace:

   ```bash
   kubectl get pods -n kube-system
   ```

   You will see AKS-managed system components including:

   | Component | Role |
   | --- | --- |
   | `coredns-*` | In-cluster DNS — resolves Service names to ClusterIP addresses |
   | `coredns-autoscaler-*` | Adjusts CoreDNS replica count as the cluster grows |
   | `metrics-server-*` | Exposes resource metrics (CPU/memory) used by HPA and `kubectl top` |
   | `cloud-node-manager-*` | Integrates nodes with Azure (attaches Managed Disks, assigns LoadBalancer IPs) |
   | `cilium-*` / `cilium-operator-*` | eBPF-based networking and service routing — replaces kube-proxy on this cluster |
   | `azure-cns-*` | Azure Container Networking Service — manages pod IP allocation for CNI Overlay |
   | `csi-azuredisk-node-*` | CSI driver that mounts Azure Managed Disks into Pods |
   | `csi-azurefile-node-*` | CSI driver that mounts Azure Files shares into Pods |
   | `konnectivity-agent-*` | Secure tunnel between the AKS control plane and agent nodes |
   | `azure-policy-*` | Enforces Azure Policy rules inside the cluster |

   > **Note:** You will see more Pods than listed above — AKS deploys additional
   > components such as the workload-identity webhook (`azure-wi-webhook-*`) and the
   > image-cleaner (`eraser-*`). All Pods should show **Running** status. A small
   > number of recent restarts (1–2) on webhook Pods during startup is normal.

   > **Important:** Never delete or modify resources in `kube-system`. This namespace
   > is managed entirely by AKS. Changes may break cluster networking, DNS resolution,
   > or storage attachment.

### Create application namespaces

4. Create two namespaces for your workloads:

   ```bash
   kubectl create namespace frontend
   kubectl create namespace backend
   ```

5. Confirm they appear in the namespace list:

   ```bash
   kubectl get namespaces
   ```

---

## Task 3: Deploy an Application Using a Kubernetes Deployment

A **Deployment** is the standard Kubernetes object for managing stateless application
workloads. It declares the desired state — which container image to run, how many
replicas, resource limits — and the Deployment controller continuously reconciles
actual state to match.

### Create the front-end ConfigMap

A **ConfigMap** stores non-sensitive configuration data as key-value pairs. Here you
use one to inject a custom `index.html` into the nginx container via a volume mount,
so the page reflects the retail scenario rather than the generic nginx placeholder.

1. In Cloud Shell, create the ConfigMap manifest:

   ```bash
   cat <<'EOF' > web-frontend-configmap.yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: web-frontend-html
     namespace: frontend
   data:
     index.html: |
       <!DOCTYPE html>
       <html lang="en">
       <head>
         <meta charset="UTF-8" />
         <title>Retail Order Entry — MYOB</title>
         <style>
           body { font-family: Arial, sans-serif; background: #f4f6f8; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; }
           .card { background: white; border-radius: 8px; padding: 48px 64px; box-shadow: 0 2px 12px rgba(0,0,0,0.1); text-align: center; max-width: 480px; }
           h1 { color: #0078d4; margin-bottom: 8px; }
           p  { color: #555; line-height: 1.6; }
           .badge { display: inline-block; margin-top: 24px; padding: 6px 16px; background: #e6f2fb; color: #0078d4; border-radius: 4px; font-size: 0.85rem; }
         </style>
       </head>
       <body>
         <div class="card">
           <h1>Retail Order Entry</h1>
           <p>Welcome to the MYOB order management portal.<br/>
              This front-end is served by <strong>nginx</strong> running inside an
              AKS Pod, exposed via an Azure Load Balancer.</p>
           <div class="badge">Azure Kubernetes Service — Lab Day 5</div>
         </div>
       </body>
       </html>
   EOF
   ```

   > **What this manifest does:** It creates a ConfigMap named `web-frontend-html` in
   > the `frontend` namespace with a single key, `index.html`, whose value is the full
   > HTML page. Later, the Deployment mounts this key as a file at
   > `/usr/share/nginx/html/index.html` inside the nginx container, replacing the
   > default nginx placeholder page with the custom Retail Order Entry page.

2. Apply the ConfigMap:

   ```bash
   kubectl apply -f web-frontend-configmap.yaml
   ```

   Confirm it was created:

   ```bash
   kubectl get configmap web-frontend-html -n frontend
   ```

### Create the front-end Deployment

3. Create a manifest file for the front-end Deployment. Note the `volumeMounts` and
   `volumes` sections that mount the ConfigMap's `index.html` over nginx's default
   web root:

   ```bash
   cat <<'EOF' > web-frontend-deployment.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: web-frontend
     namespace: frontend
     labels:
       app: web-frontend
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: web-frontend
     template:
       metadata:
         labels:
           app: web-frontend
       spec:
         containers:
           - name: nginx
             image: nginx:1.25
             ports:
               - containerPort: 80
             resources:
               requests:
                 cpu: "100m"
                 memory: "64Mi"
               limits:
                 cpu: "250m"
                 memory: "128Mi"
             volumeMounts:
               - name: html-content
                 mountPath: /usr/share/nginx/html/index.html
                 subPath: index.html
         volumes:
           - name: html-content
             configMap:
               name: web-frontend-html
   EOF
   ```

   > **What this manifest does:** It creates a Deployment named `web-frontend` in the
   > `frontend` namespace with `replicas: 2` — meaning Kubernetes will always keep two
   > Pods running. The `selector.matchLabels` ties the Deployment to Pods labelled
   > `app: web-frontend`. Each Pod runs the `nginx:1.25` container with CPU and memory
   > requests/limits set, and mounts the `web-frontend-html` ConfigMap as a file at
   > `/usr/share/nginx/html/index.html` using `volumeMounts` + `subPath`, so nginx
   > serves the custom page instead of its default placeholder.

4. Apply the manifest:

   ```bash
   kubectl apply -f web-frontend-deployment.yaml
   ```

5. Watch the Pods come up (press `Ctrl+C` to stop watching):

   ```bash
   kubectl get pods -n frontend --watch
   ```

   Within 30–60 seconds both Pods should reach the **Running** state.

### Inspect the Deployment and Pods

6. Describe the Deployment to review its status:

   ```bash
   kubectl describe deployment web-frontend -n frontend
   ```

   Note the **Replicas** line: `2 desired | 2 updated | 2 total | 2 available`.

7. List the Pods with additional detail:

   ```bash
   kubectl get pods -n frontend -o wide
   ```

   The `-o wide` flag shows which **node** each Pod is running on. With two nodes
   and two Pods, the scheduler will typically spread them across both nodes for
   fault tolerance.

8. View the logs for one of the Pods (substitute the Pod name from the previous output):

   ```bash
   kubectl logs -n frontend <pod-name>
   ```

   You will see the nginx access log. The Pod has not received any traffic yet, so
   the log will be near-empty.

### Simulate a Pod failure

9. Delete one of the Pods to observe self-healing:

   ```bash
   kubectl delete pod -n frontend <pod-name>
   ```

10. Immediately run:

    ```bash
    kubectl get pods -n frontend
    ```

    The deleted Pod disappears and the Deployment controller immediately schedules a
    replacement. Within a few seconds you will see a new Pod in the **ContainerCreating**
    → **Running** state. The Deployment always reconciles back to `replicas: 2`.

    > **Key concept:** You never manage individual Pods directly in production.
    > You manage Deployments. The Deployment controller manages Pods on your behalf
    > and replaces any that fail or are deleted.

---

## Task 4: Expose the Application with a LoadBalancer Service

Pods have ephemeral IP addresses that change each time a Pod is replaced. A
**Service** provides a stable virtual IP (ClusterIP) and DNS name that load-balances
traffic across all healthy Pods matching its selector. A Service of type
**LoadBalancer** additionally provisions an Azure Public Load Balancer with a public IP.

### Create the LoadBalancer Service

1. Create the Service manifest:

   ```bash
   cat <<'EOF' > web-frontend-service.yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: web-svc
     namespace: frontend
   spec:
     type: LoadBalancer
     selector:
       app: web-frontend
     ports:
       - protocol: TCP
         port: 80
         targetPort: 80
   EOF
   ```

   > **What this manifest does:** It creates a Service named `web-svc` in the `frontend`
   > namespace of type `LoadBalancer`. The `selector` matches any Pod labelled
   > `app: web-frontend` — the same label the Deployment applies to its Pods. Kubernetes
   > uses this selector to build an Endpoints list and load-balance incoming traffic across
   > all matching Pods. The `port: 80` is the port the Azure Load Balancer listens on
   > externally; `targetPort: 80` is the port inside each Pod (where nginx is listening).
   > Because the type is `LoadBalancer`, AKS instructs the Azure cloud-controller-manager
   > to provision a public Azure Load Balancer and assign it a public IP — which becomes
   > the Service's `EXTERNAL-IP`.

2. Apply the manifest:

   ```bash
   kubectl apply -f web-frontend-service.yaml
   ```

3. Watch until an external IP is assigned (this triggers Azure to provision a Public
   Load Balancer — takes 60–90 seconds):

   ```bash
   kubectl get service web-svc -n frontend --watch
   ```

   When the **EXTERNAL-IP** column changes from `<pending>` to an IP address, press
   `Ctrl+C`.

4. Copy the external IP address and open it in a browser tab. You should see the
   **Retail Order Entry** page — the custom HTML injected via the ConfigMap, served
   by nginx, and publicly accessible through the Azure Load Balancer. This confirms
   that the ConfigMap volume mount replaced nginx's default `index.html` correctly.

### Understand Service DNS

5. Examine the Service details:

   ```bash
   kubectl describe service web-svc -n frontend
   ```

   Note the following fields in the output:
   - **`IP`** — the Service's ClusterIP. This is the stable internal IP that other
     Pods inside the cluster use to reach this Service.
   - **`LoadBalancer Ingress`** — the public IP assigned by Azure (shown as `(VIP)`).
     This is what you opened in the browser in step 4.
   - **`NodePort`** — a port automatically allocated on every node. Kubernetes uses
     this internally to route Load Balancer traffic from the Azure LB through each
     node to the Pod. You do not need to use or open this port directly.
   - **`Endpoints`** — the actual Pod IPs currently registered behind the Service
     (one per running `web-frontend` Pod). If a Pod is unhealthy, its IP is removed
     from this list automatically.

   The Service is also reachable inside the cluster by DNS name:

   ```
   web-svc.frontend.svc.cluster.local
   ```

   CoreDNS resolves this name to the ClusterIP automatically. Any Pod in any
   namespace can reach the Service using this FQDN without knowing the IP address.

**Service type comparison:**

| Type | Accessibility | Azure resource provisioned | Use case |
| --- | --- | --- | --- |
| **ClusterIP** | Internal only (within cluster) | None | Service-to-service communication |
| **NodePort** | Via node IP + port | None | Dev/test; direct node access |
| **LoadBalancer** | Public internet or internal VNet | Azure Public (or Internal) Load Balancer | External-facing applications |
| **ExternalName** | DNS alias to external FQDN | None | Routing to external services by DNS |

---

## Task 5: Perform a Rolling Update and Observe Zero-Downtime Behaviour

A **rolling update** replaces Pods one at a time (or in small batches) so that the
application remains available throughout the update. This is the default update
strategy for Kubernetes Deployments.

### Trigger a rolling update

1. Update the Deployment to use nginx version 1.27:

   ```bash
   kubectl set image deployment/web-frontend \
     nginx=nginx:1.27 \
     -n frontend
   ```

2. Immediately watch the rollout progress:

   ```bash
   kubectl rollout status deployment/web-frontend -n frontend
   ```

   Kubernetes replaces one Pod at a time. The old Pod is only terminated after the
   new Pod has passed its readiness check. This ensures there is always at least
   one Pod serving traffic during the update.

3. Once the rollout completes, confirm both Pods are running the new image:

   ```bash
   kubectl get pods -n frontend -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
   ```

   Both Pods should show `nginx:1.27`.

### Review rollout history

4. View the rollout history:

   ```bash
   kubectl rollout history deployment/web-frontend -n frontend
   ```

   Kubernetes records each revision. You will see **REVISION 1** (nginx:1.25) and
   **REVISION 2** (nginx:1.27).

### Roll back to the previous version

5. Roll back to the previous revision:

   ```bash
   kubectl rollout undo deployment/web-frontend -n frontend
   ```

6. Confirm the Pods are running nginx:1.25 again:

   ```bash
   kubectl get pods -n frontend -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
   ```

   > **Key concept:** `kubectl rollout undo` does not require you to know the previous
   > image tag — Kubernetes stores the previous Deployment spec in the ReplicaSet
   > history and uses it directly. For this reason, avoid deleting old ReplicaSets
   > manually; they are the mechanism that enables rollback.

---

## Task 6: Attach Persistent Storage with a PersistentVolumeClaim

Containers are ephemeral — any data written to a container's file system is lost
when the container restarts. A **PersistentVolume (PV)** is a piece of storage provisioned
independently of any individual Pod. A **PersistentVolumeClaim (PVC)** is a request for
storage that Kubernetes binds to a PV. When a Pod mounts a PVC, the data persists
across Pod restarts and rescheduling.

In AKS, the default StorageClass provisions **Azure Managed Disks** automatically
when a PVC is created.

### Review available StorageClasses

1. List the StorageClasses available in the cluster:

   ```bash
   kubectl get storageclasses
   ```

   Note the StorageClasses provided by AKS:

   | StorageClass | Disk type | Reclaim policy | Use case |
   | --- | --- | --- | --- |
   | `managed-csi` | Azure Standard SSD | Delete | General-purpose workloads (default) |
   | `managed-csi-premium` | Azure Premium SSD | Delete | Latency-sensitive, high-IOPS workloads |
   | `azurefile-csi` | Azure Files (SMB) | Delete | ReadWriteMany — multiple Pods sharing one volume |
   | `azurefile-csi-premium` | Azure Files Premium | Delete | High-throughput shared file workloads |

   > **ReadWriteOnce vs ReadWriteMany:** Azure Managed Disks can only be attached
   > to **one node** at a time (ReadWriteOnce). If your workload needs multiple Pods
   > on different nodes to write to the same volume simultaneously, use
   > `azurefile-csi` (ReadWriteMany via SMB).

### Create a PersistentVolumeClaim

2. Create the PVC manifest:

   ```bash
   cat <<'EOF' > orders-pvc.yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: orders-pvc
     namespace: backend
   spec:
     accessModes:
       - ReadWriteOnce
     storageClassName: managed-csi
     resources:
       requests:
         storage: 1Gi
   EOF
   ```

3. Apply it:

   ```bash
   kubectl apply -f orders-pvc.yaml
   ```

4. Check the PVC status:

   ```bash
   kubectl get pvc -n backend
   ```

   The status will initially show **Pending**. The Azure Managed Disk is provisioned
   when the first Pod that requests the PVC is scheduled (dynamic provisioning). It
   moves to **Bound** once the disk is created and attached.

### Deploy the order-processor with the PVC mounted

5. Create the order-processor Deployment:

   ```bash
   cat <<'EOF' > order-processor-deployment.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: order-processor
     namespace: backend
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: order-processor
     template:
       metadata:
         labels:
           app: order-processor
       spec:
         containers:
           - name: processor
             image: busybox:1.36
             command: ["/bin/sh", "-c", "while true; do echo \"order written at $(date)\" >> /data/orders.log; sleep 10; done"]
             resources:
               requests:
                 cpu: "50m"
                 memory: "32Mi"
               limits:
                 cpu: "100m"
                 memory: "64Mi"
             volumeMounts:
               - name: orders-storage
                 mountPath: /data
         volumes:
           - name: orders-storage
             persistentVolumeClaim:
               claimName: orders-pvc
   EOF
   ```

6. Apply the Deployment:

   ```bash
   kubectl apply -f order-processor-deployment.yaml
   ```

7. Create a ClusterIP Service so other Pods inside the cluster can reach the order-processor by DNS name:

   ```bash
   cat <<'EOF' > order-svc.yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: order-svc
     namespace: backend
   spec:
     type: ClusterIP
     selector:
       app: order-processor
     ports:
       - protocol: TCP
         port: 8080
         targetPort: 8080
   EOF
   kubectl apply -f order-svc.yaml
   ```

   Confirm the Service was created:

   ```bash
   kubectl get service order-svc -n backend
   ```

8. Wait for the Pod to reach Running status (the disk provisioning adds ~30 seconds):

   ```bash
   kubectl get pods -n backend --watch
   ```

   Press `Ctrl+C` once the Pod shows **Running**.

### Verify data persistence

9. Exec into the running Pod and inspect the log file:

   ```bash
   kubectl exec -n backend \
     $(kubectl get pod -n backend -l app=order-processor -o jsonpath='{.items[0].metadata.name}') \
     -- cat /data/orders.log
   ```

   You should see several timestamped entries written by the container's loop.

10. Delete the Pod to simulate a crash:

   ```bash
   kubectl delete pod -n backend \
     $(kubectl get pod -n backend -l app=order-processor -o jsonpath='{.items[0].metadata.name}')
   ```

11. Wait for the replacement Pod to start, then read the log file again:

    ```bash
    kubectl get pods -n backend --watch
    # Press Ctrl+C once Running, then:
    kubectl exec -n backend \
      $(kubectl get pod -n backend -l app=order-processor -o jsonpath='{.items[0].metadata.name}') \
      -- cat /data/orders.log
    ```

    The log file contains all entries from before the Pod was deleted. The Azure
    Managed Disk was detached from the old node and reattached to the new node
    automatically by the AKS cloud-node-manager. Data survived the Pod restart.

---

## Task 7: Apply Operational Patterns — Resource Limits, Autoscaling, and Namespace Isolation

This task demonstrates three production-readiness patterns that prevent runaway
workloads, enable automatic scale-out, and enforce team boundary isolation.

### Resource Quotas — prevent namespace resource exhaustion

A **ResourceQuota** caps the total CPU and memory that all Pods in a namespace can
consume. This prevents one team's workload from starving another's on a shared cluster.

1. Apply a ResourceQuota to the `frontend` namespace:

   ```bash
   cat <<'EOF' | kubectl apply -f -
   apiVersion: v1
   kind: ResourceQuota
   metadata:
     name: frontend-quota
     namespace: frontend
   spec:
     hard:
       requests.cpu: "500m"
       requests.memory: "256Mi"
       limits.cpu: "1"
       limits.memory: "512Mi"
       pods: "10"
   EOF
   ```

2. Describe the quota to see current usage vs limits:

   ```bash
   kubectl describe resourcequota frontend-quota -n frontend
   ```

   The output shows how much of the quota is consumed by the running `web-frontend`
   Pods. The two Pods combined request 200m CPU and 128Mi memory, well within the
   500m / 256Mi request quota.

### Horizontal Pod Autoscaler — scale out under load

A **HorizontalPodAutoscaler (HPA)** automatically adjusts the number of Pod replicas
based on observed CPU utilisation. It reads metrics from the `metrics-server` that
AKS deploys by default.

3. Apply an HPA to the `web-frontend` Deployment:

   ```bash
   kubectl autoscale deployment web-frontend \
     --namespace frontend \
     --min 2 \
     --max 5 \
     --cpu-percent 50
   ```

4. Check the HPA status:

   ```bash
   kubectl get hpa -n frontend
   ```

   The **TARGETS** column shows current CPU utilisation vs the 50% threshold.
   Under normal lab conditions the load is near zero, so the HPA will maintain
   the minimum of 2 replicas. In production, a load test would trigger scale-out.

   > **HPA and ResourceQuota interaction:** The HPA can only scale up to the point
   > where the ResourceQuota permits additional Pods. If the quota is exhausted, the
   > HPA will not be able to add replicas and will log a warning event. Always
   > size your quota to accommodate the maximum target replica count.

### Namespace isolation — verify inter-namespace network behaviour

5. Confirm that the `backend` namespace cannot be reached from the internet. The
   `order-svc` ClusterIP Service was never exposed as a LoadBalancer, so it has no
   public IP:

   ```bash
   kubectl get service -n backend
   ```

   The `order-svc` Service will show type **ClusterIP** with
   no external IP. This is intentional — the order processor is an internal service
   that should only be called by other Pods within the cluster, not exposed publicly.

6. (Optional) Demonstrate inter-namespace DNS resolution. Exec into the frontend Pod
   and resolve the backend Service by its FQDN:

   ```bash
   kubectl exec -n frontend \
     $(kubectl get pod -n frontend -l app=web-frontend -o jsonpath='{.items[0].metadata.name}') \
     -- nslookup order-svc.backend.svc.cluster.local
   ```

   > If `nslookup` is not available in the nginx container, install it with
   > `apt-get update && apt-get install -y dnsutils` first, or skip this step.

   CoreDNS resolves the FQDN to the ClusterIP of `order-svc`. Network-level
   isolation between namespaces requires **NetworkPolicy** objects (not covered
   in this lab), but DNS isolation out of the box demonstrates the logical
   separation that namespaces provide.

### Review cluster events

7. View recent events across all namespaces to understand what Kubernetes has been doing:

   ```bash
   kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -20
   ```

   Events record scheduler decisions, image pulls, volume attachments, HPA scale
   actions, and any warnings. This is the first place to look when a Pod fails to
   start or a rollout stalls.

**Operational best practices summary:**

| Practice | Kubernetes mechanism | Why it matters |
| --- | --- | --- |
| Always set resource requests and limits | `resources.requests` / `resources.limits` | Scheduler places Pods correctly; prevents CPU/memory starvation |
| Use Deployments, not bare Pods | `kind: Deployment` | Enables rolling updates, rollbacks, and self-healing |
| Separate workloads into namespaces | `kubectl create namespace` | Logical isolation; quota enforcement; RBAC scope |
| Cap namespace resource consumption | `ResourceQuota` | Prevents one team from starving others on shared clusters |
| Automate scale-out | `HorizontalPodAutoscaler` | Handles traffic spikes without manual intervention |
| Use PVCs for stateful data | `PersistentVolumeClaim` | Data survives Pod restarts and node failures |
| Monitor rollout status | `kubectl rollout status` | Catch failed rollouts before they affect all replicas |

---

## Cleanup

**Note:** AKS clusters and attached Managed Disks incur ongoing charges. Delete
resources promptly after the lab.

1. In the portal, navigate to **RG-Lab5**.

2. Select **Delete resource group**.

3. Copy and paste the resource group name to confirm deletion.

4. Select **Delete**.

   This removes the AKS cluster, all node VMs, the Managed Disk provisioned for
   the PVC, the Public Load Balancer and its IP, and the virtual network created
   by AKS, in a single operation.

---

## Key Takeaways

- **AKS manages the control plane** (API server, etcd, scheduler) at no charge.
  You pay only for the agent nodes and any Azure resources provisioned by your
  workloads (Load Balancers, Managed Disks).

- **Deployments are the standard unit of deployment** for stateless workloads.
  They provide self-healing (restart failed Pods), rolling updates (zero-downtime
  image changes), and rollback (revert to any previous revision via ReplicaSet history).

- **Services provide stable networking.** Pod IPs are ephemeral; Kubernetes Services
  provide a stable ClusterIP and DNS name. LoadBalancer Services extend this to
  the public internet by provisioning an Azure Load Balancer automatically.

- **PersistentVolumeClaims decouple storage from Pods.** Azure Managed Disks are
  provisioned dynamically via StorageClasses and survive Pod restarts and node failure.
  Use `azurefile-csi` when multiple Pods need simultaneous write access (ReadWriteMany).

- **Namespaces are the primary isolation boundary.** Combine namespaces with
  ResourceQuotas to prevent resource exhaustion on shared clusters, and with
  NetworkPolicy to enforce traffic rules between teams.

- **Kubernetes is declarative.** You describe the desired state in YAML manifests
  and `kubectl apply` them. The Kubernetes controllers continuously reconcile actual
  state to match your declared intent — without you needing to script individual
  steps.

- **ConfigMaps externalise configuration from container images.** Mounting a ConfigMap
  as a file allows you to change application content or configuration without rebuilding
  the image — just update the ConfigMap and roll out the Deployment.

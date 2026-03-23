---
lab:
  title: 'Lab Day 6: Event-Driven Architecture & Observability'
  module: 'Advanced Azure Bootcamp – Day 6'
---

# Lab Day 6 – Event-Driven Architecture & Observability

## Lab Introduction

In this lab you extend the retail order management platform from Day 5 by adding
event-driven integration and a production-grade observability stack. You will work
through the core patterns a platform engineer applies when building loosely coupled,
observable systems on Azure. You will work through the following tasks:

- Provision a Log Analytics Workspace and Application Insights instance as the
  observability foundation
- Deploy an Azure Service Bus namespace with a queue and a topic/subscription pair,
  and verify reliable message delivery
- Configure an Event Grid System Topic to capture Azure Blob Storage events and route
  them to a Service Bus queue
- Build a Logic App Standard workflow that reacts to Service Bus messages and writes
  enriched order records to Azure Table Storage
- Instrument a containerised application with the Application Insights SDK and explore
  distributed traces, dependency maps, and Live Metrics
- Configure diagnostic settings, metric alerts, and a shared Azure Dashboard for
  operational visibility

## Pre-requisites

- An Azure subscription
- A resource group named **RG-Lab6**.
  Note: You will be assigned your own unique resource group name in the sandbox
  environment to avoid conflicts with other participants. Create a resource group in
  the same region you will deploy resources to (e.g., Australia East). Name the
  resource group RG-Lab6-yourname. Note the resource group name for use in all
  lab tasks.
- Owner rights at the resource group level
- Azure Cloud Shell (browser-based) — no local tooling installation required

## Estimated Timing: 90 minutes

## Lab Scenario

The retail order management platform is now running on AKS (Day 5). The operations
team has identified two gaps:

1. **Integration gap:** When a customer submits an order, the front-end writes a JSON
   file to Blob Storage. Downstream services — an invoice generator and a warehouse
   picker — must be notified reliably and independently. A direct HTTP call is fragile;
   if a downstream service is temporarily unavailable, the order is lost.

2. **Visibility gap:** The engineering team has no insight into request latency,
   dependency failures, or error rates across the microservices. Incidents are discovered
   by customers, not by monitoring.

Your task is to build the event-driven backbone that decouples the order pipeline, and
to layer a full observability stack on top so the platform team can detect and diagnose
issues before they reach customers.

## Architecture Overview

```
RG-Lab6 (Resource Group)
│
├── Log Analytics Workspace (logs-lab6-yourname)
│   └── Application Insights (appinsights-lab6-yourname)
│         ├── Live Metrics Stream
│         ├── Application Map (distributed trace topology)
│         └── Transaction Search (end-to-end traces)
│
├── Storage Account (orderslab6yourname)
│   └── Blob Container: orders-drop
│         └── Event Grid System Topic  ──► routes BlobCreated events
│
├── Service Bus Namespace (servicebus-lab6-yourname)      [Standard SKU]
│   ├── Queue: order-intake
│   │     └── receives BlobCreated events from Event Grid
│   └── Topic: order-notifications
│         ├── Subscription: invoice-svc
│         └── Subscription: warehouse-svc
│
├── Logic App Standard (logicapp-lab6-yourname)
│   └── Workflow: process-order
│         ├── Trigger:  Service Bus — When a message arrives in order-intake
│         ├── Action 1: Parse JSON message body
│         ├── Action 2: Forward to order-notifications Topic (fan-out)
│         └── Action 3: Insert enriched record into Azure Table Storage
│
└── Storage Account – Table (orderslab6yourname)
      └── Table: OrderAudit
```

## Job Skills

- Task 1: Provision the observability foundation — Log Analytics and Application Insights
- Task 2: Deploy Azure Service Bus and verify reliable message delivery
- Task 3: Configure Event Grid to capture Blob Storage events
- Task 4: Build a Logic App workflow to process order events
- Task 5: Instrument an application with Application Insights and explore distributed tracing
- Task 6: Configure alerts, dashboards, and operational visibility

---

## Task 1: Provision the Observability Foundation

Every service you deploy in this lab will emit logs and metrics to a shared **Log
Analytics Workspace**. **Application Insights** sits on top, providing application
performance management (APM) — request rates, failure rates, dependency latency, and
distributed trace correlation.

> **Why Log Analytics first?** Log Analytics is the storage and query engine.
> Application Insights is a curated application monitoring layer that stores its
> data *inside* a Log Analytics workspace (workspace-based mode, the current default).
> Creating the workspace first lets you point both Application Insights and every
> Azure service's diagnostic settings to a single store, enabling cross-service
> queries in a single Kusto (KQL) statement.

### Create the Log Analytics Workspace

1. In the Azure portal, search for and select **Log Analytics workspaces**.

2. Select **+ Create** and configure:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab6** |
   | Name | `logs-lab6-yourname` |
   | Region | **Australia East** |

3. Select **Review + Create**, then **Create**. Provisioning takes under a minute.

4. Once deployed, navigate to the workspace. In the left menu under **Settings**, select **Properties**. Note the **Workspace ID** shown on this pane — copy it somewhere accessible.

### Create Application Insights

5. In the Azure portal, search for and select **Application Insights**.

6. Select **+ Create** and configure:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab6** |
   | Name | `appinsights-lab6-yourname` |
   | Region | **Australia East** |
   | Log Analytics Workspace | **logs-lab6-yourname** (the workspace you just created) |

7. Select **Review + Create**, then **Create**.

8. Once deployed, navigate to `appinsights-lab6-yourname`. On the **Overview** pane, copy the **Connection String**. Store it somewhere accessible — you will paste it into an environment variable in Task 5.

### Explore the Log Analytics query environment

9. Navigate to **logs-lab6-yourname**. In the left menu select **Logs**.

   The portal opens in **Simple mode** by default, showing a "Query history" panel.
   To enter KQL, select the **Simple mode** drop-down in the top-right corner of the
   query pane and switch to **KQL mode**. A blank text editor will appear.

10. Paste the following query into the editor and select **Run** (or press **Shift+Enter**):

    ```kusto
    union isfuzzy=true AzureActivity, AppRequests, AppDependencies
    | limit 10
    ```

    At this stage the workspace is empty — the query will return zero rows. This is
    expected. After completing subsequent tasks, you will return here to run
    cross-service queries.

    > **Kusto Query Language (KQL) basics:** KQL is a read-only query language for
    > Azure Monitor. Queries flow left-to-right through a pipe (`|`). Common operators:
    > - `where` — filter rows (equivalent to SQL WHERE)
    > - `summarize` — aggregate (equivalent to SQL GROUP BY)
    > - `project` — select columns (equivalent to SQL SELECT)
    > - `order by` — sort results
    > - `render timechart` — visualise results as a chart in the portal

---

## Task 2: Deploy Azure Service Bus and Verify Reliable Message Delivery

**Azure Service Bus** is an enterprise-grade message broker that decouples producers
from consumers. Unlike a direct HTTP call (where the caller blocks until the callee
responds), Service Bus persists the message on the broker until a consumer retrieves
and acknowledges it. If the consumer is offline or slow, the message waits — it is
never lost.

Two core primitives:

| Primitive | Behaviour | Use case |
| --- | --- | --- |
| **Queue** | First-in, first-out. Each message is delivered to exactly one consumer. | Point-to-point: one producer, one consumer. |
| **Topic + Subscriptions** | Each subscription receives a complete, independent copy of every message published to the topic. | Fan-out: one producer, multiple independent consumers. |

### Create the Service Bus Namespace

1. In the Azure portal, search for and select **Service Bus**.

2. Select **+ Create** and configure the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab6** |
   | Namespace name | `servicebus-lab6-yourname` (must be globally unique) |
   | Region | **Australia East** |
   | Pricing tier | **Standard** |

   > **Why Standard?** Standard is required for Topics and Subscriptions. Basic tier
   > supports only Queues. Premium tier adds VNET integration, larger message sizes,
   > and geo-disaster recovery — not required for this lab.

3. Select **Review + Create**, then **Create**. Provisioning takes 1–2 minutes.

### Create the order-intake Queue

4. Navigate to the `servicebus-lab6-yourname` namespace. In the left menu, under **Entities**,
   select **Queues**.

5. Select **+ Queue** and configure:

   | Setting | Value |
   | --- | --- |
   | Name | `order-intake` |
   | Maximum delivery count | `3` |
   | Message time to live | **1 day** |
   | Lock duration | **30 seconds** |
   | Enable dead-lettering on message expiration | **Checked** |

   Select **Create**.

   > **Queue parameter explanations:**
   >
   > | Setting | Why this value |
   > | --- | --- |
   > | **Name** | The queue identifier used by producers (Event Grid) and consumers (Logic App) to send and receive messages. Must be unique within the namespace. |
   > | **Maximum delivery count** | How many times Service Bus retries delivering a message before giving up. Set to `3` — after 3 failed processing attempts, the message is moved to the dead-letter queue instead of looping forever. |
   > | **Message time to live** | How long a message waits in the queue before expiring if no consumer picks it up. `1 day` prevents stale orders from being processed long after they were submitted. |
   > | **Lock duration** | When a consumer reads a message using peek-lock, it holds an exclusive lock for this duration. Set to `30 seconds` — if the consumer crashes or times out before completing the message, the lock expires and Service Bus makes the message available for redelivery automatically. |
   > | **Enable dead-lettering on message expiration** | When checked, messages that reach their time-to-live without being consumed are moved to the dead-letter sub-queue instead of being silently deleted. This ensures no order is ever lost — expired messages can be inspected, corrected, and replayed by an operator. |

### Create the order-notifications Topic and Subscriptions

6. In the left menu, select **Topics**.

7. Select **+ Topic** and configure:

   | Setting | Value |
   | --- | --- |
   | Name | `order-notifications` |
   | Maximum topic size | **1 GB** |
   | Message time to live | **1 day** |

   Select **Create**.

8. Select the `order-notifications` topic. Under **Entities**, select **Subscriptions**.

9. Select **+ Subscription** and create two subscriptions:

   **First subscription:**

   | Setting | Value |
   | --- | --- |
   | Name | `invoice-svc` |
   | Maximum delivery count | `3` |

   Select **Create**.

   **Second subscription:**

   | Setting | Value |
   | --- | --- |
   | Name | `warehouse-svc` |
   | Maximum delivery count | `3` |

   Select **Create**.

   > **Why two subscriptions?** Topics implement the publish-subscribe pattern. When
   > a message is published to `order-notifications`, Service Bus creates an
   > independent copy for each subscription. The `invoice-svc` subscription's copy
   > is completely independent of the `warehouse-svc` subscription's copy — one
   > subscriber processing (or failing) its copy has no effect on the other.
   >
   > **Real-world analogy:** Think of `order-notifications` as a company-wide email
   > distribution list. When a new order arrives, one notification is sent — and two
   > teams each get their own private copy simultaneously:
   > - **`invoice-svc`** — the billing team's inbox: generate an invoice and take payment
   > - **`warehouse-svc`** — the warehouse team's inbox: pick, pack, and ship the items
   >
   > If the billing team is slow that day, the warehouse team doesn't wait — they're
   > working from their own copy. If the warehouse system crashes, billing still got
   > paid. Neither team interferes with the other.

### Retrieve the connection string

10. Navigate back to the `servicebus-lab6-yourname` namespace. In the left menu, under
    **Settings**, select **Shared access policies**.

11. Select **RootManageSharedAccessKey**.

12. Copy the **Primary Connection String**. You will use this in Task 3 when
    configuring the Event Grid subscription, and in Cloud Shell when sending test
    messages.

### Send and receive a test message using Service Bus Explorer

13. In the portal, navigate to `servicebus-lab6-yourname` → **Queues** → `order-intake`. Select **Service Bus Explorer**.

14. Select the **Send message** tab. In the **Message body** field, paste the
    following JSON and select **Send**:

    ```json
    {"orderId":"ORD-001","customerId":"C-42","amount":149.99,"status":"received"}
    ```

15. Select **Peek from start**. The message body appears
    in the detail pane on the right, confirming the message is sitting in the queue
    waiting for a consumer. Peeking is non-destructive — the message remains in the
    queue.

    > **Service Bus Explorer in the portal** is the fastest way to inspect and
    > debug queue content during development. In production, messages are consumed
    > by application code or through Logic Apps connectors — not by operators peeking
    > manually.

### Enable diagnostic settings for Service Bus

16. Navigate to `servicebus-lab6-yourname`. In the left menu, under **Monitoring**, select
    **Diagnostic settings**.

17. Select **+ Add diagnostic setting** and configure:

    | Setting | Value |
    | --- | --- |
    | Diagnostic setting name | `servicebus-to-logs` |
    | Logs – Category groups | Check **allLogs** |
    | Metrics | Check **AllMetrics** |
    | Destination | **Send to Log Analytics workspace** → `logs-lab6-yourname` |

    Select **Save**.

    > After saving, all Service Bus operational logs (message sends, receives,
    > dead-letter activity) and metrics (active message count, incoming/outgoing
    > message rates) will flow into the shared Log Analytics workspace, where KQL
    > queries can correlate them with Application Insights data from Task 5.

---

## Task 3: Configure Event Grid to Capture Blob Storage Events

**Azure Event Grid** is a fully managed event routing service. It connects *event
sources* (Azure services that emit events — Blob Storage, Resource Manager, AKS, and
others) to *event handlers* (Service Bus, Azure Functions, Logic Apps, webhooks).
Unlike Service Bus (which is message-broker-centric), Event Grid is optimised for
lightweight, near-real-time event notification at scale.

### Create the Storage Account and Blob Container

1. In the Azure portal, search for and select **Storage accounts**.

2. Select **+ Create** and configure:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab6** |
   | Storage account name | `orderslab6yourname` (lowercase, no hyphens, globally unique) |
   | Region | **Australia East** |
   | Performance | **Standard** |
   | Redundancy | **Locally-redundant storage (LRS)** |

3. Select **Review + Create**, then **Create**.

4. Once deployed, navigate to `orderslab6yourname`. In the left menu, under **Data storage**,
   select **Containers**.

5. Select **+ Container**, name it `orders-drop` and select **Create**.

   > **orders-drop** acts as the order landing zone. The front-end application (or
   > a customer-facing API) drops a JSON file here each time an order is submitted.
   > Event Grid watches this container and fires a `BlobCreated` event the moment
   > any file lands — triggering the downstream processing pipeline automatically,
   > without the order producer needing to know who the consumers are.

### Create the Event Grid System Topic

6. Navigate to `orderslab6yourname`. In the left menu, select **Events**.

7. On the **Events** pane, select **+ Event Subscription**.

8. Configure the event subscription:

   **Event Subscription Details:**

   | Setting | Value |
   | --- | --- |
   | Name | `order-blob-to-sb` |
   | Event Schema | **Event Grid Schema** |

   **Topic Details** (auto-populated — confirm these values, and provide a System Topic Name):

   | Setting | Value |
   | --- | --- |
   | Topic Type | Storage account |
   | Source Resource | orderslab6yourname |
   | System Topic Name | `events-lab6-yourname` |

   **Event Types:**

   Uncheck all and check only **Blob Created**.

   > Filtering to `BlobCreated` only means deletes, overwrites, and metadata changes
   > do not trigger the workflow — only net-new order files do.

   **Endpoint Type:**

   Select **Service Bus Queue** as the endpoint type. Select **Configure an endpoint**
   and choose:

   | Setting | Value |
   | --- | --- |
   | Subscription | your Azure subscription |
   | Resource group | **RG-Lab6** |
   | Service Bus Namespace | `servicebus-lab6-yourname` |
   | Service Bus Queue | `order-intake` |

   Select **Confirm Selection**, then **Create**.

   > **What happens now:** Every time a blob is uploaded to `orderslab6yourname`, Event
   > Grid calls the Service Bus data plane and enqueues a `BlobCreated` event message
   > into `order-intake`. The event message is a JSON envelope that includes the blob
   > URL, content length, and etag. The order pipeline can read the blob URL from the
   > event, then download the actual order JSON from Blob Storage.

### Test the end-to-end event flow

9. Create a test order file on your local machine and upload it to the `orders-drop`
   container using the portal.

   **a.** On your local machine, open any text editor (Notepad, VS Code, etc.) and
   paste the following JSON, then save the file as `order-002.json`:

   ```json
   {
     "orderId": "ORD-002",
     "customerId": "C-17",
     "items": [
       { "sku": "WIDGET-A", "qty": 3, "unitPrice": 24.99 },
       { "sku": "WIDGET-B", "qty": 1, "unitPrice": 89.99 }
     ],
     "total": 164.96,
     "submittedAt": "2026-03-21T09:15:00Z"
   }
   ```

   **b.** In the Azure portal, navigate to `orderslab6yourname` → **Containers** →
   `orders-drop`. Select **Upload**, browse to your `order-002.json` file, and
   select **Upload**.

10. Within 5–15 seconds, verify that Event Grid delivered a message to the queue.
    In the portal, navigate to `servicebus-lab6-yourname` → **Queues** → `order-intake` →
    **Service Bus Explorer** → **Peek from start**.

    You should now see **two messages** in the queue:
    - The test message sent manually in Task 2 (Step 15)
    - The `BlobCreated` event just delivered by Event Grid

    Inspect the Event Grid message body. It will be a JSON envelope similar to:

    ```json
    {
      "topic": "/subscriptions/.../resourceGroups/RG-Lab6/providers/Microsoft.Storage/storageAccounts/orderslab6yourname",
      "subject": "/blobServices/default/containers/orders-drop/blobs/order-002.json",
      "eventType": "Microsoft.Storage.BlobCreated",
      "id": "8328ff23-901e-004a-23a3-bad93b060102",
      "eventTime": "2026-03-23T09:01:10.7747937Z",
      "dataVersion": "",
      "metadataVersion": "1",
      "data": {
        "api": "PutBlob",
        "requestId": "8328ff23-901e-004a-23a3-bad93b000000",
        "eTag": "0x8DE88BABA84A18B",
        "contentType": "application/json",
        "contentLength": 250,
        "blobType": "BlockBlob",
        "accessTier": "Default",
        "url": "https://orderslab6yourname.blob.core.windows.net/orders-drop/order-002.json",
        "sequencer": "000000000000000000000000000143E20000000001a59957",
        "storageDiagnostics": {
          "batchId": "b57d099a-e006-001d-00a3-ba7708000000"
        }
      }
    }
    ```

    > **Event Grid vs Service Bus — when to use which:**
    >
    > | Dimension | Event Grid | Service Bus |
    > | --- | --- | --- |
    > | Message size | Up to 1 MB | Up to 100 MB (Premium: 100 GB) |
    > | Delivery guarantee | At-least-once | At-least-once (queue) or exactly-once with sessions |
    > | Ordering | Not guaranteed | Guaranteed within a session |
    > | Primary pattern | Reactive eventing (something happened) | Reliable command/data messaging |
    > | Fan-out | Via multiple subscriptions per topic | Via Topics |
    > | Retry policy | Configurable exponential back-off (up to 24 hours) | Lock duration + dead-letter |
    >
    > In this lab, Event Grid handles the thin notification layer (blob created →
    > enqueue an event pointer), while Service Bus handles reliable processing
    > and the fan-out to downstream services.

---

## Task 4: Build a Logic App Workflow to Process Order Events

**Azure Logic Apps** is a low-code integration platform for building automated
workflows. A Logic App **workflow** consists of a **trigger** that starts the run and
one or more **actions** that execute sequentially (or in parallel, with branching) in
response. Logic Apps Workflow Service Plan runs as a single-tenant, container-based host in an App
Service Environment, giving you network isolation and deterministic scaling.

In this task you build a workflow that:
1. Triggers when a message arrives in the `order-intake` queue
2. Parses the event body to extract the blob URL
3. Publishes a notification to the `order-notifications` topic (fan-out to `invoice-svc`
   and `warehouse-svc` subscribers)
4. Writes an audit record to Azure Table Storage

### Create a Storage Table for order auditing

1. Navigate to `orderslab6yourname`. In the left menu, under **Data storage**, select
   **Tables**.

2. Select **+ Table**, name it `OrderAudit`, and select **OK**.

### Create the Logic App resource

3. In the Azure portal, search for and select **Logic apps**.

4. Select **+ Add**. A **Select a hosting option** screen appears. Select **Workflow Service Plan** (under the Standard column) and select **Select**.

   > **Why Workflow Service Plan?** This is the single-tenant Standard hosting option. It provides a dedicated compute host with VNET integration, which is required for the private networking model in this lab. The Consumption (Multi-tenant) option runs on shared Microsoft infrastructure with no VNET support and is not suitable here.

5. Configure the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab6** |
   | Logic App name | `logicapp-lab6-yourname` |
   | Region | **Australia East** |
   | Windows Plan | (New) accept the default name |
   | Pricing plan | **WS1** (smallest, sufficient for lab) |

6. Select the **Monitoring** tab and configure:

   | Setting | Value |
   | --- | --- |
   | Enable Application Insights | **Yes** |
   | Application Insights | **appinsights-lab6-yourname** |

   > Enabling Application Insights here automatically instruments all Logic App
   > workflow runs with distributed trace context. Each run generates a dependency
   > telemetry item visible in the Application Map (Task 5).

7. Select **Review + Create**, then **Create**. Provisioning takes 2–3 minutes.

### Create the process-order workflow

8. Once deployed, navigate to `logicapp-lab6-yourname`. In the left menu, under
   **Workflows**, select **Workflows**.

9. Select **+ Create** and configure:

   | Setting | Value |
   | --- | --- |
   | Workflow name | `process-order` |
   | State type | **Stateful** |

   Select **Create**.

   > **Stateful vs Stateless:** Stateful workflows persist their run history and
   > inputs/outputs to Azure Storage, enabling full run history replay and long-running
   > (days/weeks) workflows. Stateless workflows run entirely in memory —
   > faster and cheaper, but no durable run history. Use stateless for high-throughput,
   > short-lived workflows; stateful for auditable, long-running business processes.

10. Select the `process-order` workflow and then select **Designer** to open the
    visual workflow editor.

### Add the Service Bus trigger

11. In the designer, select **Add a trigger** and search for `Service Bus`.

12. Select **When messages are available in a queue (peek-lock)**.

13. Create a new Service Bus connection:

    | Setting | Value |
    | --- | --- |
    | Connection name | `sb-connection` |
    | Authentication | **Connection String** |
    | Connection String | paste the Primary Connection String from Task 2 Step 12 |

    Select **Create new**.

14. Configure the trigger:

    | Setting | Value |
    | --- | --- |
    | Queue name | `order-intake` |

    Under **Advanced parameters**, select **Show all** to reveal hidden settings, then set **Maximum message batch size** to `1`.

    > **Peek-lock vs Receive-and-Delete:** Peek-lock takes a temporary lock on the
    > message without removing it from the queue. The message is only deleted (settled)
    > after the Logic App explicitly completes it with a **Complete message** action at
    > the end of the workflow. If the workflow fails before completing, the lock expires
    > after 30 seconds and Service Bus makes the message available again for redelivery.
    > This guarantees at-least-once processing — no messages are lost even if Logic Apps
    > crashes mid-run.

### Add the Parse JSON action

15. Below the trigger, select **+** and then **Add an action**. Search for `Parse JSON`
    and select it.

16. Configure:

    | Field | Value |
    | --- | --- |
    | Content | Select the lightning-bolt icon and choose **Message Body** from the trigger's dynamic content |
    | Schema | Select **Use sample payload to generate schema** and paste: |

    > **Where does this sample come from?** This is the actual Event Grid message body delivered to the `order-intake` queue, which you can inspect in Service Bus Explorer (Task 3, Step 10). Paste your own message from there, or use the sample below.

    ```json
    {
      "topic": "/subscriptions/xxx/resourceGroups/RG-Lab6/providers/Microsoft.Storage/storageAccounts/orderslab6yourname",
      "subject": "/blobServices/default/containers/orders-drop/blobs/order-002.json",
      "eventType": "Microsoft.Storage.BlobCreated",
      "id": "7a48eca8-b01e-0005-4dc0-ba5f1f06d7cd",
      "data": {
        "api": "PutBlob",
        "requestId": "7a48eca8-b01e-0005-4dc0-ba5f1f000000",
        "eTag": "0x8DE88D8105829EC",
        "contentType": "application/json",
        "contentLength": 250,
        "blobType": "BlockBlob",
        "accessTier": "Default",
        "url": "https://orderslab6yourname.blob.core.windows.net/orders-drop/order-002.json",
        "sequencer": "000000000000000000000000000219880000000000a171a8",
        "storageDiagnostics": {
          "batchId": "78a44d18-d006-0071-00c0-ba6bef000000"
        }
      },
      "dataVersion": "",
      "metadataVersion": "1",
      "eventTime": "2026-03-23T12:31:10.1725913Z"
    }
    ```

    Select **Done** to generate the schema automatically.

### Add the Service Bus publish action (fan-out)

17. Add another action below Parse JSON. Search for `Service Bus` and select
    **Send message**.

18. Configure:

    | Setting | Value |
    | --- | --- |
    | Connection | **sb-connection** (reuse the existing connection) |
    | Entity type | **Topic** |
    | Topic name | `order-notifications` |
    | Message body | Use dynamic content to compose a JSON string: |

    In the **Message** field, select the expression editor (the `fx` icon) and enter:

    ```
    json(concat('{"event":"OrderReceived","blobUrl":"', body('Parse_JSON')?['data']?['url'], '","processedAt":"', utcNow(), '"}'))
    ```

    > This expression reads the blob URL parsed from the Event Grid message envelope
    > and constructs a richer notification message for downstream subscribers. Both
    > `invoice-svc` and `warehouse-svc` subscriptions receive a copy automatically —
    > that is the publish-subscribe fan-out without any additional workflow steps.

### Add the Table Storage write action (audit)

19. Add a third action. Search for `Azure Table Storage` and select **Insert or replace
    entity**.

20. Create a new connection:

    | Setting | Value |
    | --- | --- |
    | Connection name | `storage-connection` |
    | Authentication | **Access Key** |
    | Storage Account Name | `orderslab6yourname` |
    | Shared Storage Key | Copy from orderslab6yourname → **Access keys** → **key1** |

    Select **Create new**.

21. Configure the action:

    | Setting | Value |
    | --- | --- |
    | Table name | `OrderAudit` |
    | Partition key | `orders` |
    | Row key | Use expression: `guid()` |
    | Entity | Compose this JSON (use dynamic content for blobUrl): |

    ```json
    {
      "PartitionKey": "orders",
      "RowKey": "@{guid()}",
      "BlobUrl": "@{body('Parse_JSON')?['data']?['url']}",
      "EventTime": "@{body('Parse_JSON')?['eventTime']}",
      "ProcessedAt": "@{utcNow()}"
    }
    ```

### Add the Complete Message action

22. Add a final action. Search for `Service Bus` and select **Complete the message in a queue**.

23. Configure:

    | Setting | Value |
    | --- | --- |
    | Connection | **sb-connection** |
    | Queue name | `order-intake` |
    | Lock token | Select dynamic content → **Lock Token** from the trigger |

    > This action settles the message. Once completed, Service Bus permanently removes
    > it from the queue. If this action is never reached (because a previous action
    > threw an error), the lock expires and the message is redelivered — guaranteed
    > at-least-once processing.

24. Select **Save** in the designer toolbar.

### Upload a new order blob and observe the workflow run

25. In Cloud Shell, upload another order file:

    ```bash
    cat <<'EOF' > order-003.json
    {
      "orderId": "ORD-003",
      "customerId": "C-88",
      "items": [
        { "sku": "GADGET-X", "qty": 2, "unitPrice": 59.99 }
      ],
      "total": 119.98,
      "submittedAt": "2026-03-21T09:30:00Z"
    }
    EOF

    az storage blob upload \
      --account-name orderslab6yourname \
      --container-name orders-drop \
      --name order-003.json \
      --file order-003.json \
      --auth-mode login
    ```

26. In the portal, navigate to `logicapp-lab6-yourname` → **Workflows** → `process-order`
    → **Run history** (left menu).

    Within 30–60 seconds a new run should appear with status **Succeeded**.

27. Select the run to open the run detail view. Each action is shown with its
    duration and inputs/outputs. Expand **Parse JSON** to see the parsed blob URL.
    Expand **Insert or replace entity** to confirm the Table Storage write.

28. Verify the audit record was written. In Cloud Shell:

    ```bash
    az storage entity query \
      --account-name orderslab6yourname \
      --table-name OrderAudit \
      --auth-mode login
    ```

    You should see one or more rows with the `BlobUrl` and `ProcessedAt` fields.

29. Verify the fan-out. In the portal, navigate to `servicebus-lab6-yourname` →
    **Topics** → `order-notifications` → **Subscriptions**. Both `invoice-svc`
    and `warehouse-svc` should show **1** as the **Active Message Count**.

---

## Task 5: Instrument an Application with Application Insights and Explore Distributed Tracing

**Application Insights** provides end-to-end distributed tracing by injecting a
correlation context (an `Operation ID`) into every request. Each service that handles
the request adds a child span — a **dependency telemetry item** — that records how long
its work took and whether it succeeded. All spans sharing the same Operation ID are
assembled into an **end-to-end transaction trace**, visualised in the portal as a
waterfall diagram and in the **Application Map** as a live topology of all services
and their dependency relationships.

In this task you deploy a simple Python application to Azure Container Instances (ACI)
with the Application Insights SDK configured, send traffic through it, and explore
the resulting traces and metrics.

### Deploy a sample instrumented application

1. In Cloud Shell, create the application files:

   ```bash
   mkdir -p ~/lab6-app && cd ~/lab6-app
   ```

2. Create the application code:

   ```bash
   cat <<'EOF' > app.py
   import os
   import time
   import random
   import json
   from http.server import HTTPServer, BaseHTTPRequestHandler
   from azure.monitor.opentelemetry import configure_azure_monitor
   from opentelemetry import trace
   from opentelemetry.instrumentation.requests import RequestsInstrumentor

   # Configure the Azure Monitor exporter using the connection string
   configure_azure_monitor(
       connection_string=os.environ["APPLICATIONINSIGHTS_CONNECTION_STRING"]
   )
   RequestsInstrumentor().instrument()

   tracer = trace.get_tracer(__name__)

   class OrderHandler(BaseHTTPRequestHandler):
       def do_GET(self):
           if self.path == "/health":
               self.send_response(200)
               self.end_headers()
               self.wfile.write(b'{"status":"healthy"}')
               return

           if self.path.startswith("/order"):
               with tracer.start_as_current_span("process-order") as span:
                   order_id = f"ORD-{random.randint(1000, 9999)}"
                   span.set_attribute("order.id", order_id)

                   # Simulate downstream dependency call latency
                   with tracer.start_as_current_span("query-inventory-db") as dep_span:
                       latency = random.uniform(0.02, 0.15)
                       time.sleep(latency)
                       dep_span.set_attribute("db.latency_ms", round(latency * 1000))

                   # Simulate occasional downstream errors for observability demo
                   if random.random() < 0.15:
                       span.set_attribute("error", True)
                       self.send_response(500)
                       self.end_headers()
                       self.wfile.write(json.dumps({
                           "error": "inventory-db timeout",
                           "orderId": order_id
                       }).encode())
                       return

                   self.send_response(200)
                   self.end_headers()
                   self.wfile.write(json.dumps({
                       "orderId": order_id,
                       "status": "accepted"
                   }).encode())
               return

           self.send_response(404)
           self.end_headers()

       def log_message(self, format, *args):
           pass  # suppress default access log — Application Insights captures it

   if __name__ == "__main__":
       print("Starting order API on port 8080")
       HTTPServer(("0.0.0.0", 8080), OrderHandler).serve_forever()
   EOF
   ```

3. Create the `requirements.txt`:

   ```bash
   cat <<'EOF' > requirements.txt
   azure-monitor-opentelemetry==1.6.4
   opentelemetry-instrumentation-requests==0.49b0
   EOF
   ```

4. Create the Dockerfile:

   ```bash
   cat <<'EOF' > Dockerfile
   FROM python:3.12-slim
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY app.py .
   EXPOSE 8080
   CMD ["python", "app.py"]
   EOF
   ```

5. Build the container image using Azure Container Registry (ACR) Tasks — no local
   Docker installation required:

   ```bash
   # Create a container registry (name must be globally unique, lowercase, no hyphens)
   az acr create \
     --resource-group RG-Lab6 \
     --name acrlab6yourname \
     --sku Basic

   # Build and push the image to ACR using a cloud build
   az acr build \
     --registry acrlab6yourname \
     --image lab6/order-api:v1 \
     .
   ```

   The build runs in the cloud — output streams back to Cloud Shell. Expect 2–3 minutes.

6. Retrieve your Application Insights connection string (substitute the value you
   saved in Task 1 Step 8):

   ```bash
   AI_CONN_STR=$(az monitor app-insights component show \
     --app appinsights-lab6-yourname \
     --resource-group RG-Lab6 \
     --query connectionString -o tsv)

   echo "Connection string: $AI_CONN_STR"
   ```

7. Deploy the instrumented application to Azure Container Instances:

   ```bash
   # Get the ACR login server and credentials
   ACR_SERVER=$(az acr show --name acrlab6yourname --query loginServer -o tsv)
   ACR_PASSWORD=$(az acr credential show --name acrlab6yourname --query "passwords[0].value" -o tsv)

   az container create \
     --resource-group RG-Lab6 \
     --name order-api \
     --image ${ACR_SERVER}/lab6/order-api:v1 \
     --registry-login-server ${ACR_SERVER} \
     --registry-username acrlab6yourname \
     --registry-password "${ACR_PASSWORD}" \
     --environment-variables APPLICATIONINSIGHTS_CONNECTION_STRING="${AI_CONN_STR}" \
     --ports 8080 \
     --dns-name-label order-api-yourname \
     --cpu 0.5 \
     --memory 0.5
   ```

8. Retrieve the public FQDN:

   ```bash
   FQDN=$(az container show \
     --resource-group RG-Lab6 \
     --name order-api \
     --query ipAddress.fqdn -o tsv)

   echo "App URL: http://${FQDN}:8080/order"
   ```

### Generate traffic to produce telemetry

9. Send a burst of requests to generate traces and surface the intentional errors
   (approximately 15% of requests will return HTTP 500):

   ```bash
   for i in $(seq 1 40); do
     curl -s "http://${FQDN}:8080/order" | python3 -m json.tool --no-ensure-ascii
     sleep 0.5
   done
   ```

   You should see a mix of `"status": "accepted"` (HTTP 200) and
   `"error": "inventory-db timeout"` (HTTP 500) responses.

### Explore Application Insights in the portal

10. Navigate to `appinsights-lab6-yourname` in the portal. Select **Overview**. After 1–2
    minutes of telemetry ingestion you will see live summary tiles:

    | Tile | What it shows |
    | --- | --- |
    | **Failed requests** | Count of HTTP 5xx responses over the last hour |
    | **Server response time** | Average end-to-end latency including all child spans |
    | **Server requests** | Requests per second |
    | **Availability** | Results of configured availability tests (none yet — configured in Task 6) |

11. Select **Application Map** in the left menu.

    The Application Map renders a real-time topology graph. You should see:
    - A node for **order-api** (the ACI container)
    - A node for **query-inventory-db** (the child span representing the simulated
      database call), connected by an edge labelled with average latency

    Select a node to see call rate, failure rate, and average duration. This map
    is auto-generated from distributed trace data — no manual configuration required.

    > **Application Map in a microservices architecture:** As you add more services,
    > each service that propagates the `traceparent` W3C header (done automatically
    > by OpenTelemetry SDK instrumentation) appears as a node. The map becomes a
    > live architecture diagram that reflects actual runtime behaviour, not just
    > what was documented at design time.

12. Select **Transaction search** (left menu, under **Investigate**). In the
    **Time range**, select **Last 30 minutes**.

    You will see a list of recent requests. Select one with **Result code 500** to
    open the end-to-end transaction detail. The waterfall view shows:
    - The top-level `process-order` span (the outer trace)
    - The child `query-inventory-db` span nested below it
    - The duration of each span and the point at which the error occurred

    > **Distributed tracing — how it works:** The Application Insights / OpenTelemetry
    > SDK generates a unique `Operation ID` (trace ID) for each inbound request, plus
    > a `Span ID` for each unit of work. When the application calls a downstream
    > service, it injects both IDs into the outgoing HTTP header (`traceparent`).
    > The downstream service reads the header, creates a child span with the same
    > `Operation ID` but a new `Span ID`, and emits it to Application Insights.
    > The portal then assembles all spans with the same `Operation ID` into a single
    > waterfall, regardless of which service or process emitted them.

13. Select **Failures** in the left menu. The **Operations** tab shows which
    operations are producing the most errors. Select `GET /order` to drill into
    the failure detail. Select **Drill into → Top 3 exception types** to see
    the stack trace captured at the point of failure.

14. Select **Performance** in the left menu. The scatter chart plots individual
    request durations over time. Select **Drill into → Top performing operations**
    to see dependency breakdown — how much of the end-to-end latency is attributable
    to the simulated inventory DB call versus the application's own processing time.

15. Select **Live Metrics** in the left menu. This stream shows real-time telemetry
    with sub-second latency — incoming request rate, outgoing dependency call rate,
    CPU, memory, and a live feed of failed requests. Send another burst of traffic
    to see the tiles update in real time:

    ```bash
    for i in $(seq 1 20); do curl -s "http://${FQDN}:8080/order" > /dev/null; sleep 0.3; done
    ```

    > **Live Metrics** uses a persistent streaming connection, not the standard
    > 2-minute telemetry batch cycle. This makes it useful for watching a deployment
    > roll out in real time or triaging an active incident.

### Query traces in Log Analytics with KQL

16. Navigate to **logs-lab6-yourname** → **Logs** and run the following query to
    find all failed requests in the last hour:

    ```kusto
    AppRequests
    | where TimeGenerated > ago(1h)
    | where Success == false
    | project TimeGenerated, Name, ResultCode, DurationMs, OperationId
    | order by TimeGenerated desc
    ```

17. Find average latency per operation, broken down by success/failure:

    ```kusto
    AppRequests
    | where TimeGenerated > ago(1h)
    | summarize
        AvgDurationMs = avg(DurationMs),
        P95DurationMs = percentile(DurationMs, 95),
        RequestCount = count()
      by Name, Success
    | order by AvgDurationMs desc
    ```

18. Correlate request traces with dependency spans in a single query:

    ```kusto
    AppRequests
    | where TimeGenerated > ago(1h)
    | join kind=inner (
        AppDependencies
        | where TimeGenerated > ago(1h)
        | project DepOperationId = OperationId, DepName = Name, DepDurationMs = DurationMs, DepSuccess = Success
    ) on $left.OperationId == $right.DepOperationId
    | project TimeGenerated, RequestName = Name, DepName, DurationMs, DepDurationMs, Success, DepSuccess
    | order by TimeGenerated desc
    ```

    This join links each HTTP request to the downstream dependency call it triggered,
    showing in a single row how much of the total latency came from the database call.

---

## Task 6: Configure Alerts, Dashboards, and Operational Visibility

Observability is not only about collecting data — it is about acting on it. In this
task you configure proactive alerts so the team is notified before a problem escalates,
and build a shared dashboard that gives the whole organisation a single view of
platform health.

### Create a metric alert for high error rate

1. Navigate to `appinsights-lab6-yourname`. In the left menu, under **Monitoring**, select
   **Alerts**.

2. Select **+ Create** → **Alert rule**.

3. On the **Condition** tab, select **Add condition** and search for
   `Failed requests`. Select it.

4. Configure the signal logic:

   | Setting | Value |
   | --- | --- |
   | Threshold | **Static** |
   | Aggregation type | **Count** |
   | Operator | **Greater than** |
   | Threshold value | `3` |
   | Check every | **1 minute** |
   | Lookback period | **5 minutes** |

   > A threshold of 3 failures in 5 minutes is deliberately low for the lab so you
   > can trigger it immediately. In production, calibrate the threshold to the
   > expected baseline error rate — typically expressed as a percentage using a
   > dynamic threshold, not a fixed count.

5. Select **Next: Actions**. Select **+ Create action group** and configure:

   | Setting | Value |
   | --- | --- |
   | Action group name | `lab6-ops-team` |
   | Display name | `Lab6-Ops` |

   Under **Notifications**, add:

   | Notification type | Name | Email |
   | --- | --- | --- |
   | Email/SMS message/Push/Voice | `email-on-call` | your own email address |

   Select **Review + Create** → **Create**.

6. Back on the alert rule creation page, set:

   | Setting | Value |
   | --- | --- |
   | Alert rule name | `High error rate — order-api` |
   | Severity | **1 – Error** |
   | Enable upon creation | **Yes** |

   Select **Review + Create** → **Create**.

### Create an availability test

7. Navigate to `appinsights-lab6-yourname` → **Availability** (left menu). Select
   **+ Add Standard test**.

8. Configure:

   | Setting | Value |
   | --- | --- |
   | Test name | `order-api-health` |
   | URL | `http://<your-FQDN>:8080/health` (use the FQDN from Task 5 Step 8) |
   | Test frequency | **5 minutes** |
   | Test locations | Select at least **3** locations — e.g., Australia East, Southeast Asia, East US |
   | Success criteria – HTTP status | `200` |

   Select **Create**.

   > **Availability tests** call your endpoint from Azure-managed global test agents.
   > If fewer than a configurable number of locations can reach your endpoint, an
   > availability alert fires. This catches regional outages — not just single-region
   > failures — because tests run from multiple geographically distinct locations
   > simultaneously.

9. After 5–10 minutes, navigate to **Availability** to see the results. Each dot
   on the scatter chart represents a single test run from a single location. Green
   dots = pass, red dots = fail.

### Create a Log Analytics alert for Service Bus dead-letter messages

10. Navigate to **logs-lab6-yourname** → **Alerts** → **+ Create** → **Alert rule**.

11. On the **Condition** tab, select **Custom log search** and enter:

    ```kusto
    AzureMetrics
    | where ResourceType == "MICROSOFT.SERVICEBUS/NAMESPACES"
    | where MetricName == "DeadletteredMessages"
    | where Total > 0
    | summarize DeadLetterCount = sum(Total) by bin(TimeGenerated, 5m), Resource
    ```

12. Set the alert condition:

    | Setting | Value |
    | --- | --- |
    | Operator | **Greater than** |
    | Threshold | `0` |
    | Check every | **5 minutes** |
    | Lookback period | **5 minutes** |

13. Assign the `lab6-ops-team` action group. Name the alert
    `Dead-letter messages detected — Service Bus`. Select **Severity 2 – Warning**.

    Select **Review + Create** → **Create**.

    > **Why dead-letter alerts matter:** A dead-letter message is a message that
    > failed processing `Maximum delivery count` times. It means the workflow (Logic
    > App) crashed repeatably on a particular message. Without a DLQ alert, these
    > failures are silent — the order disappears from the queue but was never
    > processed. An alert ensures the team investigates before the backlog grows.

### Build an Azure Dashboard

14. In the Azure portal, select the **Dashboard** icon from the left navigation bar
    (the grid icon). Select **+ New dashboard** → **Blank dashboard**.

15. Name the dashboard `Lab6 — Platform Health`.

16. Select **+ Add tile** and add the following tiles by searching for each resource:

    | Tile type | Resource | Metric / Content |
    | --- | --- | --- |
    | **Metrics chart** | `appinsights-lab6-yourname` | Metric: **Failed requests**, Time: Last 24h |
    | **Metrics chart** | `appinsights-lab6-yourname` | Metric: **Server response time**, Time: Last 24h |
    | **Metrics chart** | `servicebus-lab6-yourname` | Metric: **Active Messages**, Time: Last 24h |
    | **Metrics chart** | `servicebus-lab6-yourname` | Metric: **Dead-lettered Messages**, Time: Last 24h |
    | **Log Analytics query** | `logs-lab6-yourname` | Paste the failed requests KQL from Task 5 Step 16 |
    | **Application Insights Application Map** | `appinsights-lab6-yourname` | Pin directly from the Application Map blade |

17. Arrange and resize the tiles to your preference. Select **Save**.

18. Select **Share** → **Publish dashboard** to make the dashboard visible to all
    users in your Azure Active Directory tenant with read access to the resource group.

    > **Shared dashboards** appear under the **Shared dashboards** tab for any portal
    > user with `Reader` role on the dashboard resource. Combined with Azure RBAC,
    > this means the operations team sees the platform health view without needing
    > access to the underlying resource configuration blades.

### Validate the full end-to-end telemetry chain

19. Trigger the complete pipeline one more time and confirm all observability layers captured it:

    ```bash
    # Upload a new order blob (triggers Event Grid → Service Bus → Logic App)
    cat > order-004.json <<'EOF'
    {"orderId":"ORD-004","customerId":"C-99","total":299.99}
    EOF

    az storage blob upload \
      --account-name orderslab6yourname \
      --container-name orders-drop \
      --name order-004.json \
      --file order-004.json \
      --auth-mode login

    # Send traffic to the order API (generates Application Insights traces)
    for i in $(seq 1 10); do curl -s "http://${FQDN}:8080/order" > /dev/null; sleep 1; done
    ```

20. Confirm all observability layers captured the activity:

    | Layer | Where to check | Expected result |
    | --- | --- | --- |
    | Event Grid delivery | `orderslab6yourname` → **Events** → **Metrics** → **Published Events** | Shows 1 event published |
    | Service Bus message | `servicebus-lab6-yourname` → `order-intake` → **Metrics** → **Messages** | Incoming message count increases |
    | Logic App run | `logicapp-lab6-yourname` → Workflows → `process-order` → **Run history** | New Succeeded run for order-004 |
    | Application Insights | `appinsights-lab6-yourname` → **Transaction search** | New request traces visible |
    | Log Analytics | **logs-lab6-yourname** → **Logs** → run the KQL from Task 5 Step 16 | New rows with recent timestamps |
    | Dashboard | **Lab6 — Platform Health** | Metrics charts reflect recent activity |

---

## Cleanup

**Note:** Azure Service Bus (Standard), Logic Apps (Standard), Container Instances,
and Application Insights all incur ongoing charges. Delete resources promptly after the lab.

1. In the portal, navigate to **RG-Lab6**.

2. Select **Delete resource group**.

3. Copy and paste the resource group name to confirm deletion.

4. Select **Delete**.

   This removes all resources provisioned during the lab — the Service Bus namespace
   and its queues/topics, the Logic App and its hosting plan, the Container Instance,
   the Storage Account (blobs and tables), the Event Grid subscription, the Log
   Analytics workspace, and Application Insights — in a single operation.

---

## Key Takeaways

- **Event Grid decouples event producers from consumers.** When a blob lands in
  storage, Event Grid fires an event to Service Bus without the uploading application
  needing to know which downstream services consume it. Adding a new consumer is
  a matter of adding a new Event Grid subscription — no application code changes.

- **Service Bus provides reliable, ordered, durable messaging.** The peek-lock
  pattern ensures messages are never lost, even if Logic Apps crashes mid-processing.
  The dead-letter queue captures poison messages for investigation without stalling
  the rest of the pipeline.

- **Topics + Subscriptions implement fan-out without coupling.** Publishing a single
  message to the `order-notifications` topic delivers an independent copy to both
  `invoice-svc` and `warehouse-svc`. Each subscriber processes at its own pace and
  failure rate without affecting the other.

- **Logic Apps orchestrates integration without custom code.** The visual designer
  and 400+ built-in connectors (Service Bus, Blob Storage, Table Storage, HTTP,
  Office 365, Salesforce, and more) allow platform engineers to wire systems together
  declaratively. Stateful workflow runs are durably persisted and fully auditable.

- **A workspace-based Application Insights instance centralises all telemetry in
  Log Analytics.** This allows KQL queries to join application traces (`AppRequests`,
  `AppDependencies`) with infrastructure logs (`AzureActivity`, `AzureMetrics`) in
  a single query — correlation across the full stack, not just within one service.

- **Distributed tracing (OpenTelemetry + W3C traceparent) makes microservice
  failures diagnosable.** The Application Map shows the live runtime topology.
  The end-to-end transaction view pinpoints which service and which dependency
  caused a failure. Without tracing, a 500 error at the API gateway tells you
  something is wrong but not where or why.

- **Alerts must be proactive, not reactive.** Metric alerts on failed requests and
  KQL-based alerts on dead-letter messages notify the team before customers report
  failures. Availability tests from multiple global locations catch regional outages
  that single-region monitoring misses.

- **Observability is a platform capability, not an application feature.** By
  routing diagnostic settings from every Azure service into a single Log Analytics
  workspace and pointing Application Insights at the same workspace, the platform
  team gains a unified, queryable view of the entire system — independent of which
  team owns each service. This is the foundation of a mature Site Reliability
  Engineering (SRE) practice.

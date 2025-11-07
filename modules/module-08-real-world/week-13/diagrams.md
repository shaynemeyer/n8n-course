# Week 13: Complex Business Automations - Visual Guides

This document contains all visual diagrams for Week 13 content.

## Table of Contents

1. [E-commerce Order Processing Flow](#e-commerce-order-processing-flow)
2. [Customer Onboarding Journey](#customer-onboarding-journey)
3. [Multi-Stage Approval Workflow](#multi-stage-approval-workflow)
4. [Event-Driven Architecture](#event-driven-architecture)
5. [Financial Reconciliation Process](#financial-reconciliation-process)

---

## E-commerce Order Processing Flow

### Complete Order Fulfillment Process

```mermaid
flowchart TD
    START[Customer Places Order] --> WEBHOOK[Order Webhook Received]

    WEBHOOK --> VALIDATE[Validate Order Data]

    VALIDATE -->|Invalid| ERROR1[Send Error Email<br/>Log Failed Order]
    VALIDATE -->|Valid| FRAUD[Fraud Check]

    FRAUD -->|High Risk| MANUAL[Flag for Manual Review<br/>Notify Team]
    FRAUD -->|Low Risk| INVENTORY[Check Inventory]

    INVENTORY -->|Insufficient| BACKORDER[Create Backorder<br/>Notify Customer<br/>Alert Purchasing]
    INVENTORY -->|Available| RESERVE[Reserve Inventory]

    RESERVE --> PAYMENT[Process Payment]

    PAYMENT -->|Failed| PAYMENT_FAILED[Release Inventory<br/>Send Payment Failed Email<br/>Retry Logic]
    PAYMENT -->|Success| FULFILL[Create Fulfillment Order]

    FULFILL --> WAREHOUSE[Route to Optimal Warehouse]

    WAREHOUSE --> PICK[Generate Pick List]
    PICK --> PACK[Generate Packing Slip]
    PACK --> SHIP[Create Shipping Label]

    SHIP --> NOTIFY_SHIPPED[Send Shipping Confirmation<br/>Include Tracking Number]

    NOTIFY_SHIPPED --> UPDATE_INVENTORY[Update Inventory Count]
    UPDATE_INVENTORY --> UPDATE_ACCOUNTING[Update Accounting System]

    UPDATE_ACCOUNTING --> TRACK[Set Up Delivery Tracking]

    TRACK --> DELIVERED{Delivered?}

    DELIVERED -->|Yes| CONFIRM[Send Delivery Confirmation<br/>Request Review]
    DELIVERED -->|No| WAIT[Wait for Delivery]

    WAIT -->|3 Days| REMIND[Send Tracking Reminder]
    REMIND --> DELIVERED

    CONFIRM --> FOLLOWUP[Schedule Follow-up<br/>7 Days]
    FOLLOWUP --> COMPLETE[Order Complete]

    style START fill:#4CAF50,stroke:#333,color:#fff
    style PAYMENT_FAILED fill:#F44336,stroke:#333,color:#fff
    style BACKORDER fill:#FF9800,stroke:#333,color:#fff
    style COMPLETE fill:#4CAF50,stroke:#333,color:#fff
```

### Multi-Channel Order Integration

```mermaid
graph TB
    subgraph "Sales Channels"
        SHOPIFY[Shopify Store]
        AMAZON[Amazon]
        EBAY[eBay]
        WEBSITE[Direct Website]
        POS[POS System]
    end

    subgraph "n8n Order Hub"
        ROUTER[Order Router<br/>Normalize & Route]
        ORCHESTRATOR[Order Orchestrator<br/>Main Workflow]
    end

    subgraph "Processing Systems"
        INVENTORY[(Inventory Management<br/>Real-time Stock)]
        PAYMENT[Payment Gateway<br/>Stripe/PayPal]
        SHIPPING[Shipping Provider<br/>ShipStation]
        ACCOUNTING[Accounting<br/>QuickBooks]
    end

    subgraph "Fulfillment"
        WH1[Warehouse 1<br/>East Coast]
        WH2[Warehouse 2<br/>West Coast]
        WH3[Warehouse 3<br/>International]
    end

    SHOPIFY -->|Webhook| ROUTER
    AMAZON -->|API Poll| ROUTER
    EBAY -->|Webhook| ROUTER
    WEBSITE -->|Webhook| ROUTER
    POS -->|API| ROUTER

    ROUTER --> ORCHESTRATOR

    ORCHESTRATOR --> INVENTORY
    ORCHESTRATOR --> PAYMENT
    ORCHESTRATOR --> SHIPPING
    ORCHESTRATOR --> ACCOUNTING

    ORCHESTRATOR -->|Route by Location| WH1
    ORCHESTRATOR -->|Route by Location| WH2
    ORCHESTRATOR -->|Route by Location| WH3

    style ROUTER fill:#EA4B71,stroke:#333,color:#fff
    style ORCHESTRATOR fill:#FF9800,stroke:#333,color:#fff
    style INVENTORY fill:#4CAF50,stroke:#333,color:#fff
```

### Inventory Synchronization

```mermaid
sequenceDiagram
    participant S as Shopify
    participant A as Amazon
    participant n8n
    participant DB as Master Inventory
    participant WH as Warehouse System

    Note over n8n: Every 5 Minutes

    n8n->>DB: Get Current Inventory Levels
    DB-->>n8n: SKU: ABC123, Qty: 47

    n8n->>S: Get Shopify Inventory
    S-->>n8n: ABC123: 50

    n8n->>A: Get Amazon Inventory
    A-->>n8n: ABC123: 45

    Note over n8n: Calculate Differences<br/>Master: 47<br/>Shopify: 50 (needs update)<br/>Amazon: 45 (needs update)

    n8n->>S: Update to 47
    S-->>n8n: Updated ✓

    n8n->>A: Update to 47
    A-->>n8n: Updated ✓

    n8n->>DB: Log Sync Activity
    DB-->>n8n: Logged ✓

    alt Warehouse Update Received
        WH->>n8n: Inventory Change Event<br/>ABC123: -3 (sold)
        n8n->>DB: Update Master<br/>47 → 44
        n8n->>S: Update to 44
        n8n->>A: Update to 44
    end
```

---

## Customer Onboarding Journey

### Employee Onboarding Automation

```mermaid
flowchart TD
    START[New Hire Accepts Offer] --> TRIGGER[Trigger Onboarding Workflow]

    TRIGGER --> WELCOME[Send Welcome Email<br/>With Overview]

    WELCOME --> PARALLEL1{Start Parallel Tasks}

    PARALLEL1 --> DOCS[Collect Documents]
    PARALLEL1 --> ACCOUNTS[Create Accounts]
    PARALLEL1 --> EQUIPMENT[Order Equipment]
    PARALLEL1 --> TRAINING[Assign Training]

    subgraph "Document Collection"
        DOCS --> DOC1[Request Tax Forms]
        DOC1 --> DOC2[Request Emergency Contacts]
        DOC2 --> DOC3[Request Bank Details]
        DOC3 --> DOC_VERIFY[Verify Documents]
    end

    subgraph "Account Provisioning"
        ACCOUNTS --> ACC1[Create Email Account]
        ACC1 --> ACC2[Create Slack Account]
        ACC2 --> ACC3[Create Project Tools]
        ACC3 --> ACC4[Set Permissions]
    end

    subgraph "Equipment Setup"
        EQUIPMENT --> EQ1[Order Laptop]
        EQ1 --> EQ2[Order Accessories]
        EQ2 --> EQ3[Schedule Delivery]
    end

    subgraph "Training Assignment"
        TRAINING --> TR1[Assign Compliance Training]
        TR1 --> TR2[Assign Role-Specific Training]
        TR2 --> TR3[Schedule Orientation]
    end

    DOC_VERIFY --> SYNC1[Sync Point]
    ACC4 --> SYNC1
    EQ3 --> SYNC1
    TR3 --> SYNC1

    SYNC1 --> VERIFY_ALL{All Complete?}

    VERIFY_ALL -->|No| REMIND[Send Reminder<br/>To HR & Employee]
    REMIND --> WAIT[Wait 2 Days]
    WAIT --> VERIFY_ALL

    VERIFY_ALL -->|Yes| ACCESS[Grant System Access]

    ACCESS --> DAY1[Send Day 1 Instructions<br/>24 Hours Before Start]

    DAY1 --> ONBOARD[First Day Onboarding]

    ONBOARD --> CHECK30[30-Day Check-in Survey]
    CHECK30 --> CHECK90[90-Day Review Workflow]

    CHECK90 --> COMPLETE[Onboarding Complete]

    style START fill:#4CAF50,stroke:#333,color:#fff
    style PARALLEL1 fill:#EA4B71,stroke:#333,color:#fff
    style COMPLETE fill:#4CAF50,stroke:#333,color:#fff
```

### Onboarding Task Tracking

```mermaid
gantt
    title Employee Onboarding Timeline
    dateFormat YYYY-MM-DD
    axisFormat %m/%d

    section Pre-Start
    Offer Accepted              :milestone, 2024-01-01, 0d
    Send Welcome Email          :active, 2024-01-01, 1d
    Collect Documents           :active, 2024-01-01, 7d
    Background Check            :crit, 2024-01-01, 10d

    section Week 1
    Create Accounts             :2024-01-08, 3d
    Order Equipment             :2024-01-08, 5d
    Assign Training             :2024-01-10, 3d

    section Start Date
    First Day                   :milestone, 2024-01-15, 0d
    Orientation                 :active, 2024-01-15, 1d
    Manager Welcome             :active, 2024-01-15, 1d
    IT Setup                    :2024-01-15, 2d

    section Month 1
    Role Training               :2024-01-16, 14d
    Shadow Team Members         :2024-01-20, 10d
    30-Day Check-in            :milestone, 2024-02-15, 0d

    section Month 3
    Independent Work            :2024-02-16, 30d
    90-Day Review              :milestone, 2024-04-15, 0d
    Onboarding Complete        :milestone, 2024-04-15, 0d
```

---

## Multi-Stage Approval Workflow

### Dynamic Approval Routing

```mermaid
flowchart TD
    START[Request Submitted] --> VALIDATE[Validate Request Data]

    VALIDATE -->|Invalid| REJECT1[Return to Submitter<br/>With Errors]
    VALIDATE -->|Valid| AMOUNT{Request Amount?}

    AMOUNT -->|< $1,000| MGR[Manager Approval Required]
    AMOUNT -->|$1,000 - $10,000| DIR[Director Approval Required]
    AMOUNT -->|> $10,000| VP[VP Approval Required]

    MGR --> MGR_REVIEW{Manager Decision}
    MGR_REVIEW -->|Approved| COMPLETE[Process Request]
    MGR_REVIEW -->|Rejected| NOTIFY_REJECT[Notify Submitter]
    MGR_REVIEW -->|More Info| REQUEST_INFO[Request Additional Info]

    DIR --> DIR_REVIEW{Director Decision}
    DIR_REVIEW -->|Approved| COMPLETE
    DIR_REVIEW -->|Rejected| NOTIFY_REJECT
    DIR_REVIEW -->|Escalate| VP

    VP --> VP_NOTIFY[Notify VP<br/>+ Email + Slack]
    VP_NOTIFY --> VP_REVIEW{VP Decision}
    VP_REVIEW -->|Approved| FINANCE[Finance Review]
    VP_REVIEW -->|Rejected| NOTIFY_REJECT

    FINANCE --> FINANCE_CHECK{Budget Available?}
    FINANCE_CHECK -->|Yes| COMPLETE
    FINANCE_CHECK -->|No| BUDGET_REVIEW[Budget Committee Review]

    BUDGET_REVIEW --> COMPLETE

    REQUEST_INFO --> SUBMITTER[Submitter Updates Request]
    SUBMITTER --> VALIDATE

    COMPLETE --> EXECUTE[Execute Approved Action]
    EXECUTE --> NOTIFY_SUCCESS[Notify All Parties]

    NOTIFY_REJECT --> LOG[Log Rejection Reason]
    NOTIFY_SUCCESS --> LOG[Log Approval Trail]

    LOG --> ARCHIVE[Archive Request]

    style START fill:#4CAF50,stroke:#333,color:#fff
    style COMPLETE fill:#4CAF50,stroke:#333,color:#fff
    style NOTIFY_REJECT fill:#F44336,stroke:#333,color:#fff
    style VP_NOTIFY fill:#FF9800,stroke:#333,color:#fff
```

### Approval Escalation

```mermaid
sequenceDiagram
    participant Sub as Submitter
    participant Sys as n8n System
    participant Mgr as Manager
    participant Dir as Director
    participant VP as VP

    Sub->>Sys: Submit Request ($15,000)
    activate Sys

    Sys->>Sys: Determine Approval Path<br/>Amount > $10k = VP Required

    Sys->>Mgr: Request Manager Approval
    Note over Mgr: 24 Hour SLA

    alt Manager Responds in Time
        activate Mgr
        Mgr->>Sys: Approved
        deactivate Mgr
        Sys->>Dir: Request Director Approval
        activate Dir
        Dir->>Sys: Approved
        deactivate Dir
        Sys->>VP: Request VP Approval
    else No Response After 24 Hours
        Note over Sys: Escalation Triggered
        Sys->>Mgr: Reminder: Overdue Approval
        Sys->>Dir: Manager Non-Response<br/>Skip to Director
        activate Dir
        Dir->>Sys: Approved
        deactivate Dir
        Sys->>VP: Request VP Approval
    end

    activate VP
    Note over VP: 48 Hour SLA<br/>High Priority

    VP->>Sys: Approved with Conditions
    deactivate VP

    Sys->>Sub: Request Approved ✓<br/>With Conditions
    deactivate Sys

    Note over Sub,VP: Audit Trail Logged
```

---

## Event-Driven Architecture

### Event Bus Pattern

```mermaid
graph TB
    subgraph "Event Sources"
        E1[User Actions<br/>Signup, Login, etc]
        E2[System Events<br/>Workflow Complete, etc]
        E3[External Webhooks<br/>Payments, Shipments, etc]
        E4[Scheduled Events<br/>Daily Reports, etc]
    end

    subgraph "Event Bus (n8n)"
        BUS[Central Event Router<br/>Receives & Distributes Events]
        FILTER[Event Filter<br/>By Type, Priority, etc]
    end

    subgraph "Event Processors"
        P1[Notification Service<br/>Email, SMS, Slack]
        P2[Analytics Service<br/>Track & Aggregate]
        P3[Integration Service<br/>Sync to External Systems]
        P4[Workflow Triggers<br/>Start Workflows]
    end

    subgraph "Event Store"
        STORE[(Event Log<br/>Immutable Record)]
        REPLAY[Replay Capability]
    end

    E1 -->|Emit Event| BUS
    E2 -->|Emit Event| BUS
    E3 -->|Emit Event| BUS
    E4 -->|Emit Event| BUS

    BUS --> FILTER
    BUS --> STORE

    FILTER -->|Route| P1
    FILTER -->|Route| P2
    FILTER -->|Route| P3
    FILTER -->|Route| P4

    STORE --> REPLAY
    REPLAY -.->|Reprocess| FILTER

    style BUS fill:#EA4B71,stroke:#333,color:#fff
    style STORE fill:#4CAF50,stroke:#333,color:#fff
    style FILTER fill:#FF9800,stroke:#333,color:#fff
```

### Event Flow Example: Order Placed

```mermaid
sequenceDiagram
    participant Cust as Customer
    participant Shop as Shopify
    participant Bus as Event Bus (n8n)
    participant Inv as Inventory Service
    participant Email as Email Service
    participant Analytics as Analytics
    participant Accounting as Accounting

    Cust->>Shop: Place Order
    Shop->>Bus: Event: order.placed

    activate Bus
    Note over Bus: Event Received<br/>{<br/>  type: "order.placed"<br/>  order_id: "12345"<br/>  amount: 99.99<br/>  timestamp: "..."<br/>}

    Bus->>Bus: Store Event in Log

    par Parallel Processing
        Bus->>Inv: Update Inventory
        activate Inv
        Inv-->>Bus: Inventory Updated
        deactivate Inv

    and
        Bus->>Email: Send Order Confirmation
        activate Email
        Email-->>Bus: Email Sent
        deactivate Email

    and
        Bus->>Analytics: Track Purchase
        activate Analytics
        Analytics-->>Bus: Tracked
        deactivate Analytics

    and
        Bus->>Accounting: Create Invoice
        activate Accounting
        Accounting-->>Bus: Invoice Created
        deactivate Accounting
    end

    Bus-->>Shop: All Processors Complete
    deactivate Bus

    Note over Bus: Event Processing Complete<br/>Total Time: 2.3 seconds
```

### Microservices Coordination

```mermaid
graph LR
    subgraph "Services"
        USER[User Service<br/>Authentication]
        ORDER[Order Service<br/>Order Management]
        INVENTORY[Inventory Service<br/>Stock Management]
        PAYMENT[Payment Service<br/>Transactions]
        SHIPPING[Shipping Service<br/>Fulfillment]
        NOTIF[Notification Service<br/>Communications]
    end

    subgraph "n8n Orchestration"
        ORCHESTRATOR[Workflow Orchestrator<br/>Coordinates Services]
        SAGA[Saga Manager<br/>Distributed Transactions]
    end

    USER -->|User Events| ORCHESTRATOR
    ORDER -->|Order Events| ORCHESTRATOR
    INVENTORY -->|Stock Events| ORCHESTRATOR
    PAYMENT -->|Payment Events| ORCHESTRATOR
    SHIPPING -->|Shipping Events| ORCHESTRATOR

    ORCHESTRATOR -->|Commands| USER
    ORCHESTRATOR -->|Commands| ORDER
    ORCHESTRATOR -->|Commands| INVENTORY
    ORCHESTRATOR -->|Commands| PAYMENT
    ORCHESTRATOR -->|Commands| SHIPPING
    ORCHESTRATOR -->|Notify| NOTIF

    ORCHESTRATOR --> SAGA
    SAGA -.->|Compensate| ORDER
    SAGA -.->|Compensate| INVENTORY
    SAGA -.->|Compensate| PAYMENT

    style ORCHESTRATOR fill:#EA4B71,stroke:#333,color:#fff
    style SAGA fill:#FF9800,stroke:#333,color:#fff
```

---

## Financial Reconciliation Process

### Automated Reconciliation Workflow

```mermaid
flowchart TD
    START[Scheduled: Daily 3 AM] --> EXTRACT[Extract Data from Sources]

    subgraph "Data Extraction"
        BANK[Get Bank Transactions<br/>Last 24 Hours]
        STRIPE[Get Stripe Charges<br/>Last 24 Hours]
        SHOP[Get Shopify Orders<br/>Last 24 Hours]
        QB[Get QuickBooks Entries<br/>Last 24 Hours]
    end

    EXTRACT --> BANK
    EXTRACT --> STRIPE
    EXTRACT --> SHOP
    EXTRACT --> QB

    BANK --> NORMALIZE[Normalize Data<br/>Common Format]
    STRIPE --> NORMALIZE
    SHOP --> NORMALIZE
    QB --> NORMALIZE

    NORMALIZE --> MATCH[Matching Algorithm]

    MATCH --> AUTO{Auto-Match<br/>Confidence > 95%?}

    AUTO -->|Yes| MATCHED[Mark as Reconciled<br/>Update Records]
    AUTO -->|No| MANUAL[Flag for Manual Review]

    MATCHED --> CHECK_COMPLETE{All<br/>Transactions<br/>Matched?}

    MANUAL --> NOTIFY[Notify Finance Team<br/>Send Discrepancy Report]
    NOTIFY --> REVIEW[Manual Review Queue]
    REVIEW --> RESOLVE{Resolved?}

    RESOLVE -->|Yes| MATCHED
    RESOLVE -->|No| ESCALATE[Escalate to Manager]

    CHECK_COMPLETE -->|Yes| REPORT[Generate Reconciliation Report]
    CHECK_COMPLETE -->|No| DISCREPANCY[Log Discrepancies]

    DISCREPANCY --> REPORT

    REPORT --> STORE[Store Report]
    STORE --> EMAIL[Email Report to Finance]
    EMAIL --> COMPLETE[Reconciliation Complete]

    style START fill:#4CAF50,stroke:#333,color:#fff
    style MATCHED fill:#4CAF50,stroke:#333,color:#fff
    style MANUAL fill:#FF9800,stroke:#333,color:#fff
    style ESCALATE fill:#F44336,stroke:#333,color:#fff
    style COMPLETE fill:#4CAF50,stroke:#333,color:#fff
```

### Transaction Matching Logic

```mermaid
flowchart TD
    TRANS[Transaction<br/>Needs Matching] --> ID{Exact<br/>Transaction ID<br/>Match?}

    ID -->|Yes| CONF100[100% Confidence<br/>Auto-Match]

    ID -->|No| AMOUNT{Amount<br/>Match?}

    AMOUNT -->|No| UNMATCHED[Cannot Auto-Match]

    AMOUNT -->|Yes| DATE{Date Within<br/>2 Days?}

    DATE -->|No| UNMATCHED

    DATE -->|Yes| DESC{Description<br/>Similarity<br/>> 80%?}

    DESC -->|No| CONF50[50% Confidence<br/>Manual Review]

    DESC -->|Yes| MERCHANT{Merchant<br/>Name<br/>Match?}

    MERCHANT -->|Yes| CONF95[95% Confidence<br/>Auto-Match with Note]
    MERCHANT -->|No| CONF70[70% Confidence<br/>Manual Review]

    CONF100 --> RECONCILED[Reconciled ✓]
    CONF95 --> RECONCILED
    CONF70 --> QUEUE[Manual Review Queue]
    CONF50 --> QUEUE
    UNMATCHED --> INVESTIGATE[Investigate<br/>Missing Transaction]

    style CONF100 fill:#4CAF50,stroke:#333,color:#fff
    style CONF95 fill:#8BC34A,stroke:#333,color:#fff
    style CONF70 fill:#FF9800,stroke:#333,color:#fff
    style CONF50 fill:#FF9800,stroke:#333,color:#fff
    style UNMATCHED fill:#F44336,stroke:#333,color:#fff
```

### Reconciliation Dashboard

```mermaid
graph TB
    subgraph "Reconciliation Metrics"
        TOTAL["Total Transactions: 1,247"]
        AUTO_MATCH["Auto-Matched: 1,189 (95.3%)"]
        MANUAL_REV["Manual Review: 47 (3.8%)"]
        UNMATCHED_TRANS["Unmatched: 11 (0.9%)"]
    end

    subgraph "Status by Source"
        BANK_STATUS[Bank: 425 transactions<br/>✓ 420 matched<br/>⚠ 3 review<br/>✗ 2 unmatched]

        STRIPE_STATUS[Stripe: 398 transactions<br/>✓ 392 matched<br/>⚠ 4 review<br/>✗ 2 unmatched]

        SHOP_STATUS[Shopify: 424 transactions<br/>✓ 377 matched<br/>⚠ 40 review<br/>✗ 7 unmatched]
    end

    subgraph "Actions Required"
        ACTION1[Review 47 flagged transactions]
        ACTION2[Investigate 11 unmatched]
        ACTION3[Update merchant mapping]
    end

    TOTAL --> AUTO_MATCH
    TOTAL --> MANUAL_REV
    TOTAL --> UNMATCHED_TRANS

    style AUTO_MATCH fill:#4CAF50,stroke:#333,color:#fff
    style MANUAL_REV fill:#FF9800,stroke:#333,color:#fff
    style UNMATCHED_TRANS fill:#F44336,stroke:#333,color:#fff
```

---

## Quick Reference: Business Process Patterns

### Common Workflow Patterns

```mermaid
graph TB
    subgraph "Sequential Pattern"
        SEQ1[Step 1] --> SEQ2[Step 2] --> SEQ3[Step 3]
    end

    subgraph "Parallel Pattern"
        PAR_START[Start] --> PAR1[Task 1]
        PAR_START --> PAR2[Task 2]
        PAR_START --> PAR3[Task 3]
        PAR1 --> PAR_END[Sync Point]
        PAR2 --> PAR_END
        PAR3 --> PAR_END
    end

    subgraph "Conditional Pattern"
        COND_START[Start] --> COND{Condition?}
        COND -->|Yes| COND_YES[Path A]
        COND -->|No| COND_NO[Path B]
    end

    subgraph "Loop Pattern"
        LOOP_START[Start] --> LOOP_DO[Process Item]
        LOOP_DO --> LOOP_CHECK{More Items?}
        LOOP_CHECK -->|Yes| LOOP_DO
        LOOP_CHECK -->|No| LOOP_END[End]
    end

    style SEQ1 fill:#4CAF50,stroke:#333,color:#fff
    style PAR_START fill:#FF9800,stroke:#333,color:#fff
    style COND fill:#2196F3,stroke:#333,color:#fff
    style LOOP_DO fill:#9C27B0,stroke:#333,color:#fff
```

---

**Use these diagrams to design and implement complex business automation workflows!**

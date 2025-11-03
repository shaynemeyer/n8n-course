# Week 11: Self-Hosting and Administration - Visual Guides

This document contains all visual diagrams for Week 11 content.

## Table of Contents

1. [Docker Deployment Architecture](#docker-deployment-architecture)
2. [Environment Deployment Flow](#environment-deployment-flow)
3. [Database Architecture](#database-architecture)
4. [SSL/TLS Configuration Flow](#ssltls-configuration-flow)
5. [User Management & RBAC](#user-management--rbac)
6. [Queue Mode Architecture](#queue-mode-architecture)
7. [Backup & Disaster Recovery](#backup--disaster-recovery)

---

## Docker Deployment Architecture

### Simple Single-Instance Setup

```mermaid
graph TB
    subgraph "Docker Host"
        subgraph "n8n Container"
            N8N[n8n Application<br/>Port 5678]
        end

        subgraph "Database Container"
            DB[(PostgreSQL<br/>Port 5432)]
        end

        subgraph "Redis Container"
            REDIS[(Redis<br/>Port 6379)]
        end
    end

    Internet[Internet] -->|HTTPS:443| NGINX
    NGINX[Nginx Reverse Proxy] -->|HTTP:5678| N8N
    N8N -->|Database Queries| DB
    N8N -->|Cache/Queue| REDIS

    style N8N fill:#EA4B71,stroke:#333,color:#fff
    style DB fill:#336791,stroke:#333,color:#fff
    style REDIS fill:#DC382D,stroke:#333,color:#fff
    style NGINX fill:#009639,stroke:#333,color:#fff
```

### Production Multi-Instance Setup

```mermaid
graph TB
    subgraph "Load Balancer"
        LB[Load Balancer<br/>HAProxy/Nginx]
    end

    subgraph "Application Tier"
        N8N1[n8n Main<br/>Instance 1]
        N8N2[n8n Main<br/>Instance 2]
        N8N3[n8n Main<br/>Instance 3]
    end

    subgraph "Worker Tier"
        W1[Worker 1<br/>Queue Mode]
        W2[Worker 2<br/>Queue Mode]
        W3[Worker 3<br/>Queue Mode]
    end

    subgraph "Data Tier"
        DB[(PostgreSQL<br/>Primary)]
        DBR[(PostgreSQL<br/>Replica)]
        REDIS[(Redis<br/>Cluster)]
    end

    Internet[Internet] -->|HTTPS| LB
    LB --> N8N1
    LB --> N8N2
    LB --> N8N3

    N8N1 -->|Queue Jobs| REDIS
    N8N2 -->|Queue Jobs| REDIS
    N8N3 -->|Queue Jobs| REDIS

    REDIS -->|Dequeue| W1
    REDIS -->|Dequeue| W2
    REDIS -->|Dequeue| W3

    W1 -->|Write| DB
    W2 -->|Write| DB
    W3 -->|Write| DB

    DB -->|Replicate| DBR

    N8N1 -->|Read| DBR
    N8N2 -->|Read| DBR
    N8N3 -->|Read| DBR

    style LB fill:#4CAF50,stroke:#333,color:#fff
    style N8N1 fill:#EA4B71,stroke:#333,color:#fff
    style N8N2 fill:#EA4B71,stroke:#333,color:#fff
    style N8N3 fill:#EA4B71,stroke:#333,color:#fff
    style W1 fill:#FF9800,stroke:#333,color:#fff
    style W2 fill:#FF9800,stroke:#333,color:#fff
    style W3 fill:#FF9800,stroke:#333,color:#fff
    style DB fill:#336791,stroke:#333,color:#fff
    style DBR fill:#5B8BB3,stroke:#333,color:#fff
    style REDIS fill:#DC382D,stroke:#333,color:#fff
```

---

## Environment Deployment Flow

### Development to Production Pipeline

```mermaid
flowchart LR
    subgraph Development
        DEV[Development<br/>Environment]
        DEV_TEST[Local Testing]
    end

    subgraph Staging
        STG[Staging<br/>Environment]
        STG_TEST[Integration Tests]
        STG_APPROVE[Approval Gate]
    end

    subgraph Production
        PROD_CANARY[Canary<br/>10% Traffic]
        PROD_FULL[Full Production<br/>100% Traffic]
        PROD_MONITOR[Monitoring]
    end

    DEV -->|Push Changes| DEV_TEST
    DEV_TEST -->|Tests Pass| STG
    STG -->|Deploy| STG_TEST
    STG_TEST -->|Tests Pass| STG_APPROVE
    STG_APPROVE -->|Manual Approval| PROD_CANARY
    PROD_CANARY -->|Monitor Metrics| PROD_MONITOR
    PROD_MONITOR -->|No Issues| PROD_FULL
    PROD_MONITOR -->|Issues Detected| ROLLBACK[Rollback]
    ROLLBACK -->|Revert| STG

    style DEV fill:#4CAF50,stroke:#333,color:#fff
    style STG fill:#FF9800,stroke:#333,color:#fff
    style PROD_CANARY fill:#FFE082,stroke:#333
    style PROD_FULL fill:#EA4B71,stroke:#333,color:#fff
    style ROLLBACK fill:#F44336,stroke:#333,color:#fff
```

### Environment Configuration Hierarchy

```mermaid
graph TD
    subgraph "Configuration Precedence (High to Low)"
        ENV[Environment Variables<br/>Highest Priority]
        FILE[Configuration File<br/>.env or config.json]
        DEFAULT[Default Values<br/>Lowest Priority]
    end

    ENV -->|Overrides| FILE
    FILE -->|Overrides| DEFAULT

    subgraph "Environment-Specific Configs"
        ENV --> DEV_ENV[Development<br/>Debug: ON<br/>Auth: Relaxed]
        ENV --> STG_ENV[Staging<br/>Debug: ON<br/>Auth: Strict]
        ENV --> PROD_ENV[Production<br/>Debug: OFF<br/>Auth: Strict<br/>SSL: Required]
    end

    style ENV fill:#EA4B71,stroke:#333,color:#fff
    style FILE fill:#FF9800,stroke:#333,color:#fff
    style DEFAULT fill:#9E9E9E,stroke:#333,color:#fff
    style PROD_ENV fill:#4CAF50,stroke:#333,color:#fff
```

---

## Database Architecture

### PostgreSQL Schema Overview

```mermaid
erDiagram
    USERS ||--o{ WORKFLOWS : creates
    USERS ||--o{ CREDENTIALS : owns
    WORKFLOWS ||--o{ EXECUTIONS : has
    WORKFLOWS ||--o{ WORKFLOW_SHARING : shared_via
    EXECUTIONS ||--o{ EXECUTION_DATA : contains
    CREDENTIALS ||--o{ WORKFLOWS : used_in

    USERS {
        uuid id PK
        string email
        string password_hash
        string role
        timestamp created_at
        timestamp updated_at
    }

    WORKFLOWS {
        uuid id PK
        uuid user_id FK
        string name
        json nodes
        json connections
        boolean active
        timestamp created_at
        timestamp updated_at
    }

    EXECUTIONS {
        uuid id PK
        uuid workflow_id FK
        string status
        timestamp started_at
        timestamp finished_at
        json data
    }

    CREDENTIALS {
        uuid id PK
        uuid user_id FK
        string name
        string type
        text encrypted_data
        timestamp created_at
    }

    EXECUTION_DATA {
        uuid id PK
        uuid execution_id FK
        string node_name
        int item_index
        json data
    }

    WORKFLOW_SHARING {
        uuid id PK
        uuid workflow_id FK
        uuid user_id FK
        string permission
    }
```

### Database Connection Pooling

```mermaid
flowchart TB
    subgraph "n8n Instances"
        N1[n8n Instance 1]
        N2[n8n Instance 2]
        N3[n8n Instance 3]
    end

    subgraph "Connection Pool"
        POOL[Connection Pool<br/>Max: 100 connections<br/>Min: 10 connections<br/>Idle Timeout: 30s]
    end

    subgraph "Database"
        DB[(PostgreSQL<br/>Max Connections: 200)]
    end

    N1 -->|Request Connection| POOL
    N2 -->|Request Connection| POOL
    N3 -->|Request Connection| POOL

    POOL -->|Active Connections| DB
    POOL -->|Reuse| POOL

    style POOL fill:#FF9800,stroke:#333,color:#fff
    style DB fill:#336791,stroke:#333,color:#fff
```

---

## SSL/TLS Configuration Flow

### Certificate Generation and Installation

```mermaid
flowchart TD
    START[Start SSL Setup] --> CHOOSE{Certificate Type?}

    CHOOSE -->|Let's Encrypt| LE[Install Certbot]
    CHOOSE -->|Self-Signed| SS[Generate Self-Signed Cert]
    CHOOSE -->|Commercial CA| CA[Purchase from CA]

    LE --> LE_GEN[certbot certonly<br/>--standalone<br/>-d n8n.example.com]
    LE_GEN --> CERTS[Certificates Generated<br/>/etc/letsencrypt/live/]

    SS --> SS_GEN[openssl req -x509<br/>-newkey rsa:4096<br/>-nodes]
    SS_GEN --> CERTS

    CA --> CA_VERIFY[Complete Verification]
    CA_VERIFY --> CERTS

    CERTS --> CONFIG[Configure n8n/Nginx]
    CONFIG --> TEST[Test SSL<br/>https://n8n.example.com]

    TEST -->|Success| RENEW[Set Up Auto-Renewal]
    TEST -->|Failed| DEBUG[Debug Configuration]
    DEBUG --> CONFIG

    RENEW --> CRON[Cron Job: certbot renew]
    CRON --> MONITOR[Monitor Expiry]

    style START fill:#4CAF50,stroke:#333,color:#fff
    style CERTS fill:#EA4B71,stroke:#333,color:#fff
    style RENEW fill:#FF9800,stroke:#333,color:#fff
    style TEST fill:#2196F3,stroke:#333,color:#fff
```

### SSL/TLS Handshake Flow

```mermaid
sequenceDiagram
    participant Client
    participant Nginx
    participant n8n

    Client->>Nginx: HTTPS Request (443)
    activate Nginx

    Nginx->>Client: SSL Certificate
    Client->>Nginx: Verify Certificate
    Note over Client,Nginx: TLS Handshake<br/>Establish Encryption

    Nginx->>n8n: HTTP Request (5678)
    activate n8n
    Note over Nginx,n8n: Internal Network<br/>Can be HTTP

    n8n->>n8n: Process Request
    n8n-->>Nginx: HTTP Response
    deactivate n8n

    Nginx-->>Client: HTTPS Response
    deactivate Nginx
    Note over Client,Nginx: Encrypted Response
```

---

## User Management & RBAC

### Role-Based Access Control Model

```mermaid
graph TB
    subgraph "User Roles"
        OWNER[Owner<br/>Full System Access]
        ADMIN[Admin<br/>User & Workflow Management]
        MEMBER[Member<br/>Create & Edit Own Workflows]
        GUEST[Guest<br/>View Only]
    end

    subgraph "Permissions"
        P1[Create Workflows]
        P2[Edit Workflows]
        P3[Delete Workflows]
        P4[Execute Workflows]
        P5[Manage Users]
        P6[Manage Credentials]
        P7[System Settings]
        P8[View Executions]
    end

    OWNER --> P1
    OWNER --> P2
    OWNER --> P3
    OWNER --> P4
    OWNER --> P5
    OWNER --> P6
    OWNER --> P7
    OWNER --> P8

    ADMIN --> P1
    ADMIN --> P2
    ADMIN --> P3
    ADMIN --> P4
    ADMIN --> P5
    ADMIN --> P8

    MEMBER --> P1
    MEMBER --> P2
    MEMBER --> P4
    MEMBER --> P8

    GUEST --> P8

    style OWNER fill:#EA4B71,stroke:#333,color:#fff
    style ADMIN fill:#FF9800,stroke:#333,color:#fff
    style MEMBER fill:#4CAF50,stroke:#333,color:#fff
    style GUEST fill:#9E9E9E,stroke:#333,color:#fff
```

### Workflow Sharing Model

```mermaid
flowchart TD
    WORKFLOW[Workflow] --> OWNER_CHECK{Owner?}

    OWNER_CHECK -->|Yes| FULL[Full Access<br/>Edit, Delete, Share]
    OWNER_CHECK -->|No| SHARED_CHECK{Shared With You?}

    SHARED_CHECK -->|Yes| PERM_CHECK{Permission Level?}
    SHARED_CHECK -->|No| NO_ACCESS[No Access]

    PERM_CHECK -->|Edit| EDIT[Can Edit & Execute]
    PERM_CHECK -->|View| VIEW[Can View & Execute]
    PERM_CHECK -->|Execute| EXEC[Can Execute Only]

    style FULL fill:#EA4B71,stroke:#333,color:#fff
    style EDIT fill:#FF9800,stroke:#333,color:#fff
    style VIEW fill:#4CAF50,stroke:#333,color:#fff
    style EXEC fill:#2196F3,stroke:#333,color:#fff
    style NO_ACCESS fill:#F44336,stroke:#333,color:#fff
```

---

## Queue Mode Architecture

### Main Process vs Worker Processes

```mermaid
graph TB
    subgraph "Main Process (Web)"
        UI[Web Interface<br/>Port 5678]
        API[REST API]
        WEBHOOK[Webhook Endpoints]
    end

    subgraph "Queue (Redis)"
        QUEUE[(Redis Queue<br/>Job Storage)]
    end

    subgraph "Worker Processes"
        W1[Worker 1<br/>Executes Jobs]
        W2[Worker 2<br/>Executes Jobs]
        W3[Worker 3<br/>Executes Jobs]
        W4[Worker N<br/>Executes Jobs]
    end

    subgraph "Database"
        DB[(PostgreSQL<br/>Workflow & Execution Data)]
    end

    UI -->|Create/Trigger Workflow| API
    API -->|Queue Job| QUEUE
    WEBHOOK -->|Incoming Event| API

    QUEUE -->|Dequeue| W1
    QUEUE -->|Dequeue| W2
    QUEUE -->|Dequeue| W3
    QUEUE -->|Dequeue| W4

    W1 -->|Update Status| DB
    W2 -->|Update Status| DB
    W3 -->|Update Status| DB
    W4 -->|Update Status| DB

    UI -->|Read Status| DB

    style UI fill:#EA4B71,stroke:#333,color:#fff
    style QUEUE fill:#DC382D,stroke:#333,color:#fff
    style W1 fill:#FF9800,stroke:#333,color:#fff
    style W2 fill:#FF9800,stroke:#333,color:#fff
    style W3 fill:#FF9800,stroke:#333,color:#fff
    style W4 fill:#FF9800,stroke:#333,color:#fff
    style DB fill:#336791,stroke:#333,color:#fff
```

### Workflow Execution with Queue Mode

```mermaid
sequenceDiagram
    participant User
    participant Main as Main Process
    participant Queue as Redis Queue
    participant Worker as Worker Process
    participant DB as Database

    User->>Main: Trigger Workflow
    activate Main

    Main->>DB: Create Execution Record
    DB-->>Main: Execution ID

    Main->>Queue: Queue Job
    Note over Queue: Job: {executionId, workflowId}

    Main-->>User: Execution Started (ID)
    deactivate Main

    Worker->>Queue: Poll for Jobs
    activate Worker
    Queue-->>Worker: Dequeue Job

    Worker->>DB: Get Workflow Data
    DB-->>Worker: Workflow Definition

    Worker->>Worker: Execute Workflow
    Note over Worker: Run all nodes<br/>Process data

    Worker->>DB: Update Execution Status
    Note over DB: Status: Success/Failed

    deactivate Worker

    User->>Main: Check Execution Status
    Main->>DB: Get Execution Data
    DB-->>Main: Execution Result
    Main-->>User: Display Result
```

---

## Backup & Disaster Recovery

### Backup Strategy

```mermaid
flowchart TB
    subgraph "Backup Sources"
        DB[(Database<br/>PostgreSQL)]
        FILES[Workflow Files<br/>JSON Exports]
        CREDS[Encrypted Credentials]
        CONFIG[Configuration Files<br/>.env, docker-compose]
    end

    subgraph "Backup Process"
        SCHEDULE[Scheduled Backups<br/>Daily: 2 AM UTC]

        SCHEDULE -->|pg_dump| DB_BACKUP[Database Backup<br/>n8n_backup_YYYYMMDD.sql]
        SCHEDULE -->|Export All| WF_BACKUP[Workflow Backup<br/>workflows_YYYYMMDD.json]
        SCHEDULE -->|Copy| CONFIG_BACKUP[Config Backup<br/>config_YYYYMMDD.tar.gz]
    end

    subgraph "Backup Storage"
        LOCAL[Local Storage<br/>7 days retention]
        S3[Cloud Storage<br/>S3/Azure/GCS<br/>90 days retention]
        ARCHIVE[Archive Storage<br/>Glacier<br/>7 years retention]
    end

    DB --> SCHEDULE
    FILES --> SCHEDULE
    CONFIG --> SCHEDULE

    DB_BACKUP --> LOCAL
    WF_BACKUP --> LOCAL
    CONFIG_BACKUP --> LOCAL

    LOCAL -->|Daily Sync| S3
    S3 -->|Monthly Archive| ARCHIVE

    style SCHEDULE fill:#FF9800,stroke:#333,color:#fff
    style LOCAL fill:#4CAF50,stroke:#333,color:#fff
    style S3 fill:#2196F3,stroke:#333,color:#fff
    style ARCHIVE fill:#9E9E9E,stroke:#333,color:#fff
```

### Disaster Recovery Process

```mermaid
flowchart TD
    DISASTER[Disaster Detected<br/>System Down] --> ASSESS{Assess Damage}

    ASSESS -->|Database Corrupted| DB_RESTORE[Restore Database]
    ASSESS -->|Files Lost| FILE_RESTORE[Restore Files]
    ASSESS -->|Complete Loss| FULL_RESTORE[Full System Restore]

    subgraph "Database Recovery"
        DB_RESTORE --> DB_LATEST[Get Latest Backup]
        DB_LATEST --> DB_APPLY[Apply SQL Backup]
        DB_APPLY --> DB_VERIFY[Verify Data Integrity]
    end

    subgraph "File Recovery"
        FILE_RESTORE --> FILE_LATEST[Get Latest Backup]
        FILE_LATEST --> FILE_IMPORT[Import Workflows]
        FILE_IMPORT --> FILE_VERIFY[Verify Workflows]
    end

    subgraph "Full Recovery"
        FULL_RESTORE --> INFRA[Rebuild Infrastructure]
        INFRA --> DB_RESTORE2[Restore Database]
        DB_RESTORE2 --> FILE_RESTORE2[Restore Files]
        FILE_RESTORE2 --> CONFIG_RESTORE[Restore Configuration]
        CONFIG_RESTORE --> FULL_VERIFY[Full System Test]
    end

    DB_VERIFY --> ONLINE[Bring System Online]
    FILE_VERIFY --> ONLINE
    FULL_VERIFY --> ONLINE

    ONLINE --> MONITOR[Monitor System]
    MONITOR --> POST[Post-Incident Review]

    style DISASTER fill:#F44336,stroke:#333,color:#fff
    style ONLINE fill:#4CAF50,stroke:#333,color:#fff
    style MONITOR fill:#FF9800,stroke:#333,color:#fff
```

### Recovery Time Objectives (RTO)

```mermaid
gantt
    title Recovery Time Objectives by Disaster Type
    dateFormat HH:mm
    axisFormat %H:%M

    section Database Corruption
    Detect Issue           :00:00, 15m
    Get Latest Backup     :00:15, 10m
    Restore Database      :00:25, 30m
    Verify & Test         :00:55, 15m
    System Online         :01:10, 1m

    section File Loss
    Detect Issue          :00:00, 10m
    Get Backups          :00:10, 15m
    Restore Files        :00:25, 20m
    Import Workflows     :00:45, 15m
    System Online        :01:00, 1m

    section Complete Loss
    Detect & Assess      :00:00, 30m
    Provision Infra      :00:30, 60m
    Restore All Data     :01:30, 45m
    Configure System     :02:15, 30m
    Full Testing         :02:45, 30m
    System Online        :03:15, 1m
```

---

## Monitoring and Health Checks

### Health Check Flow

```mermaid
flowchart LR
    MONITOR[Health Check<br/>Every 30s] --> CHECK_DB{Database<br/>Connection?}

    CHECK_DB -->|OK| CHECK_REDIS{Redis<br/>Connection?}
    CHECK_DB -->|Failed| ALERT_DB[Alert: DB Down]

    CHECK_REDIS -->|OK| CHECK_QUEUE{Queue<br/>Processing?}
    CHECK_REDIS -->|Failed| ALERT_REDIS[Alert: Redis Down]

    CHECK_QUEUE -->|OK| CHECK_DISK{Disk<br/>Space?}
    CHECK_QUEUE -->|Stalled| ALERT_QUEUE[Alert: Queue Stalled]

    CHECK_DISK -->|OK| HEALTHY[System Healthy]
    CHECK_DISK -->|Low| ALERT_DISK[Alert: Low Disk Space]

    ALERT_DB --> NOTIFY[Send Notifications]
    ALERT_REDIS --> NOTIFY
    ALERT_QUEUE --> NOTIFY
    ALERT_DISK --> NOTIFY

    NOTIFY --> SLACK[Slack]
    NOTIFY --> EMAIL[Email]
    NOTIFY --> PAGERDUTY[PagerDuty]

    style HEALTHY fill:#4CAF50,stroke:#333,color:#fff
    style ALERT_DB fill:#F44336,stroke:#333,color:#fff
    style ALERT_REDIS fill:#F44336,stroke:#333,color:#fff
    style ALERT_QUEUE fill:#FF9800,stroke:#333,color:#fff
    style ALERT_DISK fill:#FF9800,stroke:#333,color:#fff
```

---

## Quick Reference Commands

### Docker Compose Production Setup

```yaml
# Example docker-compose.yml structure
services:
  n8n:
    image: n8nio/n8n
    environment:
      - DB_TYPE=postgresdb
      - EXECUTIONS_MODE=queue

  postgres:
    image: postgres:14

  redis:
    image: redis:7-alpine

  worker:
    image: n8nio/n8n
    command: worker
```

### Essential Commands

```bash
# View logs
docker-compose logs -f n8n

# Database backup
docker exec n8n-postgres pg_dump -U n8n n8n > backup.sql

# Restart services
docker-compose restart n8n

# Scale workers
docker-compose up -d --scale worker=5

# Check health
curl http://localhost:5678/healthz
```

---

**Use these diagrams as references when implementing your production n8n deployment!**

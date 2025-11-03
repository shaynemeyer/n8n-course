# Week 12: Security and Compliance - Visual Guides

This document contains all visual diagrams for Week 12 content.

## Table of Contents

1. [Credential Encryption Flow](#credential-encryption-flow)
2. [Environment Variable Hierarchy](#environment-variable-hierarchy)
3. [API Key Rotation Process](#api-key-rotation-process)
4. [GDPR Compliance Workflow](#gdpr-compliance-workflow)
5. [Audit Trail Architecture](#audit-trail-architecture)
6. [Security Layers](#security-layers)

---

## Credential Encryption Flow

### Credential Storage and Retrieval

```mermaid
sequenceDiagram
    participant User
    participant n8n
    participant Encryption as Encryption Service
    participant DB as Database

    Note over User,DB: Storing Credentials

    User->>n8n: Enter Credentials<br/>(API Key, Password, etc)
    activate n8n

    n8n->>Encryption: Encrypt Data
    activate Encryption
    Note over Encryption: Uses AES-256-GCM<br/>with Master Key

    Encryption-->>n8n: Encrypted Data
    deactivate Encryption

    n8n->>DB: Store Encrypted Credentials
    Note over DB: {<br/>  id: uuid<br/>  type: "api"<br/>  data: encrypted_blob<br/>  iv: initialization_vector<br/>}

    DB-->>n8n: Stored Successfully
    n8n-->>User: Credentials Saved
    deactivate n8n

    Note over User,DB: Retrieving Credentials

    User->>n8n: Execute Workflow
    activate n8n

    n8n->>DB: Get Encrypted Credentials
    DB-->>n8n: Encrypted Data + IV

    n8n->>Encryption: Decrypt Data
    activate Encryption
    Note over Encryption: Uses Master Key + IV

    Encryption-->>n8n: Decrypted Credentials
    deactivate Encryption

    n8n->>n8n: Use in Memory Only<br/>(Never Logged)

    n8n-->>User: Workflow Executed
    deactivate n8n
```

### Encryption Key Management

```mermaid
graph TB
    subgraph "Key Hierarchy"
        MASTER[Master Encryption Key<br/>N8N_ENCRYPTION_KEY]
        DEK[Data Encryption Keys<br/>Per-Environment]
    end

    subgraph "Key Storage Options"
        ENV_VAR[Environment Variable<br/>Simple but Less Secure]
        FILE[Key File<br/>Mounted Volume]
        VAULT[Secrets Manager<br/>AWS Secrets/Vault]
        HSM[Hardware Security Module<br/>Highest Security]
    end

    subgraph "Protected Data"
        CREDS[Credentials]
        TOKENS[OAuth Tokens]
        KEYS[API Keys]
        SECRETS[Webhook Secrets]
    end

    MASTER --> DEK
    DEK --> CREDS
    DEK --> TOKENS
    DEK --> KEYS
    DEK --> SECRETS

    MASTER -.->|Stored in| ENV_VAR
    MASTER -.->|Stored in| FILE
    MASTER -.->|Stored in| VAULT
    MASTER -.->|Stored in| HSM

    style MASTER fill:#EA4B71,stroke:#333,color:#fff
    style VAULT fill:#4CAF50,stroke:#333,color:#fff
    style HSM fill:#2196F3,stroke:#333,color:#fff
    style ENV_VAR fill:#FF9800,stroke:#333,color:#fff
```

---

## Environment Variable Hierarchy

### Configuration Precedence

```mermaid
flowchart TD
    START[n8n Starts] --> CHECK1{Environment<br/>Variables?}

    CHECK1 -->|Yes| ENV[Use Environment Variables<br/>Highest Priority]
    CHECK1 -->|No| CHECK2{Config File<br/>Exists?}

    CHECK2 -->|Yes| FILE[Use Config File Values]
    CHECK2 -->|No| DEFAULT[Use Default Values<br/>Lowest Priority]

    ENV --> MERGE[Merge Configuration]
    FILE --> MERGE
    DEFAULT --> MERGE

    MERGE --> FINAL[Final Configuration]

    style ENV fill:#EA4B71,stroke:#333,color:#fff
    style FILE fill:#FF9800,stroke:#333,color:#fff
    style DEFAULT fill:#9E9E9E,stroke:#333,color:#fff
    style FINAL fill:#4CAF50,stroke:#333,color:#fff
```

### Environment Variable Categories

```mermaid
graph TB
    subgraph "Core Configuration"
        DB[N8N_PROTOCOL<br/>N8N_HOST<br/>N8N_PORT]
    end

    subgraph "Database"
        DB_VARS[DB_TYPE<br/>DB_POSTGRESDB_HOST<br/>DB_POSTGRESDB_PORT<br/>DB_POSTGRESDB_DATABASE<br/>DB_POSTGRESDB_USER<br/>DB_POSTGRESDB_PASSWORD]
    end

    subgraph "Security"
        SEC[N8N_ENCRYPTION_KEY<br/>N8N_BASIC_AUTH_ACTIVE<br/>N8N_BASIC_AUTH_USER<br/>N8N_BASIC_AUTH_PASSWORD<br/>N8N_JWT_AUTH_ACTIVE]
    end

    subgraph "Execution"
        EXEC[EXECUTIONS_MODE<br/>EXECUTIONS_TIMEOUT<br/>EXECUTIONS_TIMEOUT_MAX<br/>QUEUE_BULL_REDIS_HOST]
    end

    subgraph "Features"
        FEAT[N8N_HIDE_USAGE_PAGE<br/>N8N_DISABLE_PRODUCTION_MAIN_PROCESS<br/>N8N_PERSONALIZATION_ENABLED]
    end

    subgraph "External Services"
        EXT[WEBHOOK_URL<br/>N8N_METRICS<br/>N8N_LOG_LEVEL<br/>N8N_LOG_OUTPUT]
    end

    style SEC fill:#EA4B71,stroke:#333,color:#fff
    style DB_VARS fill:#336791,stroke:#333,color:#fff
    style EXEC fill:#FF9800,stroke:#333,color:#fff
```

### Environment-Specific Configuration

```mermaid
graph LR
    subgraph "Development"
        DEV_ENV[N8N_LOG_LEVEL=debug<br/>N8N_BASIC_AUTH_ACTIVE=false<br/>EXECUTIONS_MODE=regular<br/>N8N_METRICS=false]
    end

    subgraph "Staging"
        STG_ENV[N8N_LOG_LEVEL=info<br/>N8N_BASIC_AUTH_ACTIVE=true<br/>EXECUTIONS_MODE=queue<br/>N8N_METRICS=true<br/>SSL_REQUIRED=true]
    end

    subgraph "Production"
        PROD_ENV[N8N_LOG_LEVEL=warn<br/>N8N_BASIC_AUTH_ACTIVE=true<br/>EXECUTIONS_MODE=queue<br/>N8N_METRICS=true<br/>SSL_REQUIRED=true<br/>QUEUE_HEALTH_CHECK_ACTIVE=true<br/>N8N_SECURE_COOKIE=true]
    end

    DEV_ENV -->|Promote| STG_ENV
    STG_ENV -->|Promote| PROD_ENV

    style DEV_ENV fill:#4CAF50,stroke:#333,color:#fff
    style STG_ENV fill:#FF9800,stroke:#333,color:#fff
    style PROD_ENV fill:#EA4B71,stroke:#333,color:#fff
```

---

## API Key Rotation Process

### Zero-Downtime Key Rotation

```mermaid
sequenceDiagram
    participant Admin
    participant n8n
    participant Service as External Service
    participant DB as Database

    Note over Admin,DB: Phase 1: Add New Key

    Admin->>Service: Generate New API Key
    Service-->>Admin: New Key (Key B)

    Admin->>n8n: Add New Credential (Key B)
    n8n->>DB: Store Encrypted Key B
    Note over DB: Now has:<br/>Key A (active)<br/>Key B (active)

    Note over Admin,DB: Phase 2: Dual Key Period (24-48 hours)

    n8n->>Service: Request with Key A
    Service-->>n8n: Success ✓

    n8n->>Service: Request with Key B
    Service-->>n8n: Success ✓

    Note over n8n,Service: Both keys work<br/>Monitor for issues

    Note over Admin,DB: Phase 3: Switch to New Key

    Admin->>n8n: Update Workflows to Use Key B
    n8n->>DB: Update Workflow References

    loop Test All Workflows
        n8n->>Service: Request with Key B
        Service-->>n8n: Success ✓
    end

    Note over Admin,DB: Phase 4: Remove Old Key

    Admin->>Service: Revoke Key A
    Service-->>Admin: Key A Revoked

    Admin->>n8n: Delete Old Credential (Key A)
    n8n->>DB: Delete Encrypted Key A

    Note over Admin,DB: Rotation Complete<br/>Zero Downtime ✓
```

### Automated Rotation Schedule

```mermaid
gantt
    title API Key Rotation Timeline (90-Day Cycle)
    dateFormat YYYY-MM-DD
    axisFormat %m/%d

    section Current Key
    Key A Active                :active, 2024-01-01, 90d

    section Rotation Process
    Generate Key B              :milestone, 2024-03-25, 0d
    Dual Key Period            :crit, 2024-03-25, 7d
    Switch to Key B            :milestone, 2024-04-01, 0d
    Revoke Key A               :milestone, 2024-04-02, 0d

    section New Key
    Key B Active               :active, 2024-04-01, 90d

    section Next Rotation
    Generate Key C             :milestone, 2024-06-25, 0d
```

### Rotation Workflow

```mermaid
flowchart TD
    START[Rotation Scheduled<br/>30 Days Before Expiry] --> NOTIFY[Notify Admin Team]

    NOTIFY --> GEN[Generate New Key<br/>in External Service]

    GEN --> ADD[Add New Key to n8n<br/>Mark as Secondary]

    ADD --> TEST{Test New Key<br/>in Dev/Staging}

    TEST -->|Failed| DEBUG[Debug Issues]
    DEBUG --> TEST

    TEST -->|Success| DUAL[Enable Dual-Key Mode<br/>48 Hour Window]

    DUAL --> MONITOR{Monitor<br/>Both Keys}

    MONITOR -->|Issues Detected| ROLLBACK[Keep Old Key Active]
    ROLLBACK --> DEBUG

    MONITOR -->|All Good| SWITCH[Switch Primary Key]

    SWITCH --> UPDATE[Update All Workflows<br/>to Use New Key]

    UPDATE --> VERIFY[Verify All Workflows<br/>Execute Successfully]

    VERIFY -->|Failed| EMERGENCY[Emergency Rollback]
    EMERGENCY --> ROLLBACK

    VERIFY -->|Success| REVOKE[Revoke Old Key<br/>After 7 Days]

    REVOKE --> COMPLETE[Rotation Complete]

    COMPLETE --> SCHEDULE[Schedule Next Rotation<br/>60 Days]

    style START fill:#4CAF50,stroke:#333,color:#fff
    style DUAL fill:#FF9800,stroke:#333,color:#fff
    style SWITCH fill:#EA4B71,stroke:#333,color:#fff
    style EMERGENCY fill:#F44336,stroke:#333,color:#fff
    style COMPLETE fill:#4CAF50,stroke:#333,color:#fff
```

---

## GDPR Compliance Workflow

### Data Subject Request Handling

```mermaid
flowchart TD
    REQUEST[Data Subject Request<br/>GDPR Article 15-22] --> IDENTIFY{Request Type?}

    IDENTIFY -->|Access Request| ACCESS[Article 15:<br/>Right to Access]
    IDENTIFY -->|Rectification| RECTIFY[Article 16:<br/>Right to Rectification]
    IDENTIFY -->|Erasure| ERASURE[Article 17:<br/>Right to be Forgotten]
    IDENTIFY -->|Restriction| RESTRICT[Article 18:<br/>Right to Restriction]
    IDENTIFY -->|Portability| PORT[Article 20:<br/>Right to Portability]

    ACCESS --> VERIFY[Verify Identity]
    RECTIFY --> VERIFY
    ERASURE --> VERIFY
    RESTRICT --> VERIFY
    PORT --> VERIFY

    VERIFY -->|Invalid| REJECT[Reject Request<br/>Log Attempt]

    VERIFY -->|Valid| SEARCH[Search All Systems<br/>for Personal Data]

    SEARCH --> FOUND{Data Found?}

    FOUND -->|No| NO_DATA[Inform: No Data Found]

    FOUND -->|Yes| PROCESS{Process Request}

    PROCESS -->|Access| EXPORT[Export All Data<br/>Send to User]
    PROCESS -->|Rectify| UPDATE[Update Records<br/>Notify User]
    PROCESS -->|Erase| DELETE[Delete Personal Data<br/>Keep Audit Log]
    PROCESS -->|Restrict| FLAG[Flag Account<br/>Restrict Processing]
    PROCESS -->|Port| STRUCTURED[Export in<br/>Machine-Readable Format]

    EXPORT --> LOG[Log Completion]
    UPDATE --> LOG
    DELETE --> LOG
    FLAG --> LOG
    STRUCTURED --> LOG

    LOG --> NOTIFY[Notify Data Subject<br/>Within 30 Days]

    style REQUEST fill:#EA4B71,stroke:#333,color:#fff
    style DELETE fill:#F44336,stroke:#333,color:#fff
    style NOTIFY fill:#4CAF50,stroke:#333,color:#fff
```

### Personal Data in n8n

```mermaid
graph TB
    subgraph "Personal Data Locations"
        USERS[User Accounts<br/>Email, Name, IP]
        WORKFLOWS[Workflow Data<br/>Customer Info in Executions]
        LOGS[Execution Logs<br/>May Contain PII]
        CREDS[Credentials<br/>User-Specific Auth]
    end

    subgraph "GDPR Requirements"
        LAWFUL[Lawful Basis<br/>for Processing]
        CONSENT[User Consent<br/>When Required]
        MINIMAL[Data Minimization<br/>Only What's Needed]
        SECURITY[Security Measures<br/>Encryption, Access Control]
        RETENTION[Retention Policy<br/>Delete After Period]
    end

    subgraph "Implementation"
        ENCRYPT[Data Encryption<br/>At Rest & In Transit]
        ACCESS_CTRL[Access Controls<br/>RBAC, Authentication]
        AUDIT[Audit Trails<br/>Who Accessed What]
        ANON[Anonymization<br/>For Analytics]
        PURGE[Automated Purge<br/>After Retention Period]
    end

    USERS --> LAWFUL
    WORKFLOWS --> CONSENT
    LOGS --> MINIMAL
    CREDS --> SECURITY

    LAWFUL --> ENCRYPT
    CONSENT --> ACCESS_CTRL
    MINIMAL --> AUDIT
    SECURITY --> ANON
    RETENTION --> PURGE

    style USERS fill:#EA4B71,stroke:#333,color:#fff
    style SECURITY fill:#4CAF50,stroke:#333,color:#fff
    style ENCRYPT fill:#2196F3,stroke:#333,color:#fff
```

### Data Retention Policy

```mermaid
gantt
    title Data Retention Timeline
    dateFormat YYYY-MM-DD
    axisFormat %m/%d

    section User Data
    Active User Account        :active, 2024-01-01, 365d
    Account Deletion Requested :milestone, 2025-01-01, 0d
    Grace Period (30 days)    :crit, 2025-01-01, 30d
    Permanent Deletion        :milestone, 2025-01-31, 0d

    section Execution Data
    Workflow Executions       :active, 2024-01-01, 90d
    Auto-Purge After 90 Days  :milestone, 2024-04-01, 0d

    section Audit Logs
    Security Audit Logs       :active, 2024-01-01, 2555d
    Keep 7 Years (Compliance) :2024-01-01, 2555d

    section Anonymized Analytics
    Anonymized Usage Data     :active, 2024-01-01, 3650d
    Keep Indefinitely         :2024-01-01, 3650d
```

---

## Audit Trail Architecture

### Audit Event Logging

```mermaid
sequenceDiagram
    participant User
    participant n8n
    participant Audit as Audit Service
    participant DB as Audit Database
    participant SIEM as SIEM/Log Aggregator

    User->>n8n: Perform Action<br/>(Login, Edit Workflow, etc)
    activate n8n

    n8n->>n8n: Execute Action

    n8n->>Audit: Log Audit Event
    activate Audit

    Note over Audit: Create Audit Record:<br/>{<br/>  timestamp<br/>  userId<br/>  action<br/>  resource<br/>  ip_address<br/>  user_agent<br/>  result<br/>}

    Audit->>DB: Store Audit Record
    Note over DB: Immutable Log<br/>Append Only

    Audit->>SIEM: Forward to SIEM
    Note over SIEM: Splunk, ELK, etc<br/>For Analysis

    Audit-->>n8n: Logged
    deactivate Audit

    n8n-->>User: Action Complete
    deactivate n8n
```

### Auditable Events

```mermaid
graph TB
    subgraph "User Actions"
        LOGIN[User Login/Logout]
        FAILED[Failed Login Attempts]
        PWD[Password Changes]
        ROLE[Role Changes]
    end

    subgraph "Workflow Actions"
        CREATE[Workflow Created]
        EDIT[Workflow Modified]
        DELETE[Workflow Deleted]
        EXECUTE[Workflow Executed]
        SHARE[Workflow Shared]
    end

    subgraph "Credential Actions"
        CRED_ADD[Credential Added]
        CRED_EDIT[Credential Modified]
        CRED_DEL[Credential Deleted]
        CRED_USE[Credential Used]
    end

    subgraph "System Actions"
        CONFIG[Configuration Changed]
        BACKUP[Backup Created]
        RESTORE[Restore Performed]
        UPDATE[System Updated]
    end

    subgraph "Audit Log"
        AUDIT_DB[(Audit Database<br/>Immutable Records)]
    end

    LOGIN --> AUDIT_DB
    FAILED --> AUDIT_DB
    PWD --> AUDIT_DB
    ROLE --> AUDIT_DB

    CREATE --> AUDIT_DB
    EDIT --> AUDIT_DB
    DELETE --> AUDIT_DB
    EXECUTE --> AUDIT_DB
    SHARE --> AUDIT_DB

    CRED_ADD --> AUDIT_DB
    CRED_EDIT --> AUDIT_DB
    CRED_DEL --> AUDIT_DB
    CRED_USE --> AUDIT_DB

    CONFIG --> AUDIT_DB
    BACKUP --> AUDIT_DB
    RESTORE --> AUDIT_DB
    UPDATE --> AUDIT_DB

    style AUDIT_DB fill:#EA4B71,stroke:#333,color:#fff
    style FAILED fill:#F44336,stroke:#333,color:#fff
    style CRED_USE fill:#FF9800,stroke:#333,color:#fff
```

### Audit Log Schema

```mermaid
erDiagram
    AUDIT_LOGS {
        uuid id PK
        timestamp event_time
        string user_id
        string user_email
        string action
        string resource_type
        string resource_id
        string ip_address
        string user_agent
        string result
        json metadata
        timestamp created_at
    }

    AUDIT_LOGS ||--o{ AUDIT_METADATA : contains

    AUDIT_METADATA {
        uuid id PK
        uuid audit_log_id FK
        string key
        text value
    }
```

---

## Security Layers

### Defense in Depth

```mermaid
graph TB
    subgraph "Perimeter Security"
        FIREWALL[Firewall<br/>Port Restrictions]
        WAF[Web Application Firewall<br/>OWASP Protection]
        DDOS[DDoS Protection<br/>Rate Limiting]
    end

    subgraph "Network Security"
        VPC[Virtual Private Cloud<br/>Isolated Network]
        TLS[TLS/SSL Encryption<br/>In Transit]
        VPN[VPN Access<br/>For Admin]
    end

    subgraph "Application Security"
        AUTH[Authentication<br/>OAuth2, SAML, Basic]
        AUTHZ[Authorization<br/>RBAC]
        SESSION[Session Management<br/>Secure Cookies]
    end

    subgraph "Data Security"
        ENCRYPT_REST[Encryption at Rest<br/>AES-256]
        ENCRYPT_TRANSIT[Encryption in Transit<br/>TLS 1.3]
        BACKUP_ENCRYPT[Encrypted Backups]
    end

    subgraph "Infrastructure Security"
        HARDENED[Hardened OS<br/>Minimal Services]
        UPDATES[Auto Security Updates]
        MONITORING[Security Monitoring<br/>IDS/IPS]
    end

    subgraph "Operational Security"
        AUDIT_LOG[Audit Logging]
        INCIDENT[Incident Response<br/>Plan]
        TRAINING[Security Training]
    end

    Internet[Internet] --> FIREWALL
    FIREWALL --> WAF
    WAF --> DDOS
    DDOS --> VPC

    VPC --> TLS
    TLS --> AUTH
    AUTH --> AUTHZ
    AUTHZ --> SESSION

    SESSION --> ENCRYPT_REST
    ENCRYPT_REST --> ENCRYPT_TRANSIT
    ENCRYPT_TRANSIT --> BACKUP_ENCRYPT

    BACKUP_ENCRYPT --> HARDENED
    HARDENED --> UPDATES
    UPDATES --> MONITORING

    MONITORING --> AUDIT_LOG
    AUDIT_LOG --> INCIDENT
    INCIDENT --> TRAINING

    style FIREWALL fill:#F44336,stroke:#333,color:#fff
    style AUTH fill:#FF9800,stroke:#333,color:#fff
    style ENCRYPT_REST fill:#4CAF50,stroke:#333,color:#fff
    style MONITORING fill:#2196F3,stroke:#333,color:#fff
```

### Security Checklist

```mermaid
flowchart TD
    START[Security Assessment] --> PERIMETER{Perimeter<br/>Secure?}

    PERIMETER -->|No| FIX_PERIMETER[Configure Firewall<br/>Enable WAF<br/>Set Rate Limits]
    PERIMETER -->|Yes| NETWORK{Network<br/>Secure?}

    FIX_PERIMETER --> NETWORK

    NETWORK -->|No| FIX_NETWORK[Enable TLS<br/>Configure VPC<br/>Set up VPN]
    NETWORK -->|Yes| APP{Application<br/>Secure?}

    FIX_NETWORK --> APP

    APP -->|No| FIX_APP[Enable Auth<br/>Configure RBAC<br/>Secure Sessions]
    APP -->|Yes| DATA{Data<br/>Secure?}

    FIX_APP --> DATA

    DATA -->|No| FIX_DATA[Enable Encryption<br/>Secure Backups<br/>Rotate Keys]
    DATA -->|Yes| INFRA{Infrastructure<br/>Secure?}

    FIX_DATA --> INFRA

    INFRA -->|No| FIX_INFRA[Harden OS<br/>Enable Updates<br/>Deploy Monitoring]
    INFRA -->|Yes| OPS{Operations<br/>Secure?}

    FIX_INFRA --> OPS

    OPS -->|No| FIX_OPS[Enable Audit Logs<br/>Create Incident Plan<br/>Train Team]
    OPS -->|Yes| COMPLETE[Security Assessment<br/>Complete ✓]

    FIX_OPS --> COMPLETE

    style START fill:#4CAF50,stroke:#333,color:#fff
    style COMPLETE fill:#4CAF50,stroke:#333,color:#fff
    style FIX_PERIMETER fill:#F44336,stroke:#333,color:#fff
    style FIX_NETWORK fill:#F44336,stroke:#333,color:#fff
    style FIX_APP fill:#F44336,stroke:#333,color:#fff
    style FIX_DATA fill:#F44336,stroke:#333,color:#fff
    style FIX_INFRA fill:#F44336,stroke:#333,color:#fff
    style FIX_OPS fill:#F44336,stroke:#333,color:#fff
```

---

## Compliance Frameworks

### Compliance Mapping

```mermaid
graph LR
    subgraph "n8n Security Controls"
        ENC[Data Encryption]
        ACCESS[Access Control]
        AUDIT[Audit Logging]
        BACKUP[Backup/Recovery]
        MONITOR[Monitoring]
    end

    subgraph "GDPR"
        GDPR_ART32[Article 32:<br/>Security Measures]
        GDPR_ART30[Article 30:<br/>Record Keeping]
        GDPR_ART33[Article 33:<br/>Breach Notification]
    end

    subgraph "SOC 2"
        SOC2_CC6[CC6:<br/>Logical Access]
        SOC2_CC7[CC7:<br/>System Operations]
        SOC2_CC8[CC8:<br/>Change Management]
    end

    subgraph "ISO 27001"
        ISO_A9[A.9: Access Control]
        ISO_A10[A.10: Cryptography]
        ISO_A12[A.12: Operations Security]
    end

    ENC --> GDPR_ART32
    ENC --> ISO_A10

    ACCESS --> GDPR_ART32
    ACCESS --> SOC2_CC6
    ACCESS --> ISO_A9

    AUDIT --> GDPR_ART30
    AUDIT --> SOC2_CC7
    AUDIT --> ISO_A12

    BACKUP --> SOC2_CC7

    MONITOR --> GDPR_ART33
    MONITOR --> SOC2_CC8
    MONITOR --> ISO_A12

    style ENC fill:#4CAF50,stroke:#333,color:#fff
    style ACCESS fill:#2196F3,stroke:#333,color:#fff
    style AUDIT fill:#FF9800,stroke:#333,color:#fff
```

---

## Quick Security Commands

### Security Verification Scripts

```bash
# Check encryption key
echo $N8N_ENCRYPTION_KEY | wc -c
# Should be 32+ characters

# Verify TLS certificate
openssl s_client -connect n8n.example.com:443 -showcerts

# Check file permissions
ls -la ~/.n8n/
# Should not be world-readable

# Audit recent logins
docker-compose logs n8n | grep "Login"

# Check for exposed credentials in logs
docker-compose logs n8n | grep -i "password\|api.*key\|secret"
# Should find nothing!

# Verify database encryption
docker exec n8n-db psql -U n8n -c "SELECT pg_database.datname, pg_tablespace.spcname FROM pg_database JOIN pg_tablespace ON pg_database.dattablespace = pg_tablespace.oid;"
```

---

**Use these diagrams to implement comprehensive security and compliance measures in your n8n deployment!**

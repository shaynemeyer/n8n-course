# Week 1: Getting Started - Visual Guides

This document contains all visual diagrams for Week 1 content.

## Table of Contents

1. [n8n Interface Overview](#n8n-interface-overview)
2. [Installation Options](#installation-options)
3. [First Workflow Examples](#first-workflow-examples)
4. [Node Types Overview](#node-types-overview)
5. [Data Flow Basics](#data-flow-basics)

---

## n8n Interface Overview

### Main Interface Components

```mermaid
graph TB
    subgraph "n8n Workflow Editor"
        MENU[Top Menu Bar<br/>File, Execute, Settings]
        SIDEBAR[Left Sidebar<br/>Node Panel]
        CANVAS[Central Canvas<br/>Workflow Design Area]
        SETTINGS[Right Panel<br/>Node Settings & Parameters]
        EXECUTIONS[Bottom Panel<br/>Execution History & Output]
    end

    MENU --> CANVAS
    SIDEBAR --> CANVAS
    CANVAS --> SETTINGS
    CANVAS --> EXECUTIONS

    style MENU fill:#EA4B71,stroke:#333,color:#fff
    style SIDEBAR fill:#4CAF50,stroke:#333,color:#fff
    style CANVAS fill:#fafafa,stroke:#333
    style SETTINGS fill:#2196F3,stroke:#333,color:#fff
    style EXECUTIONS fill:#FF9800,stroke:#333,color:#fff
```

### Node Panel Organization

```mermaid
graph TB
    PANEL[Node Panel] --> TRIGGERS[Triggers<br/>Start workflows]
    PANEL --> CORE[Core Nodes<br/>Logic & Data]
    PANEL --> INTEGRATIONS[Integrations<br/>External Services]
    PANEL --> HELPERS[Helpers<br/>Utilities & Tools]

    TRIGGERS --> T1[Schedule Trigger<br/>Webhook<br/>Manual Trigger]

    CORE --> C1[IF<br/>Switch<br/>Set<br/>Code]

    INTEGRATIONS --> I1[HTTP Request<br/>Gmail<br/>Slack<br/>Google Sheets]

    HELPERS --> H1[Merge<br/>Split<br/>Sort<br/>Wait]

    style PANEL fill:#EA4B71,stroke:#333,color:#fff
    style TRIGGERS fill:#4CAF50,stroke:#333,color:#fff
    style INTEGRATIONS fill:#2196F3,stroke:#333,color:#fff
```

---

## Installation Options

### Choosing Your Installation Method

```mermaid
flowchart TD
    START{What's Your Goal?}

    START -->|Quick Start<br/>No Setup| CLOUD[n8n Cloud]
    START -->|Learning & Testing<br/>Local Development| NPX[npx/npm]
    START -->|Production Deployment<br/>Self-Hosted| DOCKER[Docker]
    START -->|Desktop Application<br/>Easy Setup| DESKTOP[Desktop App]

    CLOUD --> CLOUD_STEPS[1. Visit n8n.cloud<br/>2. Sign Up<br/>3. Start Building<br/>✓ Immediate Access]

    NPX --> NPX_STEPS[1. Install Node.js<br/>2. Run: npx n8n<br/>3. Open localhost:5678<br/>✓ Quick Local Setup]

    DOCKER --> DOCKER_STEPS[1. Install Docker<br/>2. Run: docker run n8nio/n8n<br/>3. Configure persistence<br/>✓ Production Ready]

    DESKTOP --> DESKTOP_STEPS[1. Download installer<br/>2. Install app<br/>3. Launch n8n<br/>✓ No Terminal Needed]

    style CLOUD fill:#4CAF50,stroke:#333,color:#fff
    style NPX fill:#2196F3,stroke:#333,color:#fff
    style DOCKER fill:#FF9800,stroke:#333,color:#fff
    style DESKTOP fill:#9C27B0,stroke:#333,color:#fff
```

### Installation Comparison Matrix

```mermaid
graph TB
    subgraph "Cloud (n8n.cloud)"
        CLOUD_PROS[✓ No setup required<br/>✓ Auto-updates<br/>✓ Managed hosting<br/>✗ Subscription cost<br/>✗ Data in cloud]
    end

    subgraph "npm/npx"
        NPM_PROS[✓ Quick start<br/>✓ Local development<br/>✓ Free<br/>✗ Manual updates<br/>✗ Not for production]
    end

    subgraph "Docker"
        DOCKER_PROS[✓ Production-ready<br/>✓ Easy scaling<br/>✓ Isolated environment<br/>✗ Requires Docker knowledge<br/>✗ More setup]
    end

    subgraph "Desktop App"
        DESKTOP_PROS[✓ User-friendly<br/>✓ No terminal<br/>✓ Local & secure<br/>✗ Desktop only<br/>✗ Limited for servers]
    end

    style CLOUD_PROS fill:#4CAF50,stroke:#333,color:#fff
    style DOCKER_PROS fill:#FF9800,stroke:#333,color:#fff
```

---

## First Workflow Examples

### Hello World Workflow

```mermaid
flowchart LR
    MANUAL[Manual Trigger<br/>Click to Start] -->|Empty Array| SET[Set Node<br/>Add Data]
    SET -->|JSON Data| OUTPUT[Display Result]

    style MANUAL fill:#EA4B71,stroke:#333,color:#fff
    style SET fill:#4CAF50,stroke:#333,color:#fff
    style OUTPUT fill:#2196F3,stroke:#333,color:#fff
```

**Data Flow:**
```json
Manual Trigger: []
↓
Set Node: [
  {
    "message": "Hello World!",
    "timestamp": "2024-01-01T12:00:00Z"
  }
]
```

### Scheduled Email Notification

```mermaid
flowchart LR
    SCHEDULE[Schedule Trigger<br/>Daily at 9 AM] -->|Triggers| HTTP[HTTP Request<br/>Get Weather Data]
    HTTP -->|Weather Data| EMAIL[Send Email<br/>Weather Report]

    style SCHEDULE fill:#FF9800,stroke:#333,color:#fff
    style HTTP fill:#2196F3,stroke:#333,color:#fff
    style EMAIL fill:#4CAF50,stroke:#333,color:#fff
```

**Workflow Steps:**
1. **Schedule Trigger** fires daily at 9 AM
2. **HTTP Request** fetches weather data from API
3. **Send Email** sends formatted weather report

### API to Spreadsheet

```mermaid
flowchart LR
    WEBHOOK[Webhook Trigger<br/>Receive Data] -->|JSON Payload| PROCESS[Code Node<br/>Transform Data]
    PROCESS -->|Formatted Rows| SHEETS[Google Sheets<br/>Append Row]
    SHEETS -->|Success| RESPOND[Webhook Response<br/>Return Status]

    style WEBHOOK fill:#EA4B71,stroke:#333,color:#fff
    style PROCESS fill:#FF9800,stroke:#333,color:#fff
    style SHEETS fill:#4CAF50,stroke:#333,color:#fff
    style RESPOND fill:#2196F3,stroke:#333,color:#fff
```

---

## Node Types Overview

### Core Node Categories

```mermaid
mindmap
  root((n8n Nodes))
    Trigger Nodes
      Manual Trigger
      Schedule Trigger
      Webhook Trigger
      Email Trigger
    Data Nodes
      HTTP Request
      Database Nodes
      File Operations
      API Integrations
    Logic Nodes
      IF Condition
      Switch Router
      Loop
      Merge/Split
    Transform Nodes
      Set Values
      Code/Function
      Filter
      Sort
    Output Nodes
      Email
      Slack
      Database Write
      File Write
```

### Trigger Nodes Explained

```mermaid
graph TB
    subgraph "Trigger Nodes (Start Workflows)"
        MANUAL[Manual Trigger<br/>👤 User clicks Execute]
        SCHEDULE[Schedule Trigger<br/>⏰ Time-based execution]
        WEBHOOK[Webhook Trigger<br/>🔗 External HTTP requests]
        EMAIL[Email Trigger<br/>📧 New email received]
    end

    subgraph "When to Use"
        MANUAL --> M_USE[• Testing workflows<br/>• Manual operations<br/>• One-time tasks]
        SCHEDULE --> S_USE[• Daily reports<br/>• Periodic backups<br/>• Scheduled posts]
        WEBHOOK --> W_USE[• API integrations<br/>• External events<br/>• Real-time triggers]
        EMAIL --> E_USE[• Email automation<br/>• Support tickets<br/>• Notifications]
    end

    style MANUAL fill:#EA4B71,stroke:#333,color:#fff
    style SCHEDULE fill:#FF9800,stroke:#333,color:#fff
    style WEBHOOK fill:#4CAF50,stroke:#333,color:#fff
    style EMAIL fill:#2196F3,stroke:#333,color:#fff
```

### Regular Nodes Explained

```mermaid
graph TB
    subgraph "Regular Nodes (Process Data)"
        IF[IF Node<br/>Conditional Logic]
        SET[Set Node<br/>Add/Modify Data]
        HTTP[HTTP Request<br/>API Calls]
        CODE[Code Node<br/>Custom Logic]
    end

    subgraph "Function"
        IF --> IF_DESC[Split workflow based on<br/>conditions<br/>Example: if amount > 100]
        SET --> SET_DESC[Add or modify fields<br/>Example: add timestamp,<br/>calculate total]
        HTTP --> HTTP_DESC[Make API requests<br/>Example: fetch user data,<br/>send webhook]
        CODE --> CODE_DESC[Custom JavaScript/Python<br/>Example: complex calculations,<br/>data transformation]
    end

    style IF fill:#FF9800,stroke:#333,color:#fff
    style SET fill:#4CAF50,stroke:#333,color:#fff
    style HTTP fill:#2196F3,stroke:#333,color:#fff
    style CODE fill:#9C27B0,stroke:#333,color:#fff
```

---

## Data Flow Basics

### How Data Flows Through Nodes

```mermaid
sequenceDiagram
    participant T as Trigger Node
    participant N1 as Node 1
    participant N2 as Node 2
    participant N3 as Node 3

    Note over T: Workflow starts

    T->>N1: Pass data array<br/>[{item1}, {item2}]
    activate N1
    Note over N1: Process each item<br/>Transform data
    N1->>N2: Output array<br/>[{processed1}, {processed2}]
    deactivate N1

    activate N2
    Note over N2: Further processing<br/>Add fields
    N2->>N3: Enhanced array<br/>[{enhanced1}, {enhanced2}]
    deactivate N2

    activate N3
    Note over N3: Final action<br/>(send email, save, etc)
    N3->>N3: Complete ✓
    deactivate N3
```

### Data Structure: JSON Format

```mermaid
graph TB
    subgraph "n8n Data Format"
        ARRAY[Data is ALWAYS an Array<br/>Even for single items]
        ARRAY --> ITEMS[Each item is an Object<br/>with json property]
    end

    subgraph "Example Data"
        ITEMS --> EX1["Item 1:<br/>name: John<br/>email: john@example.com"]
        ITEMS --> EX2["Item 2:<br/>name: Jane<br/>email: jane@example.com"]
    end

    style ARRAY fill:#EA4B71,stroke:#333,color:#fff
    style EX1 fill:#4CAF50,stroke:#333,color:#fff
    style EX2 fill:#4CAF50,stroke:#333,color:#fff
```

### Multiple Input/Output Paths

```mermaid
flowchart TD
    START[Start Node<br/>Data: 10 items] --> SPLIT{IF Node<br/>Amount > 100?}

    SPLIT -->|True Output| HIGH[High Value Path<br/>3 items]
    SPLIT -->|False Output| LOW[Low Value Path<br/>7 items]

    HIGH --> NOTIFY[Send Manager Alert<br/>3 notifications]
    LOW --> LOG[Log to Database<br/>7 entries]

    NOTIFY --> MERGE[Merge Node<br/>Combine Results]
    LOG --> MERGE

    MERGE --> FINAL[Final Output<br/>10 items processed]

    style START fill:#4CAF50,stroke:#333,color:#fff
    style SPLIT fill:#FF9800,stroke:#333,color:#fff
    style HIGH fill:#EA4B71,stroke:#333,color:#fff
    style LOW fill:#2196F3,stroke:#333,color:#fff
    style FINAL fill:#4CAF50,stroke:#333,color:#fff
```

---

## Common Workflow Patterns

### Simple Linear Workflow

```mermaid
flowchart LR
    A[Trigger] --> B[Process] --> C[Output]

    style A fill:#4CAF50,stroke:#333,color:#fff
    style B fill:#2196F3,stroke:#333,color:#fff
    style C fill:#FF9800,stroke:#333,color:#fff
```
**Use Case:** Simple automation tasks (e.g., scheduled backup)

### Conditional Workflow

```mermaid
flowchart TD
    A[Trigger] --> B{Condition?}
    B -->|Yes| C[Action A]
    B -->|No| D[Action B]

    style A fill:#4CAF50,stroke:#333,color:#fff
    style B fill:#FF9800,stroke:#333,color:#fff
    style C fill:#2196F3,stroke:#333,color:#fff
    style D fill:#2196F3,stroke:#333,color:#fff
```
**Use Case:** Decision-based automation (e.g., route by priority)

### Parallel Processing Workflow

```mermaid
flowchart TD
    A[Trigger] --> B[Process Data]
    B --> C[Action 1]
    B --> D[Action 2]
    B --> E[Action 3]
    C --> F[Merge Results]
    D --> F
    E --> F

    style A fill:#4CAF50,stroke:#333,color:#fff
    style B fill:#2196F3,stroke:#333,color:#fff
    style F fill:#FF9800,stroke:#333,color:#fff
```
**Use Case:** Multi-channel notifications (e.g., email + Slack + SMS)

---

## Getting Started Checklist

### Your First Hour with n8n

```mermaid
journey
    title First Hour with n8n
    section Setup (10 min)
      Install n8n: 5: Setup
      Open interface: 5: Setup
    section First Workflow (20 min)
      Add Manual Trigger: 4: Building
      Add Set Node: 4: Building
      Configure node: 3: Building
      Execute workflow: 5: Building
    section Explore (15 min)
      Browse node panel: 4: Learning
      Read documentation: 4: Learning
      Try different nodes: 3: Learning
    section First Real Workflow (15 min)
      Add Schedule Trigger: 4: Creating
      Add HTTP Request: 3: Creating
      Add Email node: 4: Creating
      Test workflow: 5: Creating
```

---

## Quick Start Commands

### Installation Commands

```bash
# Option 1: npx (No installation, run directly)
npx n8n

# Option 2: Global npm install
npm install n8n -g
n8n start

# Option 3: Docker
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n

# Option 4: Docker Compose
# Create docker-compose.yml and run:
docker-compose up -d
```

### First Workflow Template

```javascript
// Manual Trigger → Set Node → Output

// Set Node Configuration:
{
  "values": {
    "string": [
      {
        "name": "message",
        "value": "Hello from n8n!"
      },
      {
        "name": "timestamp",
        "value": "={{ $now.toISO() }}"
      }
    ]
  }
}
```

---

## Key Concepts Summary

```mermaid
graph TB
    subgraph "Essential Concepts"
        W[Workflows<br/>Complete automation sequences]
        N[Nodes<br/>Individual building blocks]
        C[Connections<br/>Link nodes together]
        D[Data<br/>JSON format, always arrays]
        E[Executions<br/>Workflow runs with history]
    end

    W --> N
    N --> C
    C --> D
    D --> E

    style W fill:#EA4B71,stroke:#333,color:#fff
    style N fill:#4CAF50,stroke:#333,color:#fff
    style C fill:#2196F3,stroke:#333,color:#fff
    style D fill:#FF9800,stroke:#333,color:#fff
    style E fill:#9C27B0,stroke:#333,color:#fff
```

---

## Next Steps

After completing Week 1, you should be able to:
- ✓ Navigate the n8n interface confidently
- ✓ Understand different node types
- ✓ Create simple workflows
- ✓ Execute and test workflows
- ✓ View execution history

**Ready for Week 2?** Move on to learn about data flow, expressions, and more advanced node configurations!

---

**Welcome to n8n! You're on your way to becoming an automation expert.**

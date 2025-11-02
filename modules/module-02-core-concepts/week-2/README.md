# Week 2: Understanding Nodes and Data Flow

## Learning Objectives

- Understand the different types of nodes in n8n
- Master JSON data structure
- Write effective expressions
- Use the Expression Editor
- Debug workflows efficiently
- Work with core nodes (Set, IF, Switch, Merge, Split)

---

## Topics

### 1. Node Types Overview

```mermaid
graph TD
    A[n8n Nodes] --> B[Trigger Nodes]
    A --> C[Action Nodes]
    A --> D[Core Nodes]

    B --> B1[Schedule Trigger]
    B --> B2[Webhook Trigger]
    B --> B3[Manual Trigger]
    B --> B4[Email Trigger]

    C --> C1[HTTP Request]
    C --> C2[Gmail]
    C --> C3[Slack]
    C --> C4[Database]

    D --> D1[Set]
    D --> D2[IF]
    D --> D3[Switch]
    D --> D4[Merge]
    D --> D5[Split]
    D --> D6[Function]

    style B fill:#EA4B71,stroke:#333,color:#fff
    style C fill:#00A8A0,stroke:#333,color:#fff
    style D fill:#7D54D9,stroke:#333,color:#fff
```

#### Trigger Nodes
Start workflow execution based on events:
- **Manual Trigger**: Start manually
- **Schedule Trigger**: Time-based execution
- **Webhook Trigger**: HTTP requests
- **Email Trigger**: Incoming emails
- **File Trigger**: File system changes

#### Action Nodes
Interact with external services:
- **HTTP Request**: Any REST API
- **Gmail**: Email operations
- **Slack**: Messaging
- **Database**: CRUD operations
- **File System**: File operations

#### Core Nodes
Control flow and data manipulation:
- **Set**: Modify data structure
- **IF**: Conditional branching
- **Switch**: Multiple conditions
- **Merge**: Combine data streams
- **Split**: Divide data
- **Function**: Custom JavaScript

---

### 2. Understanding JSON Data Structure

Every node in n8n works with JSON data.

#### Basic Structure

```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "age": 30,
  "active": true,
  "tags": ["customer", "premium"],
  "address": {
    "city": "London",
    "country": "UK"
  }
}
```

#### Data Flow in n8n

```mermaid
sequenceDiagram
    participant N1 as Node 1
    participant N2 as Node 2
    participant N3 as Node 3

    Note over N1: Input: Empty {}
    N1->>N2: Output: {name: "John"}
    Note over N2: Add fields
    N2->>N3: Output: {name: "John", age: 30, email: "..."}
    Note over N3: Transform data
```

#### Items Array

n8n processes data as an array of items:

```javascript
[
  { json: { name: "John", age: 30 } },
  { json: { name: "Jane", age: 25 } },
  { json: { name: "Bob", age: 35 } }
]
```

Each item contains:
- `json`: The actual data
- `binary`: Binary data (files, images)
- `pairedItem`: Tracking information

---

### 3. Expressions and Variables

Expressions allow dynamic data access and transformation.

#### Basic Syntax

```javascript
// Access current item data
{{$json.fieldName}}

// Access specific node output
{{$node["NodeName"].json.fieldName}}

// Access all items
{{$items}}

// Item index
{{$itemIndex}}

// Run index
{{$runIndex}}
```

#### Common Expression Patterns

**Accessing Nested Data:**
```javascript
{{$json.user.address.city}}
```

**Array Access:**
```javascript
{{$json.items[0].name}}
```

**String Operations:**
```javascript
{{$json.name.toUpperCase()}}
{{$json.email.toLowerCase()}}
{{$json.text.trim()}}
{{$json.message.substring(0, 10)}}
```

**Number Operations:**
```javascript
{{Number($json.price) * 1.2}}
{{Math.round($json.value)}}
{{Math.max($json.a, $json.b)}}
```

**Date/Time:**
```javascript
{{DateTime.now().toISO()}}
{{DateTime.now().toFormat('yyyy-MM-dd')}}
{{DateTime.fromISO($json.date).plus({days: 7})}}
```

**Conditional (Ternary):**
```javascript
{{$json.age > 18 ? 'adult' : 'minor'}}
{{$json.status === 'active' ? '✓' : '✗'}}
```

---

### 4. The Expression Editor

```mermaid
graph LR
    A[Field Value] -->|Click fx icon| B[Expression Editor]
    B --> C[Variable Browser]
    B --> D[Sample Output]
    B --> E[Error Checking]

    C --> F[Current Node Data]
    C --> G[Previous Node Data]
    C --> H[Environment Vars]

    style A fill:#f0f0f0,stroke:#333
    style B fill:#EA4B71,stroke:#333,color:#fff
    style C fill:#00A8A0,stroke:#333,color:#fff
```

**Features:**
- Auto-completion
- Variable browser
- Live preview
- Error detection
- Sample data testing

**Tips:**
1. Always test expressions with sample data
2. Use the variable browser to explore available data
3. Check the sample output before saving
4. Use console.log() in Function nodes for debugging

---

### 5. Core Nodes Deep Dive

#### Set Node

Transform and restructure data.

```mermaid
graph LR
    A["Input<br/>{name: 'John', age: 30}"] -->|Set Node| B["Output<br/>{fullName: 'John Doe',<br/>category: 'adult'}"]

    style A fill:#f0f0f0,stroke:#333
    style B fill:#00A8A0,stroke:#333,color:#fff
```

**Use Cases:**
- Add new fields
- Rename fields
- Remove unnecessary fields
- Transform data types
- Create computed values

**Example Configuration:**
```
Keep Only Set: Yes (remove all other fields)

Fields:
- fullName: {{$json.name}}
- email: {{$json.email.toLowerCase()}}
- category: {{$json.age > 18 ? 'adult' : 'minor'}}
- processedAt: {{DateTime.now().toISO()}}
```

#### IF Node

Branch workflow based on conditions.

```mermaid
graph TB
    A[IF Node] -->|Condition: True| B[True Branch]
    A -->|Condition: False| C[False Branch]

    style A fill:#EA4B71,stroke:#333,color:#fff
    style B fill:#4CAF50,stroke:#333,color:#fff
    style C fill:#FF9D00,stroke:#333
```

**Conditions:**
- String: equals, contains, starts with, regex
- Number: equals, greater than, less than
- Boolean: true/false
- Date: before, after

**Example:**
```
Condition: Number
Value 1: {{$json.age}}
Operation: Larger
Value 2: 18
```

#### Switch Node

Multiple conditional branches.

```mermaid
graph TB
    A[Switch Node] -->|Rule 1| B[Output 1]
    A -->|Rule 2| C[Output 2]
    A -->|Rule 3| D[Output 3]
    A -->|Default| E[Fallback]

    style A fill:#EA4B71,stroke:#333,color:#fff
    style B fill:#4CAF50,stroke:#333,color:#fff
    style C fill:#00A8A0,stroke:#333,color:#fff
    style D fill:#7D54D9,stroke:#333,color:#fff
    style E fill:#FF9D00,stroke:#333
```

**Use Cases:**
- Route by status (pending, approved, rejected)
- Categorize by value ranges
- Handle multiple conditions
- Default fallback handling

#### Merge Node

Combine data from multiple branches.

```mermaid
graph TB
    A[Branch 1] --> C[Merge Node]
    B[Branch 2] --> C
    C --> D[Combined Output]

    style A fill:#00A8A0,stroke:#333,color:#fff
    style B fill:#7D54D9,stroke:#333,color:#fff
    style C fill:#EA4B71,stroke:#333,color:#fff
    style D fill:#4CAF50,stroke:#333,color:#fff
```

**Modes:**
- **Append**: Stack items sequentially
- **Merge By Fields**: Combine based on matching fields
- **Multiplex**: Create combinations
- **Remove Duplicates**: Filter unique items

#### Split In Batches Node

Process large datasets in chunks.

```mermaid
graph LR
    A[1000 Items] -->|Split| B[Batch 1<br/>100 items]
    A -->|Split| C[Batch 2<br/>100 items]
    A -->|Split| D[...]
    A -->|Split| E[Batch 10<br/>100 items]

    style A fill:#f0f0f0,stroke:#333
    style B fill:#00A8A0,stroke:#333,color:#fff
    style C fill:#00A8A0,stroke:#333,color:#fff
    style D fill:#00A8A0,stroke:#333,color:#fff
    style E fill:#00A8A0,stroke:#333,color:#fff
```

**Use Cases:**
- Rate limit compliance
- Memory management
- Batch API calls
- Progress tracking

---

### 6. Workflow Execution Modes

```mermaid
stateDiagram-v2
    [*] --> Manual: Click Execute
    [*] --> Active: Toggle Active
    [*] --> Production: Webhook/Trigger

    Manual --> Executed: Test Mode
    Active --> Scheduled: Time-based
    Production --> WebhookCall: HTTP Request

    Executed --> [*]
    Scheduled --> [*]
    WebhookCall --> [*]
```

**Execution Types:**

1. **Manual Execution** (Testing)
   - Click "Execute Workflow" button
   - All nodes execute
   - See full execution data
   - Doesn't count against quotas

2. **Active Execution** (Production)
   - Workflow must be activated
   - Triggered by schedule or webhook
   - Runs in background
   - Recorded in execution history

3. **Production Execution**
   - Optimized for performance
   - Limited debugging info
   - Automatic retries
   - Error notifications

---

## Hands-On Exercises

### [Exercise 1: Data Processing Pipeline](./exercises/exercise-1-data-processing.md)
Build a workflow that filters and transforms data using Set and IF nodes.

### [Exercise 2: Multi-Condition Router](./exercises/exercise-2-switch-router.md)
Use Switch node to route data based on multiple conditions.

### [Exercise 3: Data Merger](./exercises/exercise-3-merge-data.md)
Combine data from different sources using Merge node.

---

## Debugging Workflows

```mermaid
graph TD
    A[Workflow Error] --> B{Check Execution}
    B -->|View Data| C[Click on Node]
    B -->|Error Message| D[Read Error]
    C --> E[Verify Input Data]
    D --> F[Check Configuration]
    E --> G[Test Expression]
    F --> G
    G --> H{Fixed?}
    H -->|Yes| I[Success]
    H -->|No| J[Check Docs/Community]

    style A fill:#f44336,stroke:#333,color:#fff
    style I fill:#4CAF50,stroke:#333,color:#fff
```

**Debugging Tips:**
1. Execute nodes individually
2. Check input/output data
3. Verify expressions in Expression Editor
4. Use console.log() in Function nodes
5. Check execution history
6. Enable "Save Execution Progress"

---

## Best Practices

1. **Name Your Nodes**: Use descriptive names
2. **Add Notes**: Document complex logic
3. **Keep It Simple**: Don't over-complicate workflows
4. **Test Incrementally**: Build and test node by node
5. **Use Set Nodes**: Simplify data structure early
6. **Handle Errors**: Always consider error cases

---

## Key Takeaways

- ✓ Three main node types: Triggers, Actions, Core
- ✓ Data flows as JSON items through nodes
- ✓ Expressions access and transform data dynamically
- ✓ Core nodes control workflow logic and data flow
- ✓ The Expression Editor helps build and test expressions
- ✓ Debug by checking node outputs and execution data

---

## Next Steps

**Continue to:** [Week 3: Working with APIs](../week-3/README.md)

**Practice More:**
- Build complex data transformations
- Experiment with all core nodes
- Create multi-branch workflows
- Try different expression patterns

# Week 4: Popular App Integrations

## Learning Objectives

- Connect common business applications to n8n
- Master credential management and authentication
- Build multi-app workflows
- Understand webhook triggers and responses
- Implement secure credential storage practices

---

## Topics

### 1. Gmail Integration

```mermaid
graph LR
    A[Gmail Trigger] --> B[New Email]
    B --> C[Process Email]
    C --> D1[Save Attachment]
    C --> D2[Reply]
    C --> D3[Forward]
    C --> D4[Archive]

    style A fill:#EA4B71,stroke:#333,color:#fff
    style B fill:#00A8A0,stroke:#333,color:#fff
    style C fill:#7D54D9,stroke:#333,color:#fff
    style D1 fill:#4CAF50,stroke:#333,color:#fff
    style D2 fill:#4CAF50,stroke:#333,color:#fff
    style D3 fill:#4CAF50,stroke:#333,color:#fff
    style D4 fill:#4CAF50,stroke:#333,color:#fff
```

#### Gmail Node Capabilities

**Trigger Operations:**
- New email received
- Email matched by search
- Email with specific label

**Actions:**
- Send email
- Reply to email
- Forward email
- Add/remove labels
- Mark as read/unread
- Move to trash
- Get attachments

#### Authentication Setup

1. **OAuth2 Method (Recommended)**
   - Create Google Cloud Project
   - Enable Gmail API
   - Create OAuth2 credentials
   - Configure consent screen
   - Add authorized redirect URIs

2. **Service Account (Advanced)**
   - For domain-wide delegation
   - Enterprise use cases

#### Common Use Cases

- **Email Parser**: Extract data from structured emails
- **Auto-Responder**: Send automatic replies based on conditions
- **Attachment Processor**: Save attachments to cloud storage
- **Email Classifier**: Organize emails with labels
- **Notification System**: Send alerts via email

#### Example: Email to Data Workflow

```javascript
// Extract order data from email subject
const subject = $json.subject;
const orderMatch = subject.match(/Order #(\d+)/);

return {
  orderId: orderMatch ? orderMatch[1] : null,
  customerEmail: $json.from,
  receivedAt: $json.date,
  subject: subject
};
```

---

### 2. Google Sheets Integration

```mermaid
graph TB
    A[Trigger] --> B[Google Sheets]
    B --> C1[Read Rows]
    B --> C2[Append Row]
    B --> C3[Update Row]
    B --> C4[Delete Row]
    B --> C5[Clear Sheet]

    C2 --> D[Process Data]
    C3 --> D

    style A fill:#EA4B71,stroke:#333,color:#fff
    style B fill:#00A8A0,stroke:#333,color:#fff
    style C1 fill:#7D54D9,stroke:#333,color:#fff
    style C2 fill:#7D54D9,stroke:#333,color:#fff
    style C3 fill:#7D54D9,stroke:#333,color:#fff
    style C4 fill:#7D54D9,stroke:#333,color:#fff
    style C5 fill:#7D54D9,stroke:#333,color:#fff
    style D fill:#4CAF50,stroke:#333,color:#fff
```

#### Google Sheets Operations

**Read Operations:**
- Get all rows
- Get row by ID
- Get rows with filters
- Get specific range
- Lookup by column value

**Write Operations:**
- Append row(s)
- Update row
- Delete row
- Clear sheet
- Create sheet

#### Data Mapping

**Append Row Example:**
```
Sheet: Orders
Range: A:E

Columns to Send:
- Order ID: {{$json.orderId}}
- Customer: {{$json.customerName}}
- Amount: {{$json.total}}
- Date: {{DateTime.now().toFormat('yyyy-MM-dd')}}
- Status: {{$json.status}}
```

#### Best Practices

1. **Use Named Ranges**: Easier to maintain
2. **Header Row**: Always include headers
3. **Data Validation**: Validate before writing
4. **Rate Limits**: Batch operations when possible
5. **Error Handling**: Check for duplicate entries

---

### 3. Slack Integration

```mermaid
graph LR
    A[Slack Trigger] --> B[Message Posted]
    B --> C{Process}
    C --> D1[Reply in Thread]
    C --> D2[Send DM]
    C --> D3[Add Reaction]
    C --> D4[Update Message]

    E[Slack Action] --> F[Post Message]
    E --> G[Upload File]
    E --> H[Get User Info]

    style A fill:#EA4B71,stroke:#333,color:#fff
    style B fill:#00A8A0,stroke:#333,color:#fff
    style C fill:#7D54D9,stroke:#333,color:#fff
    style E fill:#00A8A0,stroke:#333,color:#fff
```

#### Slack Node Capabilities

**Trigger Events:**
- New message in channel
- Mention received
- Reaction added
- File shared
- User joined channel

**Actions:**
- Send message to channel
- Send direct message
- Reply in thread
- Add reaction
- Update message
- Delete message
- Upload file
- Get user/channel info
- Invite to channel

#### Authentication

**Bot Token (Recommended):**
```
1. Create Slack App
2. Add Bot Token Scopes:
   - chat:write
   - channels:read
   - users:read
   - files:write
3. Install app to workspace
4. Copy Bot User OAuth Token
```

**Required Scopes by Operation:**
- Post message: `chat:write`
- Read messages: `channels:history`
- Reactions: `reactions:write`
- Files: `files:write`
- User info: `users:read`

#### Message Formatting

**Basic Formatting:**
```
*bold*
_italic_
~strikethrough~
`code`
```block code```
> quote
```

**Blocks API (Rich Messages):**
```json
{
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Order Alert*\nNew order #{{$json.orderId}}"
      }
    },
    {
      "type": "section",
      "fields": [
        {
          "type": "mrkdwn",
          "text": "*Customer:*\n{{$json.customer}}"
        },
        {
          "type": "mrkdwn",
          "text": "*Amount:*\n${{$json.total}}"
        }
      ]
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": {
            "type": "plain_text",
            "text": "View Order"
          },
          "url": "{{$json.orderUrl}}"
        }
      ]
    }
  ]
}
```

#### Interactive Workflows

**Slash Commands:**
- User types `/command` in Slack
- Slack sends webhook to n8n
- n8n processes and responds
- Can include buttons and menus

**Interactive Components:**
- Buttons
- Select menus
- Date pickers
- Modals/Forms

---

### 4. Database Integrations

```mermaid
graph TB
    A[Database Node] --> B{Database Type}
    B --> C1[PostgreSQL]
    B --> C2[MySQL]
    B --> C3[MongoDB]
    B --> C4[SQLite]

    C1 --> D[Operations]
    C2 --> D
    C3 --> D

    D --> E1[SELECT]
    D --> E2[INSERT]
    D --> E3[UPDATE]
    D --> E4[DELETE]
    D --> E5[Execute Query]

    style A fill:#EA4B71,stroke:#333,color:#fff
    style B fill:#00A8A0,stroke:#333,color:#fff
    style D fill:#7D54D9,stroke:#333,color:#fff
```

#### PostgreSQL / MySQL

**Connection Parameters:**
```
Host: database.example.com
Port: 5432 (PostgreSQL) / 3306 (MySQL)
Database: myapp_production
User: n8n_user
Password: [stored securely]
SSL: Enable (production)
```

**Operations:**

**1. Execute Query**
```sql
SELECT * FROM orders
WHERE created_at > '{{$json.startDate}}'
AND status = '{{$json.status}}'
ORDER BY created_at DESC
LIMIT 100
```

**2. Insert**
```sql
INSERT INTO orders (customer_id, total, status, created_at)
VALUES (
  {{$json.customerId}},
  {{$json.total}},
  '{{$json.status}}',
  NOW()
)
RETURNING id
```

**3. Update**
```sql
UPDATE orders
SET status = '{{$json.newStatus}}',
    updated_at = NOW()
WHERE id = {{$json.orderId}}
```

**4. Delete**
```sql
DELETE FROM orders
WHERE id = {{$json.orderId}}
AND status = 'cancelled'
```

#### MongoDB

**Operations:**
- Find
- Find One
- Insert
- Update
- Delete
- Aggregate

**Example: Find Documents**
```javascript
// Filter
{
  "status": "{{$json.status}}",
  "createdAt": {
    "$gte": "{{$json.startDate}}"
  }
}

// Projection
{
  "name": 1,
  "email": 1,
  "orders": 1,
  "_id": 0
}

// Sort
{
  "createdAt": -1
}
```

**Example: Aggregation Pipeline**
```javascript
[
  {
    "$match": {
      "status": "completed"
    }
  },
  {
    "$group": {
      "_id": "$customerId",
      "totalOrders": { "$sum": 1 },
      "totalAmount": { "$sum": "$amount" }
    }
  },
  {
    "$sort": {
      "totalAmount": -1
    }
  },
  {
    "$limit": 10
  }
]
```

#### Database Best Practices

1. **Use Parameterized Queries**: Prevent SQL injection
2. **Connection Pooling**: Reuse connections
3. **Read Replicas**: Use for SELECT queries
4. **Transactions**: For multi-step operations
5. **Indexes**: Ensure queries are optimized
6. **Error Handling**: Catch constraint violations

---

### 5. Webhook Triggers and Responses

```mermaid
sequenceDiagram
    participant E as External Service
    participant W as Webhook Trigger
    participant N as n8n Workflow
    participant R as Response

    E->>W: POST /webhook/abc123
    W->>N: Trigger Workflow
    N->>N: Process Data
    N->>R: Generate Response
    R->>E: Return Response (200 OK)

    Note over E,R: Synchronous Response
```

#### Webhook Trigger Node

**Configuration:**
- **Path**: Unique webhook identifier
- **Method**: GET, POST, PUT, DELETE, PATCH
- **Authentication**: None, Basic, Header
- **Response**: Return data to caller

**Webhook URL Structure:**
```
Production: https://your-n8n.com/webhook/your-path
Test: https://your-n8n.com/webhook-test/your-path
```

#### Request Types

**1. Form Data**
```javascript
// Access form fields
{
  name: $json.body.name,
  email: $json.body.email
}
```

**2. JSON Payload**
```javascript
// Direct access to JSON
{
  orderId: $json.body.orderId,
  items: $json.body.items
}
```

**3. Query Parameters**
```javascript
// URL: /webhook/data?id=123&type=order
{
  id: $json.query.id,
  type: $json.query.type
}
```

**4. Headers**
```javascript
// Access headers
{
  apiKey: $json.headers['x-api-key'],
  contentType: $json.headers['content-type']
}
```

#### Webhook Response

**Simple Response:**
```javascript
// Respond Node
{
  "status": "success",
  "message": "Data received",
  "id": "{{$json.newId}}"
}
```

**Custom Status Code:**
```
Response Code: 201
Response Body:
{
  "created": true,
  "resourceId": "{{$json.id}}"
}
```

**Error Response:**
```
Response Code: 400
Response Body:
{
  "error": "Invalid request",
  "details": "{{$json.errorMessage}}"
}
```

#### Security Best Practices

**1. Webhook Authentication:**
```javascript
// Verify API Key
const apiKey = $json.headers['x-api-key'];
if (apiKey !== 'your-secret-key') {
  return {
    error: 'Unauthorized',
    code: 401
  };
}
```

**2. Signature Verification:**
```javascript
// Verify webhook signature (e.g., Stripe)
const crypto = require('crypto');
const signature = $json.headers['stripe-signature'];
const payload = $json.body;
const secret = 'whsec_...';

const expectedSignature = crypto
  .createHmac('sha256', secret)
  .update(JSON.stringify(payload))
  .digest('hex');

if (signature !== expectedSignature) {
  return { error: 'Invalid signature' };
}
```

**3. IP Whitelisting:**
```javascript
// Check source IP
const allowedIPs = ['192.168.1.1', '10.0.0.1'];
const sourceIP = $json.headers['x-forwarded-for'];

if (!allowedIPs.includes(sourceIP)) {
  return { error: 'Forbidden' };
}
```

---

### 6. Credential Management

```mermaid
graph TB
    A[Credential Types] --> B[OAuth2]
    A --> C[API Key]
    A --> D[Basic Auth]
    A --> E[Custom]

    B --> F[Token Storage]
    C --> F
    D --> F
    E --> F

    F --> G[Encrypted Storage]
    G --> H[Environment Variables]

    style A fill:#EA4B71,stroke:#333,color:#fff
    style F fill:#00A8A0,stroke:#333,color:#fff
    style G fill:#7D54D9,stroke:#333,color:#fff
    style H fill:#4CAF50,stroke:#333,color:#fff
```

#### Authentication Methods

**1. OAuth2 (Most Secure)**
- Google services
- Microsoft services
- Social media APIs
- Automatic token refresh

**2. API Key**
- Simple services
- Read-only access
- Rate-limited APIs

**3. Basic Authentication**
- Username/password
- Legacy systems
- Internal APIs

**4. Header Authentication**
- Custom headers
- Bearer tokens
- JWT tokens

#### Credential Security

**Best Practices:**
1. **Never Hardcode**: Use credential system
2. **Rotate Regularly**: Update keys periodically
3. **Minimal Permissions**: Least privilege principle
4. **Separate Environments**: Dev, staging, production
5. **Audit Access**: Track credential usage
6. **Encrypt at Rest**: n8n encrypts automatically

**Environment Variables:**
```bash
# .env file
DB_PASSWORD=secret123
API_KEY=abc123xyz
SLACK_TOKEN=xoxb-your-token
```

**Access in n8n:**
```javascript
{{$env.DB_PASSWORD}}
{{$env.API_KEY}}
```

#### Sharing Credentials

**Team Collaboration:**
- Centralized credential store
- Role-based access control (Enterprise)
- Credential ownership
- Usage tracking

**Per-Workflow Credentials:**
- Each workflow can use different credentials
- Test vs production credentials
- Client-specific credentials

---

## Hands-On Exercises

### [Exercise 1: Email to Google Sheets Logger](./exercises/exercise-1-email-logger.md)

Build a workflow that:
1. Monitors Gmail for emails with specific label
2. Extracts key information (sender, subject, date)
3. Appends data to Google Sheets
4. Marks email as processed

**Learning Goals:**
- Gmail trigger setup
- Data extraction from emails
- Google Sheets append operation
- Gmail label management

---

### [Exercise 2: Slack Bot Responder](./exercises/exercise-2-slack-bot.md)

Create an interactive Slack bot that:
1. Listens for mentions in a channel
2. Processes the message content
3. Queries a database for information
4. Responds in thread with formatted message
5. Adds reaction to original message

**Learning Goals:**
- Slack event triggers
- Message parsing
- Database queries
- Slack message formatting
- Thread replies and reactions

---

### [Exercise 3: Database CRUD Operations](./exercises/exercise-3-database-crud.md)

Build a complete CRUD workflow:
1. Webhook trigger for API endpoints
2. CREATE: Insert new records
3. READ: Query and filter data
4. UPDATE: Modify existing records
5. DELETE: Remove records
6. Return appropriate responses

**Learning Goals:**
- Webhook routing
- SQL query construction
- Data validation
- Error handling
- RESTful response patterns

---

## Real-World Project: Customer Support Ticketing System

```mermaid
graph TB
    A[Gmail: New Support Email] --> B[Extract Ticket Info]
    B --> C[Create Record in Database]
    C --> D[Post to Slack Channel]
    D --> E{Priority?}
    E -->|High| F[Notify Manager DM]
    E -->|Normal| G[Standard Notification]
    F --> H[Log to Google Sheets]
    G --> H
    H --> I[Send Confirmation Email]

    style A fill:#EA4B71,stroke:#333,color:#fff
    style C fill:#00A8A0,stroke:#333,color:#fff
    style D fill:#7D54D9,stroke:#333,color:#fff
    style E fill:#FF9D00,stroke:#333
    style H fill:#4CAF50,stroke:#333,color:#fff
```

**Implementation Steps:**

1. **Gmail Trigger**
   - Monitor support@company.com
   - Filter by subject pattern
   - Get email content and attachments

2. **Data Extraction**
   - Parse email subject for ticket number
   - Extract customer email
   - Categorize issue type
   - Determine priority

3. **Database Storage**
   - Insert ticket record
   - Store customer info
   - Save email content
   - Generate ticket ID

4. **Slack Notification**
   - Post to #support-tickets
   - Format with ticket details
   - Include action buttons
   - Mention relevant team members

5. **Conditional Logic**
   - Check priority level
   - Route high-priority tickets
   - Assign to team members
   - Set SLA timers

6. **Logging**
   - Record in Google Sheets
   - Track response times
   - Monitor ticket volume
   - Generate reports

7. **Confirmation**
   - Send auto-reply email
   - Include ticket number
   - Set expectations
   - Provide self-service links

---

## Integration Patterns

### Pattern 1: Event-Driven Integration

```mermaid
sequenceDiagram
    participant A as App A
    participant W as Webhook
    participant N as n8n
    participant B as App B
    participant C as App C

    A->>W: Event Occurred
    W->>N: Trigger Workflow
    N->>B: Update Record
    N->>C: Send Notification
    B-->>N: Success
    C-->>N: Success
    N->>A: Acknowledge
```

**Use Cases:**
- Form submissions
- Payment processing
- Order fulfillment
- User registrations

### Pattern 2: Scheduled Sync

```mermaid
graph LR
    A[Schedule Trigger] --> B[Fetch from Source]
    B --> C[Transform Data]
    C --> D[Update Destination]
    D --> E[Send Report]

    style A fill:#EA4B71,stroke:#333,color:#fff
    style B fill:#00A8A0,stroke:#333,color:#fff
    style C fill:#7D54D9,stroke:#333,color:#fff
    style D fill:#4CAF50,stroke:#333,color:#fff
```

**Use Cases:**
- Daily reports
- Data synchronization
- Backup automation
- Metric collection

### Pattern 3: Bidirectional Sync

```mermaid
sequenceDiagram
    participant A as System A
    participant N as n8n
    participant B as System B

    A->>N: Data Changed
    N->>B: Update
    B->>N: Data Changed
    N->>A: Update

    Note over A,B: Keep Systems in Sync
```

**Considerations:**
- Conflict resolution
- Timestamp tracking
- Deduplication
- Loop prevention

---

## Troubleshooting Common Issues

### Gmail Issues

**Problem: Trigger not firing**
- ✓ Check OAuth2 token is valid
- ✓ Verify label exists
- ✓ Confirm workflow is active
- ✓ Check filter criteria

**Problem: Attachment not downloading**
- ✓ Check file size limits
- ✓ Verify permissions
- ✓ Check storage quotas

### Google Sheets Issues

**Problem: Data not appending**
- ✓ Verify sheet name is correct
- ✓ Check column mapping
- ✓ Confirm permissions (edit access)
- ✓ Check for special characters

**Problem: Slow performance**
- ✓ Use batch operations
- ✓ Limit range reads
- ✓ Avoid excessive API calls

### Slack Issues

**Problem: Bot not responding**
- ✓ Check bot token scopes
- ✓ Verify bot is in channel
- ✓ Confirm event subscriptions
- ✓ Check webhook URL

**Problem: Message formatting broken**
- ✓ Validate JSON structure
- ✓ Escape special characters
- ✓ Test blocks in Block Kit Builder

### Database Issues

**Problem: Connection timeout**
- ✓ Check connection parameters
- ✓ Verify firewall rules
- ✓ Test connection from n8n server
- ✓ Check SSL requirements

**Problem: Query errors**
- ✓ Validate SQL syntax
- ✓ Check table/column names
- ✓ Verify data types
- ✓ Test expressions

---

## Best Practices Summary

### Security
- ✓ Use OAuth2 when available
- ✓ Store credentials properly
- ✓ Rotate API keys regularly
- ✓ Validate webhook signatures
- ✓ Use HTTPS for webhooks

### Performance
- ✓ Batch operations when possible
- ✓ Use pagination for large datasets
- ✓ Implement rate limiting
- ✓ Cache frequently used data
- ✓ Optimize database queries

### Reliability
- ✓ Implement error handling
- ✓ Add retry logic
- ✓ Log important events
- ✓ Monitor execution history
- ✓ Set up alerts

### Maintainability
- ✓ Name nodes descriptively
- ✓ Document complex workflows
- ✓ Use consistent patterns
- ✓ Version control workflows
- ✓ Test before production

---

## Key Takeaways

- ✓ Master authentication for major platforms (Gmail, Slack, Google Sheets)
- ✓ Understand CRUD operations with databases
- ✓ Use webhooks for real-time integrations
- ✓ Manage credentials securely
- ✓ Build multi-app workflows that connect business systems
- ✓ Implement proper error handling and monitoring

---

## Additional Resources

**Official Documentation:**
- [Gmail Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/)
- [Google Sheets Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/)
- [Slack Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/)
- [PostgreSQL Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.postgres/)
- [Webhook Node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/)

**API Documentation:**
- [Gmail API](https://developers.google.com/gmail/api)
- [Google Sheets API](https://developers.google.com/sheets/api)
- [Slack API](https://api.slack.com/)
- [Slack Block Kit](https://api.slack.com/block-kit)

**Community Templates:**
- Browse n8n.io workflows for integration examples
- Join n8n community forum for tips
- Check GitHub for workflow examples

---

## Next Steps

**Continue to:** [Week 5: Advanced Integrations](../week-5/README.md)

**Practice More:**
- Connect your real accounts
- Build personal automation workflows
- Experiment with webhook integrations
- Create database-backed applications
- Build Slack bots for your team

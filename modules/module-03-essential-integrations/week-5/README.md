# Week 5: Advanced Integrations

## Learning Objectives

- Integrate file storage services (Google Drive, Dropbox, AWS S3)
- Connect CRM and marketing platforms
- Handle complex webhook integrations
- Work with social media APIs
- Implement payment processing workflows
- Build advanced multi-service automations

---

## Topics

### 1. File Storage Integrations

```mermaid
graph TB
    A[File Operations] --> B[Google Drive]
    A --> C[Dropbox]
    A --> D[AWS S3]
    A --> E[Local Files]

    B --> F[Upload]
    B --> G[Download]
    B --> H[Share]
    B --> I[Search]

    C --> F
    C --> G
    C --> H

    D --> J[Put Object]
    D --> K[Get Object]
    D --> L[List Objects]

    style A fill:#EA4B71,stroke:#333,color:#fff
    style B fill:#00A8A0,stroke:#333,color:#fff
    style C fill:#7D54D9,stroke:#333,color:#fff
    style D fill:#4CAF50,stroke:#333,color:#fff
```

#### Google Drive Integration

**Operations:**
- **Upload**: Create files from workflow data
- **Download**: Get file contents
- **Create Folder**: Organize files
- **Share**: Manage permissions
- **Move**: Reorganize structure
- **Copy**: Duplicate files
- **Delete**: Remove files
- **Search**: Find files by criteria

**Authentication:**
```
OAuth2 Setup:
1. Create Google Cloud Project
2. Enable Google Drive API
3. Create OAuth2 credentials
4. Add scopes:
   - https://www.googleapis.com/auth/drive
   - https://www.googleapis.com/auth/drive.file
5. Configure redirect URI
```

**Upload File Example:**
```javascript
// From binary data
Binary Property: data
File Name: {{$json.filename}}
Parent Folder: 1aB2cD3eF4gH5iJ6kL7mN8oP9qR0sT1

// From URL
Put Output File in Field: binaryData
URL: {{$json.fileUrl}}
```

**Search Files:**
```
Query:
name contains '{{$json.searchTerm}}'
and mimeType = 'application/pdf'
and modifiedTime > '{{$json.startDate}}'
and trashed = false

Fields:
- id
- name
- mimeType
- size
- createdTime
- modifiedTime
- webViewLink
```

**Sharing and Permissions:**
```javascript
// Share file
{
  "role": "reader", // reader, writer, commenter
  "type": "user",   // user, group, domain, anyone
  "emailAddress": "{{$json.email}}"
}

// Anyone with link
{
  "role": "reader",
  "type": "anyone"
}
```

---

#### Dropbox Integration

**Key Operations:**
- Upload file
- Download file
- Create folder
- Move file
- Copy file
- Delete file
- Search
- Get temporary link

**Upload Strategies:**

**1. Direct Upload (< 150MB):**
```javascript
// Upload file
Path: /{{$json.folder}}/{{$json.filename}}
Binary Property: data
Mode: add (or overwrite)
```

**2. Chunked Upload (> 150MB):**
```javascript
// Use HTTP Request node with Dropbox API
// Upload in 4MB chunks
// Better for large files
```

**Create Shared Link:**
```
Path: /documents/report.pdf
Link Type: view (or edit)
Access: public (or team_only)

Returns:
{
  "url": "https://www.dropbox.com/s/...",
  "expires": "2024-12-31T23:59:59Z"
}
```

---

#### AWS S3 Integration

**S3 Operations:**
- **Put Object**: Upload files
- **Get Object**: Download files
- **List Objects**: Browse bucket
- **Delete Object**: Remove files
- **Copy Object**: Duplicate within S3
- **Get Presigned URL**: Temporary access

**Authentication:**
```
Access Key ID: AKIAIOSFODNN7EXAMPLE
Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Region: us-east-1
```

**Upload File:**
```javascript
// Configuration
Bucket Name: my-company-files
File Name: {{DateTime.now().toFormat('yyyy/MM/dd')}}/{{$json.filename}}
Binary Property: data
ACL: private (or public-read)
Storage Class: STANDARD (or INTELLIGENT_TIERING)
Content Type: {{$json.mimeType}}

// Metadata
{
  "uploaded-by": "n8n-workflow",
  "source": "{{$json.source}}",
  "processed-at": "{{DateTime.now().toISO()}}"
}
```

**Generate Presigned URL:**
```javascript
// Temporary download link (expires in seconds)
Bucket: my-bucket
Key: files/document.pdf
Expires In: 3600 // 1 hour

Returns:
https://my-bucket.s3.amazonaws.com/files/document.pdf?
  X-Amz-Algorithm=AWS4-HMAC-SHA256&
  X-Amz-Credential=...&
  X-Amz-Date=...&
  X-Amz-Expires=3600&
  X-Amz-Signature=...
```

**List Objects with Filtering:**
```javascript
// Configuration
Bucket: my-bucket
Prefix: 2024/01/
Max Keys: 1000
Start After: last-key-from-previous-call

// Process results
items.forEach(item => ({
  key: item.Key,
  size: item.Size,
  lastModified: item.LastModified,
  downloadUrl: generatePresignedUrl(item.Key)
}))
```

**S3 Best Practices:**
1. **Bucket Naming**: Use DNS-compliant names
2. **Versioning**: Enable for important data
3. **Lifecycle Policies**: Auto-archive old files
4. **Encryption**: Use SSE-S3 or SSE-KMS
5. **Access Control**: Use IAM roles, not keys
6. **Cost Optimization**: Use appropriate storage class

---

### 2. CRM Integrations

```mermaid
graph LR
    A[CRM Operations] --> B[HubSpot]
    A --> C[Salesforce]

    B --> D1[Contacts]
    B --> D2[Companies]
    B --> D3[Deals]
    B --> D4[Tickets]

    C --> E1[Leads]
    C --> E2[Accounts]
    C --> E3[Opportunities]
    C --> E4[Cases]

    style A fill:#EA4B71,stroke:#333,color:#fff
    style B fill:#00A8A0,stroke:#333,color:#fff
    style C fill:#7D54D9,stroke:#333,color:#fff
```

#### HubSpot Integration

**Core Objects:**
- **Contacts**: People in your database
- **Companies**: Organizations
- **Deals**: Sales opportunities
- **Tickets**: Support requests
- **Products**: Catalog items
- **Line Items**: Deal products

**Authentication:**
```
Method 1: API Key (Simpler)
- Settings → Integrations → API Key

Method 2: OAuth2 (Recommended)
- More secure
- User-specific permissions
- Automatic token refresh
```

**Create/Update Contact:**
```javascript
// Create Contact
{
  "properties": {
    "email": "{{$json.email}}",
    "firstname": "{{$json.firstName}}",
    "lastname": "{{$json.lastName}}",
    "phone": "{{$json.phone}}",
    "company": "{{$json.companyName}}",
    "website": "{{$json.website}}",
    "lifecyclestage": "lead",
    "lead_source": "{{$json.source}}"
  }
}

// Update Contact
Operation: Update
Contact ID: {{$json.contactId}}
Properties to Update:
- lifecyclestage: customer
- closedate: {{DateTime.now().toISO()}}
- deal_amount: {{$json.dealValue}}
```

**Create Deal:**
```javascript
{
  "properties": {
    "dealname": "{{$json.dealName}}",
    "amount": "{{$json.amount}}",
    "dealstage": "appointmentscheduled",
    "pipeline": "default",
    "closedate": "{{$json.expectedCloseDate}}",
    "hubspot_owner_id": "{{$json.ownerId}}"
  },
  "associations": [
    {
      "to": {
        "id": "{{$json.contactId}}"
      },
      "types": [
        {
          "associationCategory": "HUBSPOT_DEFINED",
          "associationTypeId": 3 // Deal to Contact
        }
      ]
    }
  ]
}
```

**Search Contacts:**
```javascript
// Filter Groups
{
  "filterGroups": [
    {
      "filters": [
        {
          "propertyName": "createdate",
          "operator": "GTE",
          "value": "{{$json.startDate}}"
        },
        {
          "propertyName": "lifecyclestage",
          "operator": "EQ",
          "value": "lead"
        }
      ]
    }
  ],
  "properties": [
    "firstname",
    "lastname",
    "email",
    "phone",
    "createdate"
  ],
  "limit": 100
}
```

**HubSpot Webhooks:**
```javascript
// Subscribe to events
Events to Monitor:
- contact.creation
- contact.propertyChange
- deal.creation
- deal.propertyChange
- company.creation
- ticket.creation

Webhook Payload:
{
  "objectId": 12345,
  "propertyName": "lifecyclestage",
  "propertyValue": "customer",
  "changeSource": "CRM",
  "eventId": 123456789,
  "subscriptionId": 987654,
  "portalId": 62515,
  "occurredAt": 1577466000000
}
```

---

#### Salesforce Integration

**Standard Objects:**
- **Lead**: Potential customers
- **Account**: Companies
- **Contact**: People at accounts
- **Opportunity**: Sales deals
- **Case**: Support tickets
- **Campaign**: Marketing campaigns

**Authentication:**
```
OAuth2 JWT Flow (Recommended):
1. Create Connected App
2. Enable OAuth Settings
3. Generate Certificate
4. Configure JWT Bearer Flow
5. Get Refresh Token

Scopes:
- api
- refresh_token
- full (for admin)
```

**SOQL Queries:**
```sql
-- Find Recent Opportunities
SELECT Id, Name, Amount, StageName, CloseDate, Account.Name
FROM Opportunity
WHERE CreatedDate >= {{$json.startDate}}
AND StageName NOT IN ('Closed Won', 'Closed Lost')
ORDER BY Amount DESC
LIMIT 100

-- Find Contacts by Account
SELECT Id, FirstName, LastName, Email, Phone, Account.Name
FROM Contact
WHERE AccountId = '{{$json.accountId}}'
AND Email != null

-- Complex Query with Relationships
SELECT
  Id,
  Name,
  (SELECT Id, Name, Amount FROM Opportunities WHERE StageName = 'Prospecting'),
  (SELECT Id, FirstName, LastName, Email FROM Contacts WHERE Active__c = true)
FROM Account
WHERE Industry = '{{$json.industry}}'
```

**Create/Update Records:**
```javascript
// Create Lead
{
  "FirstName": "{{$json.firstName}}",
  "LastName": "{{$json.lastName}}",
  "Company": "{{$json.company}}",
  "Email": "{{$json.email}}",
  "Phone": "{{$json.phone}}",
  "Status": "Open - Not Contacted",
  "LeadSource": "{{$json.source}}",
  "Description": "{{$json.notes}}"
}

// Upsert (Update or Insert)
External ID Field: Email__c
Matching Value: {{$json.email}}
If Match: Update
If No Match: Create
```

**Convert Lead:**
```javascript
// Convert Lead to Account/Contact/Opportunity
{
  "leadId": "{{$json.leadId}}",
  "convertedStatus": "Qualified",
  "createOpportunity": true,
  "opportunityName": "{{$json.companyName}} - New Business",
  "ownerId": "{{$json.salesRepId}}"
}
```

---

### 3. Payment Processing

```mermaid
sequenceDiagram
    participant C as Customer
    participant S as Stripe
    participant W as Webhook
    participant N as n8n
    participant D as Database
    participant E as Email

    C->>S: Make Payment
    S->>W: payment_intent.succeeded
    W->>N: Trigger Workflow
    N->>D: Update Order Status
    N->>E: Send Receipt
    N->>S: Create Invoice
    E-->>C: Receipt Email

    style S fill:#635BFF,stroke:#333,color:#fff
    style W fill:#EA4B71,stroke:#333,color:#fff
    style N fill:#00A8A0,stroke:#333,color:#fff
```

#### Stripe Integration

**Webhook Events:**
```javascript
// Common Events to Handle
Events:
- payment_intent.succeeded
- payment_intent.payment_failed
- customer.created
- customer.subscription.created
- customer.subscription.deleted
- invoice.paid
- invoice.payment_failed
- charge.refunded
- checkout.session.completed
```

**Verify Webhook Signature:**
```javascript
// Function node
const crypto = require('crypto');

const sig = $input.item.headers['stripe-signature'];
const payload = JSON.stringify($input.item.body);
const secret = 'whsec_your_webhook_secret';

try {
  const elements = sig.split(',');
  const timestamp = elements.find(e => e.startsWith('t=')).substring(2);
  const signatures = elements.filter(e => e.startsWith('v1=')).map(e => e.substring(3));

  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(`${timestamp}.${payload}`)
    .digest('hex');

  const isValid = signatures.includes(expectedSignature);

  if (!isValid) {
    throw new Error('Invalid signature');
  }

  return { json: $input.item.body };
} catch (error) {
  throw new Error('Webhook verification failed: ' + error.message);
}
```

**Create Customer:**
```javascript
// Stripe API via HTTP Request
POST https://api.stripe.com/v1/customers

Headers:
Authorization: Bearer {{$env.STRIPE_SECRET_KEY}}
Content-Type: application/x-www-form-urlencoded

Body (form-urlencoded):
email={{$json.email}}
name={{$json.name}}
phone={{$json.phone}}
metadata[user_id]={{$json.userId}}
metadata[source]=n8n_workflow
```

**Create Payment Intent:**
```javascript
POST https://api.stripe.com/v1/payment_intents

Body:
amount={{$json.amount * 100}} // Convert to cents
currency=usd
customer={{$json.stripeCustomerId}}
description={{$json.description}}
metadata[order_id]={{$json.orderId}}
automatic_payment_methods[enabled]=true
```

**Process Successful Payment:**
```javascript
// Extract from webhook
const paymentIntent = $json.data.object;

return {
  json: {
    paymentId: paymentIntent.id,
    amount: paymentIntent.amount / 100,
    currency: paymentIntent.currency,
    customerId: paymentIntent.customer,
    status: paymentIntent.status,
    orderId: paymentIntent.metadata.order_id,
    receiptEmail: paymentIntent.receipt_email,
    createdAt: new Date(paymentIntent.created * 1000).toISOString()
  }
};
```

**Handle Failed Payments:**
```javascript
// Webhook: payment_intent.payment_failed
const paymentIntent = $json.data.object;
const errorMessage = paymentIntent.last_payment_error?.message;

// Actions to take:
// 1. Update order status to "payment_failed"
// 2. Send email to customer
// 3. Notify sales team
// 4. Create retry schedule
// 5. Log to monitoring system

return {
  json: {
    orderId: paymentIntent.metadata.order_id,
    customerId: paymentIntent.customer,
    errorMessage: errorMessage,
    attemptCount: paymentIntent.charges.data.length,
    shouldRetry: paymentIntent.charges.data.length < 3
  }
};
```

**Refund Processing:**
```javascript
POST https://api.stripe.com/v1/refunds

Body:
payment_intent={{$json.paymentIntentId}}
amount={{$json.refundAmount * 100}} // Partial refund
reason=requested_by_customer
metadata[reason]={{$json.refundReason}}
metadata[requested_by]={{$json.requestedBy}}
```

---

### 4. Social Media APIs

```mermaid
graph TB
    A[Social Media] --> B[Twitter/X]
    A --> C[LinkedIn]
    A --> D[Facebook]
    A --> E[Instagram]

    B --> F[Post Tweet]
    B --> G[Search Tweets]
    B --> H[Get Mentions]

    C --> I[Share Post]
    C --> J[Get Profile]

    style A fill:#EA4B71,stroke:#333,color:#fff
    style B fill:#1DA1F2,stroke:#333,color:#fff
    style C fill:#0077B5,stroke:#333,color:#fff
```

#### Twitter/X API Integration

**Authentication:**
```
OAuth 1.0a or OAuth 2.0
- API Key
- API Secret Key
- Access Token
- Access Token Secret
- Bearer Token
```

**Post Tweet:**
```javascript
// Using HTTP Request node
POST https://api.twitter.com/2/tweets

Headers:
Authorization: Bearer {{$env.TWITTER_BEARER_TOKEN}}
Content-Type: application/json

Body:
{
  "text": "{{$json.tweetText}}",
  "reply": {
    "in_reply_to_tweet_id": "{{$json.replyToId}}"
  },
  "media": {
    "media_ids": ["{{$json.mediaId}}"]
  }
}
```

**Search Tweets:**
```javascript
GET https://api.twitter.com/2/tweets/search/recent

Query Parameters:
query={{$json.searchQuery}} -is:retweet
max_results=100
tweet.fields=created_at,author_id,public_metrics,entities
expansions=author_id
user.fields=username,name,verified

Example Queries:
- "n8n automation" -is:retweet lang:en
- from:n8n_io
- #automation has:images
- (n8n OR zapier) automation -is:reply
```

**Monitor Mentions:**
```javascript
// Get mentions of authenticated user
GET https://api.twitter.com/2/users/:id/mentions

Query:
max_results=10
since_id={{$json.lastProcessedId}}
tweet.fields=created_at,conversation_id,in_reply_to_user_id

// Process each mention
mentions.forEach(tweet => {
  // Analyze sentiment
  // Auto-reply if needed
  // Log to database
  // Notify team
});
```

**Upload Media:**
```javascript
// Step 1: Upload media
POST https://upload.twitter.com/1.1/media/upload.json

Body (multipart):
media: [binary data]
media_category: tweet_image

Response:
{
  "media_id": 123456789,
  "media_id_string": "123456789"
}

// Step 2: Use in tweet
{
  "text": "Check this out!",
  "media": {
    "media_ids": ["123456789"]
  }
}
```

---

#### LinkedIn API Integration

**Authentication:**
```
OAuth 2.0
Scopes:
- r_liteprofile (basic profile)
- r_emailaddress (email)
- w_member_social (post content)
- r_organization_social (page analytics)
```

**Share Post:**
```javascript
POST https://api.linkedin.com/v2/ugcPosts

Headers:
Authorization: Bearer {{$env.LINKEDIN_ACCESS_TOKEN}}
Content-Type: application/json
X-Restli-Protocol-Version: 2.0.0

Body:
{
  "author": "urn:li:person:{{$json.personId}}",
  "lifecycleState": "PUBLISHED",
  "specificContent": {
    "com.linkedin.ugc.ShareContent": {
      "shareCommentary": {
        "text": "{{$json.postText}}"
      },
      "shareMediaCategory": "ARTICLE",
      "media": [
        {
          "status": "READY",
          "description": {
            "text": "{{$json.linkDescription}}"
          },
          "originalUrl": "{{$json.linkUrl}}",
          "title": {
            "text": "{{$json.linkTitle}}"
          }
        }
      ]
    }
  },
  "visibility": {
    "com.linkedin.ugc.MemberNetworkVisibility": "PUBLIC"
  }
}
```

**Share Image Post:**
```javascript
// Step 1: Register upload
POST https://api.linkedin.com/v2/assets?action=registerUpload

{
  "registerUploadRequest": {
    "recipes": ["urn:li:digitalmediaRecipe:feedshare-image"],
    "owner": "urn:li:person:{{$json.personId}}",
    "serviceRelationships": [
      {
        "relationshipType": "OWNER",
        "identifier": "urn:li:userGeneratedContent"
      }
    ]
  }
}

// Step 2: Upload image to provided URL
PUT [uploadUrl from response]
Binary data

// Step 3: Create post with image
{
  "author": "urn:li:person:{{$json.personId}}",
  "lifecycleState": "PUBLISHED",
  "specificContent": {
    "com.linkedin.ugc.ShareContent": {
      "shareCommentary": {
        "text": "{{$json.postText}}"
      },
      "shareMediaCategory": "IMAGE",
      "media": [
        {
          "status": "READY",
          "media": "{{$json.assetUrn}}"
        }
      ]
    }
  },
  "visibility": {
    "com.linkedin.ugc.MemberNetworkVisibility": "PUBLIC"
  }
}
```

---

### 5. Advanced Integration Patterns

#### Multi-Step File Processing

```mermaid
graph TB
    A[Webhook: File Uploaded] --> B[Download from Google Drive]
    B --> C[Process File]
    C --> D{File Type?}
    D -->|PDF| E[Extract Text]
    D -->|Image| F[OCR Processing]
    D -->|CSV| G[Parse Data]
    E --> H[Store in S3]
    F --> H
    G --> H
    H --> I[Update Database]
    I --> J[Send Slack Notification]

    style A fill:#EA4B71,stroke:#333,color:#fff
    style D fill:#FF9D00,stroke:#333
    style H fill:#4CAF50,stroke:#333,color:#fff
```

**Implementation:**
```javascript
// 1. Detect file type
const fileExt = $json.filename.split('.').pop().toLowerCase();
const mimeType = $json.mimeType;

// 2. Route based on type
if (mimeType === 'application/pdf') {
  // PDF processing path
} else if (mimeType.startsWith('image/')) {
  // Image processing path
} else if (fileExt === 'csv') {
  // CSV processing path
}

// 3. Process and store
// 4. Update records
// 5. Notify stakeholders
```

---

#### Lead Enrichment Pipeline

```mermaid
sequenceDiagram
    participant F as Form Submission
    participant N as n8n
    participant C as Clearbit API
    participant H as HubSpot
    participant S as Slack

    F->>N: New Lead
    N->>C: Enrich Data
    C-->>N: Company Info
    N->>H: Create/Update Contact
    H-->>N: Contact ID
    N->>S: Notify Sales Team
    S-->>N: Posted

    Note over F,S: Complete Lead Flow
```

**Workflow Steps:**

1. **Form Submission (Webhook)**
```javascript
{
  "email": "john@example.com",
  "name": "John Doe",
  "company": "Example Corp"
}
```

2. **Enrich with Clearbit**
```javascript
GET https://company.clearbit.com/v2/companies/find

Params:
domain={{$json.email.split('@')[1]}}

Returns:
- Company name
- Industry
- Employee count
- Revenue
- Technologies used
- Social profiles
```

3. **Create HubSpot Contact**
```javascript
{
  "properties": {
    "email": "{{$json.email}}",
    "firstname": "{{$json.name.split(' ')[0]}}",
    "lastname": "{{$json.name.split(' ')[1]}}",
    "company": "{{$json.enrichment.name}}",
    "industry": "{{$json.enrichment.industry}}",
    "numberofemployees": "{{$json.enrichment.employees}}",
    "annualrevenue": "{{$json.enrichment.revenue}}",
    "hs_lead_status": "NEW"
  }
}
```

4. **Slack Notification**
```javascript
{
  "channel": "#sales",
  "text": "New lead from {{$json.enrichment.name}}",
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*New High-Value Lead* 🎯"
      }
    },
    {
      "type": "section",
      "fields": [
        {"type": "mrkdwn", "text": "*Name:*\n{{$json.name}}"},
        {"type": "mrkdwn", "text": "*Company:*\n{{$json.enrichment.name}}"},
        {"type": "mrkdwn", "text": "*Industry:*\n{{$json.enrichment.industry}}"},
        {"type": "mrkdwn", "text": "*Size:*\n{{$json.enrichment.employees}} employees"}
      ]
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": {"type": "plain_text", "text": "View in HubSpot"},
          "url": "https://app.hubspot.com/contacts/{{portalId}}/contact/{{$json.contactId}}"
        }
      ]
    }
  ]
}
```

---

## Hands-On Exercises

### [Exercise 1: Automated File Backup System](./exercises/exercise-1-backup-system.md)

Build a workflow that:
1. Monitors Google Drive folder for new files
2. Downloads the file
3. Uploads to AWS S3 with date-based path
4. Optionally syncs to Dropbox
5. Logs activity to Google Sheets
6. Sends daily summary via email

**Learning Goals:**
- Multi-platform file operations
- Binary data handling
- Date-based organization
- Backup strategies
- Activity logging

---

### [Exercise 2: Lead Capture and CRM Sync](./exercises/exercise-2-lead-capture.md)

Create a complete lead management workflow:
1. Webhook receives form submission
2. Validates and cleanses data
3. Enriches with external API (Clearbit/Hunter)
4. Creates/updates contact in HubSpot
5. Creates deal if qualified
6. Assigns to sales rep based on territory
7. Sends welcome email
8. Notifies team in Slack
9. Logs to database for analytics

**Learning Goals:**
- Data validation
- API enrichment
- CRM automation
- Lead routing logic
- Multi-channel notifications

---

### [Exercise 3: Social Media Post Scheduler](./exercises/exercise-3-social-scheduler.md)

Build a social media management system:
1. Google Sheets as content calendar
2. Schedule trigger checks for posts due
3. Prepares media (downloads from Drive)
4. Posts to Twitter/LinkedIn
5. Logs success/failure
6. Sends analytics report weekly
7. Handles errors and retries

**Learning Goals:**
- Content scheduling
- Multi-platform posting
- Media handling
- Error recovery
- Analytics tracking

---

## Real-World Project: E-Commerce Order Fulfillment

```mermaid
graph TB
    A[Stripe: Payment Success] --> B[Create Customer Record]
    B --> C[Update Salesforce Opportunity]
    C --> D[Create HubSpot Deal]
    D --> E[Generate Invoice PDF]
    E --> F[Upload to Google Drive]
    F --> G[Send Receipt Email]
    G --> H[Update Order in Database]
    H --> I[Notify Fulfillment via Slack]
    I --> J[Create Shipping Label via API]
    J --> K[Send Tracking Email]
    K --> L[Log to Analytics]

    style A fill:#635BFF,stroke:#333,color:#fff
    style E fill:#00A8A0,stroke:#333,color:#fff
    style I fill:#7D54D9,stroke:#333,color:#fff
    style L fill:#4CAF50,stroke:#333,color:#fff
```

**Implementation Overview:**

1. **Payment Processing** (Stripe Webhook)
2. **Customer Management** (Database + CRM)
3. **Document Generation** (Invoice PDF)
4. **File Storage** (Google Drive/S3)
5. **Email Notifications** (Gmail/SendGrid)
6. **Team Communication** (Slack)
7. **Shipping Integration** (ShipStation/EasyPost)
8. **Analytics** (Google Sheets/Database)

---

## Integration Best Practices

### File Handling

**Binary Data Management:**
```javascript
// Download file
const binaryData = $binary.data;

// Convert to base64
const base64 = binaryData.toString('base64');

// Get file info
const fileName = $binary.data.fileName;
const mimeType = $binary.data.mimeType;
const fileSize = $binary.data.fileSize;

// Stream large files
// Use chunked uploads for files > 100MB
```

**File Naming Conventions:**
```javascript
// Date-based
const filename = `${DateTime.now().toFormat('yyyy-MM-dd')}_${$json.original_name}`;

// User-based
const filename = `${$json.userId}_${$json.documentType}_${Date.now()}.pdf`;

// Organized paths
const path = `/${$json.year}/${$json.month}/${$json.category}/${filename}`;
```

### API Rate Limiting

**Implement Backoff:**
```javascript
// Check rate limit headers
const remaining = parseInt($json.headers['x-ratelimit-remaining']);
const resetTime = parseInt($json.headers['x-ratelimit-reset']);

if (remaining < 10) {
  const waitTime = (resetTime - Date.now()) / 1000;
  // Wait or queue for later
}

// Exponential backoff
const retryDelay = Math.pow(2, attemptNumber) * 1000;
```

**Batch Operations:**
```javascript
// Instead of 100 separate API calls
const items = $input.all();
const batches = [];

for (let i = 0; i < items.length; i += 50) {
  batches.push(items.slice(i, i + 50));
}

// Process batches with delays
for (const batch of batches) {
  await processBatch(batch);
  await sleep(1000); // Rate limit compliance
}
```

### Error Handling

**Graceful Degradation:**
```javascript
try {
  // Primary API call
  const result = await primaryAPI.call();
  return result;
} catch (error) {
  if (error.statusCode === 429) {
    // Rate limited - try again later
    return { retry: true, delay: 60000 };
  } else if (error.statusCode >= 500) {
    // Server error - use fallback
    return await fallbackAPI.call();
  } else {
    // Client error - log and continue
    console.error('API Error:', error);
    return { error: true, message: error.message };
  }
}
```

### Security

**Sensitive Data:**
```javascript
// Redact in logs
const logData = {
  ...data,
  creditCard: data.creditCard ? '****' + data.creditCard.slice(-4) : null,
  ssn: data.ssn ? '***-**-' + data.ssn.slice(-4) : null
};

// Don't store in workflow
// Use secure credential storage
// Encrypt before database storage
```

---

## Troubleshooting

### File Storage Issues

**Google Drive:**
- ✓ Check file size limits (750GB upload/day)
- ✓ Verify folder permissions
- ✓ Confirm OAuth scopes
- ✓ Handle shared drive differences

**AWS S3:**
- ✓ Verify IAM permissions
- ✓ Check bucket policies
- ✓ Confirm region settings
- ✓ Validate Content-Type headers

**Dropbox:**
- ✓ Monitor rate limits
- ✓ Use chunked upload for large files
- ✓ Check team vs personal account
- ✓ Verify app permissions

### CRM Issues

**HubSpot:**
- ✓ Check property permissions
- ✓ Verify contact deduplication
- ✓ Confirm association types
- ✓ Review API limits (100 req/10s)

**Salesforce:**
- ✓ Validate SOQL syntax
- ✓ Check object permissions
- ✓ Verify field-level security
- ✓ Review governor limits

### Payment Processing

**Stripe:**
- ✓ Test with test mode keys
- ✓ Verify webhook signatures
- ✓ Handle idempotency
- ✓ Check API version compatibility

---

## Key Takeaways

- ✓ File storage integrations enable document automation across platforms
- ✓ CRM integrations centralize customer data and sales processes
- ✓ Payment webhooks enable real-time order processing
- ✓ Social media APIs automate content distribution
- ✓ Multi-service workflows create powerful business automations
- ✓ Proper error handling ensures reliability
- ✓ Security best practices protect sensitive data

---

## Additional Resources

**File Storage:**
- [Google Drive API](https://developers.google.com/drive/api/guides/about-sdk)
- [Dropbox API](https://www.dropbox.com/developers/documentation)
- [AWS S3 Documentation](https://docs.aws.amazon.com/s3/)

**CRM:**
- [HubSpot API](https://developers.hubspot.com/docs/api/overview)
- [Salesforce API](https://developer.salesforce.com/docs/apis)

**Payments:**
- [Stripe API](https://stripe.com/docs/api)
- [Stripe Webhooks](https://stripe.com/docs/webhooks)

**Social Media:**
- [Twitter API v2](https://developer.twitter.com/en/docs/twitter-api)
- [LinkedIn API](https://docs.microsoft.com/en-us/linkedin/)

---

## Next Steps

**Continue to:** [Module 4: Data Transformation and Logic](../../module-04-data-transformation/README.md)

**Practice More:**
- Build a complete backup automation
- Create a lead nurturing workflow
- Implement payment processing
- Automate social media posting
- Connect your real business tools

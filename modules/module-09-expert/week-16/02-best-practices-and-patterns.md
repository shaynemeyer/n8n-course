# n8n Best Practices and Expert Patterns

This comprehensive guide consolidates all the best practices, design patterns, and expert techniques you need to build production-ready n8n workflows.

## Table of Contents

1. [Workflow Design Principles](#workflow-design-principles)
2. [Naming Conventions](#naming-conventions)
3. [Error Handling Patterns](#error-handling-patterns)
4. [Performance Optimization](#performance-optimization)
5. [Security Best Practices](#security-best-practices)
6. [Testing Strategies](#testing-strategies)
7. [Monitoring and Observability](#monitoring-and-observability)
8. [Deployment Patterns](#deployment-patterns)
9. [Advanced Design Patterns](#advanced-design-patterns)
10. [Code Organization](#code-organization)
11. [Team Collaboration](#team-collaboration)
12. [Troubleshooting Guide](#troubleshooting-guide)

---

## Workflow Design Principles

### 1. Single Responsibility Principle

Each workflow should have one clear, well-defined purpose.

**Good Example**:
```
Workflow: process-order
Purpose: Process incoming orders from webhook to fulfillment

Workflow: sync-inventory
Purpose: Synchronize inventory across sales channels

Workflow: send-order-confirmation
Purpose: Send order confirmation emails
```

**Bad Example**:
```
Workflow: order-system
Purpose: Handle orders, inventory, emails, analytics, and reporting
(Too many responsibilities)
```

**Benefits**:
- Easier to understand
- Easier to test
- Easier to maintain
- Reusable components

### 2. Modular Architecture

Break complex processes into smaller, reusable subworkflows.

**Pattern**:
```
Main Workflow: order-orchestrator
  ├─ Subworkflow: validate-order
  ├─ Subworkflow: check-inventory
  ├─ Subworkflow: process-payment
  ├─ Subworkflow: create-shipment
  └─ Subworkflow: send-notifications
```

**Implementation**:
```javascript
// Main workflow
Start
  ↓
Execute Workflow: validate-order
  ↓
IF Valid:
  ↓
  Execute Workflow: check-inventory
  ↓
  Execute Workflow: process-payment
  ↓
  Execute Workflow: create-shipment
  ↓
  Execute Workflow: send-notifications
```

**Benefits**:
- Code reuse
- Independent testing
- Easy updates
- Clear structure

### 3. Fail Fast, Fail Clearly

Validate inputs early and provide clear error messages.

**Pattern**:
```javascript
// At workflow start
Start
  ↓
Validate Required Fields
  ↓
IF Missing Fields:
  → Return Clear Error Message
  → Log Error
  → Exit
  ↓
Validate Data Types
  ↓
IF Invalid Types:
  → Return Clear Error Message
  → Log Error
  → Exit
  ↓
Validate Business Rules
  ↓
IF Rules Violated:
  → Return Clear Error Message
  → Log Error
  → Exit
  ↓
Proceed with Processing
```

**Implementation**:
```javascript
// Function Node: Validate Order
const order = $input.first().json;
const errors = [];

// Check required fields
const required = ['orderId', 'customerId', 'items', 'total'];
for (const field of required) {
  if (!order[field]) {
    errors.push(`Missing required field: ${field}`);
  }
}

// Check data types
if (typeof order.total !== 'number') {
  errors.push('Total must be a number');
}

if (!Array.isArray(order.items) || order.items.length === 0) {
  errors.push('Order must have at least one item');
}

// Check business rules
if (order.total < 0) {
  errors.push('Order total cannot be negative');
}

if (errors.length > 0) {
  throw new Error(`Validation failed: ${errors.join(', ')}`);
}

return { json: order };
```

### 4. Idempotency

Design workflows to produce the same result when executed multiple times with the same input.

**Pattern**:
```javascript
// Check if already processed
Start
  ↓
Generate Unique Request ID
  ↓
Check if ID Already Processed
  ↓
IF Already Processed:
  → Return Cached Result
  → Exit
  ↓
Process Request
  ↓
Cache Result with ID
  ↓
Return Result
```

**Implementation**:
```javascript
// Function Node: Idempotent Process
const requestId = $json.orderId + '_' + $json.timestamp;

// Check cache
const cached = await $('Cache DB').getNode({
  table: 'processed_requests',
  where: { request_id: requestId }
});

if (cached && cached.length > 0) {
  // Already processed, return cached result
  return cached[0].result;
}

// Process request
const result = await processOrder($json);

// Cache result
await $('Cache DB').insert({
  table: 'processed_requests',
  data: {
    request_id: requestId,
    result: result,
    processed_at: new Date()
  }
});

return { json: result };
```

---

## Naming Conventions

### Workflow Names

**Convention**: `action-noun` or `noun-action`

**Examples**:
```
Good:
- process-order
- sync-inventory
- send-notification
- customer-onboarding
- payment-processor

Avoid:
- workflow1
- test
- new_workflow
- untitled
```

### Node Names

**Convention**: Descriptive, action-oriented

**Examples**:
```
Good:
- Validate Order Data
- Check Inventory Availability
- Process Payment via Stripe
- Send Confirmation Email
- Update Database Record

Avoid:
- HTTP Request
- Function
- IF
- Set
```

### Variable Names

**Convention**: `camelCase` for JavaScript, `snake_case` for parameters

**Examples**:
```javascript
// Good
const orderTotal = $json.order.total;
const customerId = $json.customer.id;
const isValidOrder = validateOrder(order);

// Avoid
const x = $json.order.total;
const data = $json.customer.id;
const flag = validateOrder(order);
```

### Credential Names

**Convention**: `service-purpose` or `service-environment`

**Examples**:
```
Good:
- stripe-production
- shopify-staging
- google-sheets-reporting
- slack-notifications

Avoid:
- api-key
- token1
- credentials
```

---

## Error Handling Patterns

### Pattern 1: Try-Catch with Retry

```javascript
Main Flow
  ↓
Try:
  ↓
  Risky Operation
  ↓
  Success Path
Catch:
  ↓
  Log Error
  ↓
  IF Retryable Error:
    ↓
    Wait (with exponential backoff)
    ↓
    Retry (max 3 times)
    ↓
    IF Still Failing:
      → Error Workflow
  ELSE:
    → Error Workflow
```

**Implementation**:
```javascript
// Function Node: Retry Logic
const maxRetries = 3;
const baseDelay = 1000; // 1 second

for (let attempt = 1; attempt <= maxRetries; attempt++) {
  try {
    const result = await makeApiCall();
    return { json: result };
  } catch (error) {
    if (attempt === maxRetries) {
      // Last attempt failed
      throw new Error(`Failed after ${maxRetries} attempts: ${error.message}`);
    }

    // Check if error is retryable
    if (isRetryableError(error)) {
      // Exponential backoff
      const delay = baseDelay * Math.pow(2, attempt - 1);
      await wait(delay);
      continue;
    }

    // Non-retryable error
    throw error;
  }
}

function isRetryableError(error) {
  const retryableStatus = [408, 429, 500, 502, 503, 504];
  return retryableStatus.includes(error.statusCode);
}
```

### Pattern 2: Circuit Breaker

```javascript
// Prevent cascade failures

Check Circuit State
  ↓
IF Open:
  → Return Cached/Fallback Response
  → Exit
  ↓
IF Half-Open:
  → Try Single Request
  → IF Success:
      → Close Circuit
  → IF Failure:
      → Open Circuit
  ↓
IF Closed:
  → Execute Request
  → Track Failures
  → IF Failure Threshold Exceeded:
      → Open Circuit
```

**Implementation**:
```javascript
// Function Node: Circuit Breaker
const circuit = {
  state: 'closed', // closed, open, half-open
  failureCount: 0,
  failureThreshold: 5,
  timeout: 60000, // 1 minute
  lastFailureTime: null
};

async function callWithCircuitBreaker(fn) {
  // Check if circuit should be half-open
  if (circuit.state === 'open') {
    const timeSinceFailure = Date.now() - circuit.lastFailureTime;
    if (timeSinceFailure > circuit.timeout) {
      circuit.state = 'half-open';
    } else {
      return getFallbackResponse();
    }
  }

  try {
    const result = await fn();

    // Success - close circuit
    if (circuit.state === 'half-open') {
      circuit.state = 'closed';
      circuit.failureCount = 0;
    }

    return result;
  } catch (error) {
    circuit.failureCount++;
    circuit.lastFailureTime = Date.now();

    // Open circuit if threshold exceeded
    if (circuit.failureCount >= circuit.failureThreshold) {
      circuit.state = 'open';
    }

    throw error;
  }
}
```

### Pattern 3: Dead Letter Queue

```javascript
// Handle failed messages

Main Processing
  ↓
IF Processing Fails:
  ↓
  Increment Retry Count
  ↓
  IF Retry Count < Max:
    → Add to Retry Queue
  ELSE:
    → Add to Dead Letter Queue
    → Notify Team
    → Create Incident
```

### Pattern 4: Compensating Transactions

```javascript
// Rollback distributed transactions

Start Transaction
  ↓
Step 1: Reserve Inventory
  ↓
Step 2: Process Payment
  ↓
Step 3: Create Shipment
  ↓
IF Any Step Fails:
  ↓
  Compensate Step 2: Refund Payment
  ↓
  Compensate Step 1: Release Inventory
  ↓
  Notify User of Failure
```

---

## Performance Optimization

### 1. Batch Processing

Process multiple items together instead of one at a time.

**Before (Slow)**:
```javascript
FOR EACH User:
  ↓
  Get User Details (API call)
  ↓
  Process User
  ↓
  Update Database (API call)
// 2N API calls for N users
```

**After (Fast)**:
```javascript
Split into Batches of 100
  ↓
FOR EACH Batch:
  ↓
  Get All User Details (1 API call)
  ↓
  Process All Users
  ↓
  Bulk Update Database (1 API call)
// 2*(N/100) API calls for N users
```

**Implementation**:
```javascript
// Function Node: Batch Processing
const batchSize = 100;
const items = $input.all();
const batches = [];

for (let i = 0; i < items.length; i += batchSize) {
  batches.push(items.slice(i, i + batchSize));
}

const results = [];

for (const batch of batches) {
  // Process batch
  const batchIds = batch.map(item => item.json.id);

  // Single API call for batch
  const batchResults = await apiClient.getBatch(batchIds);

  results.push(...batchResults);
}

return results.map(r => ({ json: r }));
```

### 2. Parallel Execution

Execute independent operations concurrently.

**Before (Sequential)**:
```javascript
Get User Data → 2s
  ↓
Get Order Data → 2s
  ↓
Get Analytics Data → 2s
  ↓
Total: 6 seconds
```

**After (Parallel)**:
```javascript
         ┌─ Get User Data → 2s
Start ───┼─ Get Order Data → 2s
         └─ Get Analytics Data → 2s
           ↓
        Merge Results
        Total: 2 seconds
```

**Implementation**:
Use n8n's native parallel execution by connecting multiple nodes to the same source.

### 3. Caching

Cache frequently accessed data.

**Pattern**:
```javascript
Check Cache
  ↓
IF Cache Hit:
  → Return Cached Data
  ↓
ELSE:
  → Fetch from Source
  → Store in Cache
  → Return Data
```

**Implementation**:
```javascript
// Function Node: Caching
const cacheKey = `user_${$json.userId}`;
const cacheTTL = 3600; // 1 hour

// Check cache
const cached = await redis.get(cacheKey);
if (cached) {
  return { json: JSON.parse(cached) };
}

// Fetch from source
const userData = await api.getUser($json.userId);

// Store in cache
await redis.set(cacheKey, JSON.stringify(userData), 'EX', cacheTTL);

return { json: userData };
```

### 4. Pagination

Handle large datasets efficiently.

**Implementation**:
```javascript
// Function Node: Pagination
async function getAllResults(endpoint) {
  const allResults = [];
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const response = await api.get(endpoint, {
      params: {
        page: page,
        per_page: 100
      }
    });

    allResults.push(...response.data);

    // Check if there are more pages
    hasMore = response.data.length === 100;
    page++;

    // Safety limit
    if (page > 1000) {
      throw new Error('Too many pages');
    }
  }

  return allResults;
}
```

### 5. Database Query Optimization

**Tips**:
- Use indexes on frequently queried fields
- Limit result sets
- Use specific field selection instead of `SELECT *`
- Avoid N+1 queries

**Example**:
```javascript
// Bad: N+1 queries
for (const order of orders) {
  const customer = await db.query(
    'SELECT * FROM customers WHERE id = ?',
    [order.customer_id]
  );
}

// Good: Single query with join
const ordersWithCustomers = await db.query(`
  SELECT o.*, c.*
  FROM orders o
  JOIN customers c ON o.customer_id = c.id
  WHERE o.status = 'pending'
`);
```

---

## Security Best Practices

### 1. Credential Management

**DO**:
- Use n8n's credential system
- Rotate credentials regularly
- Use environment-specific credentials
- Grant minimum necessary permissions
- Enable credential testing

**DON'T**:
- Hardcode credentials in workflows
- Share credentials across environments
- Store credentials in plaintext
- Use overly permissive credentials

### 2. Input Validation

Always validate and sanitize inputs.

**Implementation**:
```javascript
// Function Node: Input Validation
function validateInput(data) {
  const errors = [];

  // Whitelist allowed fields
  const allowedFields = ['name', 'email', 'phone'];
  const sanitized = {};

  for (const field of allowedFields) {
    if (data[field]) {
      sanitized[field] = sanitizeString(data[field]);
    }
  }

  // Validate email
  if (sanitized.email && !isValidEmail(sanitized.email)) {
    errors.push('Invalid email format');
  }

  // Validate phone
  if (sanitized.phone && !isValidPhone(sanitized.phone)) {
    errors.push('Invalid phone format');
  }

  if (errors.length > 0) {
    throw new Error(`Validation failed: ${errors.join(', ')}`);
  }

  return sanitized;
}

function sanitizeString(str) {
  // Remove potentially dangerous characters
  return str.replace(/[<>\"']/g, '');
}
```

### 3. API Security

**Best Practices**:
- Use HTTPS always
- Validate webhook signatures
- Implement rate limiting
- Use API versioning
- Log security events

**Webhook Validation**:
```javascript
// Function Node: Validate Webhook
function validateWebhookSignature(payload, signature, secret) {
  const crypto = require('crypto');

  const computedSignature = crypto
    .createHmac('sha256', secret)
    .update(JSON.stringify(payload))
    .digest('hex');

  if (signature !== computedSignature) {
    throw new Error('Invalid webhook signature');
  }

  return true;
}
```

### 4. Data Protection

**Practices**:
- Encrypt sensitive data at rest
- Use HTTPS for data in transit
- Anonymize PII when possible
- Implement data retention policies
- Log data access

### 5. Access Control

**Implementation**:
```javascript
// Function Node: Check Permissions
function checkPermission(user, resource, action) {
  const permissions = {
    admin: ['read', 'write', 'delete'],
    editor: ['read', 'write'],
    viewer: ['read']
  };

  const userRole = user.role;
  const allowedActions = permissions[userRole] || [];

  if (!allowedActions.includes(action)) {
    throw new Error(`Unauthorized: ${userRole} cannot ${action} ${resource}`);
  }

  return true;
}
```

---

## Testing Strategies

### 1. Unit Testing

Test individual workflow components.

**Test Structure**:
```javascript
describe('Order Validation', () => {
  it('should validate complete order', () => {
    const order = {
      orderId: '123',
      customerId: '456',
      items: [{ sku: 'ABC', quantity: 1 }],
      total: 99.99
    };

    const result = validateOrder(order);
    expect(result.valid).toBe(true);
  });

  it('should reject order with missing fields', () => {
    const order = {
      orderId: '123'
      // Missing other fields
    };

    expect(() => validateOrder(order)).toThrow();
  });

  it('should reject order with negative total', () => {
    const order = {
      orderId: '123',
      customerId: '456',
      items: [{ sku: 'ABC', quantity: 1 }],
      total: -10
    };

    expect(() => validateOrder(order)).toThrow('negative');
  });
});
```

### 2. Integration Testing

Test complete workflow execution.

**Test Scenario**:
```javascript
describe('Order Processing Workflow', () => {
  it('should process valid order end-to-end', async () => {
    // Setup
    const mockOrder = createMockOrder();

    // Execute workflow
    const result = await executeWorkflow('process-order', {
      data: mockOrder
    });

    // Verify
    expect(result.status).toBe('success');
    expect(result.orderId).toBe(mockOrder.orderId);

    // Check side effects
    const dbRecord = await db.getOrder(mockOrder.orderId);
    expect(dbRecord.status).toBe('processed');

    const emailSent = await checkEmailSent(mockOrder.customer.email);
    expect(emailSent).toBe(true);
  });
});
```

### 3. Load Testing

Test performance under load.

**Approach**:
```javascript
// Load test script
async function loadTest() {
  const concurrentUsers = 100;
  const requestsPerUser = 10;

  const results = {
    total: 0,
    success: 0,
    failure: 0,
    avgResponseTime: 0
  };

  const promises = [];

  for (let i = 0; i < concurrentUsers; i++) {
    for (let j = 0; j < requestsPerUser; j++) {
      promises.push(
        executeWorkflow('process-order')
          .then(result => {
            results.success++;
            results.avgResponseTime += result.duration;
          })
          .catch(error => {
            results.failure++;
          })
      );
    }
  }

  await Promise.all(promises);

  results.total = results.success + results.failure;
  results.avgResponseTime /= results.success;

  console.log('Load Test Results:', results);
}
```

### 4. Edge Case Testing

Test boundary conditions and unusual scenarios.

**Test Cases**:
```javascript
describe('Edge Cases', () => {
  it('should handle empty input', () => {
    expect(() => processOrder(null)).toThrow();
    expect(() => processOrder({})).toThrow();
    expect(() => processOrder([])).toThrow();
  });

  it('should handle very large orders', () => {
    const largeOrder = {
      orderId: '123',
      items: Array(10000).fill({ sku: 'ABC', quantity: 1 })
    };

    const result = processOrder(largeOrder);
    expect(result.status).toBe('success');
  });

  it('should handle special characters', () => {
    const order = {
      customerName: "O'Brien & Sons <script>alert('xss')</script>"
    };

    const result = sanitizeOrder(order);
    expect(result.customerName).not.toContain('<script>');
  });

  it('should handle concurrent requests', async () => {
    const promises = Array(50).fill(null).map(() =>
      processOrder(mockOrder)
    );

    const results = await Promise.all(promises);
    expect(results.every(r => r.status === 'success')).toBe(true);
  });
});
```

---

## Monitoring and Observability

### 1. Logging Best Practices

**What to Log**:
- Workflow start/end
- Important state changes
- Error conditions
- Performance metrics
- Security events

**Log Structure**:
```javascript
// Structured logging
const log = {
  timestamp: new Date().toISOString(),
  level: 'info', // debug, info, warn, error
  workflow: 'process-order',
  executionId: $execution.id,
  action: 'payment-processed',
  data: {
    orderId: $json.orderId,
    amount: $json.total,
    paymentMethod: $json.payment.method
  },
  duration: 1234, // milliseconds
  success: true
};

await logService.write(log);
```

**Don't Log**:
- Sensitive data (passwords, API keys, PII)
- Excessive detail in production
- Duplicate information

### 2. Metrics Collection

**Key Metrics**:
```javascript
// Workflow Metrics
const metrics = {
  // Performance
  execution_time: Duration,
  items_processed: Count,
  throughput: ItemsPerSecond,

  // Reliability
  success_rate: Percentage,
  error_rate: Percentage,
  retry_count: Count,

  // Business
  orders_processed: Count,
  revenue_generated: Currency,
  conversion_rate: Percentage
};
```

**Implementation**:
```javascript
// Function Node: Collect Metrics
const startTime = Date.now();

try {
  // Execute workflow logic
  const result = await processOrder($json);

  // Record success metrics
  await metrics.increment('orders.processed');
  await metrics.gauge('orders.value', result.total);
  await metrics.timing('orders.duration', Date.now() - startTime);

  return { json: result };
} catch (error) {
  // Record error metrics
  await metrics.increment('orders.failed');
  await metrics.increment(`orders.error.${error.code}`);

  throw error;
}
```

### 3. Alerting

**Alert Rules**:
```javascript
const alerts = [
  {
    name: 'High Error Rate',
    condition: 'error_rate > 5%',
    window: '5 minutes',
    severity: 'critical',
    channels: ['slack', 'pagerduty']
  },
  {
    name: 'Slow Performance',
    condition: 'p95_latency > 5000ms',
    window: '10 minutes',
    severity: 'warning',
    channels: ['slack']
  },
  {
    name: 'Low Throughput',
    condition: 'throughput < 100 per hour',
    window: '1 hour',
    severity: 'warning',
    channels: ['email']
  }
];
```

### 4. Health Checks

**Implementation**:
```javascript
// Workflow: Health Check
Schedule: Every 1 minute
  ↓
Check Critical Components:
  ↓
  Test Database Connection
  ↓
  Test API Endpoints
  ↓
  Check Queue Lengths
  ↓
  Verify Disk Space
  ↓
Calculate Health Score
  ↓
IF Health Score < Threshold:
  → Trigger Alert
  → Update Status Page
  ↓
Record Health Metrics
```

### 5. Dashboards

**Essential Dashboards**:

**Operational Dashboard**:
- Execution rate
- Success/failure rate
- Average execution time
- Active workflows
- Error trends

**Business Dashboard**:
- Orders processed
- Revenue generated
- Customer signups
- Conversion rates
- Top products

---

## Deployment Patterns

### 1. Environment Strategy

**Setup**:
```
Development (dev)
  ↓
Staging (staging)
  ↓
Production (prod)
```

**Configuration**:
```javascript
const config = {
  dev: {
    apiUrl: 'https://api.dev.example.com',
    dbConnection: 'dev-database',
    logLevel: 'debug'
  },
  staging: {
    apiUrl: 'https://api.staging.example.com',
    dbConnection: 'staging-database',
    logLevel: 'info'
  },
  prod: {
    apiUrl: 'https://api.example.com',
    dbConnection: 'prod-database',
    logLevel: 'warn'
  }
};

const currentEnv = process.env.N8N_ENV || 'dev';
const envConfig = config[currentEnv];
```

### 2. Blue-Green Deployment

**Process**:
```
1. Deploy new version to Green environment
2. Run tests on Green
3. IF tests pass:
   - Switch traffic to Green
   - Keep Blue as backup
4. IF issues detected:
   - Switch traffic back to Blue
5. After stability period:
   - Decommission Blue
   - Green becomes new Blue
```

### 3. Canary Deployment

**Process**:
```
1. Deploy new version to small subset (5%)
2. Monitor metrics closely
3. IF metrics good:
   - Increase traffic (25%)
   - Monitor
4. Continue gradual rollout
5. IF issues detected at any stage:
   - Rollback immediately
```

### 4. Feature Flags

**Implementation**:
```javascript
// Function Node: Feature Flag
const features = await featureFlagService.getFlags();

if (features.newPaymentProcessor) {
  // Use new processor
  result = await newPaymentService.process($json);
} else {
  // Use old processor
  result = await oldPaymentService.process($json);
}
```

---

## Advanced Design Patterns

### Pattern 1: Event Sourcing

Store all changes as events, not just current state.

**Implementation**:
```javascript
// Store events
const events = [
  { type: 'OrderCreated', data: { orderId: '123', ... } },
  { type: 'PaymentReceived', data: { orderId: '123', ... } },
  { type: 'OrderShipped', data: { orderId: '123', ... } }
];

// Reconstruct state from events
function getCurrentState(events) {
  let state = {};

  for (const event of events) {
    state = applyEvent(state, event);
  }

  return state;
}
```

### Pattern 2: CQRS (Command Query Responsibility Segregation)

Separate read and write operations.

**Structure**:
```
Commands (Write):
- CreateOrder
- UpdateOrder
- CancelOrder

Queries (Read):
- GetOrder
- ListOrders
- SearchOrders

Command DB (Write-optimized)
  ↓
Event Stream
  ↓
Query DB (Read-optimized)
```

### Pattern 3: Saga Pattern

Manage distributed transactions.

```javascript
// Saga: Book Flight Journey

Step 1: Reserve Flight
  → IF Success: Continue
  → IF Failure: End

Step 2: Reserve Hotel
  → IF Success: Continue
  → IF Failure: Compensate (Cancel Flight)

Step 3: Process Payment
  → IF Success: Continue
  → IF Failure: Compensate (Cancel Hotel, Cancel Flight)

Step 4: Send Confirmation
```

### Pattern 4: Strangler Fig

Gradually replace legacy system.

```javascript
// Route requests based on capability

Incoming Request
  ↓
IF Feature Available in New System:
  → Route to New System
ELSE:
  → Route to Legacy System
  → Log for Future Migration
```

---

## Code Organization

### Workflow File Structure

```
workflows/
├── core/
│   ├── order-processing/
│   │   ├── main.json
│   │   ├── validate.json
│   │   └── fulfill.json
│   ├── inventory/
│   └── customer/
├── integrations/
│   ├── shopify/
│   ├── stripe/
│   └── sendgrid/
├── utilities/
│   ├── error-handler.json
│   ├── logger.json
│   └── metrics.json
└── monitoring/
    ├── health-check.json
    └── alerts.json
```

### Function Code Organization

```javascript
// Good: Organized, reusable functions

// Validation functions
function validateEmail(email) { ... }
function validatePhone(phone) { ... }
function validateOrder(order) { ... }

// Business logic functions
function calculateTotal(items) { ... }
function applyDiscount(total, code) { ... }
function calculateTax(amount, location) { ... }

// Utility functions
function formatCurrency(amount) { ... }
function parseDate(dateString) { ... }
function generateId() { ... }

// Main execution
const order = $json;
validateOrder(order);
const total = calculateTotal(order.items);
const taxAmount = calculateTax(total, order.location);
return { json: { total: total + taxAmount } };
```

---

## Team Collaboration

### 1. Documentation Standards

**Workflow Documentation Template**:
```markdown
# Workflow: process-order

## Purpose
Process incoming orders from webhook to fulfillment

## Trigger
Webhook from Shopify when new order is created

## Input Schema
{
  "orderId": "string",
  "customer": { ... },
  "items": [ ... ],
  "total": "number"
}

## Output
{
  "status": "success|failure",
  "orderId": "string",
  "fulfillmentId": "string"
}

## Dependencies
- Stripe (payment processing)
- ShipStation (fulfillment)
- SendGrid (notifications)

## Error Handling
- Payment failures: Notify customer, log to error queue
- Inventory issues: Backorder workflow
- API failures: Retry 3 times with exponential backoff

## Monitoring
- Dashboard: /dashboard/orders
- Alerts: Slack #ops-alerts
- Metrics: order_processing_time, order_success_rate

## Maintainer
@john-doe

## Last Updated
2024-01-15
```

### 2. Code Review Checklist

- [ ] Clear, descriptive node names
- [ ] Proper error handling
- [ ] Input validation
- [ ] Logging implemented
- [ ] Documentation updated
- [ ] Tests added/updated
- [ ] Performance considered
- [ ] Security reviewed
- [ ] No hardcoded credentials
- [ ] Consistent style

### 3. Version Control

**Git Strategy**:
```bash
# Feature development
git checkout -b feature/new-payment-processor
# Make changes
git commit -m "Add new payment processor integration"
git push origin feature/new-payment-processor
# Create pull request

# After review and approval
git checkout main
git merge feature/new-payment-processor
git tag v1.2.0
git push --tags
```

---

## Troubleshooting Guide

### Common Issues and Solutions

#### Issue 1: Workflow Timeout

**Symptoms**: Execution stops after 2 minutes

**Diagnosis**:
- Check execution history
- Look for long-running operations
- Check for infinite loops

**Solutions**:
- Break into smaller workflows
- Use background execution
- Implement pagination
- Add timeouts to external calls

#### Issue 2: Memory Errors

**Symptoms**: "Out of memory" errors

**Diagnosis**:
- Large dataset processing
- Memory leaks
- Inefficient data structures

**Solutions**:
- Process in batches
- Use streams
- Clear unused variables
- Optimize data structures

#### Issue 3: Rate Limiting

**Symptoms**: 429 Too Many Requests errors

**Diagnosis**:
- Check API rate limits
- Review call frequency
- Check concurrent executions

**Solutions**:
- Implement rate limiting
- Add delays between calls
- Use batch endpoints
- Cache responses

#### Issue 4: Data Inconsistencies

**Symptoms**: Mismatched data across systems

**Diagnosis**:
- Race conditions
- Failed transactions
- Missing error handling

**Solutions**:
- Implement locking
- Use transactions
- Add idempotency
- Implement reconciliation

---

## Conclusion

These best practices and patterns represent years of collective n8n experience. Apply them thoughtfully to your projects, adapt them to your specific needs, and always prioritize:

1. **Clarity** over cleverness
2. **Reliability** over features
3. **Maintainability** over optimization
4. **Security** over convenience

Remember: The best workflow is one that works reliably, is easy to understand, and can be maintained by your team.

**Happy automating!**

# Complete Guide to Business Automations

## Table of Contents

1. [Order Processing Automation](#order-processing-automation)
2. [Customer Onboarding Flows](#customer-onboarding-flows)
3. [Automated Reporting Systems](#automated-reporting-systems)
4. [Multi-Stage Approval Workflows](#multi-stage-approval-workflows)
5. [Event-Driven Architectures](#event-driven-architectures)

---

# Order Processing Automation

## Introduction

Order processing automation streamlines the entire lifecycle from order placement through fulfillment and delivery. This is one of the most impactful business automations for e-commerce and retail businesses.

## Complete Order Processing Workflow

### Architecture Overview

```
Order Sources (Web, Mobile, API, Manual)
    ↓
Order Intake & Validation
    ↓
Inventory Check & Reservation
    ↓
Payment Processing
    ↓
Fraud Detection (optional)
    ↓
Order Confirmation
    ↓
Fulfillment Process
    ↓
Shipping & Tracking
    ↓
Customer Notifications
    ↓
Post-Delivery Follow-up
```

## Implementation: E-commerce Order Processing

### Phase 1: Order Intake

```javascript
// Workflow: Order Intake

Webhook: POST /orders/new
  Body: {
    orderId, customerId, items[], shippingAddress,
    billingAddress, paymentMethod, total
  }
    ↓
Function: Validate Order Data
```

```javascript
// Validation logic
const order = $json;

// Required fields
const requiredFields = ['orderId', 'customerId', 'items', 'shippingAddress', 'total'];
const missingFields = requiredFields.filter(field => !order[field]);

if (missingFields.length > 0) {
  throw new Error(`Missing required fields: ${missingFields.join(', ')}`);
}

// Validate items
if (!Array.isArray(order.items) || order.items.length === 0) {
  throw new Error('Order must contain at least one item');
}

// Validate amounts
order.items.forEach(item => {
  if (item.quantity <= 0 || item.price <= 0) {
    throw new Error(`Invalid item: ${item.productId}`);
  }
});

// Calculate total
const calculatedTotal = order.items.reduce((sum, item) => {
  return sum + (item.price * item.quantity);
}, 0);

if (Math.abs(calculatedTotal - order.total) > 0.01) {
  throw new Error('Order total mismatch');
}

return {
  json: {
    ...order,
    status: 'validated',
    validatedAt: new Date().toISOString()
  }
};
```

### Phase 2: Inventory Management

```
Function: Check Inventory Availability
    ↓
Database: Query inventory levels
    ↓
Function: Calculate availability
```

```javascript
// Check inventory
const items = $json.items;
const inventoryChecks = [];

for (const item of items) {
  // Query inventory database
  const inventoryLevel = await getInventoryLevel(item.productId, item.warehouseId);

  inventoryChecks.push({
    productId: item.productId,
    requested: item.quantity,
    available: inventoryLevel,
    sufficient: inventoryLevel >= item.quantity
  });
}

const allAvailable = inventoryChecks.every(check => check.sufficient);

if (!allAvailable) {
  const outOfStock = inventoryChecks
    .filter(c => !c.sufficient)
    .map(c => c.productId);

  return {
    json: {
      status: 'insufficient_inventory',
      outOfStockItems: outOfStock,
      checks: inventoryChecks
    }
  };
}

return {
  json: {
    ...$ json,
    status: 'inventory_available',
    inventoryChecks
  }
};
```

```
IF (Inventory Available):
    ↓
    Database: Reserve Inventory
    PostgreSQL: BEGIN TRANSACTION
      UPDATE inventory
      SET reserved = reserved + quantity
      WHERE product_id = ? AND warehouse_id = ?
    ↓
    Database: Create Reservation Record
    INSERT INTO inventory_reservations
      (order_id, product_id, quantity, reserved_at)
ELSE:
    ↓
    Email: Notify customer of out-of-stock
    ↓
    Database: Update order status to 'pending_inventory'
```

### Phase 3: Payment Processing

```
Function: Prepare Payment
    ↓
HTTP Request: Stripe Create Payment Intent
  URL: https://api.stripe.com/v1/payment_intents
  Method: POST
  Headers:
    Authorization: Bearer {{$credentials.stripeSecretKey}}
  Body:
    amount: {{$json.total * 100}}  // cents
    currency: usd
    customer: {{$json.customerId}}
    metadata:
      orderId: {{$json.orderId}}
```

```javascript
// Process payment response
const paymentIntent = $json;

if (paymentIntent.status === 'succeeded') {
  return {
    json: {
      orderId: $json.metadata.orderId,
      paymentStatus: 'paid',
      paymentId: paymentIntent.id,
      amount: paymentIntent.amount / 100,
      paidAt: new Date().toISOString()
    }
  };
} else if (paymentIntent.status === 'requires_action') {
  // 3D Secure or additional verification needed
  return {
    json: {
      orderId: $json.metadata.orderId,
      paymentStatus: 'pending_verification',
      paymentId: paymentIntent.id,
      clientSecret: paymentIntent.client_secret
    }
  };
} else {
  throw new Error(`Payment failed: ${paymentIntent.status}`);
}
```

```
IF (Payment Success):
    ↓
    Database: Update order status = 'paid'
    ↓
    Database: Update inventory (committed)
    ↓
    Continue to fulfillment
ELSE:
    ↓
    Database: Release inventory reservation
    ↓
    Email: Payment failed notification
    ↓
    Database: Update order status = 'payment_failed'
```

### Phase 4: Fulfillment

```
Database: Create fulfillment record
    ↓
HTTP Request: Shipping Label API
  (e.g., ShipStation, EasyPost, Shippo)
```

```javascript
// Create shipping label
const shippingRequest = {
  to_address: {
    name: $json.shippingAddress.name,
    street1: $json.shippingAddress.street1,
    city: $json.shippingAddress.city,
    state: $json.shippingAddress.state,
    zip: $json.shippingAddress.zip,
    country: $json.shippingAddress.country
  },
  from_address: {
    name: 'Your Company',
    street1: '123 Warehouse St',
    city: 'Commerce City',
    state: 'CA',
    zip: '90001',
    country: 'US'
  },
  parcel: {
    length: 10,
    width: 8,
    height: 6,
    weight: calculateWeight($json.items)
  },
  service: $json.shippingMethod || 'USPS Priority'
};

// Call shipping API
const label = await createShippingLabel(shippingRequest);

return {
  json: {
    orderId: $json.orderId,
    trackingNumber: label.tracking_code,
    shippingLabel: label.postage_label.label_url,
    carrier: label.carrier,
    estimatedDelivery: label.estimated_delivery_date
  }
};
```

```
Webhook: Notify warehouse system
  Payload: {
    orderId, items, shippingLabel, priority
  }
    ↓
Database: Update order status = 'in_fulfillment'
```

### Phase 5: Customer Notifications

```
Email: Order Confirmation
  Template: order_confirmation.html
  Variables: {
    customerName: {{$json.customerName}},
    orderId: {{$json.orderId}},
    items: {{$json.items}},
    total: {{$json.total}},
    trackingNumber: {{$json.trackingNumber}}
  }
    ↓
Slack: Notify sales team
  Channel: #orders
  Message: "New order #{{$json.orderId}} - ${{$json.total}}"
```

Email template example:

```html
<!DOCTYPE html>
<html>
<head>
  <style>
    .order-details { font-family: Arial; }
    .item { padding: 10px; border-bottom: 1px solid #ccc; }
  </style>
</head>
<body>
  <div class="order-details">
    <h2>Thank you for your order, {{customerName}}!</h2>

    <p>Order #{{orderId}} has been confirmed.</p>

    <h3>Order Details:</h3>
    {{#each items}}
    <div class="item">
      <strong>{{this.name}}</strong><br>
      Quantity: {{this.quantity}}<br>
      Price: ${{this.price}}
    </div>
    {{/each}}

    <p><strong>Total: ${{total}}</strong></p>

    <p>Tracking: {{trackingNumber}}</p>

    <p>Estimated delivery: {{estimatedDelivery}}</p>
  </div>
</body>
</html>
```

### Phase 6: Tracking & Updates

```
Webhook: Shipping carrier updates
  (Tracking events from carrier)
    ↓
Function: Parse tracking event
    ↓
Database: Update shipment status
    ↓
IF (Status = 'delivered'):
    Email: Delivery confirmation
    Database: Update order = 'delivered'
    Schedule: Post-delivery survey (3 days)
```

### Phase 7: Returns & Refunds

```
Webhook: Return request
    ↓
Database: Create return record
    ↓
Email: Return authorization to customer
  Include: Return label, instructions
    ↓
Wait: For item to arrive at warehouse
    ↓
Webhook: Return received
    ↓
IF (Refund approved):
    HTTP Request: Stripe refund
    ↓
    Database: Update inventory (+returned qty)
    ↓
    Email: Refund confirmation
```

## Error Handling

```javascript
// Comprehensive error handler
try {
  // Main order processing logic
} catch (error) {
  // Categorize error
  const errorType = categorizeError(error);

  switch(errorType) {
    case 'inventory':
      // Release reservations
      await releaseInventory(order.orderId);
      // Notify customer
      await sendOutOfStockEmail(order);
      break;

    case 'payment':
      // Log failed payment
      await logPaymentFailure(order, error);
      // Notify customer to retry
      await sendPaymentFailedEmail(order);
      break;

    case 'shipping':
      // Mark for manual review
      await flagOrderForReview(order, 'shipping_error');
      // Alert fulfillment team
      await alertFulfillmentTeam(order, error);
      break;

    default:
      // Unknown error - escalate
      await createIncident({
        orderId: order.orderId,
        error: error.message,
        stack: error.stack
      });
      await alertEngineering(order, error);
  }

  // Always log
  await logError({
    orderId: order.orderId,
    step: currentStep,
    error: error.message,
    timestamp: new Date()
  });
}
```

---

# Customer Onboarding Flows

## Introduction

Customer onboarding is critical for user activation and retention. Automated onboarding ensures consistent experience, reduces manual work, and improves completion rates.

## Complete Onboarding Workflow

### Employee Onboarding Example

```
Trigger: New Hire Accepted Offer
    ↓
[Day -7] Pre-boarding
  - Send welcome email
  - Collect paperwork
  - Order equipment
    ↓
[Day -3] Setup
  - Create accounts (email, Slack, etc.)
  - Assign to teams
  - Schedule orientation
    ↓
[Day 1] First Day
  - Welcome message
  - Orientation meeting
  - Assign buddy
  - Provide access
    ↓
[Week 1] Initial Training
  - Training modules
  - Tool setup
  - Team introductions
    ↓
[Month 1] Check-ins
  - Manager 1:1s
  - Progress tracking
  - Feedback collection
    ↓
[Day 30] Survey
  - Onboarding feedback
  - Satisfaction score
  - Improvement suggestions
```

### Implementation

#### Step 1: Trigger & Setup

```
Webhook: POST /onboarding/start
  Body: {
    employeeId, name, email, department,
    role, startDate, manager, equipment
  }
    ↓
Database: Create onboarding record
  INSERT INTO onboarding_records
    (employee_id, status, start_date, created_at)
  VALUES (?, 'initiated', ?, NOW())
    ↓
Function: Calculate timeline
```

```javascript
// Calculate onboarding dates
const startDate = new Date($json.startDate);
const timeline = {
  preBoardingStart: new Date(startDate - 7 * 24 * 60 * 60 * 1000),
  equipmentOrder: new Date(startDate - 5 * 24 * 60 * 60 * 1000),
  accountCreation: new Date(startDate - 3 * 24 * 60 * 60 * 1000),
  firstDay: startDate,
  weekOneEnd: new Date(startDate.getTime() + 7 * 24 * 60 * 60 * 1000),
  monthOneEnd: new Date(startDate.getTime() + 30 * 24 * 60 * 60 * 1000)
};

return {
  json: {
    ...$json,
    timeline,
    tasks: generateOnboardingTasks(timeline, $json)
  }
};
```

#### Step 2: Pre-boarding (Day -7)

```
Schedule: {{timeline.preBoardingStart}}
    ↓
Email: Welcome email to new hire
  Template: pre_boarding_welcome
  Attachments: Employee handbook, Forms
    ↓
Airtable/Notion: Create onboarding checklist
  Tasks:
    - Complete tax forms
    - Upload ID documents
    - Emergency contact info
    - Direct deposit setup
    ↓
Slack: Create employee channel
  #onboarding-{{employeeName}}
    ↓
HTTP Request: Equipment order system
  POST /equipment/order
  Body: {
    employeeId, items: {{$json.equipment}},
    deliveryDate: {{$json.startDate}},
    deliveryAddress: {{$json.homeAddress}}
  }
```

#### Step 3: Account Provisioning (Day -3)

```
Schedule: {{timeline.accountCreation}}
    ↓
[Parallel Account Creation]

Branch 1: Email Account
  HTTP Request: Google Workspace Admin API
    POST /admin/directory/v1/users
    Body: {
      name: {givenName, familyName},
      primaryEmail: {{email}},
      password: {{generateTempPassword()}},
      orgUnitPath: "/{{department}}"
    }

Branch 2: Slack Account
  HTTP Request: Slack API
    POST /api/users.admin.invite
    Body: {
      email: {{email}},
      channels: {{departmentChannels}},
      real_name: {{name}}
    }

Branch 3: Project Management
  HTTP Request: Jira/Asana API
    Create user, assign to projects

Branch 4: Other Tools
  [CRM, Dev tools, Design tools, etc.]

[All branches merge]
    ↓
Database: Store account details
    ↓
Function: Generate credentials document
    ↓
Email: Send credentials to new hire
  (Password via separate secure channel)
```

#### Step 4: First Day Automation

```
Schedule: {{timeline.firstDay}} 09:00 AM
    ↓
Email: First day welcome
  - Building access instructions
  - Parking info
  - Meeting schedule
  - Who to ask for help
    ↓
Slack: Post to team channel
  "Please welcome {{name}} to the {{department}} team!"
    ↓
Slack: DM to buddy
  "You're assigned as buddy for {{name}}.
   Please reach out and schedule coffee!"
    ↓
Calendar: Create orientation meetings
  - HR orientation (10 AM)
  - Team introduction (2 PM)
  - Manager 1:1 (4 PM)
    ↓
Notion/Confluence: Grant access to docs
  - Team documentation
  - Company policies
  - Project resources
```

#### Step 5: Training & Tasks

```
Schedule: First Week
    ↓
[Each Day: Send daily task list]

Day 1:
  - Complete HR orientation
  - Set up workstation
  - Meet your team

Day 2:
  - Review product overview
  - Set up development environment
  - First team standup

Day 3:
  - Shadow team member
  - Review codebase
  - First pair programming

[Continue through week...]
    ↓
Database: Track task completion
    ↓
Slack: Daily progress update to manager
```

#### Step 6: Check-ins & Feedback

```
Schedule: End of Week 1, 2, 4
    ↓
Email: Check-in survey
  Questions:
    - How is onboarding going?
    - Do you have what you need?
    - Any blockers?
    - Feedback on process?
    ↓
Function: Analyze responses
    ↓
IF (Issues flagged):
    Slack: Alert manager and HR
    Create: Jira ticket for resolution
    ↓
Database: Store feedback
    ↓
Airtable: Update onboarding dashboard
```

#### Step 7: 30-Day Review

```
Schedule: Day 30
    ↓
Email: Comprehensive onboarding survey
    ↓
Collect responses
    ↓
Function: Calculate metrics
  - Completion rate
  - Satisfaction score
  - Time to productivity
  - Suggestions for improvement
    ↓
Google Sheets: Update onboarding analytics
    ↓
Email: Report to HR and management
    ↓
Database: Mark onboarding = 'completed'
```

### Onboarding Analytics Dashboard

```
Workflow: Onboarding Analytics

Schedule: Daily 9 AM
    ↓
Database: Query onboarding data
  SELECT
    COUNT(*) as total_onboardings,
    AVG(completion_rate) as avg_completion,
    AVG(satisfaction_score) as avg_satisfaction,
    COUNT(*) FILTER (WHERE status = 'completed') as completed,
    COUNT(*) FILTER (WHERE status = 'in_progress') as in_progress
  FROM onboarding_records
  WHERE created_at >= NOW() - INTERVAL '30 days'
    ↓
Function: Calculate trends
    ↓
Google Sheets: Update dashboard
  Sheet: Onboarding Metrics
    ↓
IF (Satisfaction < 4.0):
    Slack: Alert HR
    "Onboarding satisfaction below target"
```

---

# Automated Reporting Systems

## Introduction

Automated reporting saves time, ensures consistency, and provides timely insights. Build reports that aggregate data, generate visualizations, and distribute to stakeholders.

## Complete Reporting Workflow

### Daily Sales Report Example

```
Schedule: Every day at 8 AM
    ↓
[Data Collection - Parallel]

Branch 1: E-commerce Sales
  HTTP Request: Shopify Orders API
    /admin/api/orders.json
    Params: created_at_min= yesterday, status=any

Branch 2: In-store Sales
  Database: Query POS system
    SELECT * FROM sales
    WHERE sale_date = YESTERDAY

Branch 3: Marketplace Sales
  HTTP Request: Amazon MWS API
    GetOrders (yesterday)

[Merge all sources]
    ↓
Function: Aggregate and calculate metrics
```

```javascript
// Calculate metrics
const allSales = $json;

const metrics = {
  totalOrders: allSales.length,
  totalRevenue: allSales.reduce((sum, sale) => sum + sale.total, 0),
  averageOrderValue: 0,
  topProducts: {},
  salesByChannel: {
    online: 0,
    inStore: 0,
    marketplace: 0
  },
  salesByCategory: {},
  newCustomers: 0,
  returningCustomers: 0
};

// Calculate AOV
metrics.averageOrderValue = metrics.totalRevenue / metrics.totalOrders;

// Top products
allSales.forEach(sale => {
  sale.items.forEach(item => {
    if (!metrics.topProducts[item.productId]) {
      metrics.topProducts[item.productId] = {
        name: item.name,
        quantity: 0,
        revenue: 0
      };
    }
    metrics.topProducts[item.productId].quantity += item.quantity;
    metrics.topProducts[item.productId].revenue += item.price * item.quantity;
  });

  // Channel breakdown
  metrics.salesByChannel[sale.channel] += sale.total;

  // Customer type
  if (sale.isNewCustomer) {
    metrics.newCustomers++;
  } else {
    metrics.returningCustomers++;
  }
});

// Sort top products
metrics.topProducts = Object.values(metrics.topProducts)
  .sort((a, b) => b.revenue - a.revenue)
  .slice(0, 10);

return { json: metrics };
```

```
Function: Compare to previous periods
  Calculate: Yesterday vs day before
  Calculate: This week vs last week
  Calculate: This month vs last month
    ↓
Function: Generate insights
```

```javascript
// Generate insights
const insights = [];

// Revenue trend
if (metrics.revenueChange > 0.1) {
  insights.push(`Revenue up ${(metrics.revenueChange * 100).toFixed(1)}% vs yesterday`);
} else if (metrics.revenueChange < -0.1) {
  insights.push(`⚠️ Revenue down ${Math.abs(metrics.revenueChange * 100).toFixed(1)}% vs yesterday`);
}

// Top product
if (metrics.topProducts.length > 0) {
  const top = metrics.topProducts[0];
  insights.push(`Top seller: ${top.name} (${top.quantity} units, $${top.revenue.toFixed(2)})`);
}

// New customers
const newCustomerRate = metrics.newCustomers / metrics.totalOrders;
if (newCustomerRate > 0.5) {
  insights.push(`Strong new customer acquisition: ${(newCustomerRate * 100).toFixed(0)}%`);
}

return { json: { ...metrics, insights } };
```

```
Google Sheets: Update data
  Sheet: Daily Sales
  Range: Append row with date and metrics
    ↓
HTTP Request: Create chart
  (Using Google Sheets API or Chart.js)
    ↓
Function: Generate HTML report
```

```javascript
// Generate HTML report
const html = `
<!DOCTYPE html>
<html>
<head>
  <style>
    body { font-family: Arial, sans-serif; max-width: 800px; margin: 20px auto; }
    .metric { display: inline-block; width: 200px; padding: 20px; margin: 10px; background: #f5f5f5; border-radius: 8px; }
    .metric h3 { margin: 0; font-size: 14px; color: #666; }
    .metric .value { font-size: 32px; font-weight: bold; color: #333; }
    .insight { padding: 10px; background: #e3f2fd; border-left: 4px solid #2196f3; margin: 10px 0; }
    table { width: 100%; border-collapse: collapse; margin: 20px 0; }
    th, td { padding: 12px; text-align: left; border-bottom: 1px solid #ddd; }
    th { background: #f5f5f5; }
  </style>
</head>
<body>
  <h1>Daily Sales Report - ${new Date().toLocaleDateString()}</h1>

  <div>
    <div class="metric">
      <h3>Total Orders</h3>
      <div class="value">${metrics.totalOrders}</div>
    </div>
    <div class="metric">
      <h3>Revenue</h3>
      <div class="value">$${metrics.totalRevenue.toFixed(2)}</div>
    </div>
    <div class="metric">
      <h3>Avg Order Value</h3>
      <div class="value">$${metrics.averageOrderValue.toFixed(2)}</div>
    </div>
  </div>

  <h2>Key Insights</h2>
  ${metrics.insights.map(insight => `<div class="insight">${insight}</div>`).join('')}

  <h2>Top Products</h2>
  <table>
    <tr>
      <th>Product</th>
      <th>Units Sold</th>
      <th>Revenue</th>
    </tr>
    ${metrics.topProducts.map(product => `
      <tr>
        <td>${product.name}</td>
        <td>${product.quantity}</td>
        <td>$${product.revenue.toFixed(2)}</td>
      </tr>
    `).join('')}
  </table>

  <h2>Sales by Channel</h2>
  <table>
    <tr>
      <th>Channel</th>
      <th>Revenue</th>
      <th>% of Total</th>
    </tr>
    ${Object.entries(metrics.salesByChannel).map(([channel, revenue]) => `
      <tr>
        <td>${channel}</td>
        <td>$${revenue.toFixed(2)}</td>
        <td>${((revenue / metrics.totalRevenue) * 100).toFixed(1)}%</td>
      </tr>
    `).join('')}
  </table>
</body>
</html>
`;

return { json: { html } };
```

```
Email: Send report
  To: management-team@company.com
  Subject: Daily Sales Report - {{date}}
  HTML Body: {{$json.html}}
  Attachments: Chart image (if generated)
    ↓
Slack: Post summary
  Channel: #daily-metrics
  Message: Daily sales: {{totalOrders}} orders, ${{totalRevenue}}
    ↓
Database: Log report generation
  INSERT INTO report_log (report_type, generated_at, recipient_count)
```

### Dynamic Report Generation

```
Webhook: POST /reports/generate
  Body: {
    reportType, dateRange, filters, recipients
  }
    ↓
Function: Validate parameters
    ↓
Switch: Route by report type
  Case 'sales': → Sales report workflow
  Case 'inventory': → Inventory report workflow
  Case 'customer': → Customer report workflow
  Case 'financial': → Financial report workflow
    ↓
[Report-specific logic]
    ↓
Function: Apply filters
    ↓
Function: Generate report
    ↓
IF (Schedule for future):
    Database: Store scheduled report
ELSE:
    Distribute immediately
```

---

# Multi-Stage Approval Workflows

## Introduction

Approval workflows ensure proper authorization, create audit trails, and enforce business rules. Common use cases include expense approvals, document reviews, access requests, and procurement.

## Complete Approval Workflow

### Expense Approval Example

```
Webhook: Expense submitted
  Body: {
    employeeId, amount, category,
    description, receipts[], date
  }
    ↓
Database: Create expense record
  status = 'pending_approval'
    ↓
Function: Determine approval path
```

```javascript
// Determine approval path
const expense = $json;
let approvers = [];

// Rule-based routing
if (expense.amount < 100) {
  // Auto-approve small expenses
  approvers = ['auto-approved'];
} else if (expense.amount < 1000) {
  // Manager only
  approvers = [expense.managerId];
} else if (expense.amount < 5000) {
  // Manager → Department Head
  approvers = [expense.managerId, expense.departmentHeadId];
} else {
  // Manager → Department Head → VP Finance
  approvers = [expense.managerId, expense.departmentHeadId, 'vp-finance'];
}

// Category-specific rules
if (expense.category === 'travel' && expense.amount > 2000) {
  // Travel expenses > $2k need CFO approval
  approvers.push('cfo');
}

return {
  json: {
    ...expense,
    approvalChain: approvers,
    currentApprover: approvers[0],
    approvalStage: 0
  }
};
```

```
IF (Auto-approved):
    Database: Update status = 'approved'
    Email: Notify employee
    Trigger: Reimbursement process
ELSE:
    → Send to first approver
```

### Approval Request

```
Function: Get approver details
  Query user database for approver info
    ↓
Email: Approval request
  To: {{approver.email}}
  Subject: Expense Approval Needed - ${{amount}}
  Body: HTML template with:
    - Employee name
    - Amount
    - Category
    - Description
    - Receipt links
    - Approve/Reject buttons
```

Email template with action buttons:

```html
<div class="approval-request">
  <h2>Expense Approval Request</h2>

  <div class="expense-details">
    <p><strong>Employee:</strong> {{employeeName}}</p>
    <p><strong>Amount:</strong> ${{amount}}</p>
    <p><strong>Category:</strong> {{category}}</p>
    <p><strong>Date:</strong> {{date}}</p>
    <p><strong>Description:</strong> {{description}}</p>
  </div>

  <div class="receipts">
    <h3>Receipts:</h3>
    {{#each receipts}}
      <a href="{{this.url}}">View Receipt {{@index}}</a><br>
    {{/each}}
  </div>

  <div class="actions">
    <a href="{{approveUrl}}" class="btn-approve">Approve</a>
    <a href="{{rejectUrl}}" class="btn-reject">Reject</a>
  </div>
</div>
```

```
Slack: Notification
  Channel: DM to approver
  Message: "Expense approval needed"
  Attachment: Expense details
  Actions: Approve / Reject / Request Info
    ↓
Database: Create approval task
  INSERT INTO approval_tasks
    (expense_id, approver_id, status, created_at)
```

### Approval Response

```
Webhook: Approval action
  URL: /approvals/respond
  Body: {
    expenseId, approverId, action, comments
  }
    ↓
Database: Validate approver
  Check: Is this person authorized?
  Check: Is this the current pending approval?
    ↓
Database: Record approval decision
  UPDATE approval_tasks
  SET status = {{action}},
      comments = {{comments}},
      decided_at = NOW()
```

```javascript
// Process approval decision
const response = $json;

if (response.action === 'approved') {
  // Move to next approval stage
  const expense = await getExpense(response.expenseId);
  const nextStage = expense.approvalStage + 1;

  if (nextStage < expense.approvalChain.length) {
    // More approvals needed
    return {
      json: {
        ...expense,
        approvalStage: nextStage,
        currentApprover: expense.approvalChain[nextStage],
        status: 'pending_approval',
        previousApprover: response.approverId
      }
    };
  } else {
    // All approvals complete
    return {
      json: {
        ...expense,
        status: 'approved',
        approvedAt: new Date(),
        finalApprover: response.approverId
      }
    };
  }
} else if (response.action === 'rejected') {
  // Rejection stops the chain
  return {
    json: {
      ...expense,
      status: 'rejected',
      rejectedBy: response.approverId,
      rejectionReason: response.comments
    }
  };
} else if (response.action === 'request_info') {
  // Request more information
  return {
    json: {
      ...expense,
      status: 'info_requested',
      requestedBy: response.approverId,
      infoRequest: response.comments
    }
  };
}
```

```
IF (More approvals needed):
    → Send to next approver
ELSE IF (Fully approved):
    Database: Update expense = 'approved'
    ↓
    Email: Notify employee
    Subject: Expense Approved
    ↓
    Trigger: Reimbursement workflow
    ↓
    Database: Log to audit trail
ELSE IF (Rejected):
    Database: Update expense = 'rejected'
    ↓
    Email: Notify employee
    Subject: Expense Rejected
    Body: Reason: {{rejectionReason}}
```

### Escalation Logic

```
Schedule: Check for stale approvals
  Frequency: Every 6 hours
    ↓
Database: Query pending approvals
  SELECT * FROM approval_tasks
  WHERE status = 'pending'
    AND created_at < NOW() - INTERVAL '2 days'
    ↓
For each overdue approval:
    ↓
    Email: Reminder to approver
    Subject: REMINDER: Approval needed
    ↓
    IF (Overdue > 3 days):
        Slack: Escalate to manager
        Email: CC manager on reminder
    ↓
    IF (Overdue > 5 days):
        Database: Auto-escalate
        Current approver → Next level up
        ↓
        Email: Notify of escalation
```

### Audit Trail

```
Database: Approval audit log

CREATE TABLE approval_audit (
  id SERIAL PRIMARY KEY,
  expense_id INTEGER,
  approver_id INTEGER,
  action VARCHAR(50),
  comments TEXT,
  timestamp TIMESTAMP DEFAULT NOW(),
  ip_address INET,
  user_agent TEXT
);

-- Every approval action:
INSERT INTO approval_audit
  (expense_id, approver_id, action, comments, ip_address)
VALUES
  (?, ?, ?, ?, ?);
```

Audit report workflow:

```
Schedule: Monthly
    ↓
Database: Query approval metrics
  - Average approval time
  - Rejection rate
  - Escalation rate
  - Approvals by person
    ↓
Google Sheets: Update audit dashboard
    ↓
Email: Send to compliance team
```

---

# Event-Driven Architectures

## Introduction

Event-driven architecture (EDA) enables systems to react to events in real-time, creating responsive and scalable applications. Events trigger workflows, which process and potentially create more events.

## Event-Driven Patterns

### 1. Webhook Orchestration

```
External System → Webhook → n8n → Process → Multiple Actions

Example:
Stripe Payment → Webhook → n8n →
  - Update database
  - Send receipt email
  - Notify fulfillment
  - Update analytics
  - Trigger loyalty points
```

Implementation:

```
Webhook: POST /webhooks/stripe
  Body: Stripe event object
    ↓
Function: Verify webhook signature
```

```javascript
// Verify Stripe webhook
const stripe = require('stripe');
const payload = $json;
const signature = $headers['stripe-signature'];
const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET;

try {
  const event = stripe.webhooks.constructEvent(
    payload,
    signature,
    webhookSecret
  );

  return { json: event };
} catch (err) {
  throw new Error('Webhook signature verification failed');
}
```

```
Switch: Route by event type

  Case 'payment_intent.succeeded':
    → Payment success workflow

  Case 'payment_intent.payment_failed':
    → Payment failed workflow

  Case 'customer.subscription.created':
    → New subscription workflow

  Case 'customer.subscription.deleted':
    → Cancellation workflow
```

### 2. Event Streaming

```
Event Source → Stream → n8n → Process → Destinations

Example:
User Actions → Kafka → n8n →
  - Real-time analytics
  - Personalization engine
  - Fraud detection
  - Notification system
```

### 3. Pub/Sub Pattern

```
Publisher → Topic → Subscribers

Example:
Order Created →
  Topic: order.created →
    - Inventory system (reserve stock)
    - Payment system (process payment)
    - Email system (send confirmation)
    - Analytics system (track metrics)
```

Implementation with Redis Pub/Sub:

```
Workflow 1: Publisher

Webhook: Order created
    ↓
Redis: PUBLISH order.created {{orderData}}
    ↓
Response: Order queued
```

```
Workflow 2: Subscriber (Inventory)

Redis Trigger: SUBSCRIBE order.created
    ↓
Function: Process order event
    ↓
Database: Reserve inventory
    ↓
Redis: PUBLISH inventory.reserved {{data}}
```

```
Workflow 3: Subscriber (Payment)

Redis Trigger: SUBSCRIBE order.created
    ↓
HTTP Request: Process payment
    ↓
IF (Success):
    Redis: PUBLISH payment.completed {{data}}
ELSE:
    Redis: PUBLISH payment.failed {{data}}
```

### 4. Event Sourcing

Store all changes as sequence of events:

```
Command → Validate → Store Event → Update State → Notify

Example:
Update User Profile →
  Validate changes →
  Store: UserProfileUpdated event →
  Update: User table →
  Publish: profile.updated event
```

Implementation:

```
Webhook: POST /users/{{id}}/update
  Body: { field: value changes }
    ↓
Function: Create event
```

```javascript
// Create event
const event = {
  eventId: generateUUID(),
  eventType: 'UserProfileUpdated',
  aggregateId: $json.userId,
  timestamp: new Date().toISOString(),
  version: $json.currentVersion + 1,
  data: {
    changes: $json.changes,
    previousValues: $json.previousValues
  },
  metadata: {
    userId: $json.actorId,
    ipAddress: $json.ip,
    userAgent: $json.userAgent
  }
};

return { json: event };
```

```
Database: Store event
  INSERT INTO events
    (event_id, event_type, aggregate_id, data, metadata)
    ↓
Database: Update aggregate
  Apply event to current state
    ↓
Redis: Publish event
  PUBLISH user.profile.updated {{event}}
    ↓
Response: Success
```

Event replay for rebuilding state:

```
Workflow: Rebuild User State

Manual Trigger: userId
    ↓
Database: Get all events for user
  SELECT * FROM events
  WHERE aggregate_id = {{userId}}
  ORDER BY version ASC
    ↓
Function: Replay events
```

```javascript
// Replay events to rebuild state
const events = $json.events;
let userState = {
  id: events[0].aggregateId,
  version: 0
};

events.forEach(event => {
  switch(event.eventType) {
    case 'UserCreated':
      userState = {
        ...userState,
        ...event.data,
        createdAt: event.timestamp
      };
      break;

    case 'UserProfileUpdated':
      Object.assign(userState, event.data.changes);
      break;

    case 'EmailVerified':
      userState.emailVerified = true;
      userState.emailVerifiedAt = event.timestamp;
      break;

    // ... other event types
  }

  userState.version = event.version;
});

return { json: userState };
```

### 5. CQRS (Command Query Responsibility Segregation)

Separate read and write operations:

```
Commands (Writes) → Event Store → Event Handlers → Read Models

Queries (Reads) → Read Models (Optimized for queries)
```

Implementation:

```
Command: Create Order
    ↓
Validate command
    ↓
Create events: OrderCreated, ItemsReserved, PaymentProcessed
    ↓
Store in event store
    ↓
Publish events
    ↓
Event handlers update read models:
  - Order summary table
  - Customer order history
  - Analytics aggregates
  - Search index
```

Read model example:

```
Query: Get customer orders
    ↓
Query read model (not event store)
  SELECT * FROM customer_order_summary
  WHERE customer_id = ?
  ORDER BY created_at DESC
    ↓
Return optimized view
```

## Real-Time Processing Example

### User Activity Tracking

```
User Action → Capture → Process → Store → Notify

Frontend → POST /events/track
  Body: {
    userId, event, properties, timestamp
  }
    ↓
n8n Webhook
    ↓
Function: Enrich event
```

```javascript
// Enrich event with additional context
const event = {
  ...$ json,
  eventId: generateUUID(),
  sessionId: $json.sessionId || getSessionId($json.userId),
  userAgent: $headers['user-agent'],
  ipAddress: $headers['x-forwarded-for'],
  timestamp: $json.timestamp || new Date().toISOString()
};

// Add user context
const user = await getUserContext(event.userId);
event.userContext = {
  plan: user.plan,
  signupDate: user.signupDate,
  totalOrders: user.totalOrders
};

return { json: event };
```

```
[Parallel Processing]

Branch 1: Real-time analytics
  ClickHouse/BigQuery: INSERT event

Branch 2: Trigger-based actions
  IF (event.type === 'cart_abandoned'):
    Schedule: Send email in 1 hour
  IF (event.type === 'signup'):
    Trigger: Onboarding workflow
  IF (event.type === 'purchase'):
    Trigger: Thank you email

Branch 3: Personalization
  Update: User profile
  Recalculate: Recommendations

Branch 4: Fraud detection
  IF (Suspicious pattern):
    Alert: Security team

[All branches complete]
    ↓
Response: Event tracked
```

## Best Practices

1. **Idempotency**: Handle duplicate events gracefully
2. **Event Versioning**: Plan for schema changes
3. **Dead Letter Queues**: Handle failed event processing
4. **Monitoring**: Track event processing metrics
5. **Replay Capability**: Ability to reprocess events

---

## Summary

This guide covered five major business automation patterns:

1. **Order Processing** - End-to-end e-commerce fulfillment
2. **Customer Onboarding** - Multi-step user activation
3. **Automated Reporting** - Data aggregation and distribution
4. **Approval Workflows** - Multi-stage authorization
5. **Event-Driven Architecture** - Real-time reactive systems

Each pattern provides building blocks for complex business automations. Mix and match these patterns to build sophisticated automation solutions tailored to your business needs.

## Next Steps

1. Study the hands-on projects in `02-hands-on-projects.md`
2. Choose a real business process to automate
3. Design the workflow using patterns from this guide
4. Build incrementally, test thoroughly
5. Document and deploy

Remember: Start simple, iterate based on feedback, and always prioritize reliability over features!

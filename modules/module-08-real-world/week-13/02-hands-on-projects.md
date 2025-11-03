# Week 13 Hands-On Projects

## Overview

Complete these three comprehensive projects to demonstrate mastery of complex business automation. Each project represents a real-world scenario requiring multi-system integration, sophisticated logic, and production-ready error handling.

---

## Project 1: Complete E-commerce Order Fulfillment System

### Objective

Build an end-to-end order processing system that handles the complete lifecycle from order placement through delivery, including inventory management, payment processing, shipping, and customer notifications.

### Requirements

**Core Features:**
- Multi-channel order intake (webhook, API, manual)
- Real-time inventory validation and reservation
- Payment processing integration (Stripe or PayPal)
- Automated shipping label generation
- Order status tracking
- Customer email notifications
- Returns and refunds handling
- Admin dashboard for monitoring

**Technical Requirements:**
- PostgreSQL database for order/inventory data
- Webhook endpoints for external systems
- Error handling with retry logic
- Audit logging
- Performance: Handle 100+ orders/hour

**Deliverables:**
- Working order processing workflows (5-8 workflows)
- Database schema and setup scripts
- API documentation
- Email templates
- Dashboard/reporting
- Complete documentation

### Architecture

```
Order Sources
    ↓
Order Intake Workflow
    ↓
Inventory Management Workflow
    ↓
Payment Processing Workflow
    ↓
Fulfillment Workflow
    ↓
Shipping & Tracking Workflow
    ↓
Notifications Workflow
    ↓
Returns/Refunds Workflow (Optional)
```

### Step-by-Step Implementation

#### Phase 1: Setup (2 hours)

1. **Database Setup**

Create `setup-database.sql`:

```sql
-- Orders table
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  order_number VARCHAR(50) UNIQUE NOT NULL,
  customer_id INTEGER NOT NULL,
  customer_email VARCHAR(255) NOT NULL,
  status VARCHAR(50) NOT NULL,
  total_amount DECIMAL(10,2) NOT NULL,
  currency VARCHAR(3) DEFAULT 'USD',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  metadata JSONB
);

-- Order items
CREATE TABLE order_items (
  id SERIAL PRIMARY KEY,
  order_id INTEGER REFERENCES orders(id),
  product_id INTEGER NOT NULL,
  product_name VARCHAR(255) NOT NULL,
  quantity INTEGER NOT NULL,
  unit_price DECIMAL(10,2) NOT NULL,
  total_price DECIMAL(10,2) NOT NULL
);

-- Inventory
CREATE TABLE inventory (
  id SERIAL PRIMARY KEY,
  product_id INTEGER UNIQUE NOT NULL,
  product_name VARCHAR(255) NOT NULL,
  sku VARCHAR(100) UNIQUE,
  quantity_available INTEGER NOT NULL DEFAULT 0,
  quantity_reserved INTEGER NOT NULL DEFAULT 0,
  reorder_point INTEGER DEFAULT 10,
  last_updated TIMESTAMP DEFAULT NOW()
);

-- Inventory reservations
CREATE TABLE inventory_reservations (
  id SERIAL PRIMARY KEY,
  order_id INTEGER REFERENCES orders(id),
  product_id INTEGER NOT NULL,
  quantity INTEGER NOT NULL,
  reserved_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP,
  released BOOLEAN DEFAULT FALSE
);

-- Shipments
CREATE TABLE shipments (
  id SERIAL PRIMARY KEY,
  order_id INTEGER REFERENCES orders(id),
  tracking_number VARCHAR(255),
  carrier VARCHAR(100),
  label_url TEXT,
  estimated_delivery DATE,
  actual_delivery DATE,
  status VARCHAR(50),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Order status history
CREATE TABLE order_status_history (
  id SERIAL PRIMARY KEY,
  order_id INTEGER REFERENCES orders(id),
  from_status VARCHAR(50),
  to_status VARCHAR(50),
  changed_at TIMESTAMP DEFAULT NOW(),
  notes TEXT
);

-- Indexes
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_inventory_product ON inventory(product_id);
CREATE INDEX idx_reservations_order ON inventory_reservations(order_id);

-- Seed some inventory
INSERT INTO inventory (product_id, product_name, sku, quantity_available) VALUES
  (1, 'Widget Pro', 'WGT-PRO-001', 100),
  (2, 'Gadget Plus', 'GDG-PLS-002', 50),
  (3, 'Tool Master', 'TL-MST-003', 75);
```

Run setup:
```bash
docker exec -i n8n-postgres psql -U n8n -d n8n < setup-database.sql
```

2. **Configure Credentials**

In n8n, create credentials for:
- PostgreSQL (your database)
- Stripe or PayPal (payment processing)
- SendGrid or SMTP (email)
- Shipping API (EasyPost, Shippo, or ShipStation)

3. **Create Order Number Generator**

```javascript
// Reusable function for generating order numbers
function generateOrderNumber() {
  const date = new Date();
  const year = date.getFullYear().toString().slice(-2);
  const month = (date.getMonth() + 1).toString().padStart(2, '0');
  const day = date.getDate().toString().padStart(2, '0');
  const random = Math.floor(Math.random() * 10000).toString().padStart(4, '0');

  return `ORD-${year}${month}${day}-${random}`;
}
```

#### Phase 2: Order Intake (3 hours)

**Workflow 1: Order Intake**

```
Webhook: POST /orders/new
  Headers: X-API-Key (for authentication)
  Body: {
    customerId: 123,
    customerEmail: "customer@example.com",
    items: [
      {productId: 1, quantity: 2},
      {productId: 2, quantity: 1}
    ],
    shippingAddress: {...},
    paymentMethod: {...}
  }
    ↓
Function: Validate Request
  - Check required fields
  - Validate item quantities > 0
  - Validate email format
  - Calculate total
    ↓
Function: Generate Order Number
  orderNumber = generateOrderNumber()
    ↓
PostgreSQL: Get Product Details
  SELECT * FROM inventory
  WHERE product_id IN ({{productIds}})
    ↓
Function: Calculate Order Total
  For each item:
    Get current price from inventory
    Calculate line total
  Sum all line totals
    ↓
PostgreSQL: Create Order
  INSERT INTO orders
    (order_number, customer_id, customer_email,
     status, total_amount, metadata)
  VALUES (?, ?, ?, 'pending', ?, ?)
  RETURNING id
    ↓
PostgreSQL: Create Order Items
  INSERT INTO order_items
    (order_id, product_id, product_name, quantity, unit_price, total_price)
  VALUES ...
    ↓
Execute Workflow: Inventory Check
  Pass: orderId
    ↓
Webhook Response:
  Status: 202 Accepted
  Body: {
    orderNumber, orderId,
    status: "processing",
    message: "Order received"
  }
```

Error handling:
```javascript
try {
  // Main workflow logic
} catch (error) {
  // Log error
  await logError({
    workflow: 'order-intake',
    error: error.message,
    data: $json
  });

  // Return error response
  return {
    json: {
      error: true,
      message: error.message,
      code: error.code || 'ORDER_CREATION_FAILED'
    },
    statusCode: 400
  };
}
```

#### Phase 3: Inventory Management (2 hours)

**Workflow 2: Inventory Check & Reservation**

```
Workflow Trigger: Called by Order Intake
  Input: orderId
    ↓
PostgreSQL: Get Order Items
  SELECT * FROM order_items
  WHERE order_id = {{orderId}}
    ↓
PostgreSQL: Check Inventory
  SELECT product_id, quantity_available
  FROM inventory
  WHERE product_id IN ({{productIds}})
  FOR UPDATE  -- Lock rows
    ↓
Function: Validate Availability
```

```javascript
const orderItems = items[0].json;
const inventory = items[1].json;

const availabilityChecks = orderItems.map(item => {
  const inv = inventory.find(i => i.product_id === item.product_id);

  return {
    productId: item.product_id,
    productName: item.product_name,
    requested: item.quantity,
    available: inv ? inv.quantity_available : 0,
    sufficient: inv && inv.quantity_available >= item.quantity
  };
});

const allAvailable = availabilityChecks.every(c => c.sufficient);

return {
  json: {
    orderId: $json.orderId,
    allAvailable,
    checks: availabilityChecks,
    insufficientItems: availabilityChecks.filter(c => !c.sufficient)
  }
};
```

```
IF (All items available):
    ↓
    PostgreSQL: Reserve Inventory
      BEGIN;

      -- Update inventory
      UPDATE inventory
      SET quantity_reserved = quantity_reserved + {{quantity}},
          quantity_available = quantity_available - {{quantity}}
      WHERE product_id = {{productId}};

      -- Create reservation record
      INSERT INTO inventory_reservations
        (order_id, product_id, quantity, expires_at)
      VALUES ({{orderId}}, {{productId}}, {{quantity}},
              NOW() + INTERVAL '1 hour');

      COMMIT;
    ↓
    PostgreSQL: Update Order Status
      UPDATE orders
      SET status = 'inventory_reserved'
      WHERE id = {{orderId}}
    ↓
    Execute Workflow: Process Payment
ELSE:
    ↓
    PostgreSQL: Update Order Status
      UPDATE orders
      SET status = 'insufficient_inventory'
      WHERE id = {{orderId}}
    ↓
    Email: Notify Customer
      Template: out_of_stock
    ↓
    Slack: Alert operations team
```

Reservation expiry cleanup:
```
Schedule: Every 15 minutes
    ↓
PostgreSQL: Find Expired Reservations
  SELECT * FROM inventory_reservations
  WHERE expires_at < NOW()
    AND released = FALSE
    ↓
For Each Reservation:
    PostgreSQL: Release Inventory
      UPDATE inventory
      SET quantity_reserved = quantity_reserved - {{qty}},
          quantity_available = quantity_available + {{qty}}
      WHERE product_id = {{productId}}
    ↓
    PostgreSQL: Mark Released
      UPDATE inventory_reservations
      SET released = TRUE
```

#### Phase 4: Payment Processing (2 hours)

**Workflow 3: Payment Processing**

```
Workflow Trigger: Called after inventory reserved
  Input: orderId
    ↓
PostgreSQL: Get Order Details
    ↓
Function: Prepare Payment Request
```

```javascript
const order = $json;

return {
  json: {
    amount: Math.round(order.total_amount * 100), // cents
    currency: order.currency || 'usd',
    customer: order.customer_id,
    metadata: {
      order_id: order.id,
      order_number: order.order_number
    },
    description: `Order ${order.order_number}`
  }
};
```

```
HTTP Request: Stripe Create Payment Intent
  URL: https://api.stripe.com/v1/payment_intents
  Method: POST
  Auth: Bearer {{$credentials.stripe}}
  Body: {{prepared payment}}
    ↓
Function: Process Payment Response
```

```javascript
const paymentIntent = $json;

if (paymentIntent.status === 'succeeded') {
  return {
    json: {
      success: true,
      orderId: paymentIntent.metadata.order_id,
      paymentId: paymentIntent.id,
      amount: paymentIntent.amount / 100
    }
  };
} else if (['requires_payment_method', 'requires_action'].includes(paymentIntent.status)) {
  return {
    json: {
      success: false,
      requiresAction: true,
      orderId: paymentIntent.metadata.order_id,
      clientSecret: paymentIntent.client_secret
    }
  };
} else {
  throw new Error(`Payment failed: ${paymentIntent.status}`);
}
```

```
IF (Payment Succeeded):
    ↓
    PostgreSQL: Update Order
      UPDATE orders
      SET status = 'paid',
          metadata = jsonb_set(metadata, '{payment}', {{paymentData}})
      WHERE id = {{orderId}}
    ↓
    PostgreSQL: Finalize Inventory
      UPDATE inventory_reservations
      SET released = TRUE
      WHERE order_id = {{orderId}}
      -- Inventory already adjusted, just mark reservation complete
    ↓
    Execute Workflow: Create Shipment
ELSE:
    ↓
    Execute Workflow: Release Inventory
    ↓
    PostgreSQL: Update Order
      UPDATE orders SET status = 'payment_failed'
    ↓
    Email: Payment Failed Notification
```

#### Phase 5: Fulfillment & Shipping (3 hours)

**Workflow 4: Create Shipment**

```
Workflow Trigger: After payment success
  Input: orderId
    ↓
PostgreSQL: Get Order & Items
  SELECT o.*, array_agg(oi.*) as items
  FROM orders o
  JOIN order_items oi ON o.id = oi.order_id
  WHERE o.id = {{orderId}}
  GROUP BY o.id
    ↓
Function: Prepare Shipping Request
```

```javascript
const order = $json;
const shippingAddress = order.metadata.shippingAddress;

return {
  json: {
    to_address: {
      name: shippingAddress.name,
      street1: shippingAddress.street1,
      street2: shippingAddress.street2 || '',
      city: shippingAddress.city,
      state: shippingAddress.state,
      zip: shippingAddress.zip,
      country: shippingAddress.country || 'US',
      phone: shippingAddress.phone
    },
    from_address: {
      company: 'Your Company',
      street1: '123 Warehouse St',
      city: 'Commerce City',
      state: 'CA',
      zip: '90001',
      country: 'US',
      phone: '555-123-4567'
    },
    parcel: {
      length: 12,
      width: 8,
      height: 6,
      weight: calculateTotalWeight(order.items) || 16
    }
  }
};
```

```
HTTP Request: Create Shipping Label
  API: EasyPost, Shippo, or ShipStation
  Method: POST
  Body: {{shipping request}}
    ↓
Function: Parse Shipping Response
    ↓
PostgreSQL: Create Shipment Record
  INSERT INTO shipments
    (order_id, tracking_number, carrier, label_url, estimated_delivery, status)
  VALUES (?, ?, ?, ?, ?, 'label_created')
    ↓
PostgreSQL: Update Order Status
  UPDATE orders SET status = 'shipped'
    ↓
Execute Workflow: Send Notifications
```

**Workflow 5: Order Notifications**

```
Workflow Trigger: Various order events
  Input: orderId, eventType
    ↓
PostgreSQL: Get Order Details
    ↓
Switch: Route by Event Type

  Case 'order_confirmed':
    → Email: Order Confirmation
       Subject: Order Confirmed - {{orderNumber}}
       Template: order_confirmed.html

  Case 'shipped':
    → Email: Shipping Confirmation
       Subject: Your order has shipped!
       Template: shipped.html
       Include: Tracking link

  Case 'delivered':
    → Email: Delivery Confirmation
       Template: delivered.html
    → Schedule: Follow-up survey (3 days)

  Case 'payment_failed':
    → Email: Payment Issue
       Template: payment_failed.html

  Case 'refunded':
    → Email: Refund Confirmation
       Template: refund_confirmed.html
```

Email template example (`shipped.html`):

```html
<!DOCTYPE html>
<html>
<head>
  <style>
    body { font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; }
    .header { background: #4CAF50; color: white; padding: 20px; }
    .content { padding: 20px; }
    .tracking { background: #f5f5f5; padding: 15px; margin: 20px 0; border-radius: 5px; }
    .button { background: #4CAF50; color: white; padding: 12px 30px; text-decoration: none; border-radius: 5px; display: inline-block; }
  </style>
</head>
<body>
  <div class="header">
    <h1>Your Order Has Shipped!</h1>
  </div>
  <div class="content">
    <p>Hi {{customerName}},</p>

    <p>Great news! Your order #{{orderNumber}} is on its way!</p>

    <div class="tracking">
      <strong>Tracking Number:</strong> {{trackingNumber}}<br>
      <strong>Carrier:</strong> {{carrier}}<br>
      <strong>Estimated Delivery:</strong> {{estimatedDelivery}}
    </div>

    <p>
      <a href="{{trackingUrl}}" class="button">Track Your Package</a>
    </p>

    <p>Questions? Contact us at support@example.com</p>
  </div>
</body>
</html>
```

#### Phase 6: Monitoring & Dashboard (2 hours)

**Workflow 6: Order Metrics Dashboard**

```
Schedule: Every 15 minutes
    ↓
PostgreSQL: Get Metrics
  SELECT
    COUNT(*) FILTER (WHERE status = 'pending') as pending_orders,
    COUNT(*) FILTER (WHERE status = 'paid') as paid_orders,
    COUNT(*) FILTER (WHERE status = 'shipped') as shipped_orders,
    COUNT(*) FILTER (WHERE status = 'payment_failed') as failed_payments,
    SUM(total_amount) FILTER (WHERE status = 'paid') as revenue_today,
    AVG(EXTRACT(EPOCH FROM (updated_at - created_at))/60)
      FILTER (WHERE status = 'shipped') as avg_fulfillment_time_minutes
  FROM orders
  WHERE created_at >= CURRENT_DATE
    ↓
Google Sheets: Update Dashboard
  Sheet: Order Metrics
  Range: A2:F2 (overwrite latest)
    ↓
IF (pending_orders > 50):
    Slack: Alert ops team
    "High volume of pending orders: {{count}}"
    ↓
IF (failed_payments > 10):
    Slack: Alert finance team
    "Unusual payment failure rate"
```

**Real-time Order Monitor**:

```
Schedule: Every minute
    ↓
PostgreSQL: Check for Issues
  -- Orders stuck in inventory_reserved > 30 min
  SELECT * FROM orders
  WHERE status = 'inventory_reserved'
    AND updated_at < NOW() - INTERVAL '30 minutes'

  -- Shipments without tracking > 2 hours
  SELECT s.* FROM shipments s
  JOIN orders o ON s.order_id = o.id
  WHERE s.tracking_number IS NULL
    AND o.status = 'paid'
    AND o.updated_at < NOW() - INTERVAL '2 hours'
    ↓
For Each Issue:
    Create alert ticket
    Notify appropriate team
    Log to monitoring dashboard
```

#### Phase 7: Testing (2 hours)

**Test Cases**:

1. **Happy Path**
   ```bash
   # Submit test order
   curl -X POST http://localhost:5678/webhook/orders/new \
     -H "Content-Type: application/json" \
     -d '{
       "customerId": 123,
       "customerEmail": "test@example.com",
       "items": [{"productId": 1, "quantity": 2}],
       "shippingAddress": {...},
       "paymentMethod": {...}
     }'

   # Verify:
   # - Order created in database
   # - Inventory reserved
   # - Payment processed
   # - Shipping label created
   # - Customer received emails
   ```

2. **Insufficient Inventory**
   ```bash
   # Order quantity > available
   # Verify:
   # - Order marked insufficient_inventory
   # - Customer notified
   # - Ops team alerted
   ```

3. **Payment Failure**
   ```bash
   # Use test card that fails
   # Verify:
   # - Inventory released
   # - Order marked payment_failed
   # - Customer notified
   ```

4. **Concurrent Orders**
   ```bash
   # Submit 10 orders simultaneously
   # Verify:
   # - No overselling
   # - All processed correctly
   # - Inventory accurate
   ```

### Deliverables Checklist

- [ ] All 6 workflows created and tested
- [ ] Database schema implemented
- [ ] Payment integration working
- [ ] Shipping integration working
- [ ] Email notifications sending
- [ ] Dashboard updating
- [ ] Error handling comprehensive
- [ ] Test cases pass
- [ ] Documentation complete
- [ ] Code commented

---

## Project 2: Employee Onboarding Automation

### Objective

Create a complete employee onboarding system that automates the entire process from offer acceptance through the first 30 days, including account provisioning, task tracking, and engagement.

### Requirements

**Core Features:**
- Multi-stage onboarding workflow
- Automated account creation (email, Slack, tools)
- Document collection and tracking
- Task assignment and reminders
- Manager/buddy notifications
- Progress dashboard
- 30-day survey

**Integrations:**
- HRIS (BambooHR, Workday, or Airtable)
- Google Workspace (email, calendar)
- Slack
- Project management (Asana, Monday, or Notion)
- DocuSign or similar (document signing)

**Deliverables:**
- Complete onboarding workflow system
- Integration with HR tools
- Email templates
- Task templates
- Dashboard
- Documentation

### Implementation

(Follow similar detailed structure as Project 1, with workflows for: Pre-boarding, Account Setup, First Day, Weekly Check-ins, Survey Collection)

---

## Project 3: Financial Reconciliation Workflow

### Objective

Build an automated financial reconciliation system that matches transactions across multiple sources, identifies discrepancies, and routes exceptions for approval.

### Requirements

**Core Features:**
- Multi-source data import (bank, payment processors, accounting)
- Automated transaction matching
- Discrepancy detection and classification
- Exception handling workflow
- Approval process for adjustments
- Automated reporting
- Audit trail

**Data Sources:**
- Bank statements (CSV import or API)
- Payment processors (Stripe, PayPal APIs)
- Accounting system (QuickBooks, Xero APIs)

**Deliverables:**
- Reconciliation workflows
- Matching algorithm
- Exception handling system
- Approval workflow
- Reports and dashboards
- Documentation

### Implementation

(Follow similar detailed structure as Project 1, with workflows for: Data Import, Transaction Matching, Discrepancy Detection, Approval Process, Reporting)

---

## Submission Guidelines

### What to Submit

For each project:

1. **Workflows**
   - Export all n8n workflows (JSON)
   - Include screenshots of key workflows

2. **Database**
   - SQL schema scripts
   - Seed data (if applicable)
   - ER diagram

3. **Documentation**
   - Architecture overview
   - Setup instructions
   - API documentation
   - User guide

4. **Testing**
   - Test cases and results
   - Performance metrics
   - Error scenarios handled

5. **Code**
   - Custom functions
   - Scripts
   - Templates

### Evaluation Criteria

- **Functionality** (40%): Does it work as specified?
- **Code Quality** (20%): Clean, maintainable, well-documented
- **Error Handling** (15%): Comprehensive, graceful failures
- **Integration** (15%): Proper use of external systems
- **Documentation** (10%): Clear and complete

## Bonus Challenges

If you complete all three projects:

1. **Add Internationalization**
   - Multi-language support
   - Currency conversion
   - Timezone handling

2. **Build Admin Interface**
   - Order management UI
   - Onboarding dashboard
   - Reconciliation review interface

3. **Add Analytics**
   - Business intelligence reports
   - Trend analysis
   - Predictive insights

## Resources

- n8n Community workflows
- API documentation for integrated services
- Sample datasets (if needed)
- Course Slack channel for questions

## Tips for Success

1. **Start Small**: Build core functionality first
2. **Test Incrementally**: Test each workflow as you build
3. **Document as You Go**: Don't wait until the end
4. **Handle Errors Early**: Build error handling from the start
5. **Ask for Help**: Use the community when stuck

## Conclusion

These projects represent real-world business automation scenarios. Take your time, build quality over quantity, and create solutions you'd be proud to deploy in production!

**Remember**: The goal is not just to make it work, but to build maintainable, scalable, production-ready automation systems.

Good luck!

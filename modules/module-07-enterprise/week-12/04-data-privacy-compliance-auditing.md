# Data Privacy, Compliance, and Security Auditing

## Table of Contents

1. [Data Privacy and Logging](#data-privacy-and-logging)
2. [Compliance Patterns (GDPR, HIPAA)](#compliance-patterns)
3. [Security Audit Procedures](#security-audit-procedures)

---

# Data Privacy and Logging

## Introduction

Handling sensitive data properly is critical for security, privacy, and compliance. This section covers identifying sensitive data, implementing proper logging practices, and protecting privacy in workflows.

## Identifying Sensitive Data

### Personal Identifiable Information (PII)

Data that can identify an individual:

- **Direct identifiers**:
  - Full name
  - Social Security Number (SSN)
  - Driver's license number
  - Passport number
  - Email address (in many contexts)
  - Phone number

- **Quasi-identifiers** (combined can identify):
  - Date of birth
  - ZIP code
  - Gender
  - Job title
  - IP address

### Protected Health Information (PHI)

Under HIPAA:
- Medical records
- Health insurance information
- Prescriptions
- Lab results
- Diagnoses
- Treatment information

### Financial Data

Under PCI DSS:
- Credit card numbers (PAN)
- CVV codes
- Bank account numbers
- Transaction data
- PIN codes

### Business Sensitive

- Trade secrets
- Customer lists
- Pricing information
- Strategic plans
- Employee data

## Data Minimization

### Principles

1. **Collect only what's needed**
   ```javascript
   // ❌ Bad - collecting unnecessary data
   const userData = {
     ssn: user.ssn,
     creditCard: user.creditCard,
     favoriteColor: user.favoriteColor
   };

   // ✅ Good - only collecting necessary data
   const userData = {
     userId: user.id,  // Use ID instead of PII
     email: user.email  // Only if genuinely needed
   };
   ```

2. **Keep only as long as necessary**
   ```javascript
   // Set retention policy
   const RETENTION_DAYS = 90;

   // Auto-delete old data
   DELETE FROM user_data
   WHERE created_at < NOW() - INTERVAL '90 days';
   ```

3. **Delete when no longer needed**
   ```javascript
   // After processing
   await processOrder(orderData);
   await deleteCustomerPII(orderId);  // Remove PII, keep order ID
   ```

## Secure Logging Practices

### What NOT to Log

```javascript
// ❌ NEVER log these:
console.log("SSN:", user.ssn);
console.log("Password:", password);
console.log("Credit Card:", creditCard);
console.log("API Key:", apiKey);
console.log("Full user object:", user);  // May contain PII
```

### What to Log

```javascript
// ✅ Safe to log:
console.log("User ID:", userId);
console.log("Action:", "login_successful");
console.log("Timestamp:", new Date());
console.log("IP (hashed):", hashIP(ip Address));
console.log("Request ID:", requestId);
```

### Data Masking

```javascript
// Function: Mask sensitive data
function maskData(data, type) {
  switch(type) {
    case 'ssn':
      // XXX-XX-1234
      return 'XXX-XX-' + data.slice(-4);

    case 'email':
      // u***@example.com
      const [local, domain] = data.split('@');
      return local[0] + '***@' + domain;

    case 'creditCard':
      // ****-****-****-1234
      return '****-****-****-' + data.slice(-4);

    case 'phone':
      // (XXX) XXX-1234
      return '(XXX) XXX-' + data.slice(-4);

    default:
      return '***REDACTED***';
  }
}

// Usage
console.log("SSN:", maskData(user.ssn, 'ssn'));
console.log("Email:", maskData(user.email, 'email'));
```

### Structured Logging

```javascript
// n8n Function Node: Secure logging

const logEvent = {
  timestamp: new Date().toISOString(),
  eventType: 'user_action',
  action: 'profile_update',
  userId: $json.userId,  // Use ID, not email
  success: true,
  ip: hashIP($json.ipAddress),  // Hash IP
  // Don't include: name, email, phone, etc.
};

// Send to logging service
return { json: logEvent };
```

## PII Handling Workflows

### Example: User Registration with PII Protection

```
Webhook: New User Registration
    ↓
Function: Validate input
    ↓
Function: Separate PII from non-PII
  PII → Encrypted storage
  Non-PII → Regular database
    ↓
Database: Store encrypted PII
    ↓
Database: Store non-PII with reference
    ↓
Function: Create masked log entry
  userId, action, timestamp (no PII)
    ↓
Log: Write to audit log
    ↓
Response: Success (no PII in response)
```

### PII Encryption Workflow

```javascript
// n8n Function Node: Encrypt PII

const crypto = require('crypto');

const ENCRYPTION_KEY = Buffer.from(process.env.PII_ENCRYPTION_KEY, 'hex');
const IV_LENGTH = 16;

function encryptPII(text) {
  const iv = crypto.randomBytes(IV_LENGTH);
  const cipher = crypto.createCipheriv('aes-256-cbc', ENCRYPTION_KEY, iv);
  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  return iv.toString('hex') + ':' + encrypted;
}

// Encrypt sensitive fields
const encryptedData = {
  userId: $json.userId,  // Not encrypted - needed for queries
  ssn_encrypted: encryptPII($json.ssn),
  email_encrypted: encryptPII($json.email),
  phone_encrypted: encryptPII($json.phone),
  created_at: new Date()
};

return { json: encryptedData };
```

## Log Retention Policies

### Retention Matrix

| Data Type | Retention Period | Reason |
|-----------|------------------|--------|
| Access logs | 90 days | Security monitoring |
| Audit logs | 7 years | Compliance (varies) |
| Error logs | 30 days | Troubleshooting |
| PII data | As needed + 30 days | Business need |
| Financial records | 7 years | Legal requirement |

### Automated Cleanup

```sql
-- PostgreSQL: Auto-delete old logs

CREATE OR REPLACE FUNCTION cleanup_old_logs()
RETURNS void AS $$
BEGIN
  -- Delete old access logs (> 90 days)
  DELETE FROM access_logs
  WHERE created_at < NOW() - INTERVAL '90 days';

  -- Delete old error logs (> 30 days)
  DELETE FROM error_logs
  WHERE created_at < NOW() - INTERVAL '30 days';

  -- Anonymize old PII (> retention period)
  UPDATE user_data
  SET
    email = 'deleted_' || id || '@example.com',
    phone = NULL,
    ssn = NULL
  WHERE
    deleted_at < NOW() - INTERVAL '30 days'
    AND email NOT LIKE 'deleted_%';

END;
$$ LANGUAGE plpgsql;

-- Schedule daily
SELECT cron.schedule('cleanup-logs', '0 2 * * *', 'SELECT cleanup_old_logs()');
```

---

# Compliance Patterns

## GDPR (General Data Protection Regulation)

### Key Requirements

#### 1. Lawful Basis for Processing

You must have one of these:
- **Consent**: User explicitly agreed
- **Contract**: Necessary to fulfill contract
- **Legal obligation**: Required by law
- **Vital interests**: Protect someone's life
- **Public task**: Performing official function
- **Legitimate interests**: Your business need (balanced against user rights)

#### 2. User Rights

Must support:
- **Right to access**: Users can request their data
- **Right to rectification**: Users can correct data
- **Right to erasure** ("right to be forgotten"): Users can request deletion
- **Right to portability**: Export data in machine-readable format
- **Right to restrict processing**: Temporarily stop processing
- **Right to object**: Opt-out of processing

#### 3. Privacy by Design

- Data protection built into systems from the start
- Privacy as default setting
- Data minimization
- Pseudonymization where possible

### GDPR-Compliant Workflows

#### Data Subject Access Request (DSAR)

```
Webhook: DSAR Request
  Input: User email
    ↓
Validate: Verify user identity
    ↓
Database: Query all user data
  - Profile data
  - Activity logs
  - Preferences
  - Purchase history
    ↓
Function: Compile data export
  Format: JSON
    ↓
Function: Remove internal IDs
  Only include user-relevant data
    ↓
Email: Send data export to user
  Encrypted attachment
    ↓
Log: Record DSAR fulfilled
  Timestamp, user ID
```

#### Right to Erasure (Deletion)

```
Webhook: Deletion Request
    ↓
Validate: Verify user identity
    ↓
Check: Legal obligations to retain?
  (e.g., financial records)
    ↓
IF can delete:
    ↓
    Database: Delete from users table
    Database: Delete from activity logs
    Database: Anonymize transaction history
      (keep financial data, remove PII)
    Database: Delete from email lists
    API: Remove from external services
    ↓
    Email: Confirm deletion
    ↓
    Log: Record deletion
      User ID, timestamp, reason
ELSE:
    ↓
    Email: Explain why can't delete yet
    Schedule: Auto-delete after retention period
```

### GDPR Compliance Checklist

- [ ] Privacy policy published and accessible
- [ ] Consent mechanism implemented
- [ ] DSAR process documented and tested
- [ ] Data deletion process implemented
- [ ] Data export functionality working
- [ ] Data processing inventory maintained
- [ ] DPO (Data Protection Officer) appointed (if required)
- [ ] Breach notification procedure ready (<72 hours)
- [ ] Data Processing Agreements with vendors
- [ ] Regular privacy impact assessments

## HIPAA (Health Insurance Portability and Accountability Act)

### Key Requirements

#### 1. Administrative Safeguards

- **Security management**: Risk analysis and management
- **Workforce security**: Authorization and supervision
- **Information access**: Limit access to minimum necessary
- **Security awareness**: Training and incident response
- **Business Associate Agreements**: Contracts with vendors

#### 2. Physical Safeguards

- **Facility access**: Controls for physical access
- **Workstation use**: Policies for device use
- **Device controls**: Hardware inventory and disposal

#### 3. Technical Safeguards

- **Access control**: Unique user IDs, emergency access
- **Audit controls**: Log access to PHI
- **Integrity**: Ensure data isn't altered
- **Transmission security**: Encrypt PHI in transit

### HIPAA-Compliant Workflows

#### Secure PHI Processing

```
API Trigger: New medical record
    ↓
Validate: Check authorization
  - Valid user credentials
  - Appropriate role
  - Business need (minimum necessary)
    ↓
Log: Record access attempt
  Who, what, when, why
    ↓
Encrypt: PHI data
  AES-256 encryption
    ↓
Database: Store encrypted PHI
  Separate from non-PHI data
    ↓
Audit: Log successful storage
    ↓
IF (sharing with external party):
    ↓
    Verify: BAA in place
    Encrypt: For transmission (TLS 1.2+)
    Log: Disclosure
```

#### Audit Logging

```javascript
// n8n Function Node: HIPAA Audit Log

const auditLog = {
  timestamp: new Date().toISOString(),
  user_id: $json.userId,
  user_role: $json.userRole,
  action: $json.action,  // view, create, update, delete
  resource_type: 'patient_record',
  resource_id: $json.patientId,
  ip_address: $json.ipAddress,
  success: true,
  reason: $json.businessJustification  // Why accessing
};

// Store in tamper-proof audit log
return { json: auditLog };
```

### HIPAA Compliance Checklist

- [ ] Risk assessment completed
- [ ] Policies and procedures documented
- [ ] Workforce training completed
- [ ] BAAs with all vendors
- [ ] Encryption at rest implemented (PHI)
- [ ] Encryption in transit (TLS 1.2+)
- [ ] Access controls implemented
- [ ] Audit logging enabled
- [ ] Audit logs reviewed regularly
- [ ] Incident response plan documented
- [ ] Breach notification procedure (<60 days)
- [ ] Physical security controls
- [ ] Device inventory maintained
- [ ] Secure disposal procedures

## SOC 2 Compliance

### Trust Service Criteria

#### 1. Security

- Protect against unauthorized access
- Logical and physical access controls
- System monitoring

#### 2. Availability

- System uptime and performance
- Incident management
- Change management

#### 3. Processing Integrity

- Data processing is complete, accurate, timely
- Error handling and monitoring

#### 4. Confidentiality

- Confidential information protected
- Encryption and access controls

#### 5. Privacy

- Personal information collected, used, disclosed per privacy notice
- User consent and choice

### SOC 2 Controls in n8n

```javascript
// Control: Access Control
// Document who can access what

const accessControl = {
  workflow: "Customer Data Processing",
  owner: "data-team@company.com",
  approvedUsers: [
    { email: "analyst1@company.com", role: "viewer" },
    { email: "engineer1@company.com", role: "editor" }
  ],
  lastReviewed: "2024-01-15",
  nextReview: "2024-04-15"
};
```

---

# Security Audit Procedures

## Comprehensive Security Audit

### 1. Workflow Security Review

#### Checklist

```markdown
## Workflow: [Name]

### Sensitive Data Handling
- [ ] No PII in logs
- [ ] No credentials in code
- [ ] Sensitive data encrypted
- [ ] Data minimization applied
- [ ] Retention policy enforced

### Access Control
- [ ] Appropriate sharing settings
- [ ] Least privilege applied
- [ ] Owner documented
- [ ] Access reviewed quarterly

### Error Handling
- [ ] Errors don't expose sensitive data
- [ ] Error notifications configured
- [ ] Retry logic appropriate
- [ ] Timeout settings reasonable

### External Integrations
- [ ] Credentials rotated
- [ ] APIs use current versions
- [ ] TLS/HTTPS enforced
- [ ] Rate limiting considered

### Documentation
- [ ] Purpose documented
- [ ] Dependencies listed
- [ ] Owner assigned
- [ ] Last reviewed date
```

#### Automated Workflow Scan

```javascript
// n8n Function Node: Scan workflow for issues

const workflow = $json.workflow;
const issues = [];

// Check for hardcoded secrets
const secretPatterns = [
  /sk_live_\w+/,  // Stripe
  /AKIA\w{16}/,  // AWS
  /(password|pwd|secret|key)\s*=\s*['"][^'"]+['"]/i
];

for (const pattern of secretPatterns) {
  if (pattern.test(JSON.stringify(workflow))) {
    issues.push({
      severity: 'CRITICAL',
      type: 'hardcoded_secret',
      message: 'Potential hardcoded secret detected'
    });
  }
}

// Check for PII logging
const piiPatterns = [
  /console\.log.*ssn/i,
  /console\.log.*password/i,
  /console\.log.*email.*@/i
];

// Check for insecure protocols
if (workflow.includes('http://') && !workflow.includes('localhost')) {
  issues.push({
    severity: 'HIGH',
    type: 'insecure_protocol',
    message: 'HTTP used instead of HTTPS'
  });
}

return { json: { workflow: workflow.name, issues } };
```

### 2. Credential Audit

```sql
-- Audit all credentials

SELECT
  c.id,
  c.name,
  c.type,
  u.email as owner,
  c.created_at,
  c.updated_at,
  COUNT(DISTINCT wc.workflow_id) as workflow_count,
  COUNT(DISTINCT cs.user_id) as shared_with_count,
  MAX(e.finished_at) as last_used
FROM credentials_entity c
JOIN user u ON c.user_id = u.id
LEFT JOIN workflow_credential wc ON c.id = wc.credential_id
LEFT JOIN credentials_sharing cs ON c.id = cs.credential_id
LEFT JOIN execution_entity e ON wc.workflow_id = e.workflow_id
GROUP BY c.id, u.email
ORDER BY last_used DESC NULLS LAST;

-- Find unused credentials (> 90 days)
SELECT name, type, last_used
FROM credentials_usage_view
WHERE last_used < NOW() - INTERVAL '90 days'
   OR last_used IS NULL;

-- Find overshared credentials
SELECT name, type, shared_with_count
FROM credentials_usage_view
WHERE shared_with_count > 5;
```

### 3. Access Control Review

```sql
-- User access audit

SELECT
  u.email,
  u.role,
  u.created_at,
  u.last_login,
  COUNT(DISTINCT w.id) as owned_workflows,
  COUNT(DISTINCT c.id) as owned_credentials
FROM user u
LEFT JOIN workflow_entity w ON u.id = w.user_id
LEFT JOIN credentials_entity c ON u.id = c.user_id
GROUP BY u.id
ORDER BY u.last_login DESC NULLS LAST;

-- Find inactive users
SELECT email, role, last_login
FROM user
WHERE last_login < NOW() - INTERVAL '90 days'
   OR last_login IS NULL;
```

### 4. Vulnerability Scanning

```bash
#!/bin/bash
# security-scan.sh

echo "Running security scans..."

# 1. Container scanning
echo "Scanning Docker images..."
docker scan n8nio/n8n:latest

# 2. Dependency scanning
echo "Checking for vulnerable dependencies..."
npm audit

# 3. SSL/TLS check
echo "Checking SSL configuration..."
sslscan n8n.example.com

# 4. Port scan
echo "Scanning open ports..."
nmap -sV localhost

# 5. Database security
echo "Checking database..."
# Check for default passwords, encryption, etc.

echo "Scan complete!"
```

### 5. Penetration Testing

#### Common Tests

1. **Authentication bypass**
   - Try accessing without credentials
   - Test for default credentials
   - Check session management

2. **Authorization bypass**
   - Try accessing other users' workflows
   - Test horizontal privilege escalation
   - Test vertical privilege escalation

3. **Injection attacks**
   - SQL injection in custom queries
   - Command injection in Execute Command nodes
   - Script injection in Function nodes

4. **XSS (Cross-Site Scripting)**
   - Test input sanitization
   - Check for reflected XSS
   - Check for stored XSS

5. **CSRF (Cross-Site Request Forgery)**
   - Check for CSRF tokens
   - Test state parameter in OAuth

### 6. Security Reporting

#### Audit Report Template

```markdown
# n8n Security Audit Report

**Date**: 2024-01-15
**Auditor**: Security Team
**Scope**: Production n8n instance

## Executive Summary

- Total workflows audited: 127
- Critical issues: 3
- High issues: 8
- Medium issues: 15
- Low issues: 23

## Critical Issues

### 1. Hardcoded API Keys in Workflow
- **Workflow**: Customer Data Sync
- **Issue**: Stripe API key hardcoded in Function node
- **Impact**: Credential exposure risk
- **Remediation**: Move to credential store
- **Deadline**: Immediate
- **Owner**: Data Team

## Recommendations

1. Implement automated credential rotation
2. Enable MFA for all admin users
3. Conduct security training
4. Update incident response plan
5. Schedule quarterly audits

## Compliance Status

- GDPR: Compliant ✓
- HIPAA: Needs attention (3 findings)
- SOC 2: In progress

## Next Steps

1. Remediate critical issues (Week 1)
2. Address high priority issues (Week 2-3)
3. Plan for medium/low issues (Month 2)
4. Re-audit (Month 3)
```

## Quick Security Checklist

### Daily
- [ ] Review failed login attempts
- [ ] Check error logs for anomalies
- [ ] Monitor resource usage

### Weekly
- [ ] Review new workflows
- [ ] Check credential usage
- [ ] Review audit logs

### Monthly
- [ ] Rotate critical credentials
- [ ] Review user access
- [ ] Update dependencies
- [ ] Check for security updates

### Quarterly
- [ ] Full security audit
- [ ] Access control review
- [ ] Credential audit
- [ ] Vulnerability scan
- [ ] Update documentation

### Annually
- [ ] Penetration test
- [ ] Disaster recovery test
- [ ] Compliance audit
- [ ] Security training
- [ ] Review policies

## Next Steps

1. Complete hands-on projects (see `05-hands-on-projects.md`)
2. Implement security controls
3. Document procedures
4. Train team
5. Schedule regular audits

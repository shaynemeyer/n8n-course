# Week 12 Hands-On Projects

## Overview

These hands-on projects will help you apply security and compliance concepts learned this week. Complete all three projects to demonstrate mastery of security best practices for production n8n deployments.

---

## Project 1: Security Audit of Existing Workflows

### Objective

Conduct a comprehensive security audit of your n8n deployment, identify vulnerabilities, and implement fixes.

### Requirements

**Audit Scope:**
- All active workflows
- All credentials
- User access controls
- Logging practices
- Compliance with best practices

**Deliverables:**
- Security audit report
- Prioritized remediation list
- Evidence of fixes implemented
- Updated security documentation

### Step-by-Step Guide

#### Phase 1: Preparation (30 minutes)

1. **Create Audit Checklist**

   ```markdown
   # n8n Security Audit Checklist

   ## Workflow Security
   - [ ] No hardcoded credentials
   - [ ] No PII in logs
   - [ ] Proper error handling
   - [ ] HTTPS for external calls
   - [ ] Appropriate timeouts

   ## Credential Security
   - [ ] No unused credentials
   - [ ] Credentials properly named
   - [ ] Appropriate sharing settings
   - [ ] Rotation schedule documented
   - [ ] Service accounts used

   ## Access Control
   - [ ] Users have appropriate roles
   - [ ] No inactive users
   - [ ] Workflows properly shared
   - [ ] Regular access reviews

   ## Data Privacy
   - [ ] PII properly handled
   - [ ] Data minimization applied
   - [ ] Retention policies implemented
   - [ ] Secure logging practices

   ## Infrastructure
   - [ ] Latest n8n version
   - [ ] Dependencies updated
   - [ ] SSL/TLS configured
   - [ ] Backups working
   - [ ] Monitoring enabled
   ```

2. **Set Up Audit Workspace**

   ```bash
   mkdir ~/n8n-security-audit
   cd ~/n8n-security-audit

   # Create directories
   mkdir -p {findings,evidence,reports}

   # Create template
   cat > audit-template.md << 'EOF'
   # Security Finding

   **ID**: FINDING-001
   **Severity**: Critical/High/Medium/Low
   **Category**: Credentials/Access/Data/Infrastructure
   **Workflow**: [Name]

   ## Description
   [What is the issue?]

   ## Risk
   [What could happen?]

   ## Evidence
   [Screenshot or code snippet]

   ## Remediation
   [How to fix]

   ## Status
   [ ] Open
   [ ] In Progress
   [ ] Fixed
   [ ] Verified

   **Assigned to**: [Person]
   **Due date**: [Date]
   EOF
   ```

#### Phase 2: Workflow Audit (2 hours)

1. **Export All Workflows**

   ```bash
   # Via n8n UI: Settings → Workflows → Export All
   # Or via API
   curl -X GET https://n8n.example.com/api/v1/workflows \
     -H "X-N8N-API-KEY: $API_KEY" \
     > workflows.json
   ```

2. **Scan for Common Issues**

   Create `scan-workflows.js`:

   ```javascript
   const fs = require('fs');

   const workflows = JSON.parse(fs.readFileSync('workflows.json'));
   const findings = [];

   workflows.forEach(workflow => {
     const workflowStr = JSON.stringify(workflow);

     // Check for hardcoded secrets
     const secretPatterns = {
       'Stripe Key': /sk_(live|test)_\w{24,}/,
       'AWS Key': /AKIA[0-9A-Z]{16}/,
       'Generic Secret': /(password|secret|key)\s*[:=]\s*['"][^'"]{8,}['"]/i,
       'Email': /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/
     };

     for (const [name, pattern] of Object.entries(secretPatterns)) {
       if (pattern.test(workflowStr)) {
         findings.push({
           id: `FINDING-${findings.length + 1}`,
           severity: 'CRITICAL',
           workflow: workflow.name,
           type: 'Hardcoded Secret',
           description: `Potential ${name} found in workflow code`,
           location: 'Code review required'
         });
       }
     }

     // Check for HTTP (not HTTPS)
     if (workflowStr.includes('http://') && !workflowStr.includes('localhost')) {
       findings.push({
         id: `FINDING-${findings.length + 1}`,
         severity: 'HIGH',
         workflow: workflow.name,
         type: 'Insecure Protocol',
         description: 'HTTP used instead of HTTPS',
         location: 'External API calls'
       });
     }

     // Check for console.log with potential PII
     if (/console\.log.*\$json\./i.test(workflowStr)) {
       findings.push({
         id: `FINDING-${findings.length + 1}`,
         severity: 'MEDIUM',
         workflow: workflow.name,
         type: 'Potential PII Logging',
         description: 'Logging full JSON objects may expose PII',
         location: 'Function nodes'
       });
     }

     // Check for error handling
     const hasErrorWorkflow = workflow.settings?.errorWorkflow;
     if (!hasErrorWorkflow) {
       findings.push({
         id: `FINDING-${findings.length + 1}`,
         severity: 'LOW',
         workflow: workflow.name,
         type: 'Missing Error Handler',
         description: 'No error workflow configured',
         location: 'Workflow settings'
       });
     }
   });

   // Generate report
   console.log(`\n=== Security Scan Results ===\n`);
   console.log(`Total workflows scanned: ${workflows.length}`);
   console.log(`Total findings: ${findings.length}\n`);

   const bySeverity = findings.reduce((acc, f) => {
     acc[f.severity] = (acc[f.severity] || 0) + 1;
     return acc;
   }, {});

   console.log('Findings by severity:');
   Object.entries(bySeverity).forEach(([severity, count]) => {
     console.log(`  ${severity}: ${count}`);
   });

   // Save to file
   fs.writeFileSync('findings.json', JSON.stringify(findings, null, 2));
   console.log('\nFindings saved to findings.json');
   ```

   Run:
   ```bash
   node scan-workflows.js
   ```

3. **Manual Review**

   For each workflow, check:
   - Function nodes for sensitive data handling
   - HTTP Request nodes for secure configurations
   - Credential usage and sharing
   - Error handling implementation
   - Logging practices

#### Phase 3: Credential Audit (1 hour)

1. **Query Credential Usage**

   ```sql
   -- Connect to n8n database
   docker exec -it n8n-postgres psql -U n8n -d n8n

   -- Credential audit query
   \copy (
     SELECT
       c.id,
       c.name,
       c.type,
       u.email as owner,
       c.created_at::date,
       c.updated_at::date,
       COALESCE(COUNT(DISTINCT wc.workflow_id), 0) as workflow_count,
       COALESCE(COUNT(DISTINCT cs.user_id), 0) as shared_with_count,
       MAX(e.finished_at)::date as last_used,
       CASE
         WHEN MAX(e.finished_at) IS NULL THEN 'Never used'
         WHEN MAX(e.finished_at) < NOW() - INTERVAL '90 days' THEN 'Unused (>90 days)'
         ELSE 'Active'
       END as status
     FROM credentials_entity c
     JOIN "user" u ON c.user_id = u.id
     LEFT JOIN workflow_credential wc ON c.id = wc.credential_id
     LEFT JOIN credentials_sharing cs ON c.id = cs.credential_id
     LEFT JOIN execution_entity e ON wc.workflow_id = e.workflow_id
     GROUP BY c.id, u.email
     ORDER BY last_used DESC NULLS LAST
   ) TO '/tmp/credential-audit.csv' WITH CSV HEADER;
   ```

   ```bash
   # Copy from container
   docker cp n8n-postgres:/tmp/credential-audit.csv ./evidence/
   ```

2. **Identify Issues**

   Review for:
   - [ ] Unused credentials (>90 days)
   - [ ] Overshared credentials (>5 users)
   - [ ] Credentials with generic names
   - [ ] Personal account credentials
   - [ ] No rotation schedule

#### Phase 4: Access Control Review (45 minutes)

1. **User Access Audit**

   ```sql
   -- User access report
   \copy (
     SELECT
       u.email,
       u.role,
       u.created_at::date,
       CASE
         WHEN u.disabled THEN 'Disabled'
         WHEN u.last_login IS NULL THEN 'Never logged in'
         WHEN u.last_login < NOW() - INTERVAL '90 days' THEN 'Inactive (>90 days)'
         ELSE 'Active'
       END as status,
       COUNT(DISTINCT w.id) as owned_workflows,
       COUNT(DISTINCT c.id) as owned_credentials
     FROM "user" u
     LEFT JOIN workflow_entity w ON u.id = w.user_id
     LEFT JOIN credentials_entity c ON u.id = c.user_id
     GROUP BY u.id
     ORDER BY u.last_login DESC NULLS LAST
   ) TO '/tmp/user-audit.csv' WITH CSV HEADER;
   ```

2. **Workflow Sharing Review**

   ```sql
   -- Workflow sharing audit
   SELECT
     w.name as workflow,
     owner.email as owner,
     shared.email as shared_with,
     ws.role
   FROM workflow_entity w
   JOIN "user" owner ON w.user_id = owner.id
   LEFT JOIN workflow_sharing ws ON w.id = ws.workflow_id
   LEFT JOIN "user" shared ON ws.user_id = shared.id
   WHERE ws.user_id IS NOT NULL
   ORDER BY w.name;
   ```

#### Phase 5: Generate Report (1 hour)

Create `security-audit-report.md`:

```markdown
# n8n Security Audit Report

**Date**: [Current Date]
**Auditor**: [Your Name]
**Instance**: Production n8n
**Version**: [n8n version]

## Executive Summary

- **Total Workflows**: [X]
- **Total Credentials**: [Y]
- **Total Users**: [Z]

### Findings Summary

| Severity | Count | % of Total |
|----------|-------|------------|
| Critical | X | XX% |
| High | X | XX% |
| Medium | X | XX% |
| Low | X | XX% |
| **Total** | **X** | **100%** |

## Critical Findings

### FINDING-001: Hardcoded API Key in Customer Sync Workflow

- **Severity**: CRITICAL
- **Workflow**: Customer Data Sync
- **Risk**: API key exposure, potential data breach
- **Evidence**: See screenshot in evidence/finding-001.png
- **Remediation**:
  1. Create credential in n8n credential store
  2. Update workflow to use credential
  3. Rotate API key
  4. Monitor for unauthorized usage
- **Assigned to**: Data Team
- **Due Date**: Immediate (24 hours)

[Continue for each critical finding...]

## High Priority Findings

[List all high priority findings...]

## Medium/Low Priority Findings

[Summary of medium and low findings...]

## Positive Observations

- All workflows use HTTPS
- Error workflows configured
- Regular backups in place
- SSL certificate valid

## Recommendations

### Immediate (This Week)
1. Fix all critical findings
2. Rotate compromised credentials
3. Remove inactive users

### Short Term (This Month)
1. Address high priority findings
2. Implement credential rotation schedule
3. Update security documentation

### Long Term (This Quarter)
1. Implement automated security scanning
2. Set up quarterly audit schedule
3. Conduct team security training

## Compliance Status

- **GDPR**: ⚠️ Needs attention (PII logging issues)
- **SOC 2**: ✅ Controls in place
- **Internal Policy**: ⚠️ 3 violations found

## Next Audit

**Scheduled**: [3 months from now]
**Focus Areas**:
- Re-verify critical fixes
- New workflows since this audit
- Credential rotation compliance

## Appendices

- Appendix A: Complete findings list (findings.json)
- Appendix B: Credential audit (credential-audit.csv)
- Appendix C: User access audit (user-audit.csv)
- Appendix D: Evidence screenshots
```

#### Phase 6: Remediation (Varies)

1. **Create Remediation Plan**

   ```markdown
   # Remediation Plan

   ## Week 1: Critical Issues
   - [ ] FINDING-001: Remove hardcoded keys
   - [ ] FINDING-002: Fix authentication bypass
   - [ ] FINDING-003: Rotate compromised credential

   ## Week 2: High Priority
   - [ ] FINDING-004-010: Address high priority items

   ## Week 3-4: Medium/Low Priority
   - [ ] Remaining findings

   ## Verification
   - [ ] Re-scan all workflows
   - [ ] Test all fixes
   - [ ] Update documentation
   ```

2. **Implement Fixes**

   Example: Fix hardcoded credential

   ```
   Before (❌):
   Function Node:
     const stripe = require('stripe')('sk_live_abc123...');

   After (✅):
   1. Create Stripe credential in n8n
   2. Use credential in HTTP Request node
   3. Remove hardcoded key
   4. Rotate Stripe key in Stripe dashboard
   ```

3. **Verify Fixes**

   - Test affected workflows
   - Re-run security scan
   - Update audit status

### Deliverables Checklist

- [ ] Complete audit report
- [ ] Findings spreadsheet/CSV
- [ ] Evidence (screenshots, code samples)
- [ ] Remediation plan with timeline
- [ ] Documentation of fixes
- [ ] Updated security procedures
- [ ] Lessons learned document

---

## Project 2: Implement Secure Credential Management System

### Objective

Set up HashiCorp Vault for centralized secrets management, migrate existing credentials, and implement automated rotation.

### Requirements

**Infrastructure:**
- HashiCorp Vault deployed
- Vault integration with n8n
- Automated credential rotation
- Audit logging enabled

**Deliverables:**
- Working Vault deployment
- Migrated credentials
- Rotation workflows
- Documentation

### Step-by-Step Guide

#### Phase 1: Deploy Vault (1 hour)

1. **Install Vault**

   ```bash
   # Docker Compose
   cat > docker-compose.vault.yml << 'EOF'
   version: '3.8'

   services:
     vault:
       image: vault:1.15
       container_name: n8n-vault
       restart: unless-stopped
       ports:
         - "8200:8200"
       environment:
         VAULT_DEV_ROOT_TOKEN_ID: ${VAULT_ROOT_TOKEN}
         VAULT_DEV_LISTEN_ADDRESS: 0.0.0.0:8200
       cap_add:
         - IPC_LOCK
       volumes:
         - vault-data:/vault/file
         - ./vault/config:/vault/config
         - ./vault/logs:/vault/logs
       command: server -config=/vault/config/vault.hcl

   volumes:
     vault-data:
   EOF

   # Production config
   mkdir -p vault/config
   cat > vault/config/vault.hcl << 'EOF'
   storage "file" {
     path = "/vault/file"
   }

   listener "tcp" {
     address     = "0.0.0.0:8200"
     tls_disable = 1
   }

   ui = true
   EOF

   # Start Vault
   docker compose -f docker-compose.vault.yml up -d
   ```

2. **Initialize Vault**

   ```bash
   # Set Vault address
   export VAULT_ADDR='http://127.0.0.1:8200'

   # Initialize (production - do this once!)
   vault operator init

   # Save unseal keys and root token securely!
   # Example output:
   # Unseal Key 1: xxx...
   # Unseal Key 2: yyy...
   # Unseal Key 3: zzz...
   # Root Token: aaa...

   # Unseal (requires 3 of 5 keys)
   vault operator unseal <key1>
   vault operator unseal <key2>
   vault operator unseal <key3>

   # Login
   vault login <root-token>
   ```

3. **Enable Secrets Engine**

   ```bash
   # Enable KV v2 secrets engine
   vault secrets enable -path=secret kv-v2

   # Test
   vault kv put secret/test foo=bar
   vault kv get secret/test
   ```

#### Phase 2: Migrate Credentials (2 hours)

1. **Export Current Credentials**

   ```bash
   # Via n8n API
   curl -X GET https://n8n.example.com/api/v1/credentials \
     -H "X-N8N-API-KEY: $N8N_API_KEY" \
     > credentials-export.json
   ```

2. **Store in Vault**

   ```bash
   #!/bin/bash
   # migrate-to-vault.sh

   # Read credentials export
   CREDS=$(cat credentials-export.json)

   # For each credential
   echo "$CREDS" | jq -c '.[]' | while read cred; do
     NAME=$(echo $cred | jq -r '.name')
     TYPE=$(echo $cred | jq -r '.type')

     echo "Migrating: $NAME ($TYPE)"

     # Store in Vault
     vault kv put "secret/n8n/credentials/${NAME}" \
       type="$TYPE" \
       data="$(echo $cred | jq -r '.data')"

     echo "✓ Migrated: $NAME"
   done

   echo "Migration complete!"
   ```

3. **Create n8n Fetch Workflow**

   ```
   Workflow: Fetch Credentials from Vault

   Manual Trigger
       ↓
   Function: Set credential name
     Input: credentialName
       ↓
   HTTP Request: Vault API
     URL: http://vault:8200/v1/secret/data/n8n/credentials/{{$json.credentialName}}
     Method: GET
     Headers:
       X-Vault-Token: {{$env.VAULT_TOKEN}}
       ↓
   Function: Extract credential data
     return {
       json: $json.data.data
     };
       ↓
   [Use credential in subsequent nodes]
   ```

#### Phase 3: Implement Rotation (2 hours)

1. **Create Rotation Workflow**

   ```
   Workflow: Credential Rotation

   Schedule Trigger: Monthly
       ↓
   Database: Get credentials needing rotation
     Query: SELECT * FROM credential_rotation_tracker
            WHERE next_rotation_due <= CURRENT_DATE
       ↓
   Loop: For each credential
       ↓
       [Generate New Credential]
       Service-specific API call
       Extract new key
       ↓
       [Store in Vault]
       HTTP Request: Vault PUT
       Path: secret/n8n/credentials/{{name}}
       ↓
       [Test New Credential]
       Service-specific test call
       ↓
       [Update n8n]
       n8n API: Update credential
       ↓
       [Schedule Old Key Revocation]
       Wait 30 days
       Service API: Revoke old key
       ↓
       [Log Rotation]
       Database: UPDATE credential_rotation_tracker
       ↓
   [End Loop]
       ↓
   Slack: Notify team of rotations
   ```

2. **Example: Stripe Rotation**

   ```javascript
   // Function Node: Rotate Stripe Key

   const stripe = require('stripe')(items[0].json.currentKey);

   // Create new restricted key
   const newKey = await stripe.apiKeys.create({
     name: `n8n-auto-${Date.now()}`,
     scope: {
       type: 'restricted',
       permissions: {
         'charges': 'read',
         'customers': 'write'
       }
     }
   });

   // Test new key
   const testStripe = require('stripe')(newKey.secret);
   await testStripe.customers.list({ limit: 1 });

   return {
     json: {
       serviceName: 'stripe',
       oldKeyId: items[0].json.currentKeyId,
       newKey: newKey.secret,
       newKeyId: newKey.id,
       rotatedAt: new Date()
     }
   };
   ```

3. **Create Monitoring Dashboard**

   ```
   Workflow: Credential Rotation Dashboard

   Schedule: Daily 9 AM
       ↓
   Database: Query rotation status
       ↓
   Function: Calculate metrics
     - Total credentials
     - Rotated this month
     - Due for rotation (next 7 days)
     - Overdue
       ↓
   Google Sheets: Update dashboard
       ↓
   IF (credentials overdue):
       Slack: Alert security team
   ```

#### Phase 4: Documentation (1 hour)

Create comprehensive documentation:

```markdown
# Vault Secrets Management Guide

## Architecture

[Diagram showing n8n ↔ Vault integration]

## Accessing Secrets

### Via n8n Workflow
1. Use HTTP Request node
2. Set URL: http://vault:8200/v1/secret/data/n8n/credentials/{name}
3. Add header: X-Vault-Token: {{$env.VAULT_TOKEN}}
4. Parse response

### Via CLI
```bash
vault kv get secret/n8n/credentials/stripe
```

## Credential Rotation

### Schedule
- Critical credentials: Every 30 days
- Standard credentials: Every 90 days
- Low-risk credentials: Every 180 days

### Process
1. Rotation workflow runs automatically
2. New credential generated
3. Stored in Vault
4. n8n updated
5. Old credential revoked after 30-day grace period

### Manual Rotation
[Emergency rotation procedures]

## Troubleshooting

### Vault Sealed
```bash
vault operator unseal <key>
```

### Token Expired
[Token renewal procedures]

## Emergency Procedures

### Vault Compromise
1. Immediately seal Vault
2. Rotate all credentials
3. Investigate breach
4. Restore from backup if needed
```

### Deliverables Checklist

- [ ] Vault deployed and initialized
- [ ] All credentials migrated
- [ ] Rotation workflow implemented
- [ ] Monitoring dashboard created
- [ ] Documentation complete
- [ ] Team trained on Vault usage
- [ ] Emergency procedures tested

---

## Project 3: Create Compliance-Ready Data Handling Workflow

### Objective

Build a workflow that demonstrates GDPR compliance, including proper PII handling, data minimization, and user rights implementation.

### Requirements

**Compliance Features:**
- Consent management
- Data minimization
- PII encryption
- Right to access (DSAR)
- Right to erasure
- Audit logging

**Deliverables:**
- Compliant workflow
- Data flow diagram
- Privacy impact assessment
- Documentation

### Step-by-Step Guide

#### Phase 1: Design (1 hour)

1. **Define Data Flow**

   ```
   User Registration
       ↓
   Collect Data (with consent)
       ↓
   Minimize Data (only essentials)
       ↓
   Encrypt PII
       ↓
   Store Separately:
     - PII (encrypted) → Secure DB
     - Non-PII → Regular DB
       ↓
   Log Activity (no PII in logs)
   ```

2. **Create Privacy Impact Assessment**

   ```markdown
   # Privacy Impact Assessment

   ## Purpose
   User registration and profile management system

   ## Data Collected
   - Email (required - for account creation)
   - Name (required - for personalization)
   - Phone (optional - for 2FA)
   - Date of Birth (optional - for age verification)

   ## Legal Basis
   - Contract performance (account creation)
   - Legitimate interest (service improvement)
   - Consent (marketing communications)

   ## Data Minimization
   - No SSN collected
   - No unnecessary fields
   - Optional fields clearly marked

   ## Security Measures
   - PII encrypted at rest (AES-256)
   - Encrypted in transit (TLS 1.3)
   - Access controls implemented
   - Audit logging enabled

   ## Retention
   - Active users: Until account deletion
   - Deleted accounts: 30 days (compliance)
   - Then: Complete erasure

   ## User Rights
   - Access: DSAR workflow
   - Rectification: Profile update workflow
   - Erasure: Account deletion workflow
   - Portability: Data export workflow

   ## Risks and Mitigation
   - Risk: Data breach
     Mitigation: Encryption, access controls, monitoring

   - Risk: Unauthorized access
     Mitigation: MFA, role-based access, audit logs
   ```

#### Phase 2: Implementation (3 hours)

1. **User Registration Workflow**

   ```
   Webhook: POST /register
     Body: { email, name, phone, dob, marketingConsent }
       ↓
   Function: Validate Input
     - Email valid format
     - Required fields present
     - Age >= 13 (if dob provided)
       ↓
   Function: Minimize Data
     - Remove fields not consented
     - Hash IP address
     - Generate user ID (UUID)
       ↓
   Function: Separate PII
     PII: email, name, phone, dob
     Non-PII: userId, createdAt, preferences
       ↓
   Function: Encrypt PII
     ```javascript
     const crypto = require('crypto');
     const ENCRYPTION_KEY = Buffer.from(process.env.PII_KEY, 'hex');

     function encryptPII(text) {
       const iv = crypto.randomBytes(16);
       const cipher = crypto.createCipheriv('aes-256-cbc', ENCRYPTION_KEY, iv);
       let encrypted = cipher.update(text, 'utf8', 'hex');
       encrypted += cipher.final('hex');
       return iv.toString('hex') + ':' + encrypted;
     }

     return {
       json: {
         userId: $json.userId,
         email_encrypted: encryptPII($json.email),
         name_encrypted: encryptPII($json.name),
         phone_encrypted: $json.phone ? encryptPII($json.phone) : null,
         dob_encrypted: $json.dob ? encryptPII($json.dob) : null
       }
     };
     ```
       ↓
   PostgreSQL: Store Encrypted PII
     Table: user_pii_encrypted
     Columns: userId, email_encrypted, name_encrypted, ...
       ↓
   PostgreSQL: Store Non-PII
     Table: user_accounts
     Columns: userId, createdAt, preferences, ...
       ↓
   PostgreSQL: Log Consent
     Table: consent_log
     Columns: userId, consentType, granted, timestamp
       ↓
   Function: Create Audit Log (NO PII)
     ```javascript
     return {
       json: {
         timestamp: new Date(),
         eventType: 'user_registration',
         userId: $json.userId,  // ID only, not email!
         ipHash: hashIP($json.ip),
         success: true
       }
     };
     ```
       ↓
   PostgreSQL: Store Audit Log
       ↓
   Response: Success (no PII in response)
   ```

2. **Data Subject Access Request (DSAR) Workflow**

   ```
   Webhook: POST /dsar
     Body: { email }
       ↓
   Function: Validate Identity
     - Email verification
     - Optional: Additional verification
       ↓
   PostgreSQL: Get User ID
     FROM user_pii_encrypted
     WHERE email_encrypted LIKE '%...'  (approximate match)
       ↓
   Function: Decrypt PII
     ```javascript
     function decryptPII(encrypted) {
       const [ivHex, encryptedText] = encrypted.split(':');
       const iv = Buffer.from(ivHex, 'hex');
       const decipher = crypto.createDecipheriv('aes-256-cbc', ENCRYPTION_KEY, iv);
       let decrypted = decipher.update(encryptedText, 'hex', 'utf8');
       decrypted += decipher.final('utf8');
       return decrypted;
     }

     return {
       json: {
         email: decryptPII($json.email_encrypted),
         name: decryptPII($json.name_encrypted),
         phone: $json.phone_encrypted ? decryptPII($json.phone_encrypted) : null
       }
     };
     ```
       ↓
   PostgreSQL: Get All User Data
     - Profile data
     - Activity history (sanitized)
     - Consent records
     - (No internal IDs or technical data)
       ↓
   Function: Compile Data Export
     JSON format, user-friendly
       ↓
   Function: Encrypt Export File
     Password-protected zip
       ↓
   Email: Send to User
     Encrypted attachment
     Password via SMS (if phone on file)
       ↓
   PostgreSQL: Log DSAR Fulfillment
     userId, timestamp, delivered
   ```

3. **Right to Erasure Workflow**

   ```
   Webhook: POST /delete-account
     Body: { email, confirmation }
       ↓
   Function: Validate Identity
     - Verify user
     - Confirm deletion intent
       ↓
   PostgreSQL: Check Retention Obligations
     - Any active orders?
     - Legal holds?
     - Financial records retention?
       ↓
   IF (can delete immediately):
       ↓
       PostgreSQL: DELETE user_pii_encrypted
       PostgreSQL: DELETE user_accounts
       PostgreSQL: ANONYMIZE transaction history
         (keep for financial records, remove PII)
       PostgreSQL: DELETE consent_log
       ↓
       External APIs: Remove from services
         - Email list (Mailchimp)
         - CRM (if applicable)
         - Analytics (anonymize)
       ↓
       PostgreSQL: Log Deletion
         Action: "user_deleted"
         Timestamp, userId (before deletion)
       ↓
       Email: Confirmation
   ELSE:
       ↓
       PostgreSQL: Mark for deletion
         deletion_scheduled = NOW() + retention_period
       ↓
       Schedule: Auto-delete after retention
       ↓
       Email: Explain delay
   ```

4. **Audit Logging (GDPR-Compliant)**

   ```javascript
   // Function Node: Create Audit Log

   // ✅ GOOD - No PII
   const auditLog = {
     timestamp: new Date().toISOString(),
     eventType: $json.eventType,  // 'registration', 'dsar', 'deletion'
     userId: $json.userId,  // UUID, not email
     action: $json.action,
     result: $json.success ? 'success' : 'failure',
     ipHash: crypto.createHash('sha256').update($json.ip).digest('hex').substr(0, 16)
   };

   // ❌ BAD - Contains PII
   // email: $json.email,
   // name: $json.name,
   // ip: $json.ip

   return { json: auditLog };
   ```

#### Phase 3: Testing (1 hour)

1. **Test User Registration**
   - Register new user
   - Verify PII encrypted in database
   - Check audit log has no PII
   - Verify consent recorded

2. **Test DSAR**
   - Request data export
   - Verify all data included
   - Check encryption of export file
   - Confirm delivery

3. **Test Deletion**
   - Request account deletion
   - Verify data removed
   - Check anonymization of retained data
   - Confirm external systems updated

#### Phase 4: Documentation (1 hour)

Create documentation:

```markdown
# GDPR-Compliant Workflow Documentation

## Overview
This workflow system implements GDPR requirements for user data handling.

## Data Flow Diagram
[Insert visual diagram]

## Compliance Matrix

| GDPR Requirement | Implementation |
|------------------|----------------|
| Lawful basis | Consent captured, contract performance |
| Data minimization | Only essential fields collected |
| Purpose limitation | Data used only for stated purposes |
| Storage limitation | Retention policies implemented |
| Integrity/confidentiality | Encryption at rest and in transit |
| Right to access | DSAR workflow |
| Right to erasure | Deletion workflow |
| Right to portability | Data export in JSON format |
| Data breach notification | Incident response plan (separate doc) |

## PII Handling

### What is PII in this system?
- Email address
- Full name
- Phone number
- Date of birth

### How is PII protected?
- Encrypted at rest (AES-256-CBC)
- Encrypted in transit (TLS 1.3)
- Separated from non-PII data
- Never logged in plain text
- Access controls enforced

## User Rights

### How to submit requests:
- DSAR: POST /dsar with email
- Deletion: POST /delete-account with email

### Response time:
- Within 30 days (GDPR requirement)
- Usually within 7 days

## Retention Policy

- Active accounts: Until user requests deletion
- Deleted accounts: 30 days for recovery
- Financial records: 7 years (anonymized)
- Audit logs: 7 years (no PII)

## Audit Trail

All actions logged:
- User registration
- Data access (DSAR)
- Data modification
- Data deletion

Logs contain: timestamp, userId, action, result (NO PII)

## Privacy Impact Assessment

See: privacy-impact-assessment.md

## Contact

Data Protection Officer: dpo@example.com
Privacy questions: privacy@example.com
```

### Deliverables Checklist

- [ ] Workflows implemented (registration, DSAR, deletion)
- [ ] PII encryption working
- [ ] Audit logging (no PII)
- [ ] Data flow diagram created
- [ ] Privacy impact assessment completed
- [ ] Documentation complete
- [ ] Testing completed
- [ ] Compliance verification

---

## Submission Guidelines

### What to Submit

For each project:

1. **Documentation**
   - Security audit report (Project 1)
   - Vault setup guide (Project 2)
   - Privacy documentation (Project 3)

2. **Evidence**
   - Screenshots of implementations
   - Sample data/logs (sanitized)
   - Test results

3. **Code/Configurations**
   - Workflow exports
   - Scripts used
   - Configuration files (no secrets!)

4. **Verification**
   - Checklist completion proof
   - Test results
   - Before/after comparisons

### Evaluation Criteria

- **Security**: Proper implementation of security controls
- **Compliance**: Adherence to regulations
- **Documentation**: Clear and comprehensive
- **Best Practices**: Industry standards followed
- **Testing**: Thorough verification

## Bonus Challenges

If you complete all three projects:

1. **Automated Security Scanning**
   - Integrate OWASP ZAP or similar
   - Run daily scans
   - Alert on new findings

2. **Compliance Dashboard**
   - Real-time compliance metrics
   - Automated reporting
   - Trend analysis

3. **Incident Response Automation**
   - Automated credential revocation
   - Breach notification workflow
   - Forensics data collection

## Resources

- [GDPR Official Text](https://gdpr-info.eu/)
- [HIPAA Security Rule](https://www.hhs.gov/hipaa/for-professionals/security/)
- [HashiCorp Vault Docs](https://www.vaultproject.io/docs)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [n8n Security Docs](https://docs.n8n.io/hosting/security/)

## Conclusion

Completing these three projects demonstrates mastery of security and compliance in n8n production environments. Take your time, document thoroughly, and don't hesitate to ask for help!

**Remember**: Security and compliance are ongoing processes, not one-time tasks!

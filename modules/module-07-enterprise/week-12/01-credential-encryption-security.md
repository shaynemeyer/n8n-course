# Credential Encryption and Security

## Introduction

Credentials are the keys to your integrations and data. Properly securing them is critical for protecting your systems and data. This guide covers how n8n encrypts credentials and best practices for credential security.

## How n8n Encrypts Credentials

### Encryption Mechanism

n8n uses AES-256-CBC encryption to protect credentials:

1. **Encryption Key**: 256-bit key derived from `N8N_ENCRYPTION_KEY`
2. **Algorithm**: AES-256 in CBC (Cipher Block Chaining) mode
3. **Storage**: Encrypted credentials stored in database
4. **Decryption**: Only decrypted when needed during workflow execution

### What Gets Encrypted

- API keys and tokens
- Passwords
- OAuth tokens and refresh tokens
- Private keys
- Database connection strings
- Any credential data

### What Doesn't Get Encrypted

- Credential names
- Credential types
- Metadata (creation date, owner, etc.)
- Sharing settings

## The Encryption Key

### Critical Importance

The `N8N_ENCRYPTION_KEY` is the **most critical** security component:

- Lost key = lost credentials (cannot be recovered)
- Compromised key = all credentials compromised
- Must be identical across all n8n instances in a cluster
- Must be backed up securely

### Generating a Secure Encryption Key

```bash
# Generate 32-byte key (recommended)
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"

# Alternative using openssl
openssl rand -base64 32

# Alternative using /dev/urandom
head -c 32 /dev/urandom | base64
```

Example output:
```
YourEncryptionKeyHere123456789ABCDEF==
```

### Setting the Encryption Key

#### Docker Compose

```yaml
services:
  n8n:
    environment:
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
```

.env file:
```bash
N8N_ENCRYPTION_KEY=YourEncryptionKeyHere123456789ABCDEF==
```

#### Direct Environment Variable

```bash
export N8N_ENCRYPTION_KEY="YourEncryptionKeyHere123456789ABCDEF=="
```

### Key Rotation

**Warning**: Rotating the encryption key requires re-encrypting all credentials.

#### Migration Script

```javascript
// key-rotation.js
const { Credentials } = require('n8n-core');
const { Db } = require('n8n-workflow');

async function rotateEncryptionKey(oldKey, newKey) {
  // Initialize database
  await Db.init();

  // Get all credentials
  const credentials = await Db.collections.Credentials.find();

  console.log(`Rotating ${credentials.length} credentials...`);

  for (const credential of credentials) {
    try {
      // Decrypt with old key
      process.env.N8N_ENCRYPTION_KEY = oldKey;
      const decrypted = credential.getData();

      // Re-encrypt with new key
      process.env.N8N_ENCRYPTION_KEY = newKey;
      credential.setData(decrypted);

      // Save
      await credential.save();

      console.log(`✓ Rotated: ${credential.name}`);
    } catch (error) {
      console.error(`✗ Failed: ${credential.name}`, error.message);
    }
  }

  console.log('Key rotation complete');
}

// Usage
const OLD_KEY = process.argv[2];
const NEW_KEY = process.argv[3];

rotateEncryptionKey(OLD_KEY, NEW_KEY);
```

Run:
```bash
node key-rotation.js "old-key-here" "new-key-here"
```

## Encryption Key Management

### Backup Strategy

#### 1. Secure Offline Backup

```bash
# Print key to paper (air-gapped)
echo $N8N_ENCRYPTION_KEY

# Store in safe or safety deposit box
# Never store in digital form on same system
```

#### 2. Password Manager

```bash
# Store in 1Password, LastPass, Bitwarden, etc.
# Use separate vault from regular passwords
# Enable 2FA on password manager
```

#### 3. Hardware Security Module (HSM)

For enterprise deployments:

```bash
# Store key in HSM
# Access via PKCS#11 interface
# Physical security for key material
```

#### 4. Cloud Key Management

**AWS KMS**:
```bash
# Store encryption key in AWS Secrets Manager
aws secretsmanager create-secret \
  --name n8n-encryption-key \
  --secret-string "$N8N_ENCRYPTION_KEY"

# Retrieve in startup script
N8N_ENCRYPTION_KEY=$(aws secretsmanager get-secret-value \
  --secret-id n8n-encryption-key \
  --query SecretString \
  --output text)
```

**Azure Key Vault**:
```bash
# Store in Key Vault
az keyvault secret set \
  --vault-name MyKeyVault \
  --name n8n-encryption-key \
  --value "$N8N_ENCRYPTION_KEY"

# Retrieve
az keyvault secret show \
  --vault-name MyKeyVault \
  --name n8n-encryption-key \
  --query value -o tsv
```

### Key Protection Best Practices

1. **Never commit to version control**
   ```bash
   # .gitignore
   .env
   .env.*
   secrets/
   *.key
   ```

2. **Restrict file permissions**
   ```bash
   chmod 600 .env
   chown n8n:n8n .env
   ```

3. **Use separate keys per environment**
   ```bash
   # Development
   N8N_ENCRYPTION_KEY_DEV=dev-key-here

   # Production
   N8N_ENCRYPTION_KEY_PROD=prod-key-here
   ```

4. **Rotate periodically**
   - Schedule: Annually or after security incident
   - Process: Test migration, execute during maintenance window
   - Verification: Test all credentials after rotation

## Credential Storage Best Practices

### 1. Use Service Accounts

```
❌ Bad: personal.email@gmail.com
✅ Good: n8n-automation@company.com
```

Benefits:
- No impact when team members leave
- Clearer audit trails
- Dedicated permissions
- Easier to manage

### 2. Least Privilege Principle

Grant minimum permissions necessary:

```
❌ Bad: Admin access to entire system
✅ Good: Read-only access to specific resources
```

Example (AWS):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::specific-bucket/*"
      ]
    }
  ]
}
```

### 3. Credential Naming Convention

Use descriptive, standardized names:

```
❌ Bad: "Gmail", "API Key", "Token"
✅ Good: "Gmail - Team Notifications", "Stripe - Production", "GitHub - Deploy Bot"
```

Format:
```
[Service] - [Purpose] - [Environment]
```

Examples:
- "Salesforce - Lead Sync - Production"
- "Slack - Alerts - Development"
- "AWS - S3 Backup - Production"

### 4. Regular Credential Audits

Create audit workflow:

```
Schedule (Monthly)
  ↓
Database Query (Get all credentials)
  ↓
Function (Check last used date)
  ↓
Filter (Unused > 90 days)
  ↓
Email (Alert admin of unused credentials)
```

SQL Query:
```sql
SELECT
  c.name,
  c.type,
  c.created_at,
  c.updated_at,
  u.email as owner,
  COUNT(e.id) as usage_count,
  MAX(e.finished_at) as last_used
FROM credentials_entity c
JOIN user u ON c.user_id = u.id
LEFT JOIN execution_entity e ON e.workflow_id IN (
  SELECT workflow_id FROM workflow_credential WHERE credential_id = c.id
)
GROUP BY c.id, u.email
ORDER BY last_used ASC;
```

## Credential Sharing Security

### Sharing Policies

#### Policy Template

```markdown
# Credential Sharing Policy

## Allowed
- Share service account credentials with team members
- Share with specific users only (not all members)
- Document reason for sharing in credential notes

## Not Allowed
- Share personal credentials
- Share with external users
- Share across departments without approval

## Process
1. Request approval from security team
2. Document business justification
3. Set expiration date for access
4. Review quarterly
```

### Sharing Levels

#### 1. Private (Default)
```
Owner: You
Shared with: None
Use: Personal workflows only
```

#### 2. Team Sharing
```
Owner: Team service account
Shared with: Team members
Use: Team workflows
Review: Quarterly
```

#### 3. Organization-Wide
```
Owner: Org admin
Shared with: All members
Use: Common integrations
Review: Monthly
Examples: Company Slack, Email
```

### Implementing Sharing Controls

```sql
-- Check credential sharing
SELECT
  c.name as credential_name,
  c.type,
  owner.email as owner,
  shared.email as shared_with,
  cs.role
FROM credentials_entity c
JOIN user owner ON c.user_id = owner.id
LEFT JOIN credentials_sharing cs ON c.id = cs.credential_id
LEFT JOIN user shared ON cs.user_id = shared.id
WHERE c.id = ?;
```

## OAuth Credentials

### Special Considerations

OAuth credentials require extra security:

1. **Refresh Tokens**: Long-lived, must be protected
2. **Access Tokens**: Short-lived, auto-refreshed
3. **Client Secrets**: Never expose in URLs or logs

### OAuth Security Checklist

- [ ] Use HTTPS for OAuth callbacks
- [ ] Validate redirect URIs
- [ ] Store client secrets encrypted
- [ ] Implement token refresh
- [ ] Monitor token usage
- [ ] Revoke tokens when no longer needed
- [ ] Use state parameter to prevent CSRF

### OAuth Token Rotation

```javascript
// Automatic token refresh
// n8n handles this, but verify:

// In credential configuration
{
  "oauthTokenData": {
    "access_token": "...",
    "refresh_token": "...",
    "expires_at": 1234567890
  }
}

// n8n automatically refreshes when expires_at is reached
```

## Database-Level Security

### Encryption at Rest

#### PostgreSQL Transparent Data Encryption (TDE)

```sql
-- Enable encryption at filesystem level
-- Use encrypted volumes (LUKS, dm-crypt)

-- Or use PostgreSQL extensions
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Additional column encryption (if needed)
SELECT pgp_sym_encrypt('sensitive data', 'encryption-key');
```

#### Encrypted Docker Volumes

```yaml
services:
  postgres:
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
    driver: local
    driver_opts:
      type: none
      o: bind,encryption=aes-xts-plain64
      device: /encrypted/path
```

### Network Encryption

#### SSL/TLS for Database Connections

```bash
# PostgreSQL with SSL
DB_POSTGRESDB_SSL=true
DB_POSTGRESDB_SSL_REJECT_UNAUTHORIZED=true

# Connection string
postgresql://user:pass@host:5432/db?sslmode=require
```

Generate certificates:
```bash
# Server certificate
openssl req -new -x509 -days 365 -nodes \
  -out server.crt -keyout server.key

# Client certificate
openssl req -new -x509 -days 365 -nodes \
  -out client.crt -keyout client.key
```

PostgreSQL configuration:
```conf
ssl = on
ssl_cert_file = '/path/to/server.crt'
ssl_key_file = '/path/to/server.key'
ssl_ca_file = '/path/to/root.crt'
```

## Secrets Management Integration

### HashiCorp Vault

#### Setup

```bash
# Start Vault
docker run -d --name vault \
  -p 8200:8200 \
  -e VAULT_DEV_ROOT_TOKEN_ID=mytoken \
  vault:latest

# Initialize
vault operator init

# Unseal
vault operator unseal <key1>
vault operator unseal <key2>
vault operator unseal <key3>
```

#### Store Credentials

```bash
# Login
vault login mytoken

# Store secret
vault kv put secret/n8n/stripe \
  api_key="sk_live_..." \
  publishable_key="pk_live_..."

# Read secret
vault kv get secret/n8n/stripe
```

#### n8n Integration

Create workflow to fetch from Vault:

```
Webhook/Schedule
  ↓
HTTP Request to Vault
  URL: http://vault:8200/v1/secret/data/n8n/stripe
  Headers: X-Vault-Token: <token>
  ↓
Set Node (Extract secrets)
  ↓
Use in subsequent nodes
```

### AWS Secrets Manager

```bash
# Create secret
aws secretsmanager create-secret \
  --name n8n/stripe \
  --secret-string '{"api_key":"sk_live_..."}'

# Retrieve secret
aws secretsmanager get-secret-value \
  --secret-id n8n/stripe \
  --query SecretString \
  --output text
```

n8n workflow:
```
Schedule
  ↓
AWS Lambda (Get secret)
  or
HTTP Request to Secrets Manager API
  ↓
Parse JSON
  ↓
Use credentials
```

### Azure Key Vault

```bash
# Create key vault
az keyvault create \
  --name myKeyVault \
  --resource-group myResourceGroup

# Store secret
az keyvault secret set \
  --vault-name myKeyVault \
  --name stripe-api-key \
  --value "sk_live_..."

# Retrieve secret
az keyvault secret show \
  --vault-name myKeyVault \
  --name stripe-api-key
```

## Monitoring and Alerting

### Credential Usage Monitoring

Create monitoring workflow:

```
Schedule (Daily)
  ↓
Database Query (Get credential usage)
  ↓
Function (Analyze patterns)
  ↓
IF (Unusual activity)
  ↓
  Slack Alert
  ↓
  Email Security Team
```

### Anomaly Detection

Monitor for:
- Credentials used from unusual IP addresses
- High-frequency credential usage (potential leak)
- Credentials used at unusual times
- Failed authentication attempts
- Credentials used in disabled workflows

### Audit Logging

```sql
-- Create audit log table
CREATE TABLE credential_audit_log (
  id SERIAL PRIMARY KEY,
  credential_id INTEGER,
  action VARCHAR(50),
  user_id INTEGER,
  ip_address INET,
  timestamp TIMESTAMP DEFAULT NOW(),
  details JSONB
);

-- Log credential access
INSERT INTO credential_audit_log
  (credential_id, action, user_id, ip_address, details)
VALUES
  (123, 'accessed', 456, '192.168.1.100', '{"workflow_id": 789}');
```

## Incident Response

### Credential Compromise Response Plan

#### 1. Immediate Actions (0-1 hour)

```bash
# Revoke compromised credential
# In affected service (e.g., Stripe)
stripe api_keys delete <key_id>

# Disable in n8n
# Mark credential as inactive in database
UPDATE credentials_entity
SET active = false
WHERE id = <credential_id>;

# Stop affected workflows
# In n8n UI or via API
```

#### 2. Investigation (1-24 hours)

- Check access logs
- Identify scope of compromise
- Determine attack vector
- Document timeline

#### 3. Remediation (24-72 hours)

- Create new credentials
- Update workflows
- Test functionality
- Monitor for issues

#### 4. Follow-up (1-2 weeks)

- Post-incident review
- Update security procedures
- Implement additional controls
- Train team on lessons learned

### Emergency Credential Rotation

```bash
#!/bin/bash
# emergency-rotate.sh

CREDENTIAL_NAME=$1

echo "EMERGENCY CREDENTIAL ROTATION"
echo "Credential: $CREDENTIAL_NAME"
echo "Started: $(date)"

# 1. Disable workflow
echo "Disabling affected workflows..."
# Use n8n API to disable

# 2. Revoke in external service
echo "Revoking in external service..."
# Service-specific revocation

# 3. Generate new credential
echo "Generating new credential..."
# Service-specific generation

# 4. Update in n8n
echo "Updating n8n..."
# Use n8n API to update

# 5. Re-enable workflows
echo "Re-enabling workflows..."
# Use n8n API to enable

echo "Completed: $(date)"
```

## Best Practices Checklist

### Encryption
- [ ] Strong encryption key generated (32+ bytes)
- [ ] Encryption key backed up securely
- [ ] Encryption key not in version control
- [ ] Separate keys for dev/staging/prod
- [ ] Key rotation plan documented

### Storage
- [ ] Use service accounts, not personal
- [ ] Least privilege permissions
- [ ] Descriptive credential names
- [ ] Regular credential audits
- [ ] Unused credentials removed

### Sharing
- [ ] Credential sharing policy documented
- [ ] Share with specific users only
- [ ] Quarterly access reviews
- [ ] Document sharing justification
- [ ] Remove access when no longer needed

### Monitoring
- [ ] Credential usage logged
- [ ] Anomaly detection in place
- [ ] Alerts configured
- [ ] Regular audit log reviews
- [ ] Incident response plan documented

### Integration
- [ ] Consider secrets management tool
- [ ] Implement token rotation
- [ ] Monitor OAuth token usage
- [ ] Database encryption enabled
- [ ] SSL/TLS for database connections

## Quick Reference

### Generate Encryption Key
```bash
openssl rand -base64 32
```

### Check Credential Usage
```sql
SELECT c.name, COUNT(e.id) as executions
FROM credentials_entity c
LEFT JOIN workflow_credential wc ON c.id = wc.credential_id
LEFT JOIN execution_entity e ON wc.workflow_id = e.workflow_id
GROUP BY c.id;
```

### Rotate Credential
1. Create new in external service
2. Update in n8n
3. Test workflows
4. Revoke old credential
5. Monitor for issues

## Next Steps

After mastering credential security:
1. Implement environment variable management (see `02-environment-variable-management.md`)
2. Set up API key rotation (see `03-api-key-rotation.md`)
3. Configure secrets management tool
4. Conduct credential security audit
5. Document procedures for team

# API Key Rotation

## Introduction

API key rotation is a critical security practice that limits the damage from compromised credentials. This guide covers strategies, automation, and best practices for rotating API keys in n8n workflows.

## Why Rotate API Keys?

### Security Benefits

1. **Limits exposure window**: Compromised keys have limited lifetime
2. **Reduces impact**: Old keys automatically expire
3. **Audit trail**: Track which keys were used when
4. **Compliance**: Many regulations require regular rotation
5. **Best practice**: Defense in depth security

### Risk Mitigation

**Without rotation:**
- Leaked key remains valid forever
- No way to know if key is compromised
- Single point of failure

**With rotation:**
- Old keys expire automatically
- Limited window for exploitation
- Forces regular security review

## Rotation Strategies

### 1. Time-Based Rotation

Rotate on fixed schedule:

| Sensitivity | Rotation Frequency |
|-------------|-------------------|
| Critical (Production DB, Payment APIs) | 30-90 days |
| High (User data, Auth systems) | 90-180 days |
| Medium (Analytics, Logging) | 180-365 days |
| Low (Public APIs, Read-only) | Annually or as needed |

### 2. Event-Based Rotation

Rotate when:
- Employee leaves company
- Security incident occurs
- Suspicious activity detected
- Compliance audit scheduled
- System compromise suspected

### 3. Usage-Based Rotation

Rotate after:
- X number of API calls
- Specific data volume processed
- Time period + usage threshold

## Zero-Downtime Rotation Process

### The Challenge

```
Old Key Active
    ↓
Generate New Key
    ↓
Update n8n
    ↓
??? Downtime while updating ???
    ↓
Revoke Old Key
```

### The Solution: Overlapping Keys

```
Old Key Active
    ↓
Generate New Key (both keys active)
    ↓
Update n8n to use New Key
    ↓
Test with New Key
    ↓
Monitor for Old Key usage
    ↓
Grace period (7-30 days)
    ↓
Revoke Old Key
```

## Rotation Workflows

### Manual Rotation Template

```
Manual Trigger: "Rotate API Key"
    ↓
[1] Generate New Key
    HTTP Request to API
    Extract new key
    ↓
[2] Test New Key
    HTTP Request with new key
    Verify response
    ↓
[3] Update n8n Credential
    n8n API: Update credential
    ↓
[4] Update Secrets Manager
    Vault/AWS: Store new key
    ↓
[5] Notify Team
    Slack: "API key rotated"
    ↓
[6] Schedule Old Key Revocation
    Wait 7 days
    HTTP Request: Revoke old key
    ↓
[7] Verify
    Test workflows still work
```

### Automated Rotation Workflow

```
Schedule Trigger: Monthly (1st at 2 AM)
    ↓
[1] Check Services Needing Rotation
    Query database for keys > 30 days old
    ↓
[2] For Each Service
    ↓
    [2a] Generate New Key
        Service-specific API call
        ↓
    [2b] Store in Secrets Manager
        Vault KV put
        ↓
    [2c] Update n8n Credential
        n8n API update
        ↓
    [2d] Test
        Execute test workflow
        ↓
    [2e] Schedule Revocation
        Create reminder workflow
        ↓
[3] Generate Report
    List all rotated keys
    Success/failure status
    ↓
[4] Notify
    Email to security team
    Slack notification
```

## Service-Specific Rotation

### AWS Access Keys

#### Rotation Workflow

```javascript
// n8n Function Node: Rotate AWS Keys

const AWS = require('aws-sdk');
const iam = new AWS.IAM();

// 1. Create new access key
const newKey = await iam.createAccessKey({
  UserName: 'n8n-automation'
}).promise();

// 2. Store new key
const newAccessKey = newKey.AccessKey.AccessKeyId;
const newSecretKey = newKey.AccessKey.SecretAccessKey;

// 3. Test new key
AWS.config.update({
  accessKeyId: newAccessKey,
  secretAccessKey: newSecretKey
});

const s3 = new AWS.S3();
await s3.listBuckets().promise(); // Test

// 4. Schedule old key deletion (30 days)
setTimeout(async () => {
  await iam.deleteAccessKey({
    UserName: 'n8n-automation',
    AccessKeyId: oldAccessKey
  }).promise();
}, 30 * 24 * 60 * 60 * 1000);

return {
  newAccessKey,
  newSecretKey,
  oldAccessKey: $env.OLD_AWS_ACCESS_KEY
};
```

#### Complete AWS Rotation

```bash
#!/bin/bash
# rotate-aws-keys.sh

USER_NAME="n8n-automation"

# Create new key
NEW_KEY=$(aws iam create-access-key --user-name $USER_NAME)
NEW_ACCESS_KEY=$(echo $NEW_KEY | jq -r '.AccessKey.AccessKeyId')
NEW_SECRET_KEY=$(echo $NEW_KEY | jq -r '.AccessKey.SecretAccessKey')

echo "New key created: $NEW_ACCESS_KEY"

# Update in secrets manager
vault kv put secret/n8n/aws \
  access_key="$NEW_ACCESS_KEY" \
  secret_key="$NEW_SECRET_KEY"

# Update n8n credential via API
curl -X PATCH https://n8n.example.com/api/v1/credentials/123 \
  -H "Authorization: Bearer $N8N_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"data\": {
      \"accessKeyId\": \"$NEW_ACCESS_KEY\",
      \"secretAccessKey\": \"$NEW_SECRET_KEY\"
    }
  }"

# Test
aws s3 ls --profile n8n-new

# Wait 30 days, then revoke old key
echo "aws iam delete-access-key --user-name $USER_NAME --access-key-id $OLD_KEY" | at now + 30 days

echo "Rotation complete!"
```

### Stripe API Keys

#### Rotation via API

```javascript
// n8n Function Node: Rotate Stripe Key

const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);

// 1. Create new restricted key
const newKey = await stripe.apiKeys.create({
  name: `n8n-key-${Date.now()}`,
  scope: {
    type: 'restricted',
    permissions: {
      'charges': 'read',
      'customers': 'write'
    }
  }
});

// 2. Test new key
const testStripe = require('stripe')(newKey.secret);
await testStripe.customers.list({ limit: 1 });

// 3. Update n8n
// (via n8n API or manual)

// 4. Schedule deletion of old key
setTimeout(() => {
  stripe.apiKeys.del(oldKeyId);
}, 30 * 24 * 60 * 60 * 1000);

return { newKey: newKey.secret };
```

### GitHub Personal Access Tokens

#### Manual Process

1. **Generate new token**
   - Go to Settings → Developer settings → Personal access tokens
   - Generate new token with same scopes
   - Copy token immediately

2. **Update n8n**
   ```bash
   # Via n8n UI or API
   ```

3. **Test**
   ```bash
   curl -H "Authorization: token NEW_TOKEN" \
     https://api.github.com/user
   ```

4. **Revoke old token** (after grace period)
   - Return to GitHub settings
   - Delete old token

#### Automated with GitHub CLI

```bash
#!/bin/bash
# rotate-github-token.sh

# Requires: gh (GitHub CLI)

# Create new token (requires manual approval)
echo "Creating new GitHub token..."
gh auth login --scopes repo,workflow

# Get new token
NEW_TOKEN=$(gh auth token)

# Update in n8n
vault kv put secret/n8n/github token="$NEW_TOKEN"

# Update n8n credential
# (Use n8n API)

echo "Token rotated. Revoke old token in GitHub settings."
```

### Google Cloud Service Account Keys

#### Rotation Script

```bash
#!/bin/bash
# rotate-gcp-key.sh

PROJECT_ID="my-project"
SERVICE_ACCOUNT="n8n-automation@my-project.iam.gserviceaccount.com"

# Create new key
gcloud iam service-accounts keys create new-key.json \
  --iam-account=$SERVICE_ACCOUNT

# Extract key ID
NEW_KEY_ID=$(jq -r '.private_key_id' new-key.json)

# Update secrets manager
vault kv put secret/n8n/gcp \
  credentials=@new-key.json

# Test new key
export GOOGLE_APPLICATION_CREDENTIALS="new-key.json"
gcloud projects list

# Delete old key (after testing)
# List keys to find old one
gcloud iam service-accounts keys list \
  --iam-account=$SERVICE_ACCOUNT

# Delete (after grace period)
# gcloud iam service-accounts keys delete $OLD_KEY_ID \
#   --iam-account=$SERVICE_ACCOUNT

echo "Rotation complete!"
```

## Tracking Key Rotation

### Rotation Tracker Database

```sql
CREATE TABLE api_key_rotation_log (
  id SERIAL PRIMARY KEY,
  service_name VARCHAR(100) NOT NULL,
  key_id VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW(),
  rotated_at TIMESTAMP,
  next_rotation_due DATE,
  rotated_by VARCHAR(100),
  status VARCHAR(50), -- active, rotated, revoked
  notes TEXT
);

-- Insert new key
INSERT INTO api_key_rotation_log
  (service_name, key_id, next_rotation_due, status)
VALUES
  ('Stripe', 'sk_live_abc123', NOW() + INTERVAL '90 days', 'active');

-- Mark as rotated
UPDATE api_key_rotation_log
SET
  rotated_at = NOW(),
  status = 'rotated'
WHERE service_name = 'Stripe' AND status = 'active';
```

### Rotation Dashboard Workflow

```
Schedule: Daily
    ↓
Database Query: Get all keys
    ↓
Function: Calculate days until rotation
    ↓
Filter: Keys expiring < 7 days
    ↓
Slack: Alert team
    ↓
Google Sheets: Update dashboard
```

## Automation Best Practices

### 1. Test Before Revocation

```javascript
// Always test new key before revoking old

async function rotateKey() {
  // Generate new
  const newKey = await generateNewKey();

  // Test new key
  try {
    await testKeyWorks(newKey);
  } catch (error) {
    console.error('New key failed test!', error);
    // Don't proceed with rotation
    return;
  }

  // Update systems
  await updateN8n(newKey);
  await updateSecretsManager(newKey);

  // Wait grace period before revoking old
  scheduleRevocation(oldKey, 7 * 24 * 60 * 60 * 1000);
}
```

### 2. Grace Period

```javascript
// Keep both keys active for grace period

const GRACE_PERIOD_DAYS = 7;

// After updating to new key
setTimeout(() => {
  revokeOldKey();
}, GRACE_PERIOD_DAYS * 24 * 60 * 60 * 1000);
```

### 3. Rollback Plan

```javascript
// Keep old key info for rollback

const rotation = {
  oldKey: process.env.OLD_KEY,
  newKey: newlyGeneratedKey,
  rotatedAt: new Date(),
  serviceName: 'Stripe'
};

// Store rotation details
await db.saveRotationRecord(rotation);

// Rollback function
async function rollbackRotation(rotationId) {
  const rotation = await db.getRotation(rotationId);

  // Restore old key
  await updateN8n(rotation.oldKey);

  // Revoke new key
  await revokeKey(rotation.newKey);

  console.log('Rolled back to old key');
}
```

## Monitoring Key Usage

### Usage Tracking Workflow

```
Webhook: API Call Made
    ↓
Function: Extract key ID from request
    ↓
Database: Log usage
    ↓
IF (Old key still being used)
    ↓
    Slack: Alert team
    "Old key XYZ still in use!"
```

### Anomaly Detection

```javascript
// Detect unusual key usage patterns

const usage = await getKeyUsage(keyId, last24Hours);

const anomalies = {
  unusualVolume: usage.count > average * 3,
  unusualTime: usage.outsideBusinessHours,
  unusualLocation: usage.fromNewIP,
  afterRotation: usage.oldKeyAfterGracePeriod
};

if (Object.values(anomalies).some(x => x)) {
  await alertSecurityTeam(keyId, anomalies);
}
```

## Emergency Rotation

### Immediate Revocation Process

```bash
#!/bin/bash
# emergency-revoke.sh

SERVICE=$1
KEY_ID=$2

echo "EMERGENCY KEY REVOCATION"
echo "Service: $SERVICE"
echo "Key: $KEY_ID"
echo ""

read -p "Are you sure? (yes/no): " CONFIRM

if [ "$CONFIRM" != "yes" ]; then
  echo "Aborted"
  exit 1
fi

# 1. Immediately revoke key
case $SERVICE in
  "aws")
    aws iam delete-access-key --access-key-id $KEY_ID
    ;;
  "stripe")
    curl -X DELETE https://api.stripe.com/v1/api_keys/$KEY_ID \
      -u $STRIPE_SECRET_KEY:
    ;;
  *)
    echo "Unknown service"
    exit 1
    ;;
esac

# 2. Generate new key
echo "Generating new key..."
# Service-specific generation

# 3. Update systems
echo "Updating n8n..."
# Update via API

# 4. Notify team
echo "Notifying team..."
curl -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer $SLACK_TOKEN" \
  -d "channel=#security" \
  -d "text=EMERGENCY: $SERVICE key $KEY_ID revoked and rotated"

# 5. Log incident
echo "[$(date)] Emergency revocation: $SERVICE $KEY_ID" >> /var/log/security-incidents.log

echo "Emergency rotation complete!"
```

## Best Practices Checklist

### Planning
- [ ] Rotation schedule defined per service
- [ ] Grace period established (7-30 days)
- [ ] Rollback procedures documented
- [ ] Team trained on rotation process
- [ ] Emergency revocation procedure ready

### Implementation
- [ ] Generate new key first
- [ ] Test new key thoroughly
- [ ] Update all systems with new key
- [ ] Keep old key active during grace period
- [ ] Monitor usage of both keys
- [ ] Revoke old key after grace period

### Automation
- [ ] Automated rotation workflows created
- [ ] Rotation tracking database setup
- [ ] Alerts for upcoming rotations
- [ ] Usage monitoring implemented
- [ ] Anomaly detection configured

### Documentation
- [ ] Rotation procedures documented per service
- [ ] API documentation referenced
- [ ] Troubleshooting guide created
- [ ] Team contacts listed
- [ ] Incident response plan ready

## Common Pitfalls

### ❌ Don't Do This

1. **Immediate revocation without grace period**
   - Can break running workflows
   - No rollback option

2. **No testing of new key**
   - May not have necessary permissions
   - Could be malformed

3. **Forgetting to update all locations**
   - CI/CD systems
   - Backup scripts
   - Monitoring tools

4. **No tracking**
   - Don't know when last rotated
   - Can't prove compliance

## Quick Reference

### Rotation Checklist

```
□ Generate new key
□ Test new key
□ Update n8n credential
□ Update secrets manager
□ Update other systems
□ Set grace period timer
□ Monitor usage
□ Verify no old key usage
□ Revoke old key
□ Update rotation log
□ Notify team
```

### Rotation Commands

```bash
# AWS
aws iam create-access-key --user-name n8n
aws iam delete-access-key --access-key-id OLD_KEY

# Stripe (via API)
curl https://api.stripe.com/v1/api_keys -u sk_live_KEY:

# GitHub
gh auth login
gh auth token

# GCP
gcloud iam service-accounts keys create new-key.json
gcloud iam service-accounts keys delete OLD_KEY_ID
```

## Next Steps

After implementing API key rotation:
1. Review data privacy practices (see `04-data-privacy-logging.md`)
2. Understand compliance requirements (see `05-compliance-patterns.md`)
3. Conduct security audit (see `06-security-audit-procedures.md`)
4. Document rotation procedures for team
5. Schedule first rotation for all services

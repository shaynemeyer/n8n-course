# Environment Variable Management

## Introduction

Environment variables configure n8n and store sensitive information. Proper management is crucial for security, maintainability, and operational efficiency. This guide covers secure practices for managing environment variables and secrets.

## Understanding Environment Variables

### What Are Environment Variables?

Environment variables are key-value pairs that configure application behavior:

```bash
# Format
KEY=value

# Examples
DB_TYPE=postgresdb
N8N_HOST=n8n.example.com
N8N_ENCRYPTION_KEY=abc123...
```

### Why They Matter

- **Security**: Store secrets outside code
- **Flexibility**: Different configs per environment
- **Portability**: Same code, different settings
- **Compliance**: Separate code from configuration

### Types of Environment Variables

#### 1. Public Configuration
```bash
# Safe to log/display
N8N_HOST=n8n.example.com
N8N_PORT=5678
GENERIC_TIMEZONE=America/New_York
```

#### 2. Sensitive Secrets
```bash
# NEVER log/display
DB_POSTGRESDB_PASSWORD=secret123
N8N_ENCRYPTION_KEY=key456
SMTP_PASSWORD=pass789
```

#### 3. Semi-Sensitive
```bash
# Careful with logging
DB_POSTGRESDB_HOST=db.internal
DB_POSTGRESDB_USER=n8n_user
```

## Insecure Practices (Don't Do This!)

### ❌ Hardcoding in Code

```javascript
// NEVER DO THIS
const apiKey = "sk_live_1234567890";
const dbPassword = "MySecretPassword123";
```

### ❌ Committing to Git

```bash
# .env file committed to repository
git add .env
git commit -m "Add configuration"
git push  # ← Secrets now in git history forever!
```

### ❌ Logging Secrets

```javascript
// NEVER DO THIS
console.log("Database password:", process.env.DB_PASSWORD);
```

### ❌ Exposing in Error Messages

```javascript
// NEVER DO THIS
throw new Error(`Failed to connect with password ${dbPassword}`);
```

### ❌ Sharing via Insecure Channels

```
Email: "Hey, here's the production database password: MyPass123"
Slack: "The API key is: sk_live_..."
```

## Secure Practices

### ✅ Using .env Files

#### Create .env File

```bash
# /opt/n8n/.env

# Database
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=postgres
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n
DB_POSTGRESDB_PASSWORD=generate_strong_password_here

# Encryption
N8N_ENCRYPTION_KEY=generate_32_byte_key_here

# URLs
N8N_HOST=n8n.example.com
N8N_PROTOCOL=https
WEBHOOK_URL=https://n8n.example.com/

# SMTP
N8N_EMAIL_MODE=smtp
N8N_SMTP_HOST=smtp.gmail.com
N8N_SMTP_PORT=587
N8N_SMTP_USER=notifications@example.com
N8N_SMTP_PASS=app_specific_password_here
```

#### Secure the File

```bash
# Restrict permissions
chmod 600 /opt/n8n/.env
chown n8n:n8n /opt/n8n/.env

# Verify
ls -la /opt/n8n/.env
# Should show: -rw------- 1 n8n n8n
```

#### Prevent Git Commits

```bash
# .gitignore
echo ".env" >> .gitignore
echo ".env.*" >> .gitignore
echo "!.env.example" >> .gitignore
```

### ✅ Environment-Specific Files

```bash
# Different files per environment
.env.development
.env.staging
.env.production

# Load appropriate file
docker compose --env-file .env.production up -d
```

### ✅ .env.example Template

Create template without secrets:

```bash
# .env.example (safe to commit)

# Database
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=postgres
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n
DB_POSTGRESDB_PASSWORD=CHANGE_ME

# Encryption
N8N_ENCRYPTION_KEY=GENERATE_32_BYTE_KEY

# URLs
N8N_HOST=your-domain.com
N8N_PROTOCOL=https
WEBHOOK_URL=https://your-domain.com/

# SMTP
N8N_EMAIL_MODE=smtp
N8N_SMTP_HOST=smtp.example.com
N8N_SMTP_PORT=587
N8N_SMTP_USER=your-email@example.com
N8N_SMTP_PASS=CHANGE_ME
```

Team members can copy and fill in:
```bash
cp .env.example .env
nano .env  # Fill in actual values
```

## Secrets Management Tools

### HashiCorp Vault

#### Installation

```bash
# Docker
docker run -d --name vault \
  --cap-add=IPC_LOCK \
  -p 8200:8200 \
  -e VAULT_ADDR='http://0.0.0.0:8200' \
  -e VAULT_DEV_ROOT_TOKEN_ID='myroot' \
  vault:latest

# Production: Use proper init/unseal process
```

#### Store Secrets

```bash
# Set Vault address
export VAULT_ADDR='http://127.0.0.1:8200'

# Login
vault login myroot

# Enable KV secrets engine
vault secrets enable -path=secret kv-v2

# Store n8n secrets
vault kv put secret/n8n \
  db_password="SecurePass123" \
  encryption_key="abc123def456..." \
  smtp_password="EmailPass456"
```

#### Retrieve Secrets

```bash
# Get all secrets
vault kv get secret/n8n

# Get specific field
vault kv get -field=db_password secret/n8n
```

#### Integration with n8n

Create startup script:

```bash
#!/bin/bash
# start-n8n-with-vault.sh

# Fetch secrets from Vault
export DB_POSTGRESDB_PASSWORD=$(vault kv get -field=db_password secret/n8n)
export N8N_ENCRYPTION_KEY=$(vault kv get -field=encryption_key secret/n8n)
export N8N_SMTP_PASS=$(vault kv get -field=smtp_password secret/n8n)

# Start n8n
docker compose up -d
```

#### Dynamic Secrets

```bash
# Configure database secrets engine
vault secrets enable database

vault write database/config/postgresql \
  plugin_name=postgresql-database-plugin \
  allowed_roles="n8n-role" \
  connection_url="postgresql://{{username}}:{{password}}@postgres:5432/n8n" \
  username="vault" \
  password="vaultpass"

# Create role
vault write database/roles/n8n-role \
  db_name=postgresql \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';" \
  default_ttl="1h" \
  max_ttl="24h"

# Generate credentials (auto-rotate)
vault read database/creds/n8n-role
```

### AWS Secrets Manager

#### Store Secret

```bash
# Install AWS CLI
apt-get install awscli

# Configure
aws configure

# Create secret
aws secretsmanager create-secret \
  --name n8n/production \
  --description "n8n production secrets" \
  --secret-string '{
    "db_password": "SecurePass123",
    "encryption_key": "abc123def456...",
    "smtp_password": "EmailPass456"
  }'
```

#### Retrieve Secret

```bash
# Get secret
aws secretsmanager get-secret-value \
  --secret-id n8n/production \
  --query SecretString \
  --output text | jq -r '.db_password'
```

#### Integration Script

```bash
#!/bin/bash
# start-n8n-with-aws-secrets.sh

# Fetch secrets
SECRET=$(aws secretsmanager get-secret-value \
  --secret-id n8n/production \
  --query SecretString \
  --output text)

# Parse and export
export DB_POSTGRESDB_PASSWORD=$(echo $SECRET | jq -r '.db_password')
export N8N_ENCRYPTION_KEY=$(echo $SECRET | jq -r '.encryption_key')
export N8N_SMTP_PASS=$(echo $SECRET | jq -r '.smtp_password')

# Start n8n
docker compose up -d
```

#### Automatic Rotation

```python
# lambda_rotation.py
import boto3
import json

def lambda_handler(event, context):
    service_client = boto3.client('secretsmanager')

    # Get current secret
    current = service_client.get_secret_value(SecretId=event['SecretId'])
    current_dict = json.loads(current['SecretString'])

    # Generate new password
    new_password = generate_strong_password()

    # Update in database
    update_database_password(new_password)

    # Update secret
    current_dict['db_password'] = new_password
    service_client.put_secret_value(
        SecretId=event['SecretId'],
        SecretString=json.dumps(current_dict)
    )

    return {'statusCode': 200}
```

### Azure Key Vault

#### Create Key Vault

```bash
# Create resource group
az group create --name n8n-rg --location eastus

# Create key vault
az keyvault create \
  --name n8n-keyvault \
  --resource-group n8n-rg \
  --location eastus
```

#### Store Secrets

```bash
# Store individual secrets
az keyvault secret set \
  --vault-name n8n-keyvault \
  --name db-password \
  --value "SecurePass123"

az keyvault secret set \
  --vault-name n8n-keyvault \
  --name encryption-key \
  --value "abc123def456..."
```

#### Retrieve Secrets

```bash
# Get secret
az keyvault secret show \
  --vault-name n8n-keyvault \
  --name db-password \
  --query value -o tsv
```

#### Managed Identity Integration

```bash
# Assign managed identity to VM
az vm identity assign \
  --resource-group n8n-rg \
  --name n8n-vm

# Grant access to key vault
az keyvault set-policy \
  --name n8n-keyvault \
  --object-id <managed-identity-id> \
  --secret-permissions get list
```

## Docker Secrets

### Docker Swarm Secrets

#### Create Secrets

```bash
# From file
echo "SecurePass123" | docker secret create db_password -

# From stdin
docker secret create encryption_key /path/to/key/file
```

#### Use in Docker Compose

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    secrets:
      - db_password
      - encryption_key
    environment:
      - DB_POSTGRESDB_PASSWORD_FILE=/run/secrets/db_password
      - N8N_ENCRYPTION_KEY_FILE=/run/secrets/encryption_key

secrets:
  db_password:
    external: true
  encryption_key:
    external: true
```

#### Access in Application

```javascript
// n8n automatically reads _FILE variables
// Or read manually:
const fs = require('fs');
const dbPassword = fs.readFileSync('/run/secrets/db_password', 'utf8').trim();
```

### Kubernetes Secrets

#### Create Secret

```bash
# From literal
kubectl create secret generic n8n-secrets \
  --from-literal=db-password=SecurePass123 \
  --from-literal=encryption-key=abc123...

# From file
kubectl create secret generic n8n-secrets \
  --from-file=db-password=./db-pass.txt \
  --from-file=encryption-key=./enc-key.txt
```

#### Use in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: n8n
spec:
  containers:
  - name: n8n
    image: n8nio/n8n:latest
    env:
    - name: DB_POSTGRESDB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: n8n-secrets
          key: db-password
    - name: N8N_ENCRYPTION_KEY
      valueFrom:
        secretKeyRef:
          name: n8n-secrets
          key: encryption-key
```

#### Mount as Files

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: n8n
spec:
  containers:
  - name: n8n
    image: n8nio/n8n:latest
    volumeMounts:
    - name: secrets
      mountPath: "/etc/secrets"
      readOnly: true
  volumes:
  - name: secrets
    secret:
      secretName: n8n-secrets
```

## Secret Rotation

### Rotation Strategy

#### 1. Manual Rotation

```bash
#!/bin/bash
# rotate-secrets.sh

echo "Secret Rotation Process"
echo "======================="

# 1. Generate new secret
NEW_PASSWORD=$(openssl rand -base64 32)
echo "New password generated"

# 2. Update in external service (database, API, etc.)
echo "Updating external service..."
# Service-specific update command

# 3. Update in secrets manager
echo "Updating secrets manager..."
vault kv put secret/n8n db_password="$NEW_PASSWORD"

# 4. Restart n8n to pick up new secret
echo "Restarting n8n..."
docker compose restart n8n

# 5. Verify
echo "Verifying..."
# Test connection

echo "Rotation complete!"
```

#### 2. Automated Rotation

```yaml
# n8n workflow for automated rotation
name: Automated Secret Rotation

trigger:
  schedule: "0 0 1 * *"  # Monthly

steps:
  - name: Generate new password
    type: Function
    code: |
      const crypto = require('crypto');
      return {
        newPassword: crypto.randomBytes(32).toString('base64')
      };

  - name: Update database
    type: PostgreSQL
    query: |
      ALTER USER n8n WITH PASSWORD '{{newPassword}}';

  - name: Update Vault
    type: HTTP Request
    url: http://vault:8200/v1/secret/data/n8n
    method: POST
    body:
      data:
        db_password: "{{newPassword}}"

  - name: Restart n8n
    type: Execute Command
    command: docker compose restart n8n

  - name: Notify team
    type: Slack
    message: "Database password rotated successfully"
```

### Zero-Downtime Rotation

#### Database Password Rotation

```bash
#!/bin/bash
# zero-downtime-db-rotation.sh

# 1. Create temporary user with new password
NEW_PASS=$(openssl rand -base64 32)
psql -U postgres -c "CREATE USER n8n_temp WITH PASSWORD '$NEW_PASS';"
psql -U postgres -c "GRANT ALL PRIVILEGES ON DATABASE n8n TO n8n_temp;"

# 2. Update n8n to use new user
export DB_POSTGRESDB_USER=n8n_temp
export DB_POSTGRESDB_PASSWORD=$NEW_PASS
docker compose up -d

# 3. Wait for n8n to reconnect
sleep 30

# 4. Remove old user
psql -U postgres -c "DROP USER n8n;"

# 5. Rename temp user to permanent
psql -U postgres -c "ALTER USER n8n_temp RENAME TO n8n;"

# 6. Update environment with permanent name
export DB_POSTGRESDB_USER=n8n
docker compose restart n8n

echo "Zero-downtime rotation complete!"
```

## Environment Variable Validation

### Validation Script

```bash
#!/bin/bash
# validate-env.sh

ERRORS=0

# Required variables
REQUIRED_VARS=(
  "DB_TYPE"
  "DB_POSTGRESDB_HOST"
  "DB_POSTGRESDB_PASSWORD"
  "N8N_ENCRYPTION_KEY"
  "N8N_HOST"
)

echo "Validating environment variables..."
echo "==================================="

# Check required variables exist
for var in "${REQUIRED_VARS[@]}"; do
  if [ -z "${!var}" ]; then
    echo "❌ ERROR: $var is not set"
    ((ERRORS++))
  else
    echo "✓ $var is set"
  fi
done

# Validate encryption key length
if [ ${#N8N_ENCRYPTION_KEY} -lt 32 ]; then
  echo "❌ ERROR: N8N_ENCRYPTION_KEY must be at least 32 characters"
  ((ERRORS++))
fi

# Validate URLs
if [[ ! $N8N_HOST =~ ^[a-zA-Z0-9.-]+$ ]]; then
  echo "❌ ERROR: N8N_HOST contains invalid characters"
  ((ERRORS++))
fi

# Check for common mistakes
if [ "$DB_POSTGRESDB_PASSWORD" == "CHANGE_ME" ]; then
  echo "❌ ERROR: Database password not changed from default"
  ((ERRORS++))
fi

echo "==================================="
if [ $ERRORS -eq 0 ]; then
  echo "✓ All validations passed"
  exit 0
else
  echo "❌ $ERRORS error(s) found"
  exit 1
fi
```

Run before starting:
```bash
source .env
./validate-env.sh && docker compose up -d
```

## Monitoring and Auditing

### Track Secret Access

```bash
# Log when secrets are accessed
# Add to .bashrc or startup script

log_secret_access() {
  echo "[$(date)] Secret accessed: $1 by $(whoami) from $(hostname)" >> /var/log/secret-access.log
}

# Wrapper for reading secrets
get_secret() {
  local SECRET_NAME=$1
  log_secret_access $SECRET_NAME
  vault kv get -field=$SECRET_NAME secret/n8n
}
```

### Audit Log Review

Create n8n workflow:

```
Schedule (Weekly)
  ↓
Execute Command (grep /var/log/secret-access.log)
  ↓
Function (Parse and analyze)
  ↓
IF (Unusual access patterns)
  ↓
  Email security team
```

## Best Practices Checklist

### Storage
- [ ] Secrets in .env file, not code
- [ ] .env file has 600 permissions
- [ ] .env in .gitignore
- [ ] .env.example provided for team
- [ ] Different secrets per environment

### Secrets Management
- [ ] Using secrets manager (Vault, AWS, Azure)
- [ ] Secrets encrypted at rest
- [ ] Access to secrets logged
- [ ] Secrets rotated regularly
- [ ] Emergency rotation procedure documented

### Access Control
- [ ] Least privilege access to secrets
- [ ] Secrets access audited
- [ ] Team members have individual credentials
- [ ] Service accounts for automation
- [ ] MFA required for secrets access

### Monitoring
- [ ] Secret access logged
- [ ] Alerts for unusual access
- [ ] Regular audit log reviews
- [ ] Failed access attempts monitored
- [ ] Secrets expiration tracked

### Documentation
- [ ] Secrets inventory maintained
- [ ] Rotation procedures documented
- [ ] Emergency response plan
- [ ] Team training completed
- [ ] Compliance requirements met

## Troubleshooting

### Secrets Not Loading

```bash
# Check environment variables are set
docker compose exec n8n env | grep -i password

# Check file permissions
ls -la .env

# Verify docker-compose.yml references .env
grep env_file docker-compose.yml

# Check for syntax errors in .env
cat -A .env  # Shows hidden characters
```

### Rotation Failed

```bash
# Check both old and new secrets work
# Test old
echo "SELECT 1" | psql -h postgres -U n8n -d n8n

# Test new
export PGPASSWORD=new_password
echo "SELECT 1" | psql -h postgres -U n8n -d n8n

# Rollback if needed
vault kv put secret/n8n db_password="old_password"
docker compose restart n8n
```

## Quick Reference

### Generate Strong Secrets
```bash
# 32-byte random
openssl rand -base64 32

# UUID
uuidgen

# Custom length
openssl rand -base64 64 | tr -d "=+/" | cut -c1-50
```

### Secure .env File
```bash
chmod 600 .env
chown n8n:n8n .env
```

### Check Secret Age
```bash
stat -c %y .env  # Last modified
```

### Emergency Rotation
```bash
./emergency-rotate.sh <secret-name>
```

## Next Steps

After mastering environment variable management:
1. Implement API key rotation (see `03-api-key-rotation.md`)
2. Set up data privacy controls (see `04-data-privacy-logging.md`)
3. Review compliance requirements (see `05-compliance-patterns.md`)
4. Conduct security audit (see `06-security-audit-procedures.md`)

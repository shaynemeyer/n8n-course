# Environment Configuration

## Introduction

Environment variables are the primary way to configure n8n in production. Proper configuration is crucial for security, performance, and functionality. This guide covers essential environment variables and best practices.

## Environment Variable Categories

### 1. Database Configuration
### 2. Security and Authentication
### 3. URLs and Networking
### 4. Performance and Execution
### 5. Email and Notifications
### 6. Timezone and Localization
### 7. Queue Mode and Scaling
### 8. Logging and Debugging

## Essential Environment Variables

### Database Configuration

#### PostgreSQL (Recommended)

```bash
# Database Type
DB_TYPE=postgresdb

# Connection Details
DB_POSTGRESDB_HOST=postgres
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n
DB_POSTGRESDB_PASSWORD=your_secure_password

# Connection Pool
DB_POSTGRESDB_POOL_SIZE=2
```

#### MySQL/MariaDB (Alternative)

```bash
# Database Type
DB_TYPE=mysqldb

# Connection Details
DB_MYSQLDB_HOST=mysql
DB_MYSQLDB_PORT=3306
DB_MYSQLDB_DATABASE=n8n
DB_MYSQLDB_USER=n8n
DB_MYSQLDB_PASSWORD=your_secure_password
```

#### SQLite (Development Only)

```bash
# NOT recommended for production
DB_TYPE=sqlite
DB_SQLITE_DATABASE=/home/node/.n8n/database.sqlite
```

### Security and Authentication

#### Basic Authentication (Simple Setup)

```bash
# Enable basic auth
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=secure_password_here

# Password hashing
N8N_USER_MANAGEMENT_DISABLED=false
```

#### Advanced User Management

```bash
# Disable basic auth, use user management
N8N_BASIC_AUTH_ACTIVE=false
N8N_USER_MANAGEMENT_DISABLED=false

# JWT Configuration
N8N_JWT_AUTH_ACTIVE=true
N8N_JWT_AUTH_HEADER=Authorization

# Encryption Key (CRITICAL - back this up!)
N8N_ENCRYPTION_KEY=your_encryption_key_here
```

#### Generate Encryption Key

```bash
# Generate a secure encryption key
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"

# Or use openssl
openssl rand -base64 32
```

**IMPORTANT:**
- Never lose your encryption key - credentials cannot be recovered without it
- Back up this key securely
- Use the same key across all n8n instances in a cluster

### URLs and Networking

```bash
# Host Configuration
N8N_HOST=n8n.yourdomain.com
N8N_PORT=5678
N8N_PROTOCOL=https

# Webhook URLs
WEBHOOK_URL=https://n8n.yourdomain.com/

# Editor URL (if different from main URL)
N8N_EDITOR_BASE_URL=https://n8n.yourdomain.com

# Path prefix (if running behind a path-based proxy)
N8N_PATH=/n8n/
```

### Performance and Execution

#### Execution Settings

```bash
# Execution Mode (regular or queue)
EXECUTIONS_MODE=regular

# Execution Timeout (seconds)
EXECUTIONS_TIMEOUT=300
EXECUTIONS_TIMEOUT_MAX=3600

# Payload Size Limit (MB)
N8N_PAYLOAD_SIZE_MAX=16

# Manual Execution Timeout
EXECUTIONS_TIMEOUT_MANUAL=1800
```

#### Data Pruning

```bash
# Enable automatic pruning
EXECUTIONS_DATA_PRUNE=true

# Keep execution data for 7 days
EXECUTIONS_DATA_MAX_AGE=168

# Prune data this often (hours)
EXECUTIONS_DATA_PRUNE_TIMEOUT=1
```

#### Memory Management

```bash
# Node.js memory limit (MB)
NODE_OPTIONS=--max-old-space-size=2048

# Execution data saving
EXECUTIONS_DATA_SAVE_ON_ERROR=all
EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=true
```

### Email Configuration

#### SMTP Settings

```bash
# SMTP Host
N8N_EMAIL_MODE=smtp
N8N_SMTP_HOST=smtp.gmail.com
N8N_SMTP_PORT=587
N8N_SMTP_USER=your-email@gmail.com
N8N_SMTP_PASS=your-app-password

# TLS/SSL
N8N_SMTP_SSL=false
N8N_SMTP_SENDER=n8n@yourdomain.com
```

#### Email Templates

```bash
# Custom email templates
N8N_EMAIL_TEMPLATES_DIR=/home/node/.n8n/email-templates
```

### Timezone and Localization

```bash
# Generic Timezone
GENERIC_TIMEZONE=America/New_York

# Workflows Timezone
WORKFLOWS_DEFAULT_NAME=My workflow
DEFAULT_LOCALE=en
```

Common Timezones:
- America/New_York (EST/EDT)
- America/Chicago (CST/CDT)
- America/Los_Angeles (PST/PDT)
- Europe/London (GMT/BST)
- Europe/Paris (CET/CEST)
- Asia/Tokyo (JST)
- Australia/Sydney (AEST)

### Queue Mode Configuration

```bash
# Enable Queue Mode
EXECUTIONS_MODE=queue

# Redis Configuration
QUEUE_BULL_REDIS_HOST=redis
QUEUE_BULL_REDIS_PORT=6379
QUEUE_BULL_REDIS_PASSWORD=redis_password
QUEUE_BULL_REDIS_DB=0

# Queue Settings
QUEUE_BULL_REDIS_TIMEOUT_THRESHOLD=10000
QUEUE_RECOVERY_INTERVAL=60
```

### Logging and Debugging

```bash
# Log Level
N8N_LOG_LEVEL=info
# Options: error, warn, info, verbose, debug

# Log Output
N8N_LOG_OUTPUT=console
# Options: console, file

# Log File Location
N8N_LOG_FILE_LOCATION=/home/node/.n8n/logs/
N8N_LOG_FILE_COUNT_MAX=100
N8N_LOG_FILE_SIZE_MAX=16
```

## Advanced Configuration

### Custom Nodes

```bash
# Custom Node Path
N8N_CUSTOM_EXTENSIONS=/home/node/.n8n/custom

# Community Nodes
N8N_COMMUNITY_PACKAGES_ENABLED=true
```

### Workflow Settings

```bash
# Default Workflow Name
WORKFLOWS_DEFAULT_NAME=My Workflow

# Enable Workflow Activation
N8N_SKIP_WEBHOOK_DEREGISTRATION_SHUTDOWN=true
```

### External Hooks

```bash
# External Hook Files
EXTERNAL_HOOK_FILES=/home/node/.n8n/hooks/hook.js
```

### Credentials

```bash
# Credentials Override
CREDENTIALS_OVERWRITE_DATA={}
CREDENTIALS_OVERWRITE_ENDPOINT=
```

## Complete Production Configuration Example

### .env File

```bash
# ================================
# Database Configuration
# ================================
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=postgres
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n
DB_POSTGRESDB_PASSWORD=change_me_postgres_password
DB_POSTGRESDB_POOL_SIZE=4

# ================================
# Encryption & Security
# ================================
N8N_ENCRYPTION_KEY=change_me_encryption_key_32_chars

# ================================
# Authentication
# ================================
N8N_BASIC_AUTH_ACTIVE=false
N8N_USER_MANAGEMENT_DISABLED=false

# ================================
# URLs & Networking
# ================================
N8N_HOST=n8n.yourdomain.com
N8N_PORT=5678
N8N_PROTOCOL=https
WEBHOOK_URL=https://n8n.yourdomain.com/

# ================================
# Queue Mode (Redis)
# ================================
EXECUTIONS_MODE=queue
QUEUE_BULL_REDIS_HOST=redis
QUEUE_BULL_REDIS_PORT=6379
QUEUE_BULL_REDIS_PASSWORD=change_me_redis_password
QUEUE_BULL_REDIS_DB=0

# ================================
# Performance & Execution
# ================================
EXECUTIONS_TIMEOUT=300
EXECUTIONS_TIMEOUT_MAX=3600
N8N_PAYLOAD_SIZE_MAX=16
NODE_OPTIONS=--max-old-space-size=2048

# ================================
# Data Retention
# ================================
EXECUTIONS_DATA_PRUNE=true
EXECUTIONS_DATA_MAX_AGE=168
EXECUTIONS_DATA_PRUNE_TIMEOUT=1
EXECUTIONS_DATA_SAVE_ON_ERROR=all
EXECUTIONS_DATA_SAVE_ON_SUCCESS=all

# ================================
# Email (SMTP)
# ================================
N8N_EMAIL_MODE=smtp
N8N_SMTP_HOST=smtp.gmail.com
N8N_SMTP_PORT=587
N8N_SMTP_USER=your-email@gmail.com
N8N_SMTP_PASS=your-app-password
N8N_SMTP_SENDER=n8n@yourdomain.com
N8N_SMTP_SSL=false

# ================================
# Timezone
# ================================
GENERIC_TIMEZONE=America/New_York

# ================================
# Logging
# ================================
N8N_LOG_LEVEL=info
N8N_LOG_OUTPUT=console

# ================================
# Features
# ================================
N8N_COMMUNITY_PACKAGES_ENABLED=true
N8N_HIRING_BANNER_ENABLED=false
N8N_PERSONALIZATION_ENABLED=true
```

## Docker Compose Integration

### Using Environment File

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    env_file:
      - .env
    volumes:
      - n8n-data:/home/node/.n8n
```

### Inline Environment Variables

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    environment:
      - DB_TYPE=${DB_TYPE}
      - DB_POSTGRESDB_HOST=${DB_POSTGRESDB_HOST}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
```

## Security Best Practices

### 1. Never Commit Secrets

```bash
# Add to .gitignore
echo ".env" >> .gitignore
echo ".env.local" >> .gitignore
echo ".env.*.local" >> .gitignore
```

### 2. Use Strong Passwords

```bash
# Generate strong passwords
openssl rand -base64 32

# For encryption key specifically
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

### 3. Restrict File Permissions

```bash
# Protect .env file
chmod 600 .env

# Owned by appropriate user
chown 1000:1000 .env
```

### 4. Use Secret Management

For production, consider using:
- Docker Secrets
- Kubernetes Secrets
- AWS Secrets Manager
- HashiCorp Vault
- Azure Key Vault

#### Docker Secrets Example

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
    file: ./secrets/db_password.txt
  encryption_key:
    file: ./secrets/encryption_key.txt
```

## Environment Variable Validation

### Startup Validation Script

Create `validate-env.sh`:

```bash
#!/bin/bash

# Required variables
REQUIRED_VARS=(
  "DB_TYPE"
  "N8N_ENCRYPTION_KEY"
  "N8N_HOST"
  "N8N_PROTOCOL"
)

# Check each required variable
for var in "${REQUIRED_VARS[@]}"; do
  if [ -z "${!var}" ]; then
    echo "ERROR: $var is not set"
    exit 1
  fi
done

# Validate encryption key length
if [ ${#N8N_ENCRYPTION_KEY} -lt 32 ]; then
  echo "ERROR: N8N_ENCRYPTION_KEY must be at least 32 characters"
  exit 1
fi

echo "Environment validation passed"
```

Run before starting n8n:
```bash
./validate-env.sh && docker compose up -d
```

## Troubleshooting

### Common Issues

#### 1. Database Connection Failed

```bash
# Check database variables
echo $DB_TYPE
echo $DB_POSTGRESDB_HOST
echo $DB_POSTGRESDB_PORT

# Test database connectivity
docker compose exec postgres psql -U $DB_POSTGRESDB_USER -d $DB_POSTGRESDB_DATABASE
```

#### 2. Encryption Key Issues

```bash
# Verify key is set
docker compose exec n8n env | grep N8N_ENCRYPTION_KEY

# Key length check (should be base64 encoded, ~44 chars)
echo -n $N8N_ENCRYPTION_KEY | wc -c
```

#### 3. Webhook Not Working

```bash
# Verify webhook URL
echo $WEBHOOK_URL
echo $N8N_HOST
echo $N8N_PROTOCOL

# Should match: https://$N8N_HOST/
```

### Debug Mode

```bash
# Enable verbose logging
N8N_LOG_LEVEL=debug

# Check logs
docker compose logs -f n8n
```

## Performance Tuning

### Memory-Intensive Workflows

```bash
# Increase Node.js heap size
NODE_OPTIONS=--max-old-space-size=4096

# Increase payload limit
N8N_PAYLOAD_SIZE_MAX=32
```

### High-Volume Workflows

```bash
# Enable queue mode
EXECUTIONS_MODE=queue

# Increase connection pool
DB_POSTGRESDB_POOL_SIZE=10

# Aggressive data pruning
EXECUTIONS_DATA_MAX_AGE=24
EXECUTIONS_DATA_PRUNE_TIMEOUT=1
```

### Low-Resource Environments

```bash
# Reduce memory
NODE_OPTIONS=--max-old-space-size=1024

# Smaller connection pool
DB_POSTGRESDB_POOL_SIZE=2

# Shorter data retention
EXECUTIONS_DATA_MAX_AGE=48
```

## Environment Variable Reference

### Quick Reference Table

| Category | Variable | Required | Default | Description |
|----------|----------|----------|---------|-------------|
| Database | DB_TYPE | Yes | - | Database type |
| Database | DB_POSTGRESDB_HOST | Yes* | - | PostgreSQL host |
| Security | N8N_ENCRYPTION_KEY | Yes | - | Encryption key |
| Security | N8N_BASIC_AUTH_ACTIVE | No | false | Enable basic auth |
| URLs | N8N_HOST | Yes | - | Public hostname |
| URLs | N8N_PROTOCOL | Yes | https | Protocol |
| URLs | WEBHOOK_URL | Yes | - | Webhook base URL |
| Queue | EXECUTIONS_MODE | No | regular | Execution mode |
| Queue | QUEUE_BULL_REDIS_HOST | No** | - | Redis host |
| Performance | NODE_OPTIONS | No | - | Node.js options |
| Performance | EXECUTIONS_TIMEOUT | No | 300 | Default timeout |

\* Required if using PostgreSQL
\** Required if using queue mode

## Checklists

### Production Readiness Checklist

- [ ] Database configured (PostgreSQL)
- [ ] Encryption key generated and backed up
- [ ] User management enabled
- [ ] Webhook URL configured correctly
- [ ] SMTP email configured
- [ ] Queue mode enabled (for scaling)
- [ ] Data pruning configured
- [ ] Logging configured
- [ ] Timezone set correctly
- [ ] Environment file secured (600 permissions)
- [ ] Environment file excluded from git
- [ ] All secrets use strong passwords
- [ ] SSL/TLS configured (see SSL guide)
- [ ] Backup strategy implemented

### Security Checklist

- [ ] Encryption key is 32+ characters
- [ ] Database password is strong
- [ ] Redis password is set (if using queue mode)
- [ ] .env file permissions set to 600
- [ ] .env file in .gitignore
- [ ] Basic auth disabled, user management enabled
- [ ] HTTPS enforced
- [ ] SMTP credentials secured
- [ ] No hardcoded secrets in docker-compose.yml
- [ ] Regular credential rotation scheduled

## Next Steps

After configuring environment variables:
1. Set up your database (see `03-database-setup.md`)
2. Configure SSL/TLS (see `04-ssl-tls-configuration.md`)
3. Set up user management (see `05-user-management-rbac.md`)
4. Enable queue mode for scaling (see `06-queue-mode-scaling.md`)

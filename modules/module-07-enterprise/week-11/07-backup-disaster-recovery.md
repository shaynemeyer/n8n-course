# Backup and Disaster Recovery

## Introduction

Comprehensive backup and disaster recovery planning is critical for production n8n deployments. This guide covers backup strategies, automation, testing, and recovery procedures.

## What Needs to Be Backed Up

### 1. Database (Critical)

**Contains:**
- Workflow definitions
- Execution history
- User accounts
- Credentials (encrypted)
- Settings

**Priority:** HIGHEST
**Frequency:** Daily (minimum)

### 2. n8n Data Directory

**Contains:**
- Encryption key
- Custom nodes
- Binary files
- Workflow files (if not in DB)
- SSL certificates (if stored here)

**Priority:** HIGH
**Frequency:** Daily

### 3. Configuration Files

**Contains:**
- docker-compose.yml
- Environment variables (.env)
- Nginx/reverse proxy configs
- SSL certificates
- Custom scripts

**Priority:** MEDIUM
**Frequency:** On change

### 4. Redis Data (Optional)

**Contains:**
- Queued jobs (transient)
- Session data (transient)

**Priority:** LOW
**Frequency:** Usually not needed

## Database Backup Strategies

### Strategy 1: Automated PostgreSQL Dump

#### Backup Script

Create `/scripts/backup-database.sh`:

```bash
#!/bin/bash

# ================================
# Configuration
# ================================
BACKUP_DIR="/backup/database"
CONTAINER_NAME="n8n-postgres"
DB_NAME="n8n"
DB_USER="n8n"
RETENTION_DAYS=30
S3_BUCKET="your-bucket/n8n-backups"

# ================================
# Setup
# ================================
DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="n8n-db-${DATE}.sql.gz"
LOG_FILE="/var/log/n8n-backup.log"

mkdir -p "$BACKUP_DIR"

echo "[$(date)] Starting backup..." | tee -a "$LOG_FILE"

# ================================
# Create Backup
# ================================
docker exec -t "$CONTAINER_NAME" pg_dump -U "$DB_USER" "$DB_NAME" | \
  gzip > "$BACKUP_DIR/$BACKUP_FILE"

if [ $? -eq 0 ]; then
    SIZE=$(du -h "$BACKUP_DIR/$BACKUP_FILE" | cut -f1)
    echo "[$(date)] Backup successful: $BACKUP_FILE ($SIZE)" | tee -a "$LOG_FILE"
else
    echo "[$(date)] ERROR: Backup failed!" | tee -a "$LOG_FILE"
    exit 1
fi

# ================================
# Upload to S3 (Optional)
# ================================
if command -v aws &> /dev/null; then
    echo "[$(date)] Uploading to S3..." | tee -a "$LOG_FILE"
    aws s3 cp "$BACKUP_DIR/$BACKUP_FILE" "s3://$S3_BUCKET/$BACKUP_FILE"

    if [ $? -eq 0 ]; then
        echo "[$(date)] Upload successful" | tee -a "$LOG_FILE"
    else
        echo "[$(date)] WARNING: Upload failed" | tee -a "$LOG_FILE"
    fi
fi

# ================================
# Cleanup Old Backups
# ================================
echo "[$(date)] Cleaning old backups..." | tee -a "$LOG_FILE"
find "$BACKUP_DIR" -name "n8n-db-*.sql.gz" -mtime +$RETENTION_DAYS -delete

# Count remaining backups
BACKUP_COUNT=$(ls -1 "$BACKUP_DIR"/n8n-db-*.sql.gz 2>/dev/null | wc -l)
echo "[$(date)] Backup complete. Total backups: $BACKUP_COUNT" | tee -a "$LOG_FILE"
```

Make executable:
```bash
chmod +x /scripts/backup-database.sh
```

### Strategy 2: Continuous Archiving (WAL)

For point-in-time recovery:

```yaml
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: n8n
    command: |
      postgres
      -c wal_level=replica
      -c archive_mode=on
      -c archive_command='test ! -f /backup/wal/%f && cp %p /backup/wal/%f'
      -c archive_timeout=300
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./backup/wal:/backup/wal
```

### Strategy 3: Using pg_basebackup

```bash
#!/bin/bash

# Full base backup
docker exec n8n-postgres pg_basebackup \
  -U n8n \
  -D /backup/base \
  -F tar \
  -z \
  -P

echo "Base backup completed at $(date)"
```

## n8n Data Directory Backup

### Backup Script

Create `/scripts/backup-n8n-data.sh`:

```bash
#!/bin/bash

# Configuration
BACKUP_DIR="/backup/n8n-data"
VOLUME_NAME="n8n_n8n-data"
DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="n8n-data-${DATE}.tar.gz"
RETENTION_DAYS=30

mkdir -p "$BACKUP_DIR"

echo "[$(date)] Backing up n8n data volume..."

# Backup using temporary container
docker run --rm \
  -v $VOLUME_NAME:/source:ro \
  -v $BACKUP_DIR:/backup \
  alpine tar czf /backup/$BACKUP_FILE -C /source .

if [ $? -eq 0 ]; then
    SIZE=$(du -h "$BACKUP_DIR/$BACKUP_FILE" | cut -f1)
    echo "[$(date)] Backup successful: $BACKUP_FILE ($SIZE)"

    # Cleanup old backups
    find "$BACKUP_DIR" -name "n8n-data-*.tar.gz" -mtime +$RETENTION_DAYS -delete
else
    echo "[$(date)] ERROR: Backup failed!"
    exit 1
fi
```

## Complete Backup Solution

### All-in-One Backup Script

Create `/scripts/backup-all.sh`:

```bash
#!/bin/bash

# ================================
# Complete n8n Backup Script
# ================================

set -e  # Exit on error

BACKUP_ROOT="/backup"
DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="$BACKUP_ROOT/$DATE"
LOG_FILE="$BACKUP_ROOT/backup.log"

echo "========================================" | tee -a "$LOG_FILE"
echo "n8n Backup Started: $(date)" | tee -a "$LOG_FILE"
echo "========================================" | tee -a "$LOG_FILE"

mkdir -p "$BACKUP_DIR"/{database,volumes,config}

# ================================
# 1. Database Backup
# ================================
echo "[$(date)] Backing up database..." | tee -a "$LOG_FILE"
docker exec -t n8n-postgres pg_dump -U n8n n8n | \
  gzip > "$BACKUP_DIR/database/n8n-db.sql.gz"

if [ $? -eq 0 ]; then
    echo "[$(date)] ✓ Database backup successful" | tee -a "$LOG_FILE"
else
    echo "[$(date)] ✗ Database backup FAILED" | tee -a "$LOG_FILE"
    exit 1
fi

# ================================
# 2. n8n Data Volume
# ================================
echo "[$(date)] Backing up n8n data..." | tee -a "$LOG_FILE"
docker run --rm \
  -v n8n_n8n-data:/source:ro \
  -v $BACKUP_DIR/volumes:/backup \
  alpine tar czf /backup/n8n-data.tar.gz -C /source .

if [ $? -eq 0 ]; then
    echo "[$(date)] ✓ n8n data backup successful" | tee -a "$LOG_FILE"
else
    echo "[$(date)] ✗ n8n data backup FAILED" | tee -a "$LOG_FILE"
    exit 1
fi

# ================================
# 3. Configuration Files
# ================================
echo "[$(date)] Backing up configuration..." | tee -a "$LOG_FILE"

# Docker Compose
cp /opt/n8n/docker-compose.yml "$BACKUP_DIR/config/" 2>/dev/null || true

# Environment (CAREFUL - contains secrets!)
cp /opt/n8n/.env "$BACKUP_DIR/config/env.backup" 2>/dev/null || true
chmod 600 "$BACKUP_DIR/config/env.backup"

# Nginx configs
cp -r /opt/n8n/nginx "$BACKUP_DIR/config/" 2>/dev/null || true

echo "[$(date)] ✓ Configuration backup successful" | tee -a "$LOG_FILE"

# ================================
# 4. Create Backup Manifest
# ================================
cat > "$BACKUP_DIR/manifest.txt" << EOF
n8n Backup Manifest
===================
Date: $(date)
Hostname: $(hostname)

Contents:
- database/n8n-db.sql.gz (PostgreSQL dump)
- volumes/n8n-data.tar.gz (n8n data directory)
- config/ (Configuration files)

Sizes:
$(du -h "$BACKUP_DIR" | tail -1)

Database:
$(docker exec n8n-postgres psql -U n8n -d n8n -c "SELECT pg_size_pretty(pg_database_size('n8n'));" -t)

Files:
$(ls -lh "$BACKUP_DIR"/*/* 2>/dev/null | awk '{print $5, $9}')
EOF

# ================================
# 5. Create Archive
# ================================
echo "[$(date)] Creating backup archive..." | tee -a "$LOG_FILE"
cd "$BACKUP_ROOT"
tar czf "n8n-backup-${DATE}.tar.gz" "$DATE/"

if [ $? -eq 0 ]; then
    ARCHIVE_SIZE=$(du -h "n8n-backup-${DATE}.tar.gz" | cut -f1)
    echo "[$(date)] ✓ Archive created: n8n-backup-${DATE}.tar.gz ($ARCHIVE_SIZE)" | tee -a "$LOG_FILE"

    # Remove temporary directory
    rm -rf "$DATE"
else
    echo "[$(date)] ✗ Archive creation FAILED" | tee -a "$LOG_FILE"
    exit 1
fi

# ================================
# 6. Upload to Remote Storage
# ================================
if [ -n "$BACKUP_S3_BUCKET" ]; then
    echo "[$(date)] Uploading to S3..." | tee -a "$LOG_FILE"
    aws s3 cp "n8n-backup-${DATE}.tar.gz" "s3://$BACKUP_S3_BUCKET/n8n-backup-${DATE}.tar.gz"

    if [ $? -eq 0 ]; then
        echo "[$(date)] ✓ Upload successful" | tee -a "$LOG_FILE"
    else
        echo "[$(date)] ✗ Upload FAILED" | tee -a "$LOG_FILE"
    fi
fi

# ================================
# 7. Cleanup Old Backups
# ================================
echo "[$(date)] Cleaning old backups..." | tee -a "$LOG_FILE"
RETENTION_DAYS=${RETENTION_DAYS:-30}
find "$BACKUP_ROOT" -name "n8n-backup-*.tar.gz" -mtime +$RETENTION_DAYS -delete

BACKUP_COUNT=$(ls -1 "$BACKUP_ROOT"/n8n-backup-*.tar.gz 2>/dev/null | wc -l)

# ================================
# 8. Summary
# ================================
echo "========================================" | tee -a "$LOG_FILE"
echo "Backup Completed: $(date)" | tee -a "$LOG_FILE"
echo "Total backups retained: $BACKUP_COUNT" | tee -a "$LOG_FILE"
echo "========================================" | tee -a "$LOG_FILE"
```

Make executable:
```bash
chmod +x /scripts/backup-all.sh
```

## Automated Backup Scheduling

### Using Cron

```bash
# Edit crontab
crontab -e

# Daily backup at 2 AM
0 2 * * * /scripts/backup-all.sh

# Additional database backup every 6 hours
0 */6 * * * /scripts/backup-database.sh

# Weekly full backup on Sunday at 3 AM
0 3 * * 0 /scripts/backup-all.sh
```

### Using Systemd Timer

Create `/etc/systemd/system/n8n-backup.service`:

```ini
[Unit]
Description=n8n Backup Service
After=docker.service

[Service]
Type=oneshot
ExecStart=/scripts/backup-all.sh
User=root
StandardOutput=journal
StandardError=journal
```

Create `/etc/systemd/system/n8n-backup.timer`:

```ini
[Unit]
Description=n8n Backup Timer
Requires=n8n-backup.service

[Timer]
OnCalendar=daily
OnCalendar=02:00
Persistent=true

[Install]
WantedBy=timers.target
```

Enable:
```bash
sudo systemctl enable n8n-backup.timer
sudo systemctl start n8n-backup.timer
sudo systemctl status n8n-backup.timer
```

### Docker-Based Backup Container

```yaml
services:
  backup:
    image: alpine:latest
    container_name: n8n-backup
    restart: unless-stopped
    volumes:
      - postgres-data:/postgres-data:ro
      - n8n-data:/n8n-data:ro
      - ./backups:/backup
      - ./scripts:/scripts
    environment:
      - RETENTION_DAYS=30
      - BACKUP_S3_BUCKET=${BACKUP_S3_BUCKET}
    command: >
      sh -c "
      apk add --no-cache postgresql-client aws-cli &&
      while true; do
        /scripts/backup-all.sh
        sleep 86400
      done
      "
```

## Offsite Backup

### AWS S3

```bash
# Install AWS CLI
apt-get install awscli

# Configure credentials
aws configure

# Backup script with S3 upload
aws s3 cp /backup/n8n-backup-${DATE}.tar.gz \
  s3://your-bucket/n8n-backups/

# Lifecycle policy (delete after 90 days)
aws s3api put-bucket-lifecycle-configuration \
  --bucket your-bucket \
  --lifecycle-configuration file://lifecycle.json
```

lifecycle.json:
```json
{
  "Rules": [{
    "Id": "DeleteOldBackups",
    "Status": "Enabled",
    "Prefix": "n8n-backups/",
    "Expiration": {
      "Days": 90
    }
  }]
}
```

### Backblaze B2

```bash
# Install B2 CLI
pip install b2

# Authorize
b2 authorize-account <applicationKeyId> <applicationKey>

# Upload
b2 upload-file \
  --noProgress \
  your-bucket-name \
  /backup/n8n-backup-${DATE}.tar.gz \
  n8n-backups/n8n-backup-${DATE}.tar.gz
```

### Rsync to Remote Server

```bash
#!/bin/bash

REMOTE_USER="backup"
REMOTE_HOST="backup.example.com"
REMOTE_PATH="/backups/n8n"

rsync -avz --delete \
  /backup/ \
  ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_PATH}/

echo "Backup synced to remote server"
```

## Recovery Procedures

### Complete System Recovery

#### 1. Prepare Environment

```bash
# Install Docker and Docker Compose
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Create directories
mkdir -p /opt/n8n
cd /opt/n8n
```

#### 2. Restore Configuration

```bash
# Download backup
aws s3 cp s3://your-bucket/n8n-backups/n8n-backup-XXXXXXXX.tar.gz .

# Extract
tar xzf n8n-backup-XXXXXXXX.tar.gz
cd XXXXXXXX-XXXXXX

# Restore config files
cp config/docker-compose.yml /opt/n8n/
cp config/env.backup /opt/n8n/.env
chmod 600 /opt/n8n/.env
```

#### 3. Start Services

```bash
cd /opt/n8n

# Start database first
docker compose up -d postgres
sleep 10
```

#### 4. Restore Database

```bash
# Copy backup into container
docker cp /backup/XXXXXXXX/database/n8n-db.sql.gz n8n-postgres:/tmp/

# Restore
docker exec -i n8n-postgres sh -c "
  gunzip < /tmp/n8n-db.sql.gz | psql -U n8n -d n8n
"

echo "Database restored"
```

#### 5. Restore n8n Data

```bash
# Extract data into volume
docker run --rm \
  -v n8n_n8n-data:/target \
  -v /backup/XXXXXXXX/volumes:/backup \
  alpine sh -c "cd /target && tar xzf /backup/n8n-data.tar.gz"

echo "n8n data restored"
```

#### 6. Start n8n

```bash
# Start all services
docker compose up -d

# Check logs
docker compose logs -f
```

#### 7. Verify

```bash
# Check n8n is running
curl -I https://n8n.yourdomain.com

# Verify workflows exist
docker compose exec postgres psql -U n8n -d n8n -c \
  "SELECT COUNT(*) FROM workflow_entity;"

# Test login
# Open browser and login
```

### Partial Recovery

#### Restore Single Workflow

```sql
-- Export workflow from backup database
COPY (
  SELECT * FROM workflow_entity WHERE name = 'My Workflow'
) TO '/tmp/workflow.csv' WITH CSV HEADER;

-- Import into production
COPY workflow_entity FROM '/tmp/workflow.csv' WITH CSV HEADER;
```

#### Restore Credentials

```bash
# Restore from n8n data volume backup
# Extract .n8n/credentials directory only
```

## Testing Backup Integrity

### Automated Verification Script

Create `/scripts/verify-backup.sh`:

```bash
#!/bin/bash

BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup-file.tar.gz>"
    exit 1
fi

echo "Verifying backup: $BACKUP_FILE"

# Extract to temp
TEMP_DIR=$(mktemp -d)
tar xzf "$BACKUP_FILE" -C "$TEMP_DIR"

# Verify database backup
echo "Checking database backup..."
gunzip < "$TEMP_DIR"/*/database/n8n-db.sql.gz | head -10

if [ $? -eq 0 ]; then
    echo "✓ Database backup is valid"
else
    echo "✗ Database backup is INVALID"
    exit 1
fi

# Verify n8n data
echo "Checking n8n data..."
tar tzf "$TEMP_DIR"/*/volumes/n8n-data.tar.gz > /dev/null

if [ $? -eq 0 ]; then
    echo "✓ n8n data is valid"
else
    echo "✗ n8n data is INVALID"
    exit 1
fi

# Check manifest
echo "Manifest:"
cat "$TEMP_DIR"/*/manifest.txt

# Cleanup
rm -rf "$TEMP_DIR"

echo "✓ Backup verification complete"
```

### Monthly DR Test

```bash
#!/bin/bash

# Monthly disaster recovery test
# 1. Spin up test environment
# 2. Restore latest backup
# 3. Verify functionality
# 4. Report results

echo "Starting DR test..."

# Use latest backup
LATEST_BACKUP=$(ls -t /backup/n8n-backup-*.tar.gz | head -1)

# Restore to test environment
# ... restoration steps ...

# Verify
# ... verification steps ...

echo "DR test complete"
```

## Backup Monitoring

### n8n Workflow for Backup Monitoring

```
Schedule (Daily 9 AM)
  ↓
Execute Command (check last backup)
  ↓
Function (parse backup date)
  ↓
IF (backup older than 24 hours)
  ↓
  Email (alert admin)
  ↓
  Slack (notify team)
```

### Backup Health Check Script

```bash
#!/bin/bash

BACKUP_DIR="/backup"
MAX_AGE_HOURS=24
ALERT_EMAIL="admin@example.com"

LATEST_BACKUP=$(ls -t $BACKUP_DIR/n8n-backup-*.tar.gz 2>/dev/null | head -1)

if [ -z "$LATEST_BACKUP" ]; then
    echo "ERROR: No backups found!"
    # Send alert
    exit 1
fi

BACKUP_AGE_HOURS=$(( ($(date +%s) - $(stat -c %Y "$LATEST_BACKUP")) / 3600 ))

if [ $BACKUP_AGE_HOURS -gt $MAX_AGE_HOURS ]; then
    echo "WARNING: Last backup is $BACKUP_AGE_HOURS hours old"
    # Send alert
    exit 1
else
    echo "OK: Last backup is $BACKUP_AGE_HOURS hours old"
fi
```

## Best Practices Checklist

### Backup
- [ ] Daily automated database backups
- [ ] Daily n8n data volume backups
- [ ] Configuration files backed up
- [ ] Backups stored offsite
- [ ] Retention policy implemented
- [ ] Backup encryption enabled
- [ ] Monitoring for backup failures
- [ ] Backup integrity verification

### Recovery
- [ ] Recovery procedures documented
- [ ] DR test performed quarterly
- [ ] Recovery time objective (RTO) defined
- [ ] Recovery point objective (RPO) defined
- [ ] Full recovery tested end-to-end
- [ ] Partial recovery procedures tested
- [ ] Emergency contacts documented
- [ ] Runbooks created and updated

### Security
- [ ] Backup files encrypted
- [ ] Access to backups restricted
- [ ] Environment variables secured
- [ ] Encryption key backed up separately
- [ ] Offsite backups use encryption in transit
- [ ] Backup logs monitored
- [ ] Regular security audits

## Recovery Time Objectives

| Component | RTO | RPO | Priority |
|-----------|-----|-----|----------|
| Database | 1 hour | 24 hours | Critical |
| n8n Data | 2 hours | 24 hours | High |
| Configuration | 30 min | 7 days | Medium |
| SSL Certificates | 1 hour | 30 days | High |

Adjust based on your requirements.

## Quick Reference

### Backup Commands

```bash
# Quick database backup
docker exec n8n-postgres pg_dump -U n8n n8n | gzip > backup.sql.gz

# Quick n8n data backup
docker run --rm -v n8n_n8n-data:/source:ro -v $(pwd):/backup alpine \
  tar czf /backup/n8n-data.tar.gz -C /source .

# List backups
ls -lht /backup/

# Verify backup
tar tzf backup.tar.gz

# Check backup size
du -h backup.tar.gz
```

### Recovery Commands

```bash
# Restore database
gunzip < backup.sql.gz | docker exec -i n8n-postgres psql -U n8n -d n8n

# Restore n8n data
docker run --rm -v n8n_n8n-data:/target -v $(pwd):/backup alpine \
  sh -c "cd /target && tar xzf /backup/n8n-data.tar.gz"
```

## Conclusion

A comprehensive backup and disaster recovery strategy is essential for production n8n deployments. Regular testing ensures you can recover when needed.

**Remember:**
- Automate backups
- Store offsite
- Test recovery regularly
- Monitor backup health
- Document procedures
- Keep encryption key safe

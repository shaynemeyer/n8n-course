# Database Setup and Maintenance

## Introduction

Choosing and configuring the right database is critical for n8n production deployments. This guide covers database selection, setup, optimization, and maintenance for n8n.

## Database Options

### PostgreSQL (Recommended)

**Pros:**
- Best performance for n8n
- Excellent reliability
- Strong community support
- Advanced features (JSONB, full-text search)
- Well-tested with n8n

**Cons:**
- More complex setup than SQLite
- Requires separate service

**Use for:**
- All production deployments
- Multi-user environments
- High-volume workflows

### MySQL/MariaDB (Alternative)

**Pros:**
- Widely available
- Good performance
- Familiar to many developers

**Cons:**
- Slightly less optimal for n8n than PostgreSQL
- Some JSON features less efficient

**Use for:**
- Existing MySQL infrastructure
- When PostgreSQL unavailable

### SQLite (Development Only)

**Pros:**
- Zero configuration
- File-based
- Perfect for testing

**Cons:**
- NOT recommended for production
- No concurrent write support
- Not suitable for multi-user
- Limited scalability

**Use for:**
- Development/testing only
- Single-user local instances

## PostgreSQL Setup

### Docker Deployment

#### Basic Setup

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: n8n-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: n8n
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "127.0.0.1:5432:5432"  # Only expose to localhost
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U n8n']
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres-data:
```

#### Production Setup with Optimization

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: n8n-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: n8n

      # Performance tuning
      POSTGRES_INITDB_ARGS: "-E UTF8 --locale=en_US.UTF-8"

      # Shared buffers (25% of RAM)
      POSTGRES_SHARED_BUFFERS: 256MB

      # Effective cache size (50-75% of RAM)
      POSTGRES_EFFECTIVE_CACHE_SIZE: 1GB

      # Work memory
      POSTGRES_WORK_MEM: 16MB

      # Maintenance work memory
      POSTGRES_MAINTENANCE_WORK_MEM: 128MB

    command: >
      postgres
      -c shared_buffers=256MB
      -c effective_cache_size=1GB
      -c work_mem=16MB
      -c maintenance_work_mem=128MB
      -c max_connections=100
      -c random_page_cost=1.1
      -c effective_io_concurrency=200
      -c wal_buffers=8MB
      -c default_statistics_target=100
      -c checkpoint_completion_target=0.9

    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./postgres-backup:/backup

    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U n8n']
      interval: 10s
      timeout: 5s
      retries: 5

    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G
        reservations:
          cpus: '1'
          memory: 512M

volumes:
  postgres-data:
    driver: local
```

### Manual PostgreSQL Installation

#### Ubuntu/Debian

```bash
# Install PostgreSQL
sudo apt update
sudo apt install postgresql postgresql-contrib

# Start PostgreSQL
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Create database and user
sudo -u postgres psql << EOF
CREATE DATABASE n8n;
CREATE USER n8n WITH ENCRYPTED PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE n8n TO n8n;
ALTER DATABASE n8n OWNER TO n8n;
\q
EOF
```

#### CentOS/RHEL

```bash
# Install PostgreSQL
sudo dnf install postgresql-server postgresql-contrib

# Initialize database
sudo postgresql-setup --initdb

# Start PostgreSQL
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Create database and user
sudo -u postgres psql << EOF
CREATE DATABASE n8n;
CREATE USER n8n WITH ENCRYPTED PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE n8n TO n8n;
\q
EOF
```

#### macOS (Homebrew)

```bash
# Install PostgreSQL
brew install postgresql@15

# Start PostgreSQL
brew services start postgresql@15

# Create database and user
psql postgres << EOF
CREATE DATABASE n8n;
CREATE USER n8n WITH ENCRYPTED PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE n8n TO n8n;
\q
EOF
```

### n8n Database Configuration

```bash
# Environment variables for PostgreSQL
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=localhost
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n
DB_POSTGRESDB_PASSWORD=your_secure_password

# Connection pooling
DB_POSTGRESDB_POOL_SIZE=4
```

## MySQL/MariaDB Setup

### Docker Deployment

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    container_name: n8n-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: n8n
      MYSQL_USER: n8n
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    command: >
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_unicode_ci
      --max_connections=100
      --innodb_buffer_pool_size=256M
    volumes:
      - mysql-data:/var/lib/mysql
    healthcheck:
      test: ['CMD', 'mysqladmin', 'ping', '-h', 'localhost']
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mysql-data:
```

### n8n MySQL Configuration

```bash
DB_TYPE=mysqldb
DB_MYSQLDB_HOST=localhost
DB_MYSQLDB_PORT=3306
DB_MYSQLDB_DATABASE=n8n
DB_MYSQLDB_USER=n8n
DB_MYSQLDB_PASSWORD=your_secure_password
```

## Database Migrations

### Initial Setup

When n8n starts for the first time with a new database, it automatically:
1. Creates all required tables
2. Sets up indexes
3. Initializes schema

### Upgrading n8n

Before upgrading n8n:

```bash
# 1. Backup database
./backup-database.sh

# 2. Stop n8n
docker compose stop n8n

# 3. Update image
docker compose pull n8n

# 4. Start n8n (migrations run automatically)
docker compose up -d n8n

# 5. Check logs for migration success
docker compose logs -f n8n
```

### Manual Migration (if needed)

```bash
# Run n8n migration command
docker compose exec n8n n8n db:migrate
```

## Connection Pooling

### Why Connection Pooling Matters

- Reduces overhead of creating new connections
- Improves performance under load
- Prevents database connection exhaustion

### Configuration

```bash
# Pool size (number of concurrent connections)
DB_POSTGRESDB_POOL_SIZE=4

# Calculation:
# - Basic deployment: 2-4
# - Medium traffic: 4-8
# - High traffic: 8-16
# - Queue mode: 2 per worker + 4 for main
```

### Pool Size Guidelines

**Single n8n Instance:**
```bash
DB_POSTGRESDB_POOL_SIZE=4
```

**Queue Mode (Main + 2 Workers):**
```bash
# Main instance
DB_POSTGRESDB_POOL_SIZE=4

# Each worker
DB_POSTGRESDB_POOL_SIZE=2
```

**PostgreSQL max_connections should be:**
```
max_connections = (pool_size × number_of_instances) + buffer
Example: (4 × 1) + 10 = 14 minimum, set to 100 for safety
```

## Backup Strategies

### Automated PostgreSQL Backup

#### Backup Script

Create `backup-postgres.sh`:

```bash
#!/bin/bash

# Configuration
BACKUP_DIR="/backup/postgres"
CONTAINER_NAME="n8n-postgres"
DB_NAME="n8n"
DB_USER="n8n"
RETENTION_DAYS=7

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup filename with timestamp
BACKUP_FILE="$BACKUP_DIR/n8n-postgres-$(date +%Y%m%d-%H%M%S).sql.gz"

# Create backup
docker exec -t $CONTAINER_NAME pg_dump -U $DB_USER $DB_NAME | gzip > $BACKUP_FILE

# Check if backup succeeded
if [ $? -eq 0 ]; then
    echo "Backup successful: $BACKUP_FILE"

    # Delete old backups
    find $BACKUP_DIR -name "n8n-postgres-*.sql.gz" -mtime +$RETENTION_DAYS -delete
    echo "Old backups cleaned up (older than $RETENTION_DAYS days)"
else
    echo "Backup failed!"
    exit 1
fi
```

Make it executable:
```bash
chmod +x backup-postgres.sh
```

#### Automated Backups with Cron

```bash
# Edit crontab
crontab -e

# Add daily backup at 2 AM
0 2 * * * /path/to/backup-postgres.sh >> /var/log/n8n-backup.log 2>&1
```

#### Docker Compose with Backup Service

```yaml
version: '3.8'

services:
  postgres:
    # ... postgres config ...

  backup:
    image: postgres:15-alpine
    container_name: n8n-backup
    depends_on:
      - postgres
    environment:
      PGHOST: postgres
      PGDATABASE: n8n
      PGUSER: n8n
      PGPASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - ./backups:/backup
      - ./scripts:/scripts
    command: >
      sh -c "
      while true; do
        echo 'Starting backup...'
        pg_dump | gzip > /backup/n8n-$(date +%Y%m%d-%H%M%S).sql.gz
        echo 'Backup completed'
        find /backup -name 'n8n-*.sql.gz' -mtime +7 -delete
        sleep 86400
      done
      "
```

### Offsite Backup

#### Backup to S3

```bash
#!/bin/bash

# Local backup first
./backup-postgres.sh

# Upload to S3
BACKUP_FILE=$(ls -t /backup/postgres/n8n-postgres-*.sql.gz | head -1)
aws s3 cp $BACKUP_FILE s3://your-bucket/n8n-backups/

echo "Backup uploaded to S3: $BACKUP_FILE"
```

#### Backup to Backblaze B2

```bash
#!/bin/bash

# Install b2 CLI first: pip install b2

# Backup
./backup-postgres.sh

# Upload
BACKUP_FILE=$(ls -t /backup/postgres/n8n-postgres-*.sql.gz | head -1)
b2 upload-file your-bucket-name $BACKUP_FILE n8n-backups/$(basename $BACKUP_FILE)
```

## Database Restoration

### PostgreSQL Restore

#### From Docker

```bash
# Stop n8n
docker compose stop n8n

# Restore database
gunzip < backup-file.sql.gz | docker exec -i n8n-postgres psql -U n8n -d n8n

# Or for uncompressed backup
docker exec -i n8n-postgres psql -U n8n -d n8n < backup-file.sql

# Restart n8n
docker compose start n8n
```

#### Complete Database Restore Script

```bash
#!/bin/bash

BACKUP_FILE=$1
CONTAINER_NAME="n8n-postgres"
DB_NAME="n8n"
DB_USER="n8n"

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: ./restore-postgres.sh <backup-file.sql.gz>"
    exit 1
fi

echo "Stopping n8n..."
docker compose stop n8n

echo "Dropping existing database..."
docker exec -t $CONTAINER_NAME psql -U $DB_USER -c "DROP DATABASE IF EXISTS $DB_NAME;"

echo "Creating fresh database..."
docker exec -t $CONTAINER_NAME psql -U $DB_USER -c "CREATE DATABASE $DB_NAME;"

echo "Restoring backup..."
gunzip < $BACKUP_FILE | docker exec -i $CONTAINER_NAME psql -U $DB_USER -d $DB_NAME

if [ $? -eq 0 ]; then
    echo "Restore successful!"
    echo "Starting n8n..."
    docker compose start n8n
else
    echo "Restore failed!"
    exit 1
fi
```

## Performance Optimization

### PostgreSQL Tuning

#### Configuration File (`postgresql.conf`)

For a server with 4GB RAM:

```conf
# Memory Settings
shared_buffers = 1GB                    # 25% of RAM
effective_cache_size = 3GB              # 75% of RAM
work_mem = 16MB                         # RAM / max_connections / 2
maintenance_work_mem = 256MB            # RAM / 16

# Connection Settings
max_connections = 100

# Checkpoint Settings
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100

# Planner Settings
random_page_cost = 1.1                  # For SSD
effective_io_concurrency = 200          # For SSD

# Logging (optional)
log_min_duration_statement = 1000       # Log slow queries (>1s)
```

### Index Optimization

n8n creates necessary indexes automatically, but you can verify:

```sql
-- Check table sizes
SELECT
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- Check missing indexes
SELECT
    schemaname,
    tablename,
    attname,
    n_distinct,
    correlation
FROM pg_stats
WHERE schemaname = 'public'
ORDER BY n_distinct DESC;
```

### VACUUM and ANALYZE

```bash
# Create maintenance script
cat > /scripts/pg-maintenance.sh << 'EOF'
#!/bin/bash
docker exec n8n-postgres psql -U n8n -d n8n -c "VACUUM ANALYZE;"
echo "Database maintenance completed at $(date)"
EOF

chmod +x /scripts/pg-maintenance.sh

# Add to cron - run weekly
0 3 * * 0 /scripts/pg-maintenance.sh >> /var/log/pg-maintenance.log 2>&1
```

## Monitoring and Maintenance

### Database Size Monitoring

```sql
-- Total database size
SELECT pg_size_pretty(pg_database_size('n8n'));

-- Table sizes
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

### Active Connections

```sql
-- Check active connections
SELECT
    datname,
    count(*) as connections
FROM pg_stat_activity
GROUP BY datname;

-- Kill idle connections (if needed)
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'n8n'
  AND state = 'idle'
  AND state_change < current_timestamp - INTERVAL '1 hour';
```

### Slow Query Logging

Enable in `postgresql.conf`:

```conf
log_min_duration_statement = 1000
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_statement = 'all'
```

Then monitor:
```bash
docker compose exec postgres tail -f /var/lib/postgresql/data/log/postgresql-*.log
```

## Troubleshooting

### Connection Issues

```bash
# Test connection from n8n container
docker compose exec n8n sh
nc -zv postgres 5432

# Or use psql
docker compose exec n8n sh -c 'apk add postgresql-client && psql -h postgres -U n8n -d n8n'
```

### Permission Issues

```sql
-- Grant all permissions
GRANT ALL PRIVILEGES ON DATABASE n8n TO n8n;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO n8n;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO n8n;
```

### Disk Space Issues

```bash
# Check disk usage
docker exec n8n-postgres du -sh /var/lib/postgresql/data

# Clean old WAL files (PostgreSQL handles automatically, but verify)
docker exec n8n-postgres ls -lh /var/lib/postgresql/data/pg_wal/
```

## Best Practices Checklist

- [ ] Use PostgreSQL for production
- [ ] Configure appropriate connection pool size
- [ ] Set up automated daily backups
- [ ] Store backups offsite (S3, B2, etc.)
- [ ] Test restore procedures regularly
- [ ] Configure database performance tuning
- [ ] Monitor database size and growth
- [ ] Set up slow query logging
- [ ] Schedule regular VACUUM ANALYZE
- [ ] Document connection strings securely
- [ ] Use strong database passwords
- [ ] Limit database network exposure
- [ ] Monitor active connections
- [ ] Plan for scaling (replication, read replicas)

## Quick Reference

### Common Commands

```bash
# Backup
docker exec n8n-postgres pg_dump -U n8n n8n | gzip > backup.sql.gz

# Restore
gunzip < backup.sql.gz | docker exec -i n8n-postgres psql -U n8n -d n8n

# Access database
docker exec -it n8n-postgres psql -U n8n -d n8n

# Check size
docker exec n8n-postgres psql -U n8n -d n8n -c "SELECT pg_size_pretty(pg_database_size('n8n'));"

# Vacuum
docker exec n8n-postgres psql -U n8n -d n8n -c "VACUUM ANALYZE;"
```

## Next Steps

After setting up your database:
1. Configure SSL/TLS (see `04-ssl-tls-configuration.md`)
2. Set up user management (see `05-user-management-rbac.md`)
3. Implement backup automation (see `07-backup-disaster-recovery.md`)
4. Monitor performance and optimize as needed

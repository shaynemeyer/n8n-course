# Queue Mode and Scaling

## Introduction

Queue mode allows n8n to scale horizontally by distributing workflow executions across multiple worker processes. This is essential for high-availability, high-volume production deployments.

## Understanding Queue Mode

### Regular Mode vs Queue Mode

#### Regular Mode (Default)

```
User → n8n → Execute Workflow (in-process)
```

**Limitations:**
- Single process handles everything
- Limited by one server's resources
- Can become bottleneck
- Reduced reliability

#### Queue Mode

```
User → n8n Main → Redis Queue → Worker 1
                              → Worker 2
                              → Worker 3
```

**Advantages:**
- Horizontal scaling
- Better resource utilization
- High availability
- Fault tolerance
- Separate UI from execution

## Architecture

### Components

1. **Main Process**
   - Serves web UI
   - Handles webhooks
   - Manages triggers
   - Queues executions

2. **Redis**
   - Job queue
   - Message broker
   - State management

3. **Workers**
   - Execute workflows
   - Process queue jobs
   - Stateless (can scale)

### Data Flow

```
Workflow Trigger
      ↓
Main Process (receives)
      ↓
Redis Queue (stores job)
      ↓
Worker (picks up job)
      ↓
Execute Workflow
      ↓
Save Results → Database
      ↓
Update UI (via Main)
```

## Redis Setup

### Docker Deployment

```yaml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    container_name: n8n-redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    ports:
      - "127.0.0.1:6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ['CMD', 'redis-cli', '--raw', 'incr', 'ping']
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - n8n-network
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M

volumes:
  redis-data:

networks:
  n8n-network:
```

### Redis Configuration

For production, create `redis.conf`:

```conf
# Network
bind 0.0.0.0
protected-mode yes
port 6379

# Security
requirepass your_redis_password

# Memory Management
maxmemory 512mb
maxmemory-policy allkeys-lru

# Persistence
save 900 1
save 300 10
save 60 10000

# Append Only File
appendonly yes
appendfilename "appendonly.aof"

# Performance
tcp-backlog 511
timeout 0
tcp-keepalive 300
```

Use in Docker:

```yaml
redis:
  image: redis:7-alpine
  command: redis-server /usr/local/etc/redis/redis.conf
  volumes:
    - ./redis.conf:/usr/local/etc/redis/redis.conf
    - redis-data:/data
```

## Queue Mode Configuration

### Complete Docker Compose Setup

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
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U n8n']
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - n8n-network

  redis:
    image: redis:7-alpine
    container_name: n8n-redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data
    healthcheck:
      test: ['CMD', 'redis-cli', '--raw', 'incr', 'ping']
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - n8n-network

  n8n-main:
    image: n8nio/n8n:latest
    container_name: n8n-main
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      # Database
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}

      # Queue Mode (CRITICAL!)
      - EXECUTIONS_MODE=queue

      # Redis Configuration
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - QUEUE_BULL_REDIS_PASSWORD=${REDIS_PASSWORD}
      - QUEUE_BULL_REDIS_DB=0

      # Queue Settings
      - QUEUE_HEALTH_CHECK_ACTIVE=true

      # URLs
      - N8N_HOST=${N8N_HOST}
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://${N8N_HOST}/

      # General
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}

    volumes:
      - n8n-data:/home/node/.n8n
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - n8n-network

  n8n-worker:
    image: n8nio/n8n:latest
    restart: unless-stopped
    command: worker
    environment:
      # Database
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}

      # Queue Mode
      - EXECUTIONS_MODE=queue

      # Redis Configuration
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - QUEUE_BULL_REDIS_PASSWORD=${REDIS_PASSWORD}
      - QUEUE_BULL_REDIS_DB=0

      # General
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}

    volumes:
      - n8n-data:/home/node/.n8n
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - n8n-network
    deploy:
      replicas: 3  # Scale workers as needed

volumes:
  postgres-data:
  redis-data:
  n8n-data:

networks:
  n8n-network:
```

### Environment Variables

```bash
# Core Settings
POSTGRES_PASSWORD=your_secure_postgres_password
REDIS_PASSWORD=your_secure_redis_password
N8N_ENCRYPTION_KEY=your_encryption_key

# Instance Configuration
N8N_HOST=n8n.yourdomain.com
GENERIC_TIMEZONE=America/New_York
```

## Scaling Workers

### Manual Scaling

```bash
# Scale to 5 workers
docker compose up -d --scale n8n-worker=5

# Scale down to 2 workers
docker compose up -d --scale n8n-worker=2

# Check running workers
docker compose ps n8n-worker
```

### Dynamic Scaling with Docker Swarm

```yaml
version: '3.8'

services:
  n8n-worker:
    image: n8nio/n8n:latest
    command: worker
    # ... environment config ...
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G
```

Deploy:
```bash
docker stack deploy -c docker-compose.yml n8n
```

### Kubernetes Deployment

#### n8n-main Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: n8n-main
spec:
  replicas: 1
  selector:
    matchLabels:
      app: n8n-main
  template:
    metadata:
      labels:
        app: n8n-main
    spec:
      containers:
      - name: n8n
        image: n8nio/n8n:latest
        ports:
        - containerPort: 5678
        env:
        - name: EXECUTIONS_MODE
          value: "queue"
        - name: QUEUE_BULL_REDIS_HOST
          value: "redis"
        # ... other env vars from ConfigMap/Secrets ...
```

#### n8n-worker Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: n8n-worker
spec:
  replicas: 3
  selector:
    matchLabels:
      app: n8n-worker
  template:
    metadata:
      labels:
        app: n8n-worker
    spec:
      containers:
      - name: n8n
        image: n8nio/n8n:latest
        command: ["n8n", "worker"]
        env:
        - name: EXECUTIONS_MODE
          value: "queue"
        - name: QUEUE_BULL_REDIS_HOST
          value: "redis"
        # ... other env vars ...
```

#### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: n8n-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: n8n-worker
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

## Advanced Queue Configuration

### Queue Settings

```bash
# Health Check
QUEUE_HEALTH_CHECK_ACTIVE=true
QUEUE_HEALTH_CHECK_PORT=5678

# Recovery Settings
QUEUE_RECOVERY_INTERVAL=60  # Check for stalled jobs every 60s

# Redis Connection
QUEUE_BULL_REDIS_HOST=redis
QUEUE_BULL_REDIS_PORT=6379
QUEUE_BULL_REDIS_PASSWORD=password
QUEUE_BULL_REDIS_DB=0
QUEUE_BULL_REDIS_TIMEOUT_THRESHOLD=10000

# TLS (if Redis uses TLS)
QUEUE_BULL_REDIS_TLS=true
```

### Job Concurrency

Control how many jobs each worker processes simultaneously:

```bash
# Each worker handles 10 concurrent jobs (default: 1)
EXECUTIONS_PROCESS=10
```

**Guidelines:**
- Low concurrency (1-2): Memory-intensive workflows
- Medium concurrency (5-10): General purpose
- High concurrency (20+): Lightweight workflows

### Job Priorities

Workflows can have different priorities:

```javascript
// In workflow settings (advanced)
{
  "settings": {
    "executionOrder": "v1",
    "executionTimeout": -1,
    "priority": 10  // Higher = more priority
  }
}
```

## Monitoring Queue Mode

### Redis Monitoring

#### Check Queue Size

```bash
# Connect to Redis
docker compose exec redis redis-cli -a ${REDIS_PASSWORD}

# Check queue length
LLEN bull:n8n:jobs:waiting
LLEN bull:n8n:jobs:active
LLEN bull:n8n:jobs:completed
LLEN bull:n8n:jobs:failed

# View all keys
KEYS bull:n8n:*
```

#### Monitor Redis Performance

```bash
# Real-time monitoring
docker compose exec redis redis-cli -a ${REDIS_PASSWORD} --stat

# Check memory usage
docker compose exec redis redis-cli -a ${REDIS_PASSWORD} INFO memory

# Check connected clients
docker compose exec redis redis-cli -a ${REDIS_PASSWORD} CLIENT LIST
```

### Worker Monitoring

#### Check Worker Status

```bash
# View worker logs
docker compose logs -f n8n-worker

# Check worker resource usage
docker stats $(docker compose ps -q n8n-worker)

# Count active workers
docker compose ps n8n-worker | grep Up | wc -l
```

### Monitoring Workflow

Create an n8n workflow to monitor queue health:

```
Schedule (Every 5 minutes)
  ↓
Execute Command (redis-cli queue stats)
  ↓
Function (Parse stats)
  ↓
IF (Queue too long)
  ↓
  Slack (Alert team)
  ↓
  HTTP (Trigger autoscaler)
```

## Load Balancing

### Main Process Load Balancing

For high availability, run multiple main processes:

```yaml
services:
  n8n-main:
    # ... config ...
    deploy:
      replicas: 2
```

Then use a load balancer (Nginx, HAProxy, etc.):

```nginx
upstream n8n_backend {
    least_conn;
    server n8n-main-1:5678;
    server n8n-main-2:5678;
}

server {
    location / {
        proxy_pass http://n8n_backend;
        # ... other settings ...
    }
}
```

### Sticky Sessions

For webhook handling, use sticky sessions:

```nginx
upstream n8n_backend {
    ip_hash;  # Sticky sessions based on IP
    server n8n-main-1:5678;
    server n8n-main-2:5678;
}
```

## Performance Optimization

### Connection Pooling

```bash
# PostgreSQL pool size per instance
DB_POSTGRESDB_POOL_SIZE=4

# Calculate total: (main instances × 4) + (workers × 2)
# Example: (2 × 4) + (5 × 2) = 18 connections needed
# Set max_connections in PostgreSQL to 50+ for safety
```

### Redis Optimization

```conf
# redis.conf for high performance
maxmemory 2gb
maxmemory-policy allkeys-lru
maxclients 10000

# Disable disk persistence for pure queue
save ""
appendonly no
```

**Note:** Disabling persistence means jobs in queue are lost on restart. Only do this if acceptable.

### Worker Optimization

```bash
# More workers, less concurrency per worker
EXECUTIONS_PROCESS=5

# OR fewer workers, more concurrency
EXECUTIONS_PROCESS=20
```

Test and find optimal balance.

## High Availability Setup

### Complete HA Architecture

```
         Load Balancer
              ↓
    ┌─────────┴─────────┐
    ↓                   ↓
n8n-main-1         n8n-main-2
    ↓                   ↓
    └─────────┬─────────┘
              ↓
         Redis Queue
              ↓
    ┌─────────┼─────────┐
    ↓         ↓         ↓
Worker-1  Worker-2  Worker-3
    ↓         ↓         ↓
    └─────────┼─────────┘
              ↓
      PostgreSQL (Primary)
              ↓
      PostgreSQL (Replica)
```

### Database Replication

```yaml
services:
  postgres-primary:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: n8n
    command: |
      postgres
      -c wal_level=replica
      -c max_wal_senders=3
      -c max_replication_slots=3

  postgres-replica:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    command: |
      postgres
      -c hot_standby=on
```

### Redis Cluster

For mission-critical deployments:

```yaml
services:
  redis-1:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf

  redis-2:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf

  redis-3:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf
```

## Troubleshooting

### Queue Not Processing

```bash
# Check Redis is running
docker compose ps redis

# Check Redis connection
docker compose exec n8n-main sh -c 'nc -zv redis 6379'

# Verify queue mode is enabled
docker compose exec n8n-main env | grep EXECUTIONS_MODE

# Check worker logs
docker compose logs n8n-worker --tail=100
```

### Jobs Stuck in Queue

```bash
# Connect to Redis
docker compose exec redis redis-cli -a ${REDIS_PASSWORD}

# Move failed jobs back to waiting
LRANGE bull:n8n:jobs:failed 0 -1
# Manually retry or clear

# Clear all jobs (CAUTION!)
FLUSHDB
```

### Worker Not Starting

```bash
# Check worker command
docker compose ps n8n-worker

# Should show: "worker" in command

# Check logs for errors
docker compose logs n8n-worker

# Verify shared volumes
docker compose exec n8n-worker ls -la /home/node/.n8n
```

### High Memory Usage

```bash
# Check worker memory
docker stats $(docker compose ps -q n8n-worker)

# Reduce concurrency
EXECUTIONS_PROCESS=5

# Add memory limits
deploy:
  resources:
    limits:
      memory: 2G
```

## Best Practices Checklist

- [ ] Use queue mode for production
- [ ] Deploy at least 2 workers
- [ ] Configure Redis with password
- [ ] Set appropriate concurrency
- [ ] Monitor queue size
- [ ] Monitor worker health
- [ ] Configure resource limits
- [ ] Implement auto-scaling
- [ ] Use load balancer for HA
- [ ] Enable Redis persistence
- [ ] Regular queue monitoring
- [ ] Document scaling procedures
- [ ] Test failover scenarios
- [ ] Monitor Redis memory
- [ ] Plan for peak loads

## Quick Reference

### Common Commands

```bash
# Scale workers
docker compose up -d --scale n8n-worker=5

# Check queue size
docker compose exec redis redis-cli -a ${REDIS_PASSWORD} LLEN bull:n8n:jobs:waiting

# Monitor workers
docker compose logs -f n8n-worker

# Restart workers
docker compose restart n8n-worker

# Check Redis stats
docker compose exec redis redis-cli -a ${REDIS_PASSWORD} INFO stats
```

### Performance Metrics

| Workers | Concurrency | Jobs/min | Memory/Worker |
|---------|-------------|----------|---------------|
| 2 | 5 | ~100 | 1GB |
| 5 | 5 | ~250 | 1GB |
| 5 | 10 | ~500 | 1.5GB |
| 10 | 10 | ~1000 | 1.5GB |

*Approximate values, varies by workflow complexity*

## Next Steps

After setting up queue mode:
1. Implement comprehensive backups (see `07-backup-disaster-recovery.md`)
2. Set up monitoring and alerting
3. Test scaling procedures
4. Document architecture
5. Create runbooks for common issues

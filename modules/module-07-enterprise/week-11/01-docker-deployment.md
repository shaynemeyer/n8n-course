# Docker Deployment Best Practices

## Introduction

Docker is the recommended deployment method for n8n in production environments. It provides consistency, isolation, and easy version management. This guide covers best practices for deploying n8n using Docker.

## Prerequisites

- Docker installed (version 20.10 or higher)
- Docker Compose installed (version 2.0 or higher)
- Basic understanding of containers and Docker concepts
- A server or VPS with at least 2GB RAM (4GB+ recommended)

## Single Container vs Multi-Container Setup

### Single Container Setup

Best for:
- Small teams (1-5 users)
- Development/testing environments
- Simple workflows with low volume

Limitations:
- Cannot scale horizontally
- Single point of failure
- Limited to vertical scaling only

### Multi-Container Setup (Recommended for Production)

Best for:
- Production environments
- Multiple users
- High-availability requirements
- Scalable workloads

Components:
- n8n main container
- PostgreSQL database container
- Redis container (for queue mode)
- Reverse proxy container (Nginx/Traefik)

## Basic Docker Compose Configuration

### Minimal Production Setup

Create a `docker-compose.yml` file:

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

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      - N8N_HOST=${N8N_HOST}
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://${N8N_HOST}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
    volumes:
      - n8n-data:/home/node/.n8n
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  postgres-data:
  n8n-data:
```

### Environment Variables File

Create a `.env` file (never commit this to version control):

```env
# Database
POSTGRES_PASSWORD=your_secure_password_here

# n8n Authentication
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=your_secure_password_here

# n8n Configuration
N8N_HOST=n8n.yourdomain.com
GENERIC_TIMEZONE=America/New_York
```

## Advanced Docker Compose Configuration

### Production-Ready Setup with Queue Mode

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
      POSTGRES_NON_ROOT_USER: n8n_user
      POSTGRES_NON_ROOT_PASSWORD: ${POSTGRES_USER_PASSWORD}
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
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}

      # Queue Mode
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - QUEUE_BULL_REDIS_PASSWORD=${REDIS_PASSWORD}

      # Security
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}

      # URLs
      - N8N_HOST=${N8N_HOST}
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://${N8N_HOST}/

      # Performance
      - N8N_PAYLOAD_SIZE_MAX=16
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=168

      # Timezone
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
    container_name: n8n-worker
    restart: unless-stopped
    command: worker
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}

      # Queue Mode
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - QUEUE_BULL_REDIS_PASSWORD=${REDIS_PASSWORD}

      # Timezone
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
      replicas: 2

volumes:
  postgres-data:
  redis-data:
  n8n-data:

networks:
  n8n-network:
    driver: bridge
```

## Volume Management and Data Persistence

### Understanding n8n Data Volumes

n8n stores several types of data:

1. **Database Data** (PostgreSQL volume)
   - Workflow definitions
   - Execution history
   - Credentials (encrypted)
   - User information

2. **File Storage** (n8n data volume)
   - Custom nodes
   - Binary files
   - Encryption key
   - SSL certificates (if stored locally)

### Volume Best Practices

#### 1. Use Named Volumes

```yaml
volumes:
  postgres-data:
    name: n8n_postgres_data
  n8n-data:
    name: n8n_app_data
```

#### 2. Bind Mounts for Configuration

```yaml
services:
  n8n:
    volumes:
      - n8n-data:/home/node/.n8n
      - ./custom-nodes:/home/node/.n8n/custom
      - ./config:/home/node/.n8n/config:ro
```

#### 3. Backup Volume Locations

Find volume locations:
```bash
docker volume inspect n8n_postgres_data
docker volume inspect n8n_app_data
```

#### 4. Volume Permissions

Ensure correct permissions:
```bash
# n8n runs as user node (UID 1000)
sudo chown -R 1000:1000 /path/to/n8n/data
```

## Network Configuration

### Internal Network

```yaml
networks:
  n8n-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

### Expose Only Necessary Ports

```yaml
services:
  n8n:
    ports:
      - "127.0.0.1:5678:5678"  # Only localhost
    # Or use reverse proxy - no exposed ports
    expose:
      - "5678"
```

## Resource Limits and Constraints

### Setting Resource Limits

```yaml
services:
  n8n:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G

  postgres:
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
```

### Memory Optimization

```yaml
services:
  n8n:
    environment:
      - NODE_OPTIONS=--max-old-space-size=2048
```

## Deployment Commands

### Initial Deployment

```bash
# Create environment file
cp .env.example .env
nano .env  # Edit with your values

# Start services
docker compose up -d

# View logs
docker compose logs -f n8n

# Check status
docker compose ps
```

### Updates and Maintenance

```bash
# Pull latest images
docker compose pull

# Restart with new images
docker compose up -d

# View specific service logs
docker compose logs -f n8n-main

# Execute commands in container
docker compose exec n8n n8n --version
```

### Backup Before Updates

```bash
# Stop services
docker compose stop

# Backup volumes
docker run --rm \
  -v n8n_postgres_data:/source \
  -v $(pwd)/backups:/backup \
  alpine tar czf /backup/postgres-$(date +%Y%m%d).tar.gz -C /source .

# Backup n8n data
docker run --rm \
  -v n8n_app_data:/source \
  -v $(pwd)/backups:/backup \
  alpine tar czf /backup/n8n-data-$(date +%Y%m%d).tar.gz -C /source .

# Start services
docker compose up -d
```

## Monitoring and Logging

### Log Configuration

```yaml
services:
  n8n:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

### Health Checks

```yaml
services:
  n8n:
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:5678/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

## Troubleshooting

### Common Issues

#### 1. Permission Denied Errors

```bash
# Fix volume permissions
docker compose down
sudo chown -R 1000:1000 /path/to/n8n/data
docker compose up -d
```

#### 2. Database Connection Issues

```bash
# Check database is healthy
docker compose ps
docker compose logs postgres

# Test connection
docker compose exec postgres psql -U n8n -d n8n -c "SELECT 1;"
```

#### 3. Out of Memory

```bash
# Check container stats
docker stats

# Increase memory limit in docker-compose.yml
# Or add NODE_OPTIONS environment variable
```

### Debugging Commands

```bash
# Enter container shell
docker compose exec n8n sh

# Check environment variables
docker compose exec n8n env

# View real-time logs
docker compose logs -f --tail=100 n8n

# Restart specific service
docker compose restart n8n
```

## Security Considerations

### 1. Don't Expose Database Ports

```yaml
services:
  postgres:
    # NO ports section - only accessible within Docker network
    expose:
      - "5432"
```

### 2. Use Strong Passwords

```bash
# Generate secure passwords
openssl rand -base64 32
```

### 3. Keep Images Updated

```bash
# Check for updates weekly
docker compose pull
docker compose up -d
```

### 4. Limit Container Capabilities

```yaml
services:
  n8n:
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

## Best Practices Checklist

- [ ] Use Docker Compose for orchestration
- [ ] Configure persistent volumes for all data
- [ ] Set resource limits appropriately
- [ ] Implement health checks
- [ ] Configure log rotation
- [ ] Use environment variables for configuration
- [ ] Enable queue mode for scalability
- [ ] Set up proper networking
- [ ] Implement regular backups
- [ ] Keep images updated
- [ ] Monitor container performance
- [ ] Document your configuration
- [ ] Test disaster recovery procedures

## Next Steps

After mastering Docker deployment:
1. Configure environment variables (see `02-environment-configuration.md`)
2. Set up SSL/TLS with reverse proxy (see `04-ssl-tls-configuration.md`)
3. Implement backup automation (see `07-backup-disaster-recovery.md`)
4. Scale with queue mode (see `06-queue-mode-scaling.md`)

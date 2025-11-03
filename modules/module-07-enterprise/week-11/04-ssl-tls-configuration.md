# SSL/TLS Configuration

## Introduction

Securing n8n with SSL/TLS is essential for production deployments. This guide covers setting up HTTPS using reverse proxies, obtaining SSL certificates, and implementing security best practices.

## Why SSL/TLS is Critical

1. **Encryption**: Protects data in transit
2. **Authentication**: Verifies server identity
3. **Trust**: Required by modern browsers
4. **Webhooks**: Many services require HTTPS endpoints
5. **Compliance**: Required for many regulatory frameworks

## Architecture Overview

```
Internet
    ↓
SSL/TLS Termination (Reverse Proxy)
    ↓
n8n (HTTP on localhost)
```

## Prerequisites

- Domain name pointed to your server
- Server with public IP address
- DNS records configured (A record)
- Ports 80 and 443 open in firewall

## Option 1: Nginx + Let's Encrypt (Recommended)

### Complete Docker Compose Setup

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    container_name: n8n-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - certbot-etc:/etc/letsencrypt
      - certbot-var:/var/lib/letsencrypt
      - ./nginx/dhparam:/etc/ssl/certs
    depends_on:
      - n8n
    networks:
      - n8n-network

  certbot:
    image: certbot/certbot
    container_name: n8n-certbot
    volumes:
      - certbot-etc:/etc/letsencrypt
      - certbot-var:/var/lib/letsencrypt
      - ./nginx/webroot:/var/www/html
    depends_on:
      - nginx
    command: certonly --webroot --webroot-path=/var/www/html --email your-email@example.com --agree-tos --no-eff-email --force-renewal -d n8n.yourdomain.com

  postgres:
    image: postgres:15-alpine
    # ... postgres config ...

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - N8N_HOST=n8n.yourdomain.com
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.yourdomain.com/
      - N8N_PORT=5678
    volumes:
      - n8n-data:/home/node/.n8n
    depends_on:
      - postgres
    networks:
      - n8n-network

volumes:
  certbot-etc:
  certbot-var:
  postgres-data:
  n8n-data:

networks:
  n8n-network:
```

### Nginx Configuration

#### Create `nginx/nginx.conf`

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 16M;

    # SSL Settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers off;

    # Gzip Settings
    gzip on;
    gzip_disable "msie6";
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript application/json application/javascript application/xml+rss application/rss+xml font/truetype font/opentype application/vnd.ms-fontobject image/svg+xml;

    include /etc/nginx/conf.d/*.conf;
}
```

#### Create `nginx/conf.d/n8n.conf`

```nginx
# Redirect HTTP to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name n8n.yourdomain.com;

    # Let's Encrypt verification
    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    # Redirect all other traffic to HTTPS
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS Server
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name n8n.yourdomain.com;

    # SSL Certificate Configuration
    ssl_certificate /etc/letsencrypt/live/n8n.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.yourdomain.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/n8n.yourdomain.com/chain.pem;

    # SSL Security Settings
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    # Diffie-Hellman parameter
    ssl_dhparam /etc/ssl/certs/dhparam.pem;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    # Security Headers
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Logging
    access_log /var/log/nginx/n8n.access.log;
    error_log /var/log/nginx/n8n.error.log;

    # Proxy to n8n
    location / {
        proxy_pass http://n8n:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;

        # Timeouts
        proxy_connect_timeout 300;
        proxy_send_timeout 300;
        proxy_read_timeout 300;
        send_timeout 300;

        # Buffer settings
        proxy_buffering off;
        proxy_buffer_size 4k;
    }

    # Health check endpoint
    location /healthz {
        proxy_pass http://n8n:5678/healthz;
        access_log off;
    }
}
```

### Generate DH Parameters

```bash
# Generate Diffie-Hellman parameters (takes a few minutes)
mkdir -p nginx/dhparam
openssl dhparam -out nginx/dhparam/dhparam.pem 2048
```

### Obtain SSL Certificate

#### Initial Setup

```bash
# Create directories
mkdir -p nginx/webroot
mkdir -p nginx/conf.d

# Start nginx first (without SSL)
docker compose up -d nginx

# Request certificate
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path=/var/www/html \
  --email your-email@example.com \
  --agree-tos \
  --no-eff-email \
  -d n8n.yourdomain.com

# Restart nginx with SSL config
docker compose restart nginx
```

### Auto-Renewal Setup

#### Create renewal script `renew-cert.sh`

```bash
#!/bin/bash

# Renew certificates
docker compose run --rm certbot renew

# Reload nginx
docker compose exec nginx nginx -s reload

echo "Certificate renewal completed at $(date)"
```

#### Add to cron

```bash
# Make executable
chmod +x renew-cert.sh

# Add to crontab - run daily at 2:30 AM
crontab -e
30 2 * * * /path/to/renew-cert.sh >> /var/log/certbot-renewal.log 2>&1
```

## Option 2: Traefik + Let's Encrypt (Modern Alternative)

### Docker Compose with Traefik

```yaml
version: '3.8'

services:
  traefik:
    image: traefik:v2.10
    container_name: traefik
    restart: unless-stopped
    command:
      - "--api.dashboard=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.email=your-email@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik-certificates:/letsencrypt
    networks:
      - n8n-network

  postgres:
    image: postgres:15-alpine
    # ... postgres config ...

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - N8N_HOST=n8n.yourdomain.com
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.yourdomain.com/
    volumes:
      - n8n-data:/home/node/.n8n
    depends_on:
      - postgres
    networks:
      - n8n-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`n8n.yourdomain.com`)"
      - "traefik.http.routers.n8n.entrypoints=websecure"
      - "traefik.http.routers.n8n.tls.certresolver=letsencrypt"
      - "traefik.http.services.n8n.loadbalancer.server.port=5678"
      # HTTP to HTTPS redirect
      - "traefik.http.routers.n8n-http.rule=Host(`n8n.yourdomain.com`)"
      - "traefik.http.routers.n8n-http.entrypoints=web"
      - "traefik.http.routers.n8n-http.middlewares=redirect-to-https"
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"

volumes:
  traefik-certificates:
  postgres-data:
  n8n-data:

networks:
  n8n-network:
```

Traefik automatically:
- Obtains SSL certificates
- Renews certificates
- Handles HTTP to HTTPS redirect

## Option 3: Caddy (Simplest Option)

### Docker Compose with Caddy

```yaml
version: '3.8'

services:
  caddy:
    image: caddy:2-alpine
    container_name: n8n-caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy-data:/data
      - caddy-config:/config
    networks:
      - n8n-network

  n8n:
    # ... n8n config ...

volumes:
  caddy-data:
  caddy-config:
  n8n-data:

networks:
  n8n-network:
```

### Caddyfile

```
n8n.yourdomain.com {
    reverse_proxy n8n:5678
    encode gzip

    header {
        Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
        X-Frame-Options "SAMEORIGIN"
        X-Content-Type-Options "nosniff"
        X-XSS-Protection "1; mode=block"
    }
}
```

That's it! Caddy automatically obtains and renews SSL certificates.

## Security Headers Explained

### Strict-Transport-Security (HSTS)

```nginx
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

Forces browsers to use HTTPS for 2 years.

### X-Frame-Options

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
```

Prevents clickjacking attacks.

### X-Content-Type-Options

```nginx
add_header X-Content-Type-Options "nosniff" always;
```

Prevents MIME type sniffing.

### X-XSS-Protection

```nginx
add_header X-XSS-Protection "1; mode=block" always;
```

Enables browser XSS filtering.

### Referrer-Policy

```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Controls referrer information.

## Testing SSL Configuration

### Online Tools

1. **SSL Labs**: https://www.ssllabs.com/ssltest/
   - Comprehensive SSL/TLS analysis
   - Aim for A+ rating

2. **Security Headers**: https://securityheaders.com/
   - Check security headers
   - Aim for A rating

### Command Line Testing

```bash
# Test SSL certificate
openssl s_client -connect n8n.yourdomain.com:443

# Check certificate expiry
echo | openssl s_client -connect n8n.yourdomain.com:443 2>/dev/null | openssl x509 -noout -dates

# Verify HTTPS redirect
curl -I http://n8n.yourdomain.com

# Test HSTS header
curl -I https://n8n.yourdomain.com | grep -i strict
```

## Firewall Configuration

### UFW (Ubuntu)

```bash
# Allow HTTP and HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Block direct access to n8n port
sudo ufw deny 5678/tcp

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status
```

### iptables

```bash
# Allow HTTP and HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Block n8n port from external access
sudo iptables -A INPUT -p tcp --dport 5678 ! -i lo -j DROP

# Save rules
sudo netfilter-persistent save
```

## Troubleshooting

### Certificate Issues

#### Certificate not found

```bash
# Check certificate exists
docker compose exec nginx ls -la /etc/letsencrypt/live/n8n.yourdomain.com/

# Re-request if missing
docker compose run --rm certbot certonly --webroot --webroot-path=/var/www/html -d n8n.yourdomain.com
```

#### Certificate expired

```bash
# Force renewal
docker compose run --rm certbot renew --force-renewal

# Reload nginx
docker compose exec nginx nginx -s reload
```

### Connection Issues

#### Test reverse proxy

```bash
# Check nginx is running
docker compose ps nginx

# Check nginx logs
docker compose logs nginx

# Test nginx config
docker compose exec nginx nginx -t

# Reload nginx
docker compose exec nginx nginx -s reload
```

#### DNS not resolving

```bash
# Check DNS
dig n8n.yourdomain.com
nslookup n8n.yourdomain.com

# Test direct IP access (temporary)
curl -I http://YOUR_SERVER_IP
```

### Mixed Content Warnings

Ensure n8n knows it's behind HTTPS:

```bash
N8N_PROTOCOL=https
N8N_HOST=n8n.yourdomain.com
WEBHOOK_URL=https://n8n.yourdomain.com/
```

## Advanced Configuration

### Path-Based Routing

If running n8n under a path (e.g., /n8n):

```nginx
location /n8n/ {
    proxy_pass http://n8n:5678/;
    # ... other proxy settings ...
}
```

n8n configuration:
```bash
N8N_PATH=/n8n/
```

### Multiple Instances

```nginx
# Instance 1
server {
    server_name n8n-prod.yourdomain.com;
    location / {
        proxy_pass http://n8n-prod:5678;
    }
}

# Instance 2
server {
    server_name n8n-dev.yourdomain.com;
    location / {
        proxy_pass http://n8n-dev:5678;
    }
}
```

### Custom SSL Certificate

Using your own certificate:

```nginx
ssl_certificate /etc/nginx/ssl/your-cert.crt;
ssl_certificate_key /etc/nginx/ssl/your-key.key;
```

Mount certificates:
```yaml
volumes:
  - ./ssl:/etc/nginx/ssl:ro
```

## Monitoring SSL Certificates

### Expiry Monitoring Script

```bash
#!/bin/bash

DOMAIN="n8n.yourdomain.com"
ALERT_DAYS=30

EXPIRY_DATE=$(echo | openssl s_client -connect $DOMAIN:443 2>/dev/null | openssl x509 -noout -enddate | cut -d= -f2)
EXPIRY_EPOCH=$(date -d "$EXPIRY_DATE" +%s)
NOW_EPOCH=$(date +%s)
DAYS_LEFT=$(( ($EXPIRY_EPOCH - $NOW_EPOCH) / 86400 ))

if [ $DAYS_LEFT -lt $ALERT_DAYS ]; then
    echo "WARNING: SSL certificate expires in $DAYS_LEFT days!"
    # Send alert (email, Slack, etc.)
fi
```

Add to cron:
```bash
0 9 * * * /path/to/check-ssl-expiry.sh
```

## Best Practices Checklist

- [ ] Use Let's Encrypt for free SSL certificates
- [ ] Configure auto-renewal
- [ ] Implement HSTS header
- [ ] Add security headers
- [ ] Use TLS 1.2 and 1.3 only
- [ ] Generate strong DH parameters
- [ ] Configure OCSP stapling
- [ ] Redirect HTTP to HTTPS
- [ ] Test configuration with SSL Labs
- [ ] Monitor certificate expiry
- [ ] Keep reverse proxy updated
- [ ] Configure appropriate timeouts
- [ ] Limit client body size
- [ ] Enable gzip compression
- [ ] Configure proper logging

## Quick Reference

### Nginx Commands

```bash
# Test configuration
docker compose exec nginx nginx -t

# Reload configuration
docker compose exec nginx nginx -s reload

# View access logs
docker compose logs nginx

# Check certificate
docker compose exec nginx openssl x509 -in /etc/letsencrypt/live/n8n.yourdomain.com/cert.pem -text -noout
```

### Certbot Commands

```bash
# Renew all certificates
docker compose run --rm certbot renew

# Force renewal
docker compose run --rm certbot renew --force-renewal

# List certificates
docker compose run --rm certbot certificates

# Delete certificate
docker compose run --rm certbot delete --cert-name n8n.yourdomain.com
```

## Next Steps

After configuring SSL/TLS:
1. Set up user management (see `05-user-management-rbac.md`)
2. Configure queue mode for scaling (see `06-queue-mode-scaling.md`)
3. Implement comprehensive backups (see `07-backup-disaster-recovery.md`)
4. Review security practices (Week 12)

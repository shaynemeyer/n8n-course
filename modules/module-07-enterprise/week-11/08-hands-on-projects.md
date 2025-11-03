# Week 11 Hands-On Projects

## Overview

These hands-on projects will help you apply everything you've learned about self-hosting and administering n8n in production. Complete all three projects to gain practical experience.

## Project 1: Deploy n8n to a VPS or Cloud Provider

### Objective

Deploy a production-ready n8n instance to a cloud provider with Docker, PostgreSQL, SSL/TLS, and proper security.

### Requirements

**Infrastructure:**
- VPS or cloud instance (DigitalOcean, AWS, Linode, Vultr, etc.)
- Minimum 2GB RAM, 2 CPU cores, 20GB storage
- Ubuntu 22.04 LTS (recommended)
- Public IP address
- Domain name (or subdomain)

**Software:**
- Docker and Docker Compose
- PostgreSQL 15
- Nginx (reverse proxy)
- Let's Encrypt SSL certificate

**Configuration:**
- HTTPS enabled
- Environment variables properly configured
- Persistent volumes for data
- Basic monitoring

### Step-by-Step Guide

#### Phase 1: Server Setup (30 minutes)

1. **Create Server**
   ```bash
   # Example: DigitalOcean
   # - Create Droplet
   # - Choose Ubuntu 22.04
   # - Select 2GB/2CPU plan
   # - Add SSH key
   # - Create
   ```

2. **Initial Server Configuration**
   ```bash
   # SSH into server
   ssh root@your-server-ip

   # Update system
   apt update && apt upgrade -y

   # Create non-root user
   adduser n8n
   usermod -aG sudo n8n

   # Configure firewall
   ufw allow OpenSSH
   ufw allow 80/tcp
   ufw allow 443/tcp
   ufw enable

   # Switch to n8n user
   su - n8n
   ```

3. **Install Docker**
   ```bash
   curl -fsSL https://get.docker.com -o get-docker.sh
   sudo sh get-docker.sh
   sudo usermod -aG docker $USER

   # Install Docker Compose
   sudo apt install docker-compose-plugin -y

   # Verify
   docker --version
   docker compose version
   ```

#### Phase 2: DNS Configuration (10 minutes)

1. **Create DNS Record**
   - Go to your DNS provider
   - Add A record: `n8n.yourdomain.com` → `your-server-ip`
   - Wait for DNS propagation (5-15 minutes)

2. **Verify DNS**
   ```bash
   # On your local machine
   dig n8n.yourdomain.com
   nslookup n8n.yourdomain.com
   ```

#### Phase 3: n8n Deployment (45 minutes)

1. **Create Project Directory**
   ```bash
   mkdir -p ~/n8n
   cd ~/n8n
   ```

2. **Create docker-compose.yml**
   ```bash
   nano docker-compose.yml
   ```

   Paste:
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

     n8n:
       image: n8nio/n8n:latest
       container_name: n8n
       restart: unless-stopped
       ports:
         - "127.0.0.1:5678:5678"
       environment:
         - DB_TYPE=postgresdb
         - DB_POSTGRESDB_HOST=postgres
         - DB_POSTGRESDB_PORT=5432
         - DB_POSTGRESDB_DATABASE=n8n
         - DB_POSTGRESDB_USER=n8n
         - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
         - N8N_HOST=${N8N_HOST}
         - N8N_PROTOCOL=https
         - WEBHOOK_URL=https://${N8N_HOST}/
         - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
         - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
         - EXECUTIONS_DATA_PRUNE=true
         - EXECUTIONS_DATA_MAX_AGE=168
       volumes:
         - n8n-data:/home/node/.n8n
       depends_on:
         postgres:
           condition: service_healthy
       networks:
         - n8n-network

   volumes:
     postgres-data:
     n8n-data:

   networks:
     n8n-network:
   ```

3. **Create .env File**
   ```bash
   nano .env
   ```

   Paste:
   ```bash
   # Database
   POSTGRES_PASSWORD=CHANGE_ME_STRONG_PASSWORD

   # n8n
   N8N_HOST=n8n.yourdomain.com
   N8N_ENCRYPTION_KEY=CHANGE_ME_32_CHAR_KEY
   GENERIC_TIMEZONE=America/New_York
   ```

   Generate encryption key:
   ```bash
   openssl rand -base64 32
   ```

4. **Set Permissions**
   ```bash
   chmod 600 .env
   ```

#### Phase 4: SSL/TLS Setup (30 minutes)

1. **Install Nginx**
   ```bash
   sudo apt install nginx certbot python3-certbot-nginx -y
   ```

2. **Create Nginx Configuration**
   ```bash
   sudo nano /etc/nginx/sites-available/n8n
   ```

   Paste:
   ```nginx
   server {
       listen 80;
       server_name n8n.yourdomain.com;

       location / {
           proxy_pass http://localhost:5678;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection "upgrade";
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;

           proxy_connect_timeout 300;
           proxy_send_timeout 300;
           proxy_read_timeout 300;
           send_timeout 300;
       }
   }
   ```

3. **Enable Site**
   ```bash
   sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl restart nginx
   ```

4. **Obtain SSL Certificate**
   ```bash
   sudo certbot --nginx -d n8n.yourdomain.com
   ```

   Follow prompts:
   - Enter email
   - Agree to terms
   - Choose to redirect HTTP to HTTPS (2)

#### Phase 5: Start n8n (10 minutes)

1. **Start Services**
   ```bash
   cd ~/n8n
   docker compose up -d
   ```

2. **Check Logs**
   ```bash
   docker compose logs -f
   ```

   Wait for "Editor is now accessible"

3. **Verify**
   - Open browser: `https://n8n.yourdomain.com`
   - Create your owner account
   - Test with a simple workflow

#### Phase 6: Monitoring Setup (15 minutes)

1. **Create Monitoring Script**
   ```bash
   nano ~/n8n/monitor.sh
   ```

   Paste:
   ```bash
   #!/bin/bash

   echo "=== n8n Health Check ==="
   echo "Date: $(date)"
   echo ""

   echo "Container Status:"
   docker compose ps
   echo ""

   echo "Database Size:"
   docker exec n8n-postgres psql -U n8n -d n8n -c \
     "SELECT pg_size_pretty(pg_database_size('n8n'));"
   echo ""

   echo "Disk Usage:"
   df -h /
   echo ""

   echo "Memory Usage:"
   free -h
   ```

   ```bash
   chmod +x ~/n8n/monitor.sh
   ```

2. **Run Monitor**
   ```bash
   ./monitor.sh
   ```

### Deliverables

- [ ] Working n8n instance accessible via HTTPS
- [ ] SSL certificate installed (verify with SSL Labs)
- [ ] PostgreSQL database configured
- [ ] Docker Compose configuration file
- [ ] Environment variables file (secured)
- [ ] Basic monitoring script
- [ ] Documentation of deployment steps

### Testing Checklist

- [ ] Access https://n8n.yourdomain.com (no warnings)
- [ ] Create owner account
- [ ] Create test workflow with Schedule trigger
- [ ] Verify workflow executes
- [ ] Check execution history
- [ ] Restart server, verify n8n auto-starts
- [ ] Check logs for errors

---

## Project 2: Set Up Automated Backups

### Objective

Implement a comprehensive backup strategy with automated daily backups, offsite storage, and tested recovery procedures.

### Requirements

**Backups:**
- PostgreSQL database (daily)
- n8n data volume (daily)
- Configuration files (weekly)
- Retention: 30 days local, 90 days offsite

**Storage:**
- Local backup directory
- Offsite storage (S3, Backblaze B2, or similar)

**Monitoring:**
- Backup success/failure notifications
- n8n workflow for monitoring

### Step-by-Step Guide

#### Phase 1: Backup Scripts (45 minutes)

1. **Create Backup Directory**
   ```bash
   sudo mkdir -p /backup/{database,volumes,config}
   sudo chown -R n8n:n8n /backup
   ```

2. **Database Backup Script**
   ```bash
   nano ~/n8n/scripts/backup-database.sh
   ```

   Paste:
   ```bash
   #!/bin/bash

   BACKUP_DIR="/backup/database"
   DATE=$(date +%Y%m%d-%H%M%S)
   BACKUP_FILE="n8n-db-${DATE}.sql.gz"
   LOG_FILE="/var/log/n8n-backup.log"

   mkdir -p "$BACKUP_DIR"

   echo "[$(date)] Starting database backup..." | tee -a "$LOG_FILE"

   docker exec -t n8n-postgres pg_dump -U n8n n8n | \
     gzip > "$BACKUP_DIR/$BACKUP_FILE"

   if [ $? -eq 0 ]; then
       SIZE=$(du -h "$BACKUP_DIR/$BACKUP_FILE" | cut -f1)
       echo "[$(date)] Success: $BACKUP_FILE ($SIZE)" | tee -a "$LOG_FILE"

       # Cleanup old backups (30 days)
       find "$BACKUP_DIR" -name "n8n-db-*.sql.gz" -mtime +30 -delete
   else
       echo "[$(date)] ERROR: Backup failed!" | tee -a "$LOG_FILE"
       exit 1
   fi
   ```

   ```bash
   chmod +x ~/n8n/scripts/backup-database.sh
   ```

3. **n8n Data Backup Script**
   ```bash
   nano ~/n8n/scripts/backup-n8n-data.sh
   ```

   Paste:
   ```bash
   #!/bin/bash

   BACKUP_DIR="/backup/volumes"
   DATE=$(date +%Y%m%d-%H%M%S)
   BACKUP_FILE="n8n-data-${DATE}.tar.gz"

   mkdir -p "$BACKUP_DIR"

   echo "[$(date)] Backing up n8n data..."

   docker run --rm \
     -v n8n_n8n-data:/source:ro \
     -v $BACKUP_DIR:/backup \
     alpine tar czf /backup/$BACKUP_FILE -C /source .

   if [ $? -eq 0 ]; then
       echo "[$(date)] Success: $BACKUP_FILE"
       find "$BACKUP_DIR" -name "n8n-data-*.tar.gz" -mtime +30 -delete
   else
       echo "[$(date)] ERROR: Backup failed!"
       exit 1
   fi
   ```

   ```bash
   chmod +x ~/n8n/scripts/backup-n8n-data.sh
   ```

4. **Complete Backup Script**
   ```bash
   nano ~/n8n/scripts/backup-all.sh
   ```

   Paste:
   ```bash
   #!/bin/bash

   echo "========================================="
   echo "n8n Complete Backup - $(date)"
   echo "========================================="

   # Database
   /home/n8n/n8n/scripts/backup-database.sh

   # n8n Data
   /home/n8n/n8n/scripts/backup-n8n-data.sh

   # Configuration (weekly)
   if [ $(date +%u) -eq 7 ]; then
       echo "Backing up configuration..."
       cp -r /home/n8n/n8n /backup/config/n8n-config-$(date +%Y%m%d)
   fi

   echo "========================================="
   echo "Backup Complete"
   echo "========================================="
   ```

   ```bash
   chmod +x ~/n8n/scripts/backup-all.sh
   ```

#### Phase 2: Automated Scheduling (15 minutes)

1. **Setup Cron**
   ```bash
   crontab -e
   ```

   Add:
   ```cron
   # Daily backup at 2 AM
   0 2 * * * /home/n8n/n8n/scripts/backup-all.sh >> /var/log/n8n-backup.log 2>&1

   # Additional database backup every 6 hours
   0 */6 * * * /home/n8n/n8n/scripts/backup-database.sh >> /var/log/n8n-backup.log 2>&1
   ```

2. **Create Log File**
   ```bash
   sudo touch /var/log/n8n-backup.log
   sudo chown n8n:n8n /var/log/n8n-backup.log
   ```

#### Phase 3: Offsite Backup (30 minutes)

**Option A: AWS S3**

1. **Install AWS CLI**
   ```bash
   sudo apt install awscli -y
   aws configure
   ```

2. **Create Upload Script**
   ```bash
   nano ~/n8n/scripts/upload-to-s3.sh
   ```

   Paste:
   ```bash
   #!/bin/bash

   BUCKET="your-bucket/n8n-backups"
   BACKUP_DIR="/backup"

   echo "Uploading backups to S3..."

   # Upload latest database backup
   LATEST_DB=$(ls -t $BACKUP_DIR/database/n8n-db-*.sql.gz | head -1)
   aws s3 cp "$LATEST_DB" "s3://$BUCKET/database/"

   # Upload latest data backup
   LATEST_DATA=$(ls -t $BACKUP_DIR/volumes/n8n-data-*.tar.gz | head -1)
   aws s3 cp "$LATEST_DATA" "s3://$BUCKET/volumes/"

   echo "Upload complete"
   ```

**Option B: Backblaze B2**

1. **Install B2 CLI**
   ```bash
   sudo pip3 install b2
   b2 authorize-account <keyId> <applicationKey>
   ```

2. **Create Upload Script**
   ```bash
   nano ~/n8n/scripts/upload-to-b2.sh
   ```

   Similar to S3 script, using `b2 upload-file`

#### Phase 4: Backup Monitoring (30 minutes)

1. **Create n8n Backup Monitor Workflow**

   In n8n, create new workflow:

   - **Schedule Trigger**: Daily at 9 AM
   - **Execute Command Node**: Check last backup
     ```bash
     ls -lt /backup/database/n8n-db-*.sql.gz | head -1
     ```
   - **Function Node**: Parse backup age
     ```javascript
     const lastBackup = $input.item.json.stdout;
     const backupDate = lastBackup.match(/n8n-db-(\d{8})/)[1];
     const now = new Date();
     const backupTime = new Date(
       backupDate.substr(0,4),
       backupDate.substr(4,2)-1,
       backupDate.substr(6,2)
     );
     const ageHours = (now - backupTime) / (1000 * 60 * 60);

     return {
       json: {
         ageHours,
         status: ageHours < 24 ? 'OK' : 'ALERT'
       }
     };
     ```
   - **IF Node**: Check if alert needed
   - **Email/Slack Node**: Send alert if backup is old

2. **Test Workflow**
   - Execute manually
   - Verify alert logic
   - Activate workflow

#### Phase 5: Recovery Testing (45 minutes)

1. **Create Test Workflow**
   - Create a simple workflow in n8n
   - Name it "Test Recovery Workflow"
   - Activate it

2. **Perform Backup**
   ```bash
   ~/n8n/scripts/backup-all.sh
   ```

3. **Simulate Disaster**
   ```bash
   cd ~/n8n
   docker compose down -v  # WARNING: Deletes all data!
   ```

4. **Restore from Backup**
   ```bash
   # Start database
   docker compose up -d postgres
   sleep 10

   # Restore database
   LATEST_BACKUP=$(ls -t /backup/database/n8n-db-*.sql.gz | head -1)
   gunzip < "$LATEST_BACKUP" | \
     docker exec -i n8n-postgres psql -U n8n -d n8n

   # Restore n8n data
   LATEST_DATA=$(ls -t /backup/volumes/n8n-data-*.tar.gz | head -1)
   docker run --rm \
     -v n8n_n8n-data:/target \
     -v /backup/volumes:/backup \
     alpine sh -c "cd /target && tar xzf /backup/$(basename $LATEST_DATA)"

   # Start n8n
   docker compose up -d
   ```

5. **Verify Recovery**
   - Access n8n
   - Check "Test Recovery Workflow" exists
   - Verify credentials work
   - Execute a workflow

6. **Document Recovery Time**
   - Record how long recovery took
   - Document any issues encountered

### Deliverables

- [ ] Automated backup scripts
- [ ] Cron jobs configured
- [ ] Offsite backup configured
- [ ] Backup monitoring workflow
- [ ] Recovery procedures documented
- [ ] Successful recovery test completed
- [ ] Recovery time documented

### Testing Checklist

- [ ] Manual backup runs successfully
- [ ] Automated backup runs at scheduled time
- [ ] Backup files created in /backup
- [ ] Offsite upload successful
- [ ] Old backups cleaned up (>30 days)
- [ ] Monitoring workflow alerts on missing backup
- [ ] Full recovery tested and documented

---

## Project 3: Configure Multi-User Environment

### Objective

Set up a multi-user n8n environment with proper access controls, shared workflows, and credential management.

### Requirements

**Users:**
- 1 Owner (you)
- 2 Admins
- 3 Members

**Features:**
- User management enabled
- SMTP configured for invitations
- Shared workflows
- Shared credentials
- Role-based access demonstrated

### Step-by-Step Guide

#### Phase 1: Enable User Management (20 minutes)

1. **Update Environment**
   ```bash
   cd ~/n8n
   nano .env
   ```

   Add:
   ```bash
   # User Management
   N8N_BASIC_AUTH_ACTIVE=false
   N8N_USER_MANAGEMENT_DISABLED=false

   # JWT
   N8N_JWT_AUTH_ACTIVE=true

   # SMTP (Gmail example)
   N8N_EMAIL_MODE=smtp
   N8N_SMTP_HOST=smtp.gmail.com
   N8N_SMTP_PORT=587
   N8N_SMTP_USER=your-email@gmail.com
   N8N_SMTP_PASS=your-app-password
   N8N_SMTP_SENDER=n8n@yourdomain.com
   ```

2. **Restart n8n**
   ```bash
   docker compose down
   docker compose up -d
   ```

3. **Create Owner Account**
   - Access n8n
   - You'll see setup page
   - Create owner account
   - Save credentials securely

#### Phase 2: Add Users (30 minutes)

1. **Invite Admins**
   - Go to Settings → Users
   - Click "Invite User"
   - Enter email
   - Select role: Admin
   - Send invitation
   - Repeat for second admin

2. **Invite Members**
   - Same process
   - Select role: Member
   - Invite 3 members

3. **Document Users**
   Create `users.md`:
   ```markdown
   # n8n Users

   ## Owner
   - Email: owner@example.com
   - Role: Owner

   ## Admins
   - Email: admin1@example.com
   - Email: admin2@example.com

   ## Members
   - Email: member1@example.com
   - Email: member2@example.com
   - Email: member3@example.com
   ```

#### Phase 3: Create Shared Resources (45 minutes)

1. **Create Team Workflows**

   **Workflow 1: Daily Health Check** (Share with All)
   - Schedule: Daily 9 AM
   - HTTP Request to server
   - IF server is down → Email alert
   - Share with: All Members

   **Workflow 2: User Onboarding** (Share with Admins)
   - Webhook trigger
   - Create accounts
   - Send welcome email
   - Share with: Admins only

   **Workflow 3: Data Processing** (Share with specific members)
   - Manual trigger
   - Process data
   - Save to database
   - Share with: member1 and member2

2. **Create Shared Credentials**

   **Gmail Credential** (team account)
   - Create credential
   - Share with: All Members
   - Use for: Email notifications

   **Database Credential**
   - Create credential
   - Share with: Admins only
   - Use for: Data operations

   **API Credential**
   - Create credential
   - Share with: member1 only
   - Use for: Specific integration

#### Phase 4: Documentation (30 minutes)

1. **Create User Guide**
   ```bash
   nano ~/n8n/USER_GUIDE.md
   ```

   Paste:
   ```markdown
   # n8n User Guide

   ## Roles and Permissions

   ### Owner
   - Full system access
   - Can manage all users
   - Can access all workflows

   ### Admin
   - Can invite/manage users
   - Can access all workflows
   - Can manage credentials

   ### Member
   - Can create workflows
   - Can use shared credentials
   - Can view shared workflows

   ## Shared Workflows

   ### Daily Health Check
   - Purpose: Monitor system health
   - Access: All users
   - Trigger: Daily 9 AM

   ### User Onboarding
   - Purpose: Automated user setup
   - Access: Admins only
   - Trigger: Webhook

   ## Shared Credentials

   ### Team Gmail
   - Purpose: Send notifications
   - Access: All members
   - Usage: Email nodes

   ## Best Practices

   1. Don't share personal credentials
   2. Use team accounts for shared resources
   3. Request access before using others' workflows
   4. Document your workflows
   ```

2. **Create Access Control Policy**
   ```bash
   nano ~/n8n/ACCESS_POLICY.md
   ```

   Document:
   - Who can create workflows
   - Credential sharing rules
   - Workflow naming conventions
   - Data handling policies

#### Phase 5: Testing (30 minutes)

1. **Test Each Role**
   - Log in as Admin: Verify can manage users
   - Log in as Member: Verify cannot manage users
   - Test workflow sharing
   - Test credential sharing

2. **Test Scenarios**
   - Member creates workflow
   - Member uses shared credential
   - Admin shares credential with specific users
   - User leaves (deactivate and verify access removed)

### Deliverables

- [ ] User management enabled
- [ ] 6 users created (1 owner, 2 admins, 3 members)
- [ ] SMTP configured and working
- [ ] 3 workflows with different sharing levels
- [ ] 3 credentials with different sharing levels
- [ ] User guide documented
- [ ] Access control policy documented
- [ ] All roles tested

### Testing Checklist

- [ ] Owner can access everything
- [ ] Admin can invite users
- [ ] Admin can manage all workflows
- [ ] Member cannot access admin functions
- [ ] Member can use shared credentials
- [ ] Member can view shared workflows
- [ ] Member can edit workflows shared with edit permission
- [ ] Member cannot edit view-only workflows
- [ ] Email invitations working
- [ ] Password reset working (test)

---

## Submission Guidelines

### What to Submit

For each project, provide:

1. **Screenshots**
   - n8n running and accessible
   - SSL certificate (valid HTTPS)
   - Docker containers running
   - Backup files created
   - Users list
   - Workflow sharing settings

2. **Documentation**
   - Deployment steps taken
   - Any issues encountered and solutions
   - Configuration files (sanitized, no passwords)
   - Recovery test results
   - User guide and policies

3. **Verification**
   - SSL Labs test results
   - Backup verification output
   - Recovery time documentation
   - User testing results

### Evaluation Criteria

Each project will be evaluated on:

- **Completeness**: All requirements met
- **Security**: Proper configuration and best practices
- **Documentation**: Clear and thorough
- **Testing**: Evidence of testing provided
- **Best Practices**: Following industry standards

## Bonus Challenges

If you complete all three projects, try these:

1. **Advanced Monitoring**
   - Set up Prometheus + Grafana
   - Create custom dashboards
   - Configure alerts

2. **High Availability**
   - Deploy with queue mode
   - Multiple workers
   - Load balancer

3. **Automation**
   - Automated user onboarding workflow
   - Automated backup verification
   - Infrastructure as Code (Terraform/Ansible)

## Resources

- [n8n Documentation](https://docs.n8n.io)
- [Docker Documentation](https://docs.docker.com)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [Let's Encrypt](https://letsencrypt.org/docs/)

## Getting Help

If you encounter issues:

1. Check logs: `docker compose logs`
2. Review course materials
3. Search n8n community forum
4. Ask in course discussion board
5. Check project documentation in previous lessons

## Conclusion

Completing these three projects will give you hands-on experience with production n8n deployment, backup/recovery, and multi-user administration. Take your time, document everything, and don't hesitate to experiment!

Good luck!

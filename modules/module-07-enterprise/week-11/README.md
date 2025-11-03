# Week 11: Self-Hosting and Administration

## Overview

This week focuses on deploying and managing n8n in production environments. You'll learn how to self-host n8n, configure it for enterprise use, manage users, implement backup strategies, and scale your deployment for high availability.

## Learning Objectives

By the end of this week, you will be able to:

- Deploy n8n in production using Docker and other methods
- Configure environment variables for optimal performance
- Set up and manage database backends
- Configure SSL/TLS for secure communications
- Implement user management and role-based access control (RBAC)
- Deploy queue mode for horizontal scaling
- Create comprehensive backup and disaster recovery plans
- Monitor and maintain production n8n instances

## Topics Covered

1. **Docker Deployment Best Practices**
   - Single container vs multi-container setups
   - Docker Compose configurations
   - Volume management and data persistence
   - Network configuration
   - Resource limits and constraints

2. **Environment Configuration**
   - Essential environment variables
   - Performance tuning parameters
   - Security settings
   - Webhook and API endpoints
   - Email configuration

3. **Database Setup and Maintenance**
   - PostgreSQL configuration (recommended)
   - MySQL/MariaDB alternatives
   - Database migrations
   - Connection pooling
   - Backup strategies
   - Performance optimization

4. **SSL/TLS Configuration**
   - Certificate management
   - Reverse proxy setup (Nginx, Traefik)
   - Let's Encrypt integration
   - HTTPS enforcement
   - Security headers

5. **User Management and RBAC**
   - Multi-user configuration
   - Role-based access control
   - Credential sharing policies
   - Workflow permissions
   - Audit logging

6. **Queue Mode for Scaling**
   - Understanding queue mode architecture
   - Redis configuration
   - Worker deployment
   - Horizontal scaling strategies
   - Load balancing

7. **Backup and Disaster Recovery**
   - Database backup strategies
   - Workflow export/import
   - Credential backup considerations
   - Recovery procedures
   - Testing backup integrity

## Week Structure

### Day 1-2: Docker Deployment
- Learn Docker best practices for n8n
- Deploy n8n using Docker Compose
- Configure persistent volumes
- Set up basic monitoring

### Day 3: Environment and Database Configuration
- Configure essential environment variables
- Set up PostgreSQL database
- Optimize database performance
- Implement connection pooling

### Day 4: SSL/TLS and Security
- Configure reverse proxy
- Obtain and install SSL certificates
- Implement security headers
- Test secure connections

### Day 5: User Management
- Enable multi-user mode
- Configure RBAC
- Set up authentication providers
- Manage user permissions

### Day 6: Scaling with Queue Mode
- Set up Redis for queue management
- Deploy worker nodes
- Configure load balancing
- Test high-availability setup

### Day 7: Backup and Recovery
- Implement automated backups
- Create disaster recovery plan
- Test recovery procedures
- Document maintenance procedures

## Hands-On Projects

### Project 1: Deploy n8n to a VPS or Cloud Provider

Deploy a production-ready n8n instance to a cloud provider (AWS, DigitalOcean, Linode, etc.).

**Requirements:**
- Use Docker Compose
- PostgreSQL database
- SSL/TLS enabled
- Persistent volumes for data
- Basic monitoring setup

**Deliverables:**
- Working n8n instance accessible via HTTPS
- Documentation of deployment steps
- Docker Compose configuration file

### Project 2: Set Up Automated Backups

Implement a comprehensive backup strategy for your n8n instance.

**Requirements:**
- Automated database backups (daily)
- Workflow export automation
- Offsite backup storage (S3, Backblaze, etc.)
- Backup verification workflow
- Recovery testing documentation

**Deliverables:**
- Backup automation workflow
- Recovery procedure documentation
- Test recovery results

### Project 3: Configure Multi-User Environment

Set up a multi-user n8n environment with proper access controls.

**Requirements:**
- Multiple user accounts with different roles
- Credential sharing policies
- Workflow permission management
- Audit logging enabled
- User onboarding documentation

**Deliverables:**
- Configured multi-user instance
- User management documentation
- Security policies document

## Additional Resources

### Official Documentation
- [n8n Self-Hosting Guide](https://docs.n8n.io/hosting/)
- [Docker Deployment](https://docs.n8n.io/hosting/installation/docker/)
- [Environment Variables Reference](https://docs.n8n.io/hosting/environment-variables/)
- [Queue Mode Setup](https://docs.n8n.io/hosting/scaling/queue-mode/)

### Recommended Reading
- Docker best practices documentation
- PostgreSQL performance tuning guides
- Nginx reverse proxy configuration
- Let's Encrypt documentation

### Tools and Services
- Docker and Docker Compose
- PostgreSQL
- Redis (for queue mode)
- Nginx or Traefik (reverse proxy)
- Let's Encrypt (SSL certificates)
- Monitoring: Prometheus, Grafana, or similar

## Prerequisites

Before starting this week:
- Familiarity with command line/terminal
- Basic Docker knowledge
- Understanding of databases (PostgreSQL preferred)
- Access to a VPS or cloud provider account
- Domain name (optional but recommended for SSL)

## Assessment Criteria

You should be able to:
- Deploy n8n in a production environment
- Configure all essential environment variables correctly
- Set up and maintain a PostgreSQL database
- Implement SSL/TLS for secure access
- Manage multiple users with appropriate permissions
- Configure and deploy queue mode for scaling
- Create and test backup/recovery procedures
- Monitor and troubleshoot production issues

## Common Pitfalls to Avoid

- Forgetting to configure persistent volumes (data loss risk)
- Using SQLite in production (not recommended for multi-user)
- Exposing n8n without SSL/TLS
- Not setting up regular backups
- Ignoring resource limits in Docker
- Hardcoding credentials in environment files
- Not testing recovery procedures
- Insufficient monitoring and logging

## Tips for Success

- Start with a simple deployment and iterate
- Document every configuration change
- Test backups regularly
- Monitor resource usage from day one
- Use environment variables for all configuration
- Follow security best practices
- Join the n8n community for deployment help
- Keep your n8n instance updated

## Next Steps

After completing this week, you'll be ready to move on to Week 12: Security and Compliance, where you'll learn advanced security practices and compliance considerations for enterprise deployments.

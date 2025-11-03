# User Management and Role-Based Access Control (RBAC)

## Introduction

n8n's user management and RBAC features allow multiple users to collaborate securely on workflows. This guide covers setting up multi-user environments, managing permissions, and implementing access control policies.

## User Management Overview

### Available in:
- n8n Cloud (all plans)
- Self-hosted with proper configuration

### Key Features:
- Multiple user accounts
- Role-based permissions
- Workflow sharing
- Credential sharing
- Audit logging

## Enabling User Management

### Configuration

```bash
# Disable basic auth (required for user management)
N8N_BASIC_AUTH_ACTIVE=false

# Enable user management
N8N_USER_MANAGEMENT_DISABLED=false

# JWT configuration (optional but recommended)
N8N_JWT_AUTH_ACTIVE=true
N8N_JWT_AUTH_HEADER=Authorization
```

### Complete Environment Setup

```bash
# ================================
# User Management
# ================================
N8N_BASIC_AUTH_ACTIVE=false
N8N_USER_MANAGEMENT_DISABLED=false

# JWT Settings
N8N_JWT_AUTH_ACTIVE=true
N8N_JWT_AUTH_HEADER=Authorization

# Encryption (CRITICAL!)
N8N_ENCRYPTION_KEY=your_encryption_key_here

# Email for invitations
N8N_EMAIL_MODE=smtp
N8N_SMTP_HOST=smtp.gmail.com
N8N_SMTP_PORT=587
N8N_SMTP_USER=your-email@gmail.com
N8N_SMTP_PASS=your-app-password
N8N_SMTP_SENDER=n8n@yourdomain.com

# Public URL (required for invitations)
N8N_HOST=n8n.yourdomain.com
N8N_PROTOCOL=https
WEBHOOK_URL=https://n8n.yourdomain.com/
```

## User Roles and Permissions

### Role Types

#### 1. Owner

**Permissions:**
- Full system administration
- Manage all users
- Access all workflows
- Manage all credentials
- Configure system settings
- Delete workflows and data

**Limitations:**
- Only one owner per instance
- Cannot be deleted (can transfer ownership)

#### 2. Admin

**Permissions:**
- Invite and manage users
- Access all workflows
- Manage all credentials
- Configure most settings

**Limitations:**
- Cannot delete the owner
- Cannot change instance-level settings

#### 3. Member

**Permissions:**
- Create and edit own workflows
- View shared workflows
- Use shared credentials
- Create personal credentials

**Limitations:**
- Cannot access other users' private workflows
- Cannot manage users
- Cannot access credentials not shared with them

### Permission Matrix

| Action | Owner | Admin | Member |
|--------|-------|-------|--------|
| Invite users | ✅ | ✅ | ❌ |
| Delete users | ✅ | ✅ | ❌ |
| Change user roles | ✅ | ✅ | ❌ |
| Create workflows | ✅ | ✅ | ✅ |
| Edit own workflows | ✅ | ✅ | ✅ |
| View all workflows | ✅ | ✅ | ❌ |
| Delete any workflow | ✅ | ✅ | ❌ |
| Create credentials | ✅ | ✅ | ✅ |
| View all credentials | ✅ | ✅ | ❌ |
| Share credentials | ✅ | ✅ | ✅* |
| System settings | ✅ | ⚠️ | ❌ |

\* Can only share own credentials

## Initial Setup

### Creating the First User (Owner)

When user management is enabled and no users exist:

1. Navigate to your n8n URL
2. You'll see the setup page
3. Create the owner account:
   - Email address
   - First name
   - Last name
   - Password (strong password required)

**Important:** Save these credentials securely!

### Adding Additional Users

#### Method 1: Email Invitation (Recommended)

1. Click **Settings** → **Users**
2. Click **Invite User**
3. Enter email address
4. Select role (Admin or Member)
5. Click **Send Invitation**

The user receives an email with:
- Invitation link (expires in 7 days)
- Instructions to set password

#### Method 2: Direct Link

If email isn't configured:

1. Click **Settings** → **Users**
2. Click **Invite User**
3. Copy the invitation link
4. Manually send to user
5. User clicks link to complete signup

### User Onboarding Workflow

Create an n8n workflow to automate user onboarding:

```
Webhook (User Invited)
  ↓
Send Email (Welcome message)
  ↓
Slack (Notify team)
  ↓
Notion (Add to team database)
```

## Workflow Sharing

### Sharing Levels

#### 1. Private (Default)

- Only creator can access
- Not visible to other users

#### 2. Shared with specific users

- Select users to share with
- Can view and/or edit
- Controlled collaboration

#### 3. Shared with role

- Share with all Admins or Members
- Automatic access for new users in role

### How to Share a Workflow

1. Open workflow
2. Click **Share** button (top right)
3. Choose sharing option:
   - **Add User**: Share with specific users
   - **Add Role**: Share with role (Admin/Member)
4. Set permission level:
   - **Can view**: Read-only access
   - **Can edit**: Full edit access
5. Click **Save**

### Workflow Sharing Best Practices

**Do:**
- Share templates with all Members
- Keep sensitive workflows private
- Use descriptive workflow names
- Document shared workflows
- Review sharing permissions regularly

**Don't:**
- Share unnecessarily
- Give edit access to all workflows
- Share workflows with credentials you don't want to share

## Credential Sharing

### Credential Ownership

- Creator owns the credential
- Can share with specific users or roles
- Credential data always encrypted

### Sharing Credentials

1. Go to **Credentials**
2. Select credential
3. Click **Sharing** tab
4. Add users or roles
5. Save

### Credential Sharing Strategies

#### Strategy 1: Shared Service Accounts

```
Gmail (team@company.com)
  Shared with: All Members
  Use for: General email sending

Slack (company workspace)
  Shared with: All Members
  Use for: Notifications
```

#### Strategy 2: Department Credentials

```
Salesforce (sales team)
  Shared with: Sales Members
  Use for: CRM operations

AWS (engineering)
  Shared with: Engineering Admins
  Use for: Infrastructure automation
```

#### Strategy 3: Personal Credentials

```
Personal Gmail
  Shared with: None
  Use for: Individual tasks
```

### Credential Security Best Practices

- Only share credentials when necessary
- Use service accounts, not personal accounts
- Regularly audit credential sharing
- Remove access when users leave
- Use least-privilege principle
- Document credential usage

## User Administration

### Managing Existing Users

#### Change User Role

1. **Settings** → **Users**
2. Click on user
3. Change role dropdown
4. Confirm change

#### Deactivate User

1. **Settings** → **Users**
2. Click on user
3. Click **Deactivate**
4. Confirm

**Effects:**
- User cannot log in
- Workflows remain
- Can reactivate later

#### Delete User

1. **Settings** → **Users**
2. Click on user
3. Click **Delete**
4. Choose what to do with workflows:
   - **Transfer to another user**
   - **Delete all workflows**
5. Confirm

**Warning:** Deletion is permanent!

### Transfer Ownership

Only the current owner can transfer:

1. **Settings** → **Users**
2. Click on new owner
3. Click **Make Owner**
4. Confirm

Current owner becomes Admin.

## Security Policies

### Password Policy

Configure password requirements:

```bash
# Minimum password length
N8N_USER_MANAGEMENT_MINIMUM_PASSWORD_LENGTH=8

# Require special characters, numbers, etc.
# (Handled in UI settings)
```

### Session Management

```bash
# JWT expiration (seconds)
N8N_JWT_EXPIRATION_TIME=86400  # 24 hours

# Refresh token expiration
N8N_JWT_REFRESH_TIMEOUT=604800  # 7 days
```

### Account Security

#### Enforce Strong Passwords

- Minimum 12 characters
- Mix of upper/lowercase
- Include numbers
- Include special characters

#### Two-Factor Authentication

Currently not available in self-hosted n8n.
Available in n8n Cloud.

Consider:
- Reverse proxy 2FA (Nginx + auth module)
- SSO/SAML integration
- IP allowlisting

## Single Sign-On (SSO)

### SAML Configuration

n8n supports SAML for enterprise SSO:

```bash
# Enable SAML
N8N_SAML_ENABLED=true

# SAML Settings
N8N_SAML_LOGIN_ENABLED=true
N8N_SAML_LOGIN_LABEL=Login with SSO
N8N_SAML_METADATA_URL=https://your-idp.com/metadata.xml

# Or provide metadata directly
N8N_SAML_METADATA=<EntityDescriptor...>
```

### Supported Identity Providers

- Okta
- Azure AD
- Google Workspace
- OneLogin
- Auth0
- Custom SAML providers

### SAML Setup Steps

1. Configure IdP (e.g., Okta)
2. Get SAML metadata URL
3. Configure n8n environment
4. Test SSO login
5. Map user attributes
6. Roll out to users

## Audit Logging

### Available Logs

n8n logs user actions:
- User login/logout
- Workflow creation/modification
- Credential access
- User management changes
- Execution history

### Accessing Logs

#### Database Queries

```sql
-- User login history
SELECT
    u.email,
    el.event_name,
    el.timestamp
FROM execution_log el
JOIN user u ON el.user_id = u.id
WHERE el.event_name = 'user.login'
ORDER BY el.timestamp DESC;

-- Workflow modifications
SELECT
    u.email,
    w.name as workflow_name,
    w.updated_at
FROM workflow_entity w
JOIN user u ON w.updated_by = u.id
ORDER BY w.updated_at DESC
LIMIT 100;
```

#### Export Audit Log

Create an n8n workflow:

```
Schedule (Daily)
  ↓
Postgres (Query audit events)
  ↓
Filter (Process logs)
  ↓
Google Sheets (Log to sheet)
  ↓
Email (Send daily report)
```

## User Provisioning Automation

### Automated User Onboarding

Create workflow to automatically provision users:

```
Webhook (HR system)
  ↓
HTTP Request (Invite user to n8n)
  ↓
Set Credentials (Create default credentials)
  ↓
Assign to Workflows (Share default workflows)
  ↓
Send Email (Welcome email)
  ↓
Slack (Notify team)
```

### Automated Offboarding

```
Webhook (User deactivated in HR)
  ↓
HTTP Request (Deactivate n8n user)
  ↓
Transfer Workflows (Reassign to manager)
  ↓
Revoke Credentials (Remove access)
  ↓
Archive Data (Backup user data)
  ↓
Email (Notify admin)
```

## Multi-Tenant Considerations

### Separate Instances

For true multi-tenancy:

- Deploy separate n8n instance per tenant
- Separate databases
- Separate domains
- Complete isolation

### Shared Instance with RBAC

For team separation:

- Use roles to separate teams
- Workflow sharing controls
- Credential sharing policies
- Naming conventions

Example naming:
```
[TEAM] Workflow Name
[SALES] Lead Processing
[MARKETING] Email Campaign
[FINANCE] Invoice Generation
```

## Troubleshooting

### Users Can't Log In

```bash
# Check user management is enabled
echo $N8N_USER_MANAGEMENT_DISABLED  # Should be 'false'

# Check basic auth is disabled
echo $N8N_BASIC_AUTH_ACTIVE  # Should be 'false'

# Check JWT is configured
echo $N8N_JWT_AUTH_ACTIVE  # Should be 'true'

# Check database
docker compose exec postgres psql -U n8n -d n8n -c "SELECT email, role FROM user;"
```

### Invitation Emails Not Sending

```bash
# Verify SMTP settings
echo $N8N_EMAIL_MODE  # Should be 'smtp'
echo $N8N_SMTP_HOST
echo $N8N_SMTP_PORT
echo $N8N_SMTP_USER

# Test SMTP
docker compose exec n8n sh -c 'apk add mailx && echo "test" | mail -s "test" test@example.com'
```

### Credentials Not Accessible

```bash
# Check encryption key is set
echo $N8N_ENCRYPTION_KEY

# Verify credential sharing in database
docker compose exec postgres psql -U n8n -d n8n -c "
SELECT
    c.name,
    c.type,
    u.email as owner
FROM credentials_entity c
JOIN user u ON c.user_id = u.id;
"
```

## Best Practices Checklist

### Setup
- [ ] Disable basic auth
- [ ] Enable user management
- [ ] Configure SMTP for invitations
- [ ] Set strong password policy
- [ ] Configure session timeouts
- [ ] Document access control policies

### User Management
- [ ] Use descriptive user roles
- [ ] Follow least-privilege principle
- [ ] Regular access reviews
- [ ] Offboard users promptly
- [ ] Document role responsibilities
- [ ] Maintain user inventory

### Workflow Sharing
- [ ] Share thoughtfully
- [ ] Use appropriate permission levels
- [ ] Document shared workflows
- [ ] Regular sharing audits
- [ ] Clear naming conventions

### Credential Sharing
- [ ] Use service accounts
- [ ] Share minimally
- [ ] Regular credential rotation
- [ ] Audit credential access
- [ ] Document credential purposes
- [ ] Remove access when not needed

### Security
- [ ] Monitor user activity
- [ ] Review audit logs
- [ ] Implement offboarding workflow
- [ ] Regular security reviews
- [ ] Keep n8n updated
- [ ] Backup user data

## Quick Reference

### Common User Management Tasks

```bash
# Check current users
docker compose exec postgres psql -U n8n -d n8n -c \
  "SELECT id, email, role, created_at FROM user;"

# Count workflows by user
docker compose exec postgres psql -U n8n -d n8n -c \
  "SELECT u.email, COUNT(w.id) as workflow_count
   FROM user u
   LEFT JOIN workflow_entity w ON u.id = w.user_id
   GROUP BY u.email;"

# Find unused credentials
docker compose exec postgres psql -U n8n -d n8n -c \
  "SELECT c.name, c.type
   FROM credentials_entity c
   WHERE c.id NOT IN
     (SELECT DISTINCT credential_id FROM workflow_credential);"
```

## Next Steps

After setting up user management:
1. Configure queue mode for scaling (see `06-queue-mode-scaling.md`)
2. Implement backup strategies (see `07-backup-disaster-recovery.md`)
3. Review security and compliance (Week 12)
4. Create user onboarding documentation
5. Establish access control policies

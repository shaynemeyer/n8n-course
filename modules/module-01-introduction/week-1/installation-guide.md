# n8n Installation Guide

This guide provides detailed instructions for installing n8n using different methods.

## Prerequisites

- **For npm/npx**: Node.js 18.10 or later
- **For Docker**: Docker Desktop or Docker Engine installed
- **For cloud**: Just a web browser!

## Method 1: n8n Cloud (Recommended for Beginners)

The easiest way to get started with n8n.

### Steps:

1. Navigate to [https://n8n.cloud](https://n8n.cloud)
2. Click "Start Free Trial"
3. Create an account with email or social login
4. Choose a workspace name
5. Start building workflows immediately

### Pros:
- No local setup required
- Automatic updates
- Built-in SSL certificates
- Managed backups
- 24/7 uptime

### Cons:
- Requires internet connection
- Usage limits on free tier
- Monthly cost for production use

---

## Method 2: npx (Quick Start)

Perfect for trying out n8n without installation.

### Steps:

```bash
npx n8n
```

That's it! n8n will start and open in your browser at `http://localhost:5678`

### Pros:
- No installation needed
- Always uses latest version
- Quick to start

### Cons:
- Data is not persisted between sessions
- Slower startup time
- Requires internet for first run

---

## Method 3: npm Global Installation

Best for local development with persistent data.

### Steps:

1. **Install n8n globally:**
   ```bash
   npm install n8n -g
   ```

2. **Start n8n:**
   ```bash
   n8n start
   ```

3. **Access n8n:**
   Open `http://localhost:5678` in your browser

### Configuration:

Set environment variables before starting:

```bash
export N8N_BASIC_AUTH_ACTIVE=true
export N8N_BASIC_AUTH_USER=admin
export N8N_BASIC_AUTH_PASSWORD=your_password
n8n start
```

### Data Location:

By default, n8n stores data in:
- **Mac/Linux**: `~/.n8n`
- **Windows**: `%USERPROFILE%\.n8n`

### Pros:
- Persistent data storage
- Quick startup after first install
- Full control over configuration

### Cons:
- Requires Node.js knowledge
- Manual updates needed
- Shares Node.js environment

---

## Method 4: Docker (Recommended for Production)

Best for production deployments and isolated environments.

### Quick Start:

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

### Production Setup with Docker Compose:

Create a `docker-compose.yml` file:

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=changeme
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${N8N_HOST}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

Start with:
```bash
docker-compose up -d
```

### Pros:
- Isolated environment
- Easy to deploy and scale
- Consistent across systems
- Simple backup and restore

### Cons:
- Requires Docker knowledge
- Additional layer of complexity
- Resource overhead

---

## Method 5: Desktop App

n8n Desktop provides a native application experience.

### Steps:

1. Download from [n8n Desktop Releases](https://github.com/n8n-io/n8n-desktop/releases)
2. Install the application for your OS
3. Launch n8n Desktop
4. Start building workflows

### Pros:
- Native app experience
- Easy to start/stop
- Menu bar integration
- No terminal needed

### Cons:
- Limited to desktop use
- May lag behind npm version

---

## Post-Installation Setup

### 1. Create Your First User

On first launch, you'll be prompted to create an owner account:
- Email address
- Strong password
- First and last name

### 2. Configure Basic Settings

Navigate to Settings → General:
- Timezone settings
- Date format preferences
- Language selection

### 3. Test Your Installation

Create a simple test workflow:
1. Add a Manual Trigger node
2. Add a Set node with test data
3. Execute the workflow
4. Verify results appear

### 4. Set Up Security (Production)

For production deployments:

```bash
# Enable basic authentication
export N8N_BASIC_AUTH_ACTIVE=true
export N8N_BASIC_AUTH_USER=your_username
export N8N_BASIC_AUTH_PASSWORD=strong_password

# Or use environment variables file
echo "N8N_BASIC_AUTH_ACTIVE=true" >> ~/.n8n/.env
```

---

## Common Installation Issues

### Issue: Port 5678 Already in Use

**Solution:**
```bash
# Change the port
export N8N_PORT=5679
n8n start
```

### Issue: Node.js Version Too Old

**Solution:**
```bash
# Update Node.js using nvm
nvm install 18
nvm use 18
npm install n8n -g
```

### Issue: Permission Denied (Linux/Mac)

**Solution:**
```bash
# Don't use sudo! Instead, fix npm permissions
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### Issue: Docker Container Won't Start

**Solution:**
```bash
# Check logs
docker logs n8n

# Ensure volume permissions
sudo chown -R 1000:1000 ~/.n8n

# Restart container
docker restart n8n
```

---

## Updating n8n

### npm Installation:
```bash
npm update -g n8n
```

### Docker:
```bash
docker pull n8nio/n8n:latest
docker-compose down
docker-compose up -d
```

### Desktop App:
Check for updates in the app menu or download the latest version.

---

## Next Steps

Once n8n is installed and running:

1. Complete [Exercise 1: Hello World](./exercises/exercise-1-hello-world.md)
2. Explore the n8n interface
3. Check out [Example Workflows](./exercises/)
4. Join the [n8n Community](https://community.n8n.io)

## Getting Help

- [n8n Documentation](https://docs.n8n.io)
- [Community Forum](https://community.n8n.io)
- [GitHub Issues](https://github.com/n8n-io/n8n/issues)
- [Discord Server](https://discord.gg/n8n)

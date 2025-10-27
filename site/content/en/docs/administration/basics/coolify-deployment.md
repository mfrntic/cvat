---
title: 'Coolify Platform Deployment'
linkTitle: 'Coolify Deployment'
weight: 25
description: 'Deploy CVAT on the Coolify platform with one-click simplicity'
---

This guide explains how to deploy CVAT on [Coolify](https://coolify.io), an open-source, self-hosted Platform as a Service (PaaS) that simplifies deployment, SSL management, and monitoring.

## Why Coolify?

Coolify provides:
- **One-click deployment** from Git repositories
- **Automatic SSL/TLS certificates** via Let's Encrypt
- **Built-in health monitoring** and automatic restarts
- **Zero-downtime deployments** and easy rollbacks
- **Managed reverse proxy** (Traefik) with automatic routing
- **No Docker socket security concerns**

## Prerequisites

Before deploying CVAT on Coolify, ensure you have:

1. **Coolify v4.0 or later** installed on your server
   - Follow the [Coolify installation guide](https://coolify.io/docs/installation)
   - Minimum server requirements: 2 CPU cores, 4GB RAM, 20GB disk space
   - For production: 4+ CPU cores, 8GB+ RAM, 100GB+ disk space

2. **Domain name** pointed to your Coolify server
   - Configure DNS A record to point to your server's IP address
   - Example: `cvat.example.com` → `your.server.ip`

3. **Git repository access** to CVAT
   - Public repository: `https://github.com/cvat-ai/cvat.git`
   - Or your own fork for customizations

## Deployment Steps

### Step 1: Create New Resource in Coolify

1. Log into your Coolify dashboard
2. Navigate to **Projects** → **Add New Resource**
3. Select **Docker Compose** as the resource type
4. Choose **Public Repository** (or Private if using a fork)
5. Enter repository URL: `https://github.com/cvat-ai/cvat.git`
6. Set the **branch** (e.g., `develop` or a specific release tag)
7. Set **Compose File Path** to: `docker-compose.coolify.yml`
8. Click **Continue**

### Step 2: Configure Environment Variables

In the **Environment Variables** section, configure the following required variables:

#### Essential Variables

```bash
# Domain Configuration
CVAT_HOST=cvat.example.com

# Version Selection
CVAT_VERSION=v2.11.0  # Or 'dev' for latest development build

# Database Configuration (PostgreSQL)
POSTGRES_USER=root
POSTGRES_DB=cvat
POSTGRES_HOST_AUTH_METHOD=trust

# ClickHouse Configuration (Analytics)
CLICKHOUSE_DB=cvat
CLICKHOUSE_USER=user
CLICKHOUSE_PASSWORD=user
```

#### Optional Variables

```bash
# Cache Settings
CVAT_ALLOW_STATIC_CACHE=yes  # Enable static file caching

# Worker Configuration
NUMPROCS=2  # Number of worker processes

# Analytics
CVAT_ANALYTICS=1  # Enable analytics dashboard

# Proxy Settings (if needed)
no_proxy=clickhouse,grafana,vector,nuclio,opa
```

### Step 3: Configure Domain and SSL

1. In the **Domains** section, add your domain: `cvat.example.com`
2. Enable **HTTPS** (Coolify will automatically provision SSL certificates)
3. Optionally enable **Force HTTPS** to redirect HTTP to HTTPS

### Step 4: Deploy CVAT

1. Review your configuration
2. Click **Deploy** to start the deployment
3. Monitor the deployment logs in real-time
4. Wait for all services to start (typically 2-5 minutes)

### Step 5: Verify Deployment

Once deployment completes, verify that CVAT is running:

1. **Check Service Health**
   - In Coolify dashboard, verify all 17 services show as "running"
   - Services include: cvat_server, cvat_ui, cvat_db, and 14 others

2. **Access Web UI**
   - Navigate to `https://cvat.example.com`
   - You should see the CVAT login page
   - SSL certificate should be valid (check browser padlock icon)

3. **Test Functionality**
   - Register a new user account (first user becomes admin)
   - Create a test project
   - Upload sample images
   - Verify annotation tools load correctly

## Post-Deployment Configuration

### Creating the First Admin User

The first user to register will automatically become an administrator:

1. Navigate to `https://cvat.example.com/auth/register`
2. Fill in registration details
3. This user will have full administrative privileges

Alternatively, create a superuser via command line:

```bash
# In Coolify dashboard, open the cvat_server container terminal
docker exec -it cvat_server python manage.py createsuperuser
```

### Accessing Analytics Dashboard

CVAT includes a Grafana analytics dashboard:

1. Navigate to `https://cvat.example.com/analytics`
2. You'll be redirected to CVAT login (if not already logged in)
3. After authentication, Grafana dashboards will load
4. View annotation events, user activity, and performance metrics

### Configuring Email Notifications

For email notifications (password resets, webhooks):

1. Add SMTP configuration to environment variables:
   ```bash
   EMAIL_HOST=smtp.example.com
   EMAIL_PORT=587
   EMAIL_USE_TLS=true
   EMAIL_HOST_USER=notifications@example.com
   EMAIL_HOST_PASSWORD=your_password
   DEFAULT_FROM_EMAIL=noreply@example.com
   ```

2. Redeploy the stack for changes to take effect

## Coolify-Specific Features

### Automatic Updates

Coolify can automatically pull and deploy updates:

1. Go to **Settings** → **Auto Deploy**
2. Enable **Deploy on Git Push** (webhook-based)
3. Or set up **Scheduled Deployments** (e.g., weekly)

### Health Monitoring

Coolify monitors all CVAT services:

- View service status in the dashboard
- Receive notifications for unhealthy services
- Automatic restart of failed containers

### Resource Management

Configure resource limits for CVAT services:

1. Go to **Resource Limits** in Coolify
2. Set memory/CPU limits per service
3. Recommended production limits:
   - `cvat_server`: 2GB memory, 1 CPU
   - `cvat_db`: 2GB memory, 1 CPU
   - Workers: 1GB memory each

### Backups

Set up automated backups in Coolify:

1. Navigate to **Backups** → **Configure**
2. Select volumes to backup:
   - `cvat_db` (PostgreSQL database)
   - `cvat_data` (uploaded datasets)
   - `cvat_keys` (authentication keys)
   - `cvat_logs` (application logs)
3. Set backup schedule (e.g., daily at 2 AM)
4. Configure backup retention policy

## Troubleshooting

### Services Not Starting

**Symptom**: One or more services fail to start

**Solutions**:
1. Check Coolify deployment logs for error messages
2. Verify environment variables are set correctly
3. Ensure `CVAT_HOST` matches your domain configuration
4. Check that the coolify network exists: `docker network ls | grep coolify`

### SSL Certificate Issues

**Symptom**: Certificate errors or HTTPS not working

**Solutions**:
1. Verify DNS is correctly pointed to your server
2. Ensure ports 80 and 443 are open in firewall
3. Check Coolify's Traefik logs for certificate errors
4. Wait 5-10 minutes for Let's Encrypt certificate provisioning

### Cannot Access Web UI

**Symptom**: CVAT UI doesn't load or shows 502 Bad Gateway

**Solutions**:
1. Verify `cvat_server` and `cvat_ui` services are running
2. Check that services are on both `cvat` and `coolify` networks:
   ```bash
   docker inspect cvat_server | grep -A 10 Networks
   ```
3. Review Traefik routing rules in service labels
4. Restart the deployment from Coolify dashboard

### Analytics Dashboard Not Loading

**Symptom**: `/analytics` endpoint returns errors

**Solutions**:
1. Verify `cvat_grafana`, `cvat_clickhouse`, and `cvat_vector` are running
2. Check Grafana authentication middleware is working:
   ```bash
   docker logs cvat_server | grep analytics
   ```
3. Verify ClickHouse is accepting connections:
   ```bash
   docker exec -it cvat_clickhouse clickhouse-client --query "SELECT 1"
   ```

### Worker Jobs Not Processing

**Symptom**: Import/export/annotation jobs stuck in queue

**Solutions**:
1. Verify all 8 worker services are running
2. Check Redis connections:
   ```bash
   docker exec -it cvat_redis_inmem redis-cli ping
   docker exec -it cvat_redis_ondisk redis-cli -p 6666 ping
   ```
3. Review worker logs for errors:
   ```bash
   docker logs cvat_worker_import
   docker logs cvat_worker_export
   ```

### Database Connection Errors

**Symptom**: cvat_server cannot connect to PostgreSQL

**Solutions**:
1. Verify `cvat_db` service is running and healthy
2. Check database logs:
   ```bash
   docker logs cvat_db
   ```
3. Ensure services can communicate on `cvat` network:
   ```bash
   docker exec -it cvat_server ping cvat_db
   ```

## Scaling and Performance

### Horizontal Scaling

Scale worker services for better performance:

1. In Coolify, go to the service settings
2. Increase replicas for worker services:
   - `cvat_worker_import`: 2-4 replicas
   - `cvat_worker_export`: 2-4 replicas
   - `cvat_worker_chunks`: 2-4 replicas

### Vertical Scaling

Increase resources for high-traffic deployments:

```bash
# Increase server workers
NUMPROCS=4  # More Django workers

# Allocate more resources in Coolify dashboard
# cvat_server: 4GB RAM, 2 CPUs
# cvat_db: 4GB RAM, 2 CPUs
```

## Migration to Different Deployment

If you need to migrate from Coolify to standard Docker Compose deployment, see the [Migration Guide](../advanced/coolify-migration.md).

## Support and Resources

- [Coolify Documentation](https://coolify.io/docs)
- [CVAT Documentation](https://docs.cvat.ai)
- [CVAT GitHub Issues](https://github.com/cvat-ai/cvat/issues)
- [Coolify Discord Community](https://coolify.io/discord)

---
title: 'Coolify Migration Guide'
linkTitle: 'Coolify Migration'
weight: 15
description: 'Migrate CVAT between standard and Coolify deployments'
---

This guide covers migrating CVAT between standard Docker Compose deployment and Coolify platform deployment in both directions.

## Migration Scenarios

This guide covers:
1. **Standard to Coolify**: Migrating an existing docker-compose.yml deployment to Coolify
2. **Coolify to Standard**: Rolling back from Coolify to standard Docker Compose
3. **Coolify to Coolify**: Moving between Coolify instances

## Prerequisites

Before starting any migration:

- [ ] **Backup all data** (databases, volumes, configuration)
- [ ] **Document current configuration** (environment variables, custom settings)
- [ ] **Test rollback procedure** in a non-production environment
- [ ] **Schedule maintenance window** (expect 30-60 minutes downtime)
- [ ] **Notify users** of planned downtime
- [ ] **Verify target environment** meets requirements

## Migration 1: Standard to Coolify

Migrate from a standard Docker Compose deployment to Coolify platform.

### Step 1: Backup Current Deployment

#### 1.1 Stop CVAT Services

```bash
cd /path/to/cvat
docker-compose stop
```

#### 1.2 Backup PostgreSQL Database

```bash
# Create backup directory
mkdir -p ~/cvat-migration-backup

# Backup database
docker exec cvat_db pg_dumpall -U root > ~/cvat-migration-backup/cvat_db_backup.sql

# Verify backup
ls -lh ~/cvat-migration-backup/cvat_db_backup.sql
```

#### 1.3 Backup Docker Volumes

```bash
# Backup data volume (uploaded datasets, annotations)
docker run --rm \
  -v cvat_data:/data \
  -v ~/cvat-migration-backup:/backup \
  alpine tar czf /backup/cvat_data.tar.gz -C /data .

# Backup keys volume (authentication keys)
docker run --rm \
  -v cvat_keys:/data \
  -v ~/cvat-migration-backup:/backup \
  alpine tar czf /backup/cvat_keys.tar.gz -C /data .

# Backup logs volume
docker run --rm \
  -v cvat_logs:/data \
  -v ~/cvat-migration-backup:/backup \
  alpine tar czf /backup/cvat_logs.tar.gz -C /data .

# Backup ClickHouse analytics data
docker run --rm \
  -v cvat_events_db:/data \
  -v ~/cvat-migration-backup:/backup \
  alpine tar czf /backup/cvat_events_db.tar.gz -C /data .

# Verify backups
ls -lh ~/cvat-migration-backup/
```

#### 1.4 Export Environment Configuration

```bash
# Save current .env file
cp .env ~/cvat-migration-backup/env_backup

# Document custom configurations
docker-compose config > ~/cvat-migration-backup/docker-compose-resolved.yml
```

#### 1.5 Test Database Backup

```bash
# Verify backup can be read
head -50 ~/cvat-migration-backup/cvat_db_backup.sql
```

### Step 2: Prepare Coolify Environment

#### 2.1 Install Coolify

If not already installed, follow the [Coolify installation guide](https://coolify.io/docs/installation).

```bash
# Quick installation on a fresh server
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

#### 2.2 Configure DNS

Point your domain to the Coolify server:

```bash
# Example DNS A record
cvat.newdomain.com  →  your.coolify.server.ip
```

Verify DNS propagation:

```bash
dig cvat.newdomain.com +short
nslookup cvat.newdomain.com
```

#### 2.3 Transfer Backup Files

Transfer backups to Coolify server:

```bash
# From old server, transfer to Coolify server
scp -r ~/cvat-migration-backup/ user@coolify-server:/tmp/
```

### Step 3: Deploy CVAT on Coolify

#### 3.1 Create Coolify Resource

1. Log into Coolify dashboard
2. Create new **Docker Compose** resource
3. Repository: `https://github.com/cvat-ai/cvat.git`
4. Compose file: `docker-compose.coolify.yml`
5. Set branch/tag to match your current version

#### 3.2 Configure Environment Variables

Use the same values from your `env_backup` file:

```bash
CVAT_HOST=cvat.newdomain.com  # Update to new domain
CVAT_VERSION=v2.11.0  # Match current version
POSTGRES_USER=root
POSTGRES_DB=cvat
POSTGRES_HOST_AUTH_METHOD=trust
CLICKHOUSE_USER=user
CLICKHOUSE_PASSWORD=user
CLICKHOUSE_DB=cvat
```

#### 3.3 Initial Deployment

Deploy once to create volumes and containers:

1. Click **Deploy** in Coolify
2. Wait for initial deployment to complete
3. **Stop all services** before restoring data:
   ```bash
   docker-compose -f docker-compose.coolify.yml stop
   ```

### Step 4: Restore Data

#### 4.1 Restore PostgreSQL Database

```bash
# Copy backup to database container
cat /tmp/cvat-migration-backup/cvat_db_backup.sql | \
  docker exec -i cvat_db psql -U root

# Verify restoration
docker exec -it cvat_db psql -U root -d cvat -c "\dt"
```

#### 4.2 Restore Docker Volumes

```bash
# Restore data volume
docker run --rm \
  -v cvat_data:/data \
  -v /tmp/cvat-migration-backup:/backup \
  alpine sh -c "cd /data && tar xzf /backup/cvat_data.tar.gz"

# Restore keys volume
docker run --rm \
  -v cvat_keys:/data \
  -v /tmp/cvat-migration-backup:/backup \
  alpine sh -c "cd /data && tar xzf /backup/cvat_keys.tar.gz"

# Restore logs volume
docker run --rm \
  -v cvat_logs:/data \
  -v /tmp/cvat-migration-backup:/backup \
  alpine sh -c "cd /data && tar xzf /backup/cvat_logs.tar.gz"

# Restore ClickHouse data
docker run --rm \
  -v cvat_events_db:/data \
  -v /tmp/cvat-migration-backup:/backup \
  alpine sh -c "cd /data && tar xzf /backup/cvat_events_db.tar.gz"
```

#### 4.3 Fix Permissions

```bash
# Ensure correct ownership
docker run --rm -v cvat_data:/data alpine chown -R 1000:1000 /data
docker run --rm -v cvat_logs:/data alpine chown -R 1000:1000 /data
```

### Step 5: Start and Verify

#### 5.1 Restart Services via Coolify

1. In Coolify dashboard, click **Restart**
2. Monitor deployment logs for errors
3. Wait for all services to become healthy

#### 5.2 Verify Data Migration

1. **Access Web UI**: Navigate to `https://cvat.newdomain.com`
2. **Login**: Use existing credentials
3. **Verify Projects**: Check that all projects are present
4. **Verify Tasks**: Open tasks and verify annotations loaded
5. **Verify Files**: Check uploaded datasets are accessible
6. **Test Analytics**: Visit `/analytics` to verify Grafana dashboards

#### 5.3 Test Functionality

- [ ] Create a new project
- [ ] Upload test images
- [ ] Create annotations
- [ ] Export dataset
- [ ] Import dataset
- [ ] Test webhooks (if configured)
- [ ] Verify worker jobs processing

### Step 6: Update DNS and Finalize

#### 6.1 Update DNS (if changing domains)

```bash
# Update DNS A record to point to Coolify server
old.cvat.domain  →  new.cvat.domain
```

#### 6.2 Decommission Old Deployment

Once verified, clean up the old server:

```bash
# On old server
cd /path/to/cvat
docker-compose down -v  # WARNING: Deletes volumes

# Keep backups for at least 30 days
```

## Migration 2: Coolify to Standard

Rollback from Coolify to standard Docker Compose deployment.

### Step 1: Backup Coolify Deployment

#### 1.1 Stop Coolify Deployment

In Coolify dashboard:
1. Navigate to your CVAT resource
2. Click **Stop**
3. Wait for all containers to stop

#### 1.2 Backup Data

```bash
# SSH into Coolify server
ssh user@coolify-server

# Create backup directory
mkdir -p ~/cvat-coolify-backup

# Backup database
docker exec cvat_db pg_dumpall -U root > ~/cvat-coolify-backup/cvat_db_backup.sql

# Backup volumes (same commands as Migration 1, Step 1.3)
# ... (see above)
```

### Step 2: Prepare Standard Deployment Server

#### 2.1 Install Docker and Docker Compose

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

#### 2.2 Clone CVAT Repository

```bash
git clone https://github.com/cvat-ai/cvat.git
cd cvat
git checkout <same-version-as-coolify>
```

#### 2.3 Configure Environment

```bash
# Copy example environment file
cp .env.example .env

# Edit .env with same values from Coolify
nano .env
```

Example `.env`:
```bash
CVAT_HOST=localhost  # Or your domain
CVAT_VERSION=v2.11.0
POSTGRES_USER=root
POSTGRES_DB=cvat
POSTGRES_HOST_AUTH_METHOD=trust
```

### Step 3: Deploy and Restore

#### 3.1 Initial Deployment

```bash
# Start services once to create volumes
docker-compose up -d

# Stop services for data restoration
docker-compose stop
```

#### 3.2 Restore Data

```bash
# Transfer backups from Coolify server
scp -r user@coolify-server:~/cvat-coolify-backup/ /tmp/

# Restore database
cat /tmp/cvat-coolify-backup/cvat_db_backup.sql | \
  docker exec -i cvat_db psql -U root

# Restore volumes (same as Migration 1, Step 4.2)
# ...
```

#### 3.3 Start Services

```bash
docker-compose up -d

# Monitor logs
docker-compose logs -f cvat_server
```

### Step 4: Configure Reverse Proxy

If exposing to internet, configure Traefik or Nginx:

#### Option A: Use Bundled Traefik

The standard `docker-compose.yml` includes Traefik. Configure SSL:

```bash
# Edit docker-compose.yml to enable HTTPS
# Uncomment websecure entrypoint configuration
```

#### Option B: External Nginx

```nginx
server {
    listen 80;
    server_name cvat.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name cvat.example.com;

    ssl_certificate /etc/ssl/certs/cvat.crt;
    ssl_certificate_key /etc/ssl/private/cvat.key;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## Migration 3: Coolify to Coolify

Moving CVAT between Coolify instances (e.g., staging to production).

### Quick Migration Steps

1. **Backup source Coolify instance** (Migration 2, Step 1)
2. **Deploy on target Coolify** (Migration 1, Step 3)
3. **Restore data** (Migration 1, Step 4)
4. **Verify and update DNS** (Migration 1, Steps 5-6)

## Rollback Procedures

### Emergency Rollback

If migration fails and you need to quickly restore service:

#### From Coolify Back to Old Server

```bash
# On old server (if still available)
cd /path/to/cvat
docker-compose up -d

# Revert DNS
# Point domain back to old server IP
```

#### From Standard Back to Coolify

```bash
# In Coolify dashboard
# Click "Restart" to bring services back up

# Revert DNS if changed
```

### Partial Rollback

If only some services have issues:

```bash
# Restart specific service in Coolify
docker restart cvat_server

# Or via docker-compose
docker-compose restart cvat_server
```

## Common Migration Issues

### Database Connection Errors After Migration

**Symptom**: Services can't connect to database after restore

**Solution**:
```bash
# Ensure database is ready
docker exec -it cvat_db psql -U root -d cvat -c "SELECT 1"

# Check service can reach database
docker exec -it cvat_server ping cvat_db

# Restart services
docker-compose restart cvat_server
```

### Missing Annotations or Files

**Symptom**: Projects load but annotations/files are missing

**Solution**:
```bash
# Verify volume restoration
docker exec -it cvat_server ls -la /home/django/data

# Check file permissions
docker run --rm -v cvat_data:/data alpine ls -la /data

# Re-restore if needed
```

### SSL Certificate Issues After DNS Change

**Symptom**: Certificate errors after changing domain

**Solution**:
- Wait 5-10 minutes for Let's Encrypt to provision new certificate
- Clear browser cache and cookies
- Verify DNS has fully propagated: `dig cvat.newdomain.com +short`

### Analytics Dashboard Not Working

**Symptom**: `/analytics` returns errors after migration

**Solution**:
```bash
# Verify ClickHouse data restored
docker exec -it cvat_clickhouse clickhouse-client --query "SHOW DATABASES"

# Restart Grafana with fresh config
docker restart cvat_grafana cvat_vector
```

## Post-Migration Checklist

After completing migration, verify:

- [ ] All users can log in with existing credentials
- [ ] All projects and tasks are visible and accessible
- [ ] Annotations load correctly in annotation interface
- [ ] File uploads and downloads work
- [ ] Import/export functionality works
- [ ] Worker jobs process correctly (check queues)
- [ ] Webhooks deliver (if configured)
- [ ] Analytics dashboard loads (`/analytics`)
- [ ] Email notifications send (if configured)
- [ ] SSL certificate is valid and auto-renewing
- [ ] Backups are scheduled in new environment
- [ ] Monitoring/alerting is configured

## Data Integrity Verification

After migration, verify data integrity:

```bash
# Check database record counts
docker exec -it cvat_db psql -U root -d cvat -c "
  SELECT 'projects' AS table, COUNT(*) FROM public.project
  UNION ALL
  SELECT 'tasks', COUNT(*) FROM public.task
  UNION ALL
  SELECT 'jobs', COUNT(*) FROM public.job
  UNION ALL
  SELECT 'users', COUNT(*) FROM public.auth_user;
"

# Verify file counts
docker exec -it cvat_server sh -c "
  echo 'Total files:' && find /home/django/data -type f | wc -l
"

# Compare with pre-migration counts (saved during backup)
```

## Support

If you encounter issues during migration:

1. Check Coolify logs: **Dashboard** → **Logs**
2. Check CVAT logs: `docker logs cvat_server`
3. Review [CVAT Documentation](https://docs.cvat.ai)
4. Ask on [CVAT GitHub Discussions](https://github.com/cvat-ai/cvat/discussions)
5. For Coolify-specific issues: [Coolify Discord](https://coolify.io/discord)

## Best Practices

- **Always test migration in staging environment first**
- **Keep backups for at least 30 days after migration**
- **Document custom configurations before migration**
- **Schedule migrations during low-usage periods**
- **Have rollback plan ready before starting**
- **Verify data integrity before decommissioning old deployment**

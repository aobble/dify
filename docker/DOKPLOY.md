# Dokploy Configuration for Dify

This document explains the `dokploy.yaml` configuration for deploying Dify on Dokploy with persistent volumes and backup support.

## 📁 File Structure

```
/app-name/                          <- Dokploy application root
├── code/                          <- Git repository (WIPED on each deploy!)
│   ├── docker/                    <- Your dokploy.yaml is HERE
│   │   ├── dokploy.yaml          <- This file
│   │   ├── docker-compose.yaml   <- Original Dify file (don't modify)
│   │   ├── nginx/                <- Config files (static from repo)
│   │   ├── ssrf_proxy/           <- Config files (static from repo)
│   │   └── volumes/              <- Config files (static from repo)
│   └── ...                       <- Rest of Dify codebase
└── files/                        <- PERSISTENT storage (survives deploys!)
    ├── app/storage/              <- User uploads, documents
    ├── db/data/                  <- PostgreSQL data (if using bind mounts)
    ├── redis/data/               <- Redis data (if using bind mounts)
    └── ...                       <- Other persistent data
```

## 🔄 Volume Strategy: Hybrid Approach

We use a **hybrid approach** combining named volumes (for Dokploy backups) and bind mounts (for less critical data).

### Named Volumes (Critical Data - Backup These!)

These volumes can be backed up via Dokploy's Volume Backup UI:

| Volume Name | Used By | Contains | Backup Priority |
|-------------|---------|----------|-----------------|
| `dify_api_storage` | `api`, `worker` | User uploads, generated files, knowledge base documents | 🔴 Critical |
| `dify_postgres_data` | `db_postgres` | All Dify application data, users, workflows, apps | 🔴 Critical |
| `dify_mysql_data` | `db_mysql` | All Dify application data (MySQL alternative) | 🔴 Critical |
| `dify_redis_data` | `redis` | Sessions, queues, cache - important for active workflows | 🟡 High |
| `dify_weaviate_data` | `weaviate` | Vector embeddings - expensive to rebuild | 🟡 High |
| `dify_plugin_daemon_data` | `plugin_daemon` | Installed plugins and their data | 🟢 Medium |

### Bind Mounts (Less Critical Data)

These use `../../files/` paths and can be rebuilt or are config files:

| Path | Used By | Contains | Notes |
|------|---------|----------|-------|
| `../../files/sandbox/dependencies` | `sandbox` | Python packages, dependencies | Rebuildable |
| `../../files/certbot/conf` | `certbot`, `nginx` | SSL certificates | Important but manageable |
| `./volumes/sandbox/conf` | `sandbox` | Static config from repo | Reset on each deploy (expected) |
| `./nginx/` | `nginx` | Static config from repo | Reset on each deploy (expected) |
| `./ssrf_proxy/` | `ssrf_proxy` | Static config from repo | Reset on each deploy (expected) |

## 🛠️ Key Configuration Changes

### Volume Path Corrections

**Problem:** Original paths were incorrect for Dokploy's directory structure.

**Solution:** Since `dokploy.yaml` is in `/code/docker/`, we need `../../files/` to reach persistent storage:

```yaml
# ❌ Wrong (gets wiped on deploy)
- ../files/app/storage:/app/api/storage

# ❌ Wrong (invalid syntax)
- .../files/app/storage:/app/api/storage

# ✅ Correct (persistent storage)
- ../../files/app/storage:/app/api/storage

# ✅ Better (named volume for backups)
- dify_api_storage:/app/api/storage
```

### Critical Services Using Named Volumes

```yaml
# API and Worker services
volumes:
  - dify_api_storage:/app/api/storage

# Database services
volumes:
  - dify_postgres_data:/var/lib/postgresql/data
  - dify_mysql_data:/var/lib/mysql

# Cache and Vector Store
volumes:
  - dify_redis_data:/data
  - dify_weaviate_data:/var/lib/weaviate

# Plugin storage
volumes:
  - dify_plugin_daemon_data:/app/storage
```

### Volume Declarations

```yaml
volumes:
  # System volumes (for services with named volume requirements)
  oradata:
  dify_es01_data:
  
  # Critical data volumes - BACKUP THESE via Dokploy UI!
  dify_api_storage:
    # Contains: user uploads, generated files, knowledge base documents
  
  dify_postgres_data:
    # Contains: all Dify application data, users, workflows, apps
  
  dify_mysql_data:
    # Contains: all Dify application data (if using MySQL instead of PostgreSQL)
  
  dify_redis_data:
    # Contains: sessions, queues, cache - important for active workflows
  
  dify_weaviate_data:
    # Contains: vector embeddings - expensive to rebuild
  
  dify_plugin_daemon_data:
    # Contains: installed plugins and their data
```

## 🔧 Dokploy Deployment Setup

### 1. Repository Configuration

- **Repository:** `https://github.com/aobble/dify.git`
- **Branch:** `main`
- **Docker Compose File:** `docker/dokploy.yaml`
- **AutoDeploy:** Enabled (optional)

### 2. Environment Variables

Copy from `.env.example` and configure:
- Database credentials
- Redis password
- API keys
- Domain settings

### 3. Profiles

Enable required services via profiles:
```yaml
# Common profiles to enable:
- postgresql  # or mysql
- weaviate    # or other vector store
- certbot     # for SSL (optional)
```

## 💾 Backup Strategy

### Dokploy Volume Backups

1. **Configure S3 Storage:**
   - Go to Dokploy UI → Settings → Volume Backups
   - Add S3 destination (AWS S3, Backblaze B2, MinIO, etc.)

2. **Volume Names in Dokploy:**
   ```
   {app-name}_dify_api_storage
   {app-name}_dify_postgres_data
   {app-name}_dify_redis_data
   {app-name}_dify_weaviate_data
   {app-name}_dify_plugin_daemon_data
   ```

3. **Recommended Backup Schedule:**
   - **PostgreSQL:** Every 6 hours, retain 30 days
   - **API Storage:** Daily, retain 30 days
   - **Weaviate:** Daily, retain 14 days
   - **Redis:** Daily, retain 7 days
   - **Plugin Daemon:** Weekly, retain 14 days

### Manual Backup Commands

```bash
# Find volume names
docker volume ls | grep dify

# Backup volume to tar
docker run --rm -v {app-name}_dify_postgres_data:/backup alpine tar czf - /backup > postgres-backup.tar.gz

# Restore volume from tar
docker volume create {app-name}_dify_postgres_data
docker run --rm -v {app-name}_dify_postgres_data:/restore -v $(pwd):/backup alpine tar xzf /backup/postgres-backup.tar.gz -C /restore --strip-components=1
```

## 🚨 Disaster Recovery

### Complete Restore Process

1. **Stop Dify application** in Dokploy UI
2. **Remove old volumes:**
   ```bash
   docker volume rm {app-name}_dify_postgres_data
   docker volume rm {app-name}_dify_api_storage
   docker volume rm {app-name}_dify_redis_data
   # ... etc
   ```
3. **Restore via Dokploy UI:**
   - Volume Backups → Restore
   - Select backup and target volume name
4. **Redeploy application**

### Partial Recovery

For single service issues:
1. Stop affected service
2. Restore specific volume
3. Restart service

## 🔄 Updating from Upstream Dify

### Setup (One-time)

```bash
git remote add upstream https://github.com/langgenius/dify.git
```

### Regular Updates

```bash
# Fetch latest from official Dify
git fetch upstream

# Check what's new
git log --oneline main..upstream/main

# Merge updates (your dokploy.yaml is safe!)
git merge upstream/main

# Push to your fork
git push origin main
```

### Why Your Config is Safe

- ✅ `dokploy.yaml` doesn't exist in upstream
- ✅ Different filename from `docker-compose.yaml`
- ✅ No merge conflicts possible
- ✅ Your customizations are preserved

## 🐛 Troubleshooting

### Volume Issues

**Problem:** Data not persisting after redeploy
```bash
# Check volume mounts
docker inspect {container-name} | grep -A 10 "Mounts"

# Verify volume exists
docker volume ls | grep dify
```

**Problem:** Permission errors
```bash
# Check volume ownership
docker run --rm -v {volume-name}:/data alpine ls -la /data
```

### Path Issues

**Problem:** Files not found
- Verify path syntax: `../../files/` for bind mounts
- Check Dokploy file structure in Advanced → Mounts

### Backup Issues

**Problem:** Backup fails
- Ensure S3 credentials are correct
- Check volume is not in use during backup
- Verify sufficient storage space

## 📋 Maintenance Checklist

### Weekly
- [ ] Check application health
- [ ] Verify backups completed successfully
- [ ] Monitor disk usage

### Monthly
- [ ] Update from upstream Dify
- [ ] Test backup restore process
- [ ] Review backup retention policies
- [ ] Update environment variables if needed

### Quarterly
- [ ] Full disaster recovery test
- [ ] Review and update documentation
- [ ] Security audit of configurations

## 🔗 References

- [Dokploy Docker Compose Documentation](https://docs.dokploy.com/docs/core/docker-compose)
- [Dokploy Volume Backups](https://docs.dokploy.com/docs/core/volume-backups)
- [Dokploy Troubleshooting](https://docs.dokploy.com/docs/core/troubleshooting)
- [Dify Official Repository](https://github.com/langgenius/dify)

---

**Last Updated:** November 2025  
**Configuration Version:** Dify 1.10.0 + Dokploy Optimized

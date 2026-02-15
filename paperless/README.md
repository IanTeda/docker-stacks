# Paperless-ngx Docker Stack

A production-ready Docker Compose configuration for [Paperless-ngx](https://docs.paperless-ngx.com/), a document management system that transforms your physical documents into a searchable online archive.

## 📋 Table of Contents

- [Features](#features)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Services](#services)
- [Storage](#storage)
- [Backup](#backup)
- [Troubleshooting](#troubleshooting)
- [Documentation](#documentation)

## ✨ Features

- **PostgreSQL Database**: Reliable metadata storage with persistent volumes
- **Redis Cache**: Task queue management with AOF persistence
- **Tika Integration**: Advanced text extraction from various document formats
- **Gotenberg Integration**: Office document conversion (.docx, .xlsx, .pptx, .eml)
- **Health Checks**: All services include health monitoring
- **Watchtower Ready**: Optional automatic container updates
- **Traefik Compatible**: Ready for reverse proxy integration
- **Resource Optimized**: Sensible defaults with logging limits

## 📦 Prerequisites

- Docker Engine 20.10+ and Docker Compose 2.0+
- Minimum 2GB RAM (4GB+ recommended)
- External `traefik-network` (if using Traefik reverse proxy)
- Sufficient disk space for documents and database

## 🚀 Quick Start

1. **Copy environment file**:
   ```bash
   cp .env.sample .env
   ```

2. **Edit `.env` file** with your configuration:
   ```bash
   nano .env
   ```
   
   **Critical settings to change**:
   - `POSTGRES_PASSWORD`: Strong database password
   - `PAPERLESS_SECRET_KEY`: Generate with `openssl rand -base64 32`
   - `PAPERLESS_ADMIN_PASSWORD`: Strong admin password
   - `PAPERLESS_DOMAIN_NAME`: Your public URL
   - `PAPERLESS_ADMIN_MAIL`: Your email address

3. **Create storage directory** (if using custom path):
   ```bash
   mkdir -p /path/to/paperless/{data,media,export,consume,postgres,redis}
   chown -R 1000:1000 /path/to/paperless
   ```

4. **Start the stack**:
   ```bash
   docker compose up -d
   ```

5. **Check logs**:
   ```bash
   docker compose logs -f paperless
   ```

6. **Access Paperless**:
   - URL: http://localhost:8000 (or your configured domain)
   - Username: Value from `PAPERLESS_ADMIN_USER`
   - Password: Value from `PAPERLESS_ADMIN_PASSWORD`

## ⚙️ Configuration

### Environment Variables

See [.env.sample](.env.sample) for all available configuration options.

**Most commonly customized**:

| Variable | Description | Example |
|----------|-------------|---------|
| `PAPERLESS_PATH` | Storage location for all data | `/mnt/storage/paperless` |
| `TZ` | Timezone | `America/New_York` |
| `PAPERLESS_OCR_LANGUAGE` | Primary OCR language | `eng+deu` |
| `PAPERLESS_CONSUMER_RECURSIVE` | Watch subdirectories | `true` |
| `PAPERLESS_CONSUMER_SUBDIRS_AS_TAGS` | Tag from folder names | `true` |

### Version Pinning

For production stability, use specific versions instead of `latest`:

```env
REDIS_VERSION=7.2
POSTGRES_VERSION=16
GOTENBERG_VERSION=8.0
PAPERLESS_VERSION=2.6.0
```

## 🛠️ Services

| Service | Purpose | Port | Health Check |
|---------|---------|------|--------------|
| **paperless** | Main application | 8000 | ✅ Yes |
| **postgres** | Database | 5432 (internal) | ✅ Yes |
| **redis** | Cache & task queue | 6379 (internal) | ✅ Yes |
| **gotenberg** | Document conversion | 3000 (internal) | ✅ Yes |
| **tika** | Text extraction | 9998 (internal) | ✅ Yes |

## 💾 Storage

### Directory Structure

```
${PAPERLESS_PATH}/
├── data/           # Application data, SQLite (if used), search index
├── media/          # Original documents and thumbnails
├── export/         # Document exports
├── consume/        # Drop documents here for processing
├── postgres/       # PostgreSQL data
└── redis/          # Redis AOF persistence files
```

### Volume Mounts

All persistent data is stored in `${PAPERLESS_PATH}` (defaults to `./.volume`):

- Documents: `${PAPERLESS_PATH}/media`
- Database: `${PAPERLESS_PATH}/postgres`
- Cache: `${PAPERLESS_PATH}/redis`
    
## ⬆️ PostgreSQL Version Upgrades

### PostgreSQL 18+ Breaking Change

PostgreSQL 18+ changed the data directory structure. This compose file now mounts to `/var/lib/postgresql` (not `/var/lib/postgresql/data`) to support the new structure.

**If upgrading from PostgreSQL <18 to 18+:**

1. **Backup your database first**:
   ```bash
   docker compose exec postgres pg_dump -U paperless paperless > backup.sql
   ```

2. **Stop the stack**:
   ```bash
   docker compose down
   ```

3. **Move your existing data** (if you have data in the old structure):
   ```bash
   # If your data is at ${PAPERLESS_PATH}/postgres (no subdirectory)
   # No action needed - PostgreSQL 18+ will handle it
   
   # If your data was at ${PAPERLESS_PATH}/postgres/data
   mv ${PAPERLESS_PATH}/postgres ${PAPERLESS_PATH}/postgres.old
   mkdir ${PAPERLESS_PATH}/postgres
   ```

4. **Update PostgreSQL version** in `.env`:
   ```env
   POSTGRES_VERSION=18
   ```

5. **Start with new version and restore**:
   ```bash
   docker compose up -d postgres
   docker compose exec -T postgres psql -U paperless paperless < backup.sql
   ```

6. **Verify data** and start all services:
   ```bash
   docker compose up -d
   ```

### Standard PostgreSQL Upgrades

For minor version upgrades (e.g., 16.1 → 16.2), simply update the version and restart:
```bash
docker compose pull postgres
docker compose up -d postgres
```

## 🔄 Backup

### Important Data to Backup

1. **PostgreSQL Database**:
   ```bash
   docker compose exec postgres pg_dump -U paperless paperless > backup.sql
   ```

2. **Documents**:
   ```bash
   tar -czf paperless-media-backup.tar.gz ${PAPERLESS_PATH}/media/
   ```

3. **Environment file**:
   ```bash
   cp .env .env.backup
   ```

### Restore Database

```bash
docker compose exec -T postgres psql -U paperless paperless < backup.sql
```

### Full Backup Script

```bash
#!/bin/bash
BACKUP_DIR="/path/to/backups"
DATE=$(date +%Y%m%d_%H%M%S)

# Database
docker compose exec postgres pg_dump -U paperless paperless | gzip > "$BACKUP_DIR/db_$DATE.sql.gz"

# Documents
tar -czf "$BACKUP_DIR/media_$DATE.tar.gz" ${PAPERLESS_PATH}/media/

# Configuration
cp .env "$BACKUP_DIR/env_$DATE.backup"
```

## 🔍 Troubleshooting

### Check Service Status

```bash
docker compose ps
docker compose logs -f [service-name]
```

### Common Issues

**1. Permission Errors**
```bash
sudo chown -R 1000:1000 ${PAPERLESS_PATH}
```

**2. PostgreSQL Connection Issues**
- Verify `POSTGRES_PASSWORD` matches in both database and paperless service
- Check database is healthy: `docker compose ps postgres`

**3. Documents Not Processing**
- Check consume directory permissions
- View consumer logs: `docker compose logs -f paperless | grep consumer`

**4. OCR Not Working**
- Verify `PAPERLESS_OCR_LANGUAGE` is set correctly
- Check language is installed: `docker compose exec paperless bash -c "tesseract --list-langs"`

**5. Out of Memory**
- Reduce `PAPERLESS_TASK_WORKERS` in environment
- Increase Docker memory limits

### Reset Admin Password

```bash
docker compose exec paperless python3 manage.py changepassword <username>
```

### Access Container Shell

```bash
docker compose exec paperless bash
```

## 📚 Documentation

- [Official Documentation](https://docs.paperless-ngx.com/)
- [Configuration Options](https://docs.paperless-ngx.com/configuration/)
- [Advanced Usage](https://docs.paperless-ngx.com/advanced_usage/)
- [REST API](https://docs.paperless-ngx.com/api/)
- [GitHub Repository](https://github.com/paperless-ngx/paperless-ngx)

## 🔐 Security Best Practices

1. **Use strong passwords** for all services
2. **Generate unique secret key**: `openssl rand -base64 32`
3. **Set file permissions**: `chmod 600 .env`
4. **Use specific version tags** in production (not `latest`)
5. **Regular backups** of database and documents
6. **Enable HTTPS** via reverse proxy (Traefik, Nginx, etc.)
7. **Restrict network exposure** (don't expose database ports)
8. **Keep containers updated** (manually or via Watchtower)

## 🆘 Support

- [GitHub Issues](https://github.com/paperless-ngx/paperless-ngx/issues)
- [Matrix Chat](https://matrix.to/#/#paperless:matrix.org)
- [Documentation](https://docs.paperless-ngx.com/)

## 📝 License

This configuration is provided as-is. Paperless-ngx is licensed under the GPL-3.0 License.

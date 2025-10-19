# Highlight.io Deployment on Dokploy

## 🚨 Problem with `compose.hobby.yml`

The `compose.hobby.yml` file **only contains the frontend and backend services**. It does NOT include the required infrastructure services:

- ❌ PostgreSQL (Database)
- ❌ Redis (Cache)
- ❌ ClickHouse (Analytics)
- ❌ Kafka (Event Streaming)
- ❌ Zookeeper (Kafka Coordination)
- ❌ OpenTelemetry Collector

### Error You Were Getting

```
Error migrating DB: failed to connect to `host=port=5432`: hostname resolving error
```

This happens because `PSQL_HOST` is not set, so the backend can't find PostgreSQL.

---

## ✅ Solution: Use the Complete Compose File

I've created `compose.hobby.complete.yml` which includes **ALL required services**.

### Quick Start for Dokploy

#### Option 1: Use the Complete File (Recommended)

1. **In Dokploy, use this compose file:**
   ```
   compose.hobby.complete.yml
   ```

2. **Set these environment variables in Dokploy:**
   ```env
   REACT_APP_FRONTEND_URI=https://first.sume.one
   REACT_APP_PRIVATE_GRAPH_URI=https://first.sume.one:8082/private
   REACT_APP_PUBLIC_GRAPH_URI=https://first.sume.one:8082/public
   ADMIN_PASSWORD=YourSecurePassword
   BACKEND_IMAGE_NAME=ghcr.io/highlight/highlight-backend:latest
   FRONTEND_IMAGE_NAME=ghcr.io/highlight/highlight-frontend:latest
   LICENSE_KEY=
   ```

3. **Deploy!**

#### Option 2: Use Both compose.yml and compose.hobby.yml Together

If you want to use the original setup:

1. **In Dokploy, set COMPOSE_FILE:**
   ```env
   COMPOSE_FILE=compose.yml:compose.hobby.yml
   ```

2. **Or merge them manually** before deploying

---

## 🌐 Port Mappings for Your Domains

Your domain configuration:
- **3001** → `first.sume.one` (Main Frontend)
- **6006** → `second.sume.one` (Storybook - optional)
- **8081** → `third.sume.one` (Additional service - optional)
- **8082** → Backend API (should also be accessible via first.sume.one)

### Updated Services in compose.hobby.complete.yml

```yaml
frontend:
  ports:
    - '3001:3000'  # Main frontend
    - '6006:6006'  # Storybook
    - '8081:8080'  # Additional service

backend:
  ports:
    - '8082:8082'  # API server
```

---

## 📋 What's Included in compose.hobby.complete.yml

### Infrastructure Services
✅ **postgres** - PostgreSQL 15 with pgvector (Port 5432)  
✅ **redis** - Redis 8.0.2 (Port 6379)  
✅ **clickhouse** - ClickHouse 24.3 (Ports 8123, 9000)  
✅ **kafka** - Kafka 7.7.0 (Port 9092)  
✅ **zookeeper** - Zookeeper 7.7.0 (Port 2181)  
✅ **collector** - OpenTelemetry Collector (Ports 4317, 4318)

### Application Services
✅ **backend** - Highlight.io Backend API  
✅ **frontend** - Highlight.io Web Interface

### Health Checks
- PostgreSQL waits until database is ready
- Backend waits for all infrastructure services
- Frontend waits for backend to be healthy

---

## 🔧 Environment Variables

### Required Variables

```env
# Domain Configuration
REACT_APP_FRONTEND_URI=https://first.sume.one
REACT_APP_PRIVATE_GRAPH_URI=https://first.sume.one:8082/private
REACT_APP_PUBLIC_GRAPH_URI=https://first.sume.one:8082/public

# Admin Password
ADMIN_PASSWORD=YourSecurePassword

# Docker Images
BACKEND_IMAGE_NAME=ghcr.io/highlight/highlight-backend:latest
FRONTEND_IMAGE_NAME=ghcr.io/highlight/highlight-frontend:latest

# License (leave empty for hobby mode)
LICENSE_KEY=
```

### Pre-configured Internal Variables

These are already set in the compose file:

```env
PSQL_HOST=postgres
PSQL_PORT=5432
PSQL_DB=postgres
PSQL_USER=postgres
REDIS_EVENTS_STAGING_ENDPOINT=redis:6379
REDIS_PASSWORD=redispassword
CLICKHOUSE_ADDRESS=clickhouse:9000
KAFKA_SERVERS=kafka:9092
OTLP_ENDPOINT=http://collector:4318
IN_DOCKER=true
IN_DOCKER_GO=true
ON_PREM=true
SSL=false
```

---

## 🚀 Deployment Steps in Dokploy

### 1. Create a New Service/Application

In Dokploy dashboard:
- Click **"Create Application"**
- Choose **"Docker Compose"**
- Name it: `highlight-io`

### 2. Upload/Paste the Compose File

Copy the contents of `compose.hobby.complete.yml` into the compose editor.

### 3. Set Environment Variables

In the environment variables section, add:

```env
REACT_APP_FRONTEND_URI=https://first.sume.one
REACT_APP_PRIVATE_GRAPH_URI=https://first.sume.one:8082/private
REACT_APP_PUBLIC_GRAPH_URI=https://first.sume.one:8082/public
ADMIN_PASSWORD=YourSecurePassword123!
BACKEND_IMAGE_NAME=ghcr.io/highlight/highlight-backend:latest
FRONTEND_IMAGE_NAME=ghcr.io/highlight/highlight-frontend:latest
LICENSE_KEY=
```

### 4. Configure Domain Mappings in Dokploy

Set up your domain mappings:

| Service   | Container Port | Domain            |
|-----------|----------------|-------------------|
| frontend  | 3001           | first.sume.one    |
| frontend  | 6006           | second.sume.one   |
| frontend  | 8081           | third.sume.one    |
| backend   | 8082           | (via first.sume.one proxy) |

### 5. Deploy

Click **"Deploy"** and wait for all services to start (may take 2-3 minutes).

### 6. Verify Deployment

Check service health:
```bash
curl https://first.sume.one:8082/health
```

Access the frontend:
```
https://first.sume.one
```

---

## 🔍 Troubleshooting

### Backend keeps restarting

**Check logs:**
```bash
docker logs backend
```

**Common issues:**
- ✅ Make sure PostgreSQL is healthy: `docker exec postgres pg_isready -U postgres`
- ✅ Verify Redis is running: `docker exec redis redis-cli -a redispassword ping`
- ✅ Check all environment variables are set

### Can't connect to database

**Error:** `failed to connect to host=port=5432`

**Solution:** This means `PSQL_HOST` is not set. Use `compose.hobby.complete.yml` which has all variables pre-configured.

### Port already in use

**Error:** `Bind for 0.0.0.0:3000 failed: port is already allocated`

**Solution:** We've already changed the ports to 3001, 6006, 8081. Make sure you're using the updated compose file.

### Enterprise license error

**Error:** `no license key set`

**Solution:** This is just a warning. Set `LICENSE_KEY=` (empty) in environment variables for hobby mode.

---

## 📊 Resource Requirements

Minimum requirements for running all services:

- **RAM:** 4GB minimum, 8GB recommended
- **CPU:** 2 cores minimum, 4 cores recommended
- **Disk:** 20GB minimum for volumes

---

## 🔐 Security Recommendations

1. **Change the admin password** - Don't use `password`
2. **Change Redis password** - Update `REDIS_PASSWORD` in compose file
3. **Use SSL/TLS** - Set `SSL=true` and configure certificates
4. **Restrict port access** - Only expose necessary ports
5. **Regular backups** - Backup Docker volumes regularly

---

## 📚 Additional Resources

- [Highlight.io Documentation](https://www.highlight.io/docs)
- [Highlight.io Self-Hosting Guide](https://www.highlight.io/docs/getting-started/self-host/self-hosted-hobby-guide)
- [Dokploy Documentation](https://dokploy.com/docs)

---

## 💡 Why Two Compose Files?

Highlight.io's official setup uses **two compose files**:

1. **`compose.yml`** - Infrastructure services (postgres, redis, etc.)
2. **`compose.hobby.yml`** - Application services (frontend, backend)

They're meant to be used together:
```bash
docker compose -f compose.yml -f compose.hobby.yml up
```

For Dokploy (and easier deployment), I've merged them into:
- **`compose.hobby.complete.yml`** - Everything in one file ✅

---

## ✅ Summary

- ❌ **Don't use:** `compose.hobby.yml` alone (missing database!)
- ✅ **Use:** `compose.hobby.complete.yml` (includes everything)
- ✅ All ports updated to avoid conflicts (3001, 6006, 8081, 8082)
- ✅ All environment variables pre-configured
- ✅ Ready for Dokploy deployment
- ✅ Domains configured for your setup

Deploy with confidence! 🚀


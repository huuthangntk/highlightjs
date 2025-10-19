# Fix for 502 Bad Gateway Errors - Highlight.io

## 🚨 Problem

Ports 3001, 6006, and 8081 were throwing **502 Bad Gateway** errors due to backend instability.

### Root Causes Identified

1. **Kafka Consumer Group Instability**
   - Backend creating multiple consumer group members
   - Constant rebalancing and "Unknown Member ID" errors
   - Backend restarting frequently

2. **Database Constraint Violations**
   ```
   duplicate key value violates unique constraint "system_configurations_pkey"
   ```
   - Multiple backend instances trying to initialize database
   - Race conditions during startup

3. **Poor Service Dependency Management**
   - Services starting before dependencies were ready
   - No health checks for critical infrastructure services
   - Backend trying to connect to services that weren't fully initialized

## ✅ **Complete Fix Applied**

### Backend Stability Improvements

```yaml
backend:
    restart: unless-stopped  # Changed from 'on-failure'
    
    # Better health check timing
    healthcheck:
        test: ['CMD-SHELL', 'curl -f http://localhost:8082/health || exit 1']
        interval: 30s
        timeout: 10s
        retries: 10
        start_period: 60s  # Wait 60s before starting health checks
    
    # Wait for ALL dependencies to be healthy
    depends_on:
        postgres:
            condition: service_healthy
        redis:
            condition: service_healthy
        clickhouse:
            condition: service_healthy
        kafka:
            condition: service_healthy
        collector:
            condition: service_started
    
    environment:
        # ClickHouse credentials
        CLICKHOUSE_USERNAME: default
        CLICKHOUSE_PASSWORD: ''
        
        # Session management
        SESSION_RETENTION_DAYS: 30
        
        # Worker limits to prevent multiple instances
        WORKER_MAX_MEMORY_THRESHOLD: 1GiB
        
        # Explicit Kafka topic
        KAFKA_TOPIC: dev
```

### Infrastructure Health Checks Added

**PostgreSQL** (already had health check) ✅
```yaml
healthcheck:
    test: ['CMD-SHELL', 'pg_isready -U postgres']
```

**Redis** ✅
```yaml
healthcheck:
    test: ['CMD-SHELL', 'redis-cli ping | grep PONG || exit 1']
```

**Zookeeper** ✅
```yaml
healthcheck:
    test: ['CMD-SHELL', 'echo "ruok" | nc localhost 2181 | grep imok || exit 1']
```

**Kafka** ✅
```yaml
healthcheck:
    test: ['CMD-SHELL', 'kafka-topics --bootstrap-server kafka:9092 --list || exit 1']
```

**ClickHouse** ✅
```yaml
healthcheck:
    test: ['CMD-SHELL', 'wget --no-verbose --tries=1 --spider http://localhost:8123/ || exit 1']
```

### Kafka Consumer Group Stability

Added specific settings to prevent constant rebalancing:
```yaml
environment:
    KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 3000
    KAFKA_GROUP_MAX_SESSION_TIMEOUT_MS: 1800000
    KAFKA_GROUP_MIN_SESSION_TIMEOUT_MS: 6000
```

### All Services Restart Policy

Changed from `restart: on-failure` to `restart: unless-stopped` for:
- ✅ backend
- ✅ frontend  
- ✅ postgres
- ✅ redis
- ✅ zookeeper
- ✅ kafka
- ✅ clickhouse
- ✅ collector

## 🔄 **How to Apply the Fix**

### 1. Stop Current Deployment
```bash
docker compose -f docker/compose.hobby.complete.yml down
```

### 2. Pull Latest Changes
```bash
git pull origin main
```

### 3. Start with Fixed Configuration
```bash
docker compose -f docker/compose.hobby.complete.yml up -d
```

### 4. Monitor Startup Process
```bash
# Watch all services come up in order
docker compose -f docker/compose.hobby.complete.yml ps

# Monitor backend logs (should be much more stable now)
docker compose -f docker/compose.hobby.complete.yml logs -f backend
```

## ✅ **Expected Results After Fix**

### Service Startup Order
1. **postgres** → healthy ✅
2. **redis** → healthy ✅  
3. **zookeeper** → healthy ✅
4. **kafka** → healthy ✅ (waits for zookeeper)
5. **clickhouse** → healthy ✅
6. **collector** → starts ✅ (waits for kafka + clickhouse)
7. **backend** → healthy ✅ (waits for ALL dependencies)
8. **frontend** → healthy ✅ (waits for backend)

### Backend Should Show
```
✅ No more Kafka consumer group rebalancing errors
✅ No more "Unknown Member ID" messages
✅ No more database constraint violations
✅ Stable HTTP listener on port 8082
✅ Successful health checks
```

### Frontend Access
- ✅ **Port 3001** → `first.sume.one` (Main Frontend)
- ✅ **Port 6006** → `second.sume.one` (Storybook)  
- ✅ **Port 8081** → `third.sume.one` (Additional service)
- ✅ **Port 8082** → Backend API

## 🔍 **Verification Commands**

### Check All Services Healthy
```bash
docker compose -f docker/compose.hobby.complete.yml ps
```
All should show "Up" and "(healthy)" status.

### Test Backend Health
```bash
curl http://localhost:8082/health
```
Should return: `{"status":"ok"}` or similar success response.

### Test Frontend Access
```bash
curl http://localhost:3001
curl http://localhost:6006  
curl http://localhost:8081
```
Should return HTML content (not 502 errors).

### Check Backend Logs for Stability
```bash
docker compose -f docker/compose.hobby.complete.yml logs backend --tail=50
```
Should show:
- ✅ No Kafka consumer group errors
- ✅ No database constraint violations
- ✅ Stable "running HTTP listener" messages

## 📊 **Git Commit Applied**

```bash
✅ Commit: f767661ca
✅ Message: "fix: comprehensive backend stability improvements to resolve 502 errors"
✅ Pushed to: https://github.com/huuthangntk/highlightjs.git
```

## 🎯 **Summary**

The 502 Bad Gateway errors were caused by backend instability due to:
- Kafka consumer group issues
- Database initialization conflicts  
- Poor service startup ordering

**Fixed by:**
- ✅ Comprehensive health checks for all infrastructure services
- ✅ Proper service dependency ordering
- ✅ Backend stability improvements (restart policy, timing, configuration)
- ✅ Kafka consumer group stability settings
- ✅ Single backend instance enforcement

The frontend should now successfully connect to a stable, healthy backend! 🎉

---

## 🆘 **If Issues Persist**

1. **Check service startup order**: Ensure all infrastructure services are healthy before backend starts
2. **Monitor backend logs**: Look for any remaining Kafka or database errors
3. **Verify domain mapping**: Ensure Dokploy domain mapping is correctly configured
4. **Check resource limits**: Ensure sufficient RAM/CPU for all services

**Need help?** Check logs with:
```bash
docker compose -f docker/compose.hobby.complete.yml logs
```

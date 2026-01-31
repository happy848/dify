# Dify 配置总结 - 服务器独立部署

## 配置变更总览

### ✅ 已完成的配置修改

#### 1. 移除 Docker 内置服务
- ✅ 移除 Nginx 容器（使用服务器已有的 Nginx）
- ✅ 禁用 Redis 容器（使用服务器已有的 Redis）
- ⚠️ Weaviate 容器保留但不启动（使用 Qdrant）

#### 2. 使用外部服务
- ✅ Redis: `host.docker.internal:6379`
- ✅ Qdrant: `http://host.docker.internal:6333`
- ✅ PostgreSQL: Docker 容器（profile: postgresql）

#### 3. 端口映射
- ✅ API: `127.0.0.1:5001` → 容器 `5001`
- ✅ Web: `127.0.0.1:3000` → 容器 `3000`
- ✅ Plugin Daemon: `127.0.0.1:5002` → 容器 `5002`

## 服务架构

```
┌─────────────────────────────────────────────────────────┐
│                    服务器 (宿主机)                        │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐      │
│  │  Nginx   │      │  Redis   │      │ Qdrant   │      │
│  │  :80/443 │      │  :6379   │      │  :6333   │      │
│  └────┬─────┘      └────┬─────┘      └────┬─────┘      │
│       │                 │                   │            │
│       │    ┌────────────┴───────────────────┘            │
│       │    │                                             │
│       ↓    ↓                                             │
│  ┌─────────────────────────────────────────────────┐    │
│  │           Docker 容器网络                        │    │
│  │  ┌────────┐  ┌────────┐  ┌───────┐  ┌──────┐  │    │
│  │  │  API   │  │  Web   │  │Worker │  │ PG   │  │    │
│  │  │ :5001  │  │ :3000  │  │       │  │:5432 │  │    │
│  │  └────────┘  └────────┘  └───────┘  └──────┘  │    │
│  │                                                  │    │
│  │  ┌────────┐  ┌────────┐  ┌───────┐             │    │
│  │  │Plugin  │  │Sandbox │  │SSRF   │             │    │
│  │  │ :5002  │  │ :8194  │  │Proxy  │             │    │
│  │  └────────┘  └────────┘  └───────┘             │    │
│  └─────────────────────────────────────────────────┘    │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

## 关键配置文件

### 1. `.env.server-nginx` - 主要环境变量

**域名配置：**
```bash
CONSOLE_API_URL=https://dify.agentsben.com
CONSOLE_WEB_URL=https://dify.agentsben.com
APP_API_URL=https://api.dify.agentsben.com
APP_WEB_URL=https://app.agentsben.com
SERVICE_API_URL=https://api.dify.agentsben.com
FILES_URL=https://files.dify.agentsben.com
```

**Redis 配置：**
```bash
REDIS_HOST=host.docker.internal
REDIS_PORT=6379
REDIS_USERNAME=default
REDIS_PASSWORD=sd@234#$dfSDwe23
REDIS_DB=0
CELERY_BROKER_URL=redis://:sd@234#$dfSDwe23@host.docker.internal:6379/1
```

**Qdrant 配置：**
```bash
VECTOR_STORE=qdrant
QDRANT_URL=http://host.docker.internal:6333
QDRANT_API_KEY=f3f8e23r45gh606-6c1a-4c34-8929-0352d4025302
QDRANT_CLIENT_TIMEOUT=30
```

**数据库配置：**
```bash
DB_TYPE=postgresql
DB_USERNAME=dify
DB_PASSWORD=PV6jQcSCAfwdXuz4CAJg5+nVyXqhBabdFp8CDT8r0LU=
DB_HOST=db_postgres
DB_PORT=5432
DB_DATABASE=dify
```

### 2. `docker-compose.yaml` - Docker 编排配置

**主要修改：**
- 移除了 `nginx` 服务定义
- 注释了 `redis` 服务定义
- 所有服务添加了 `extra_hosts` 配置
- 移除了对 `redis` 容器的依赖
- 为 `api`, `web`, `plugin_daemon` 添加端口映射到 `127.0.0.1`

**extra_hosts 配置：**
```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

这个配置使 Docker 容器能够通过 `host.docker.internal` 访问宿主机服务。

### 3. `dify-nginx.conf` - 服务器 Nginx 配置

**主要域名路由：**

**dify.agentsben.com:**
- `/console/api` → API (5001)
- `/api` → API (5001)
- `/v1` → API (5001)
- `/files` → API (5001)
- `/explore` → Web (3000)
- `/e/` → Plugin Daemon (5002)
- `/mcp` → API (5001)
- `/triggers` → API (5001)
- `/` → Web (3000)

**api.dify.agentsben.com:**
- `/` → API (5001)

**app.dify.agentsben.com:**
- `/` → Web (3000)

**files.dify.agentsben.com:**
- `/` → API (5001)

## 部署流程

### 快速部署（推荐）

```bash
cd /Users/chunlinle/dev/dify/docker

# 使用自动化脚本
./start-dify.sh
```

### 手动部署步骤

#### 1. 准备配置文件
```bash
cd /Users/chunlinle/dev/dify/docker

# 复制配置文件
cp .env.server-nginx .env
```

#### 2. 配置服务器 Nginx
```bash
# 复制 Nginx 配置
sudo cp dify-nginx.conf /etc/nginx/sites-available/dify
sudo ln -s /etc/nginx/sites-available/dify /etc/nginx/sites-enabled/

# 获取 SSL 证书
sudo certbot certonly --nginx \
  -d dify.agentsben.com \
  -d api.dify.agentsben.com \
  -d app.dify.agentsben.com \
  -d files.dify.agentsben.com

# 测试并重载 Nginx
sudo nginx -t
sudo systemctl reload nginx
```

#### 3. 启动 Docker 服务
```bash
# 创建数据目录
sudo mkdir -p /data/dify/storage /data/dify/plugins
sudo chown -R 1000:1000 /data/dify

# 启动服务（使用 PostgreSQL profile）
docker-compose --profile postgresql up -d

# 查看日志
docker-compose logs -f api
```

#### 4. 验证部署
```bash
# 检查服务状态
docker-compose ps

# 测试 API
curl http://localhost:5001/health
curl https://api.dify.agentsben.com/health

# 测试 Web
curl http://localhost:3000
curl https://dify.agentsben.com

# 检查 Redis 连接
docker-compose exec api python -c "
import redis
r = redis.Redis(host='host.docker.internal', port=6379, password='sd@234#\$dfSDwe23')
print('Redis:', r.ping())
"

# 检查 Qdrant 连接
docker-compose exec api python -c "
import requests
r = requests.get('http://host.docker.internal:6333/')
print('Qdrant:', r.status_code, r.json())
"
```

## 故障排查清单

### ✓ 检查列表

1. **外部服务可用性**
   ```bash
   # Redis
   redis-cli -h localhost -p 6379 -a 'sd@234#$dfSDwe23' ping
   
   # Qdrant
   curl http://localhost:6333/
   ```

2. **Docker 容器状态**
   ```bash
   docker-compose ps
   docker-compose logs api | grep -i error
   ```

3. **网络连接**
   ```bash
   # 从容器内测试
   docker-compose exec api ping -c 3 host.docker.internal
   docker-compose exec api curl http://host.docker.internal:6379
   docker-compose exec api curl http://host.docker.internal:6333
   ```

4. **端口监听**
   ```bash
   sudo ss -tlnp | grep -E ':(3000|5001|5002|6379|6333)'
   ```

5. **Nginx 配置**
   ```bash
   sudo nginx -t
   sudo systemctl status nginx
   curl -I https://dify.agentsben.com
   ```

## 常见问题解决

### 问题 1: Redis 连接被拒绝

**症状：** 日志显示 `Connection refused` 或 `Error connecting to Redis`

**解决方案：**
```bash
# 检查 Redis 监听地址
sudo ss -tlnp | grep 6379

# 如果只监听 127.0.0.1，修改 Redis 配置
sudo vim /etc/redis/redis.conf
# 修改: bind 127.0.0.1 ::1 改为 bind 0.0.0.0
sudo systemctl restart redis

# 检查防火墙
sudo ufw allow from 172.17.0.0/16 to any port 6379
```

### 问题 2: Qdrant 连接失败

**症状：** 日志显示 Qdrant 连接超时或失败

**解决方案：**
```bash
# 检查 Qdrant 是否运行
sudo systemctl status qdrant
# 或
docker ps | grep qdrant

# 检查监听地址
sudo ss -tlnp | grep 6333

# 测试连接
curl http://localhost:6333/
curl http://localhost:6333/collections
```

### 问题 3: host.docker.internal 无法解析

**症状：** 容器内无法解析 `host.docker.internal`

**解决方案：**
```bash
# 方法 1: 使用宿主机 IP
ip addr show docker0 | grep inet | awk '{print $2}' | cut -d/ -f1
# 更新 .env 文件，使用实际 IP 替代 host.docker.internal

# 方法 2: 检查 docker-compose.yaml 中的 extra_hosts 配置
docker-compose exec api cat /etc/hosts | grep host.docker.internal
```

### 问题 4: 502 Bad Gateway

**症状：** Nginx 返回 502 错误

**解决方案：**
```bash
# 检查 Docker 服务是否运行
docker-compose ps

# 检查服务日志
docker-compose logs api | tail -50

# 检查端口监听
sudo ss -tlnp | grep -E ':(3000|5001|5002)'

# 重启服务
docker-compose restart api web
```

## 性能调优

### 根据服务器配置调整

**2核4GB 服务器：**
```bash
SERVER_WORKER_AMOUNT=1
CELERY_WORKER_AMOUNT=1
PM2_INSTANCES=1
POSTGRES_SHARED_BUFFERS=128MB
```

**4核8GB 服务器：**
```bash
SERVER_WORKER_AMOUNT=2
CELERY_WORKER_AMOUNT=2
PM2_INSTANCES=2
POSTGRES_SHARED_BUFFERS=256MB
```

**8核16GB 服务器：**
```bash
SERVER_WORKER_AMOUNT=4
CELERY_WORKER_AMOUNT=4
PM2_INSTANCES=4
POSTGRES_SHARED_BUFFERS=512MB
```

## 维护操作

### 更新服务
```bash
cd /Users/chunlinle/dev/dify/docker

# 拉取最新镜像
docker-compose pull

# 重启服务
docker-compose up -d
```

### 备份数据
```bash
# 备份数据库
docker-compose exec db_postgres pg_dump -U dify -d dify | gzip > backup_$(date +%Y%m%d).sql.gz

# 备份文件存储
tar -czf storage_backup_$(date +%Y%m%d).tar.gz /data/dify/storage
```

### 查看日志
```bash
# 实时日志
docker-compose logs -f

# 特定服务
docker-compose logs -f api
docker-compose logs -f worker

# 搜索错误
docker-compose logs | grep -i error
```

## 相关文档

- [DEPLOYMENT.md](./DEPLOYMENT.md) - 详细部署指南
- [NGINX-SETUP.md](./NGINX-SETUP.md) - Nginx 配置说明
- [EXTERNAL-SERVICES.md](./EXTERNAL-SERVICES.md) - 外部服务配置详解
- [start-dify.sh](./start-dify.sh) - 自动化启动脚本

## 技术支持

如遇到问题，请按以下顺序排查：

1. 查看相关文档
2. 检查服务日志：`docker-compose logs -f`
3. 验证外部服务连接（Redis、Qdrant）
4. 检查网络和防火墙配置
5. 查看 Dify 官方文档：https://docs.dify.ai

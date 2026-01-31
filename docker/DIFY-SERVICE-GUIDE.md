# Dify 服务特性与调试指南

**版本**: Dify 1.11.4  
**更新时间**: 2026-01-31  
**部署模式**: Docker Compose + 外部 Nginx

---

## 📋 目录

- [1. 服务架构](#1-服务架构)
- [2. 核心特性](#2-核心特性)
- [3. 容器组件说明](#3-容器组件说明)
- [4. 配置管理](#4-配置管理)
- [5. 调试技巧](#5-调试技巧)
- [6. 常见问题排查](#6-常见问题排查)
- [7. 性能优化](#7-性能优化)
- [8. 监控与维护](#8-监控与维护)
- [9. 最佳实践](#9-最佳实践)

---

## 1. 服务架构

### 1.1 整体架构

```
用户请求
    ↓
服务器 Nginx (443/80)
    ↓
┌─────────────────────────────────────┐
│         Dify Docker 环境             │
├─────────────────────────────────────┤
│ Web (3000) → API (5001)             │
│     ↓              ↓                 │
│ Worker + Worker Beat (Celery)       │
│     ↓              ↓                 │
│ Plugin Daemon (5002)                │
│     ↓              ↓                 │
│ PostgreSQL + Redis + Qdrant         │
│     ↓              ↓                 │
│ Sandbox + SSRF Proxy                │
└─────────────────────────────────────┘
```

### 1.2 网络架构

- **docker_default**: 默认网络,所有服务互联
- **ssrf_proxy_network**: 隔离网络,防止 SSRF 攻击
- **外部依赖**: Redis (host:10.0.0.5:6379), Qdrant (host:10.0.0.5:6333)

### 1.3 数据流向

```
用户 → Web UI → API Server → Worker (异步任务)
                    ↓
              Plugin Daemon (插件执行)
                    ↓
              AI Models (LLM/Embedding)
                    ↓
              Vector Store (Qdrant)
```

---

## 2. 核心特性

### 2.1 主要功能

1. **LLM 应用开发平台**
   - 可视化工作流编排
   - Prompt 工程与管理
   - 对话式应用构建
   - Agent 智能体开发

2. **模型管理**
   - 多模型供应商支持 (OpenAI, Anthropic, 国产大模型等)
   - 自定义模型接入
   - 插件系统扩展

3. **知识库**
   - 文档上传与解析
   - 向量化存储 (Qdrant/Weaviate/Milvus 等)
   - RAG (检索增强生成)

4. **插件生态**
   - Python 插件系统
   - 工具扩展
   - 模型供应商插件

### 2.2 技术栈

**后端**:
- Python 3.12 (FastAPI/Flask)
- Celery (异步任务队列)
- SQLAlchemy (ORM)
- PostgreSQL (主数据库)
- Redis (缓存 + 队列)

**前端**:
- Next.js 15.5.9
- React
- TypeScript

**向量存储**:
- Qdrant (默认推荐)
- 支持 20+ 向量数据库

---

## 3. 容器组件说明

### 3.1 核心服务容器

#### **api** - API 服务器
- **镜像**: `langgenius/dify-api:1.11.4`
- **端口**: `127.0.0.1:5001:5001`
- **功能**: 处理所有 API 请求,业务逻辑核心
- **资源**: ~545MB 内存
- **健康检查**: `curl http://localhost:5001/health`

#### **web** - 前端服务
- **镜像**: `langgenius/dify-web:1.11.4`
- **端口**: `127.0.0.1:3000:3000`
- **功能**: Next.js SSR 前端应用
- **资源**: ~275MB 内存
- **进程**: PM2 管理,2 个实例

#### **worker** - 异步任务处理
- **镜像**: `langgenius/dify-api:1.11.4`
- **功能**: 处理所有 Celery 队列任务
  - `dataset`: 知识库索引
  - `workflow`: 工作流执行
  - `mail`: 邮件发送
  - `app_deletion`: 应用删除
  - `plugin`: 插件任务
- **资源**: ~273MB 内存

#### **worker_beat** - 定时任务调度
- **镜像**: `langgenius/dify-api:1.11.4`
- **功能**: Celery Beat 定时任务调度器
- **资源**: ~280MB 内存

#### **plugin_daemon** - 插件运行时
- **镜像**: `langgenius/dify-plugin-daemon:0.5.3-local`
- **端口**: `127.0.0.1:5002:5002`
- **功能**: 隔离环境运行 Python 插件
- **资源**: ~28MB 内存
- **SDK**: dify_plugin 0.7.1

### 3.2 存储服务

#### **db_postgres** - PostgreSQL 数据库
- **镜像**: `postgres:15-alpine`
- **端口**: `5432` (内部)
- **功能**: 主数据库,存储应用/用户/配置等
- **资源**: ~98MB 内存
- **Profile**: `postgresql` (需显式启动)

#### **Redis** (外部容器)
- **地址**: `10.0.0.5:6379`
- **功能**: 缓存 + Celery Broker
- **配置**:
  - 内存上限: 2GB
  - 淘汰策略: allkeys-lru
- **资源**: ~545MB 内存

#### **Qdrant** (外部容器)
- **地址**: `10.0.0.5:6333`
- **功能**: 向量数据库
- **资源**: ~35MB 内存

### 3.3 辅助服务

#### **sandbox** - 代码沙箱
- **镜像**: `langgenius/dify-sandbox:0.2.12`
- **端口**: `8194` (内部)
- **功能**: 安全执行用户代码
- **资源**: ~59MB 内存

#### **ssrf_proxy** - SSRF 防护代理
- **镜像**: `ubuntu/squid:latest`
- **端口**: `3128` (内部)
- **功能**: 防止服务器端请求伪造攻击
- **网络**: ssrf_proxy_network (隔离)

---

## 4. 配置管理

### 4.1 环境变量配置

**核心配置文件**: `.env`

```bash
# 域名配置
CONSOLE_API_URL=https://dify.agentsben.com
CONSOLE_WEB_URL=https://dify.agentsben.com
APP_API_URL=https://api.dify.agentsben.com
APP_WEB_URL=https://app.dify.agentsben.com

# 数据库配置
DB_TYPE=postgresql
DB_HOST=db_postgres
DB_PORT=5432
DB_USERNAME=dify
DB_PASSWORD=<your-password>
DB_DATABASE=dify

# Redis 配置 (使用宿主机内网 IP)
REDIS_HOST=10.0.0.5
REDIS_PORT=6379
REDIS_PASSWORD=<your-password>
CELERY_BROKER_URL=redis://:password@10.0.0.5:6379/1

# 向量数据库配置
VECTOR_STORE=qdrant
QDRANT_URL=http://10.0.0.5:6333
QDRANT_API_KEY=<your-api-key>

# 存储配置
STORAGE_TYPE=opendal
OPENDAL_SCHEME=fs
OPENDAL_FS_ROOT=/app/api/storage

# 性能配置
SERVER_WORKER_AMOUNT=2
CELERY_WORKER_AMOUNT=2
PM2_INSTANCES=2
```

### 4.2 关键配置说明

**网络配置注意事项**:
- ✅ Redis 使用 host 网络 → 需要使用宿主机内网 IP (`10.0.0.5`)
- ✅ Qdrant 使用独立网络 → 同样使用宿主机内网 IP
- ❌ 不要使用 `host.docker.internal` (在 Linux 上不可靠)
- ❌ 不要使用 `127.0.0.1` (指向容器内部)

**Profile 系统**:
```bash
# PostgreSQL 需要显式启动
docker-compose --profile postgresql up -d

# 其他可用 profiles
--profile weaviate    # 使用 Weaviate 向量库
--profile qdrant      # 使用内置 Qdrant
--profile mysql       # 使用 MySQL 替代 PostgreSQL
```

### 4.3 Docker Compose 关键配置

```yaml
# 依赖关系
api:
  depends_on:
    init_permissions:
      condition: service_completed_successfully
    db_postgres:
      condition: service_healthy
      required: false  # 可选依赖

# 健康检查
db_postgres:
  healthcheck:
    test: ["CMD", "pg_isready", "-h", "db_postgres"]
    interval: 1s
    timeout: 3s
    retries: 60

# 网络隔离
sandbox:
  networks:
    - ssrf_proxy_network  # 隔离网络
```

---

## 5. 调试技巧

### 5.1 日志查看

#### 实时日志
```bash
# 查看所有服务日志
docker-compose logs -f

# 查看特定服务日志
docker logs -f docker-api-1
docker logs -f docker-worker-1
docker logs -f docker-plugin_daemon-1

# 查看最近 N 行日志
docker logs --tail 50 docker-api-1

# 查看带时间戳的日志
docker logs -f --timestamps docker-api-1
```

#### 日志过滤
```bash
# 只看错误日志
docker logs docker-api-1 2>&1 | grep -i error

# 查看特定时间范围
docker logs --since 10m docker-api-1

# 查看插件相关日志
docker logs docker-plugin_daemon-1 2>&1 | grep -E 'ERROR|plugin'
```

### 5.2 容器状态检查

#### 快速检查
```bash
# 查看所有容器状态
docker-compose ps

# 查看容器资源使用
docker stats --no-stream

# 查看容器详细信息
docker inspect docker-api-1

# 检查容器健康状态
docker ps --filter "health=unhealthy"
```

#### 健康检查
```bash
# API 健康检查
curl http://localhost:5001/health
# 预期返回: {"pid": 19, "status": "ok", "version": "1.11.4"}

# Web 前端检查
curl -I http://localhost:3000

# PostgreSQL 检查
docker exec docker-db_postgres-1 pg_isready

# Redis 检查 (外部)
redis-cli -h 10.0.0.5 -p 6379 -a 'password' ping
```

### 5.3 进入容器调试

```bash
# 进入 API 容器
docker exec -it docker-api-1 bash

# 进入 PostgreSQL 容器
docker exec -it docker-db_postgres-1 psql -U dify -d dify

# 进入 Plugin Daemon 容器
docker exec -it docker-plugin_daemon-1 sh

# 在容器内执行命令
docker exec docker-api-1 ps aux
docker exec docker-worker-1 celery -A celery_entrypoint.celery inspect active
```

### 5.4 网络调试

```bash
# 查看网络列表
docker network ls

# 查看网络详情
docker network inspect docker_default

# 检查容器连接
docker exec docker-api-1 ping db_postgres
docker exec docker-api-1 curl http://sandbox:8194/health

# 检查端口监听
docker exec docker-api-1 netstat -tlnp
```

### 5.5 数据库调试

#### PostgreSQL
```bash
# 连接数据库
docker exec -it docker-db_postgres-1 psql -U dify -d dify

# 常用 SQL
\dt              # 列出所有表
\d+ tablename    # 查看表结构
SELECT version();
SELECT COUNT(*) FROM apps;
```

#### Redis
```bash
# 连接 Redis
redis-cli -h 10.0.0.5 -p 6379 -a 'password'

# 常用命令
INFO memory              # 内存使用
CONFIG GET maxmemory     # 内存限制
KEYS celery*             # 查看 Celery 队列
DBSIZE                   # 键数量
```

### 5.6 插件调试

```bash
# 查看已安装插件
docker exec docker-plugin_daemon-1 ls -la /app/storage/cwd/langgenius/

# 查看插件日志
docker logs docker-plugin_daemon-1 2>&1 | grep -A 10 "plugin"

# 查看插件 SDK 版本
docker exec docker-plugin_daemon-1 pip list | grep dify

# 删除问题插件
docker exec docker-plugin_daemon-1 rm -rf /app/storage/cwd/langgenius/plugin-name

# 重启插件服务
docker-compose restart plugin_daemon
```

---

## 6. 常见问题排查

### 6.1 容器启动问题

#### 问题: PostgreSQL 未启动
```bash
# 现象
docker ps | grep postgres  # 无结果

# 原因
# PostgreSQL 使用 profile,默认不启动

# 解决
docker-compose --profile postgresql up -d
```

#### 问题: plugin_daemon 反复重启
```bash
# 查看日志
docker logs docker-plugin_daemon-1 2>&1 | tail -50

# 常见原因 1: Redis 连接失败
# 检查 Redis 地址配置
grep REDIS_HOST .env
# 应该是: REDIS_HOST=10.0.0.5 (内网 IP)
# 错误示例: REDIS_HOST=host.docker.internal

# 常见原因 2: 数据库连接失败
# 检查 PostgreSQL 是否运行
docker-compose ps | grep postgres

# 常见原因 3: 插件兼容性问题
# 删除不兼容的插件
docker exec docker-plugin_daemon-1 rm -rf /app/storage/cwd/langgenius/problematic-plugin
docker-compose restart plugin_daemon
```

### 6.2 网络连接问题

#### 问题: 容器无法连接 Redis/Qdrant
```bash
# 检查配置
cat .env | grep -E "REDIS_HOST|QDRANT_URL"

# 应该使用宿主机内网 IP
REDIS_HOST=10.0.0.5      # ✅ 正确
REDIS_HOST=127.0.0.1     # ❌ 错误 (指向容器内部)
REDIS_HOST=localhost     # ❌ 错误

# 检查 Redis 绑定地址
redis-cli -h 10.0.0.5 -a 'password' CONFIG GET bind
# 应该返回: bind 0.0.0.0

# 测试连接
docker exec docker-api-1 telnet 10.0.0.5 6379
```

#### 问题: 外部无法访问服务
```bash
# 检查端口绑定
docker ps | grep dify

# 正确绑定示例
127.0.0.1:5001->5001/tcp    # ✅ 只允许本地访问
0.0.0.0:5001->5001/tcp      # ⚠️  暴露到公网

# 检查 Nginx 配置
nginx -t
systemctl status nginx
```

### 6.3 性能问题

#### 问题: 内存不足
```bash
# 检查系统内存
free -h

# 检查容器内存使用
docker stats --no-stream

# 检查 Redis 内存
redis-cli -h 10.0.0.5 -a 'password' INFO memory

# 解决方案
# 1. 增加 Swap
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# 2. 限制容器内存
# 在 docker-compose.yaml 中添加
services:
  api:
    deploy:
      resources:
        limits:
          memory: 1G
```

#### 问题: Redis 内存满
```bash
# 检查 Redis 内存
redis-cli INFO memory | grep maxmemory

# 调整内存限制
redis-cli CONFIG SET maxmemory 2gb

# 调整淘汰策略
redis-cli CONFIG SET maxmemory-policy allkeys-lru

# 持久化配置 (修改 redis.conf)
maxmemory 2g
maxmemory-policy allkeys-lru
```

### 6.4 插件问题

#### 问题: 插件安装失败 - ImportError
```bash
# 现象
ERROR: ImportError: cannot import name 'MultiModalContent'

# 原因
# 插件使用的 API 与当前 dify_plugin SDK 版本不兼容

# 解决方案 1: 删除不兼容插件
docker exec docker-plugin_daemon-1 rm -rf /app/storage/cwd/langgenius/plugin-xxx
docker-compose restart plugin_daemon

# 解决方案 2: 使用 OpenAI 兼容接口
# 在 Dify UI 中配置自定义模型,无需插件
```

#### 问题: 插件运行缓慢
```bash
# 检查插件资源使用
docker stats docker-plugin_daemon-1

# 检查插件进程
docker exec docker-plugin_daemon-1 ps aux

# 增加插件超时时间
# 在 .env 中配置
PLUGIN_MAX_EXECUTION_TIMEOUT=600
```

### 6.5 数据库问题

#### 问题: PostgreSQL 连接数满
```bash
# 查看当前连接数
docker exec docker-db_postgres-1 psql -U dify -d dify -c \
  "SELECT count(*) FROM pg_stat_activity;"

# 查看最大连接数
docker exec docker-db_postgres-1 psql -U dify -d dify -c \
  "SHOW max_connections;"

# 调整连接数 (在 .env 中)
POSTGRES_MAX_CONNECTIONS=200
SQLALCHEMY_POOL_SIZE=30
SQLALCHEMY_MAX_OVERFLOW=10

# 重启服务
docker-compose restart api worker worker_beat
```

---

## 7. 性能优化

### 7.1 Worker 配置优化

```bash
# .env 配置
# 根据 CPU 核心数调整
SERVER_WORKER_AMOUNT=2              # API workers (推荐: CPU 核心数)
CELERY_WORKER_AMOUNT=2              # Celery workers
PM2_INSTANCES=2                     # Web 前端实例

# Celery 自动扩展 (可选)
CELERY_AUTO_SCALE=true
CELERY_MAX_WORKERS=4
CELERY_MIN_WORKERS=2
```

### 7.2 数据库优化

```bash
# PostgreSQL 优化 (根据服务器内存调整)
POSTGRES_SHARED_BUFFERS=256MB       # 推荐: 系统内存的 25%
POSTGRES_EFFECTIVE_CACHE_SIZE=1GB   # 推荐: 系统内存的 50%
POSTGRES_WORK_MEM=8MB
POSTGRES_MAINTENANCE_WORK_MEM=128MB

# 连接池优化
SQLALCHEMY_POOL_SIZE=30
SQLALCHEMY_MAX_OVERFLOW=10
SQLALCHEMY_POOL_RECYCLE=3600        # 1小时回收连接
```

### 7.3 Redis 优化

```bash
# 内存限制
maxmemory 2g

# 淘汰策略
maxmemory-policy allkeys-lru         # 推荐

# 其他优化
# save ""                            # 禁用 RDB (提升性能,降低持久性)
# appendonly yes                     # 启用 AOF (提升持久性,降低性能)
```

### 7.4 缓存策略

```bash
# 启用文件访问缓存
FILES_ACCESS_TIMEOUT=300

# API 超时配置
API_TOOL_DEFAULT_CONNECT_TIMEOUT=10
API_TOOL_DEFAULT_READ_TIMEOUT=60
GUNICORN_TIMEOUT=360
```

---

## 8. 监控与维护

### 8.1 监控指标

#### 系统级监控
```bash
# 内存监控
free -h
watch -n 5 'free -h'

# CPU 监控
top
htop

# 磁盘监控
df -h
du -sh /path/to/dify/*

# Swap 监控
swapon --show
cat /proc/sys/vm/swappiness
```

#### 容器级监控
```bash
# 实时监控
docker stats

# 持续监控脚本
watch -n 5 'docker stats --no-stream | head -10'

# 资源使用报告
docker stats --no-stream --format \
  "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}"
```

#### 应用级监控
```bash
# API 健康检查
curl http://localhost:5001/health

# 检查 Celery 队列
docker exec docker-worker-1 celery -A celery_entrypoint.celery inspect active

# 检查数据库连接
docker exec docker-db_postgres-1 psql -U dify -d dify -c \
  "SELECT count(*) FROM pg_stat_activity WHERE state = 'active';"

# 检查 Redis 连接
redis-cli -h 10.0.0.5 -a 'password' CLIENT LIST | wc -l
```

### 8.2 日志管理

#### 日志轮转配置
```bash
# Docker 日志配置 (在 daemon.json 中)
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}

# 重启 Docker
systemctl restart docker
```

#### 日志清理
```bash
# 查看日志大小
du -sh /var/lib/docker/containers/*

# 清理所有容器日志
truncate -s 0 /var/lib/docker/containers/*/*-json.log

# 清理特定容器日志
docker logs --tail 0 docker-api-1 2>/dev/null
```

### 8.3 备份策略

#### 数据库备份
```bash
# PostgreSQL 备份
docker exec docker-db_postgres-1 pg_dump -U dify dify > \
  backup_$(date +%Y%m%d_%H%M%S).sql

# 恢复
docker exec -i docker-db_postgres-1 psql -U dify -d dify < backup.sql
```

#### 文件备份
```bash
# 备份存储目录
tar -czf dify_storage_$(date +%Y%m%d).tar.gz \
  volumes/app/storage

# 备份配置
tar -czf dify_config_$(date +%Y%m%d).tar.gz \
  .env docker-compose.yaml dify-nginx.conf
```

#### Redis 备份
```bash
# 触发 Redis 备份
redis-cli -h 10.0.0.5 -a 'password' SAVE

# 复制 RDB 文件
docker cp redis:/data/dump.rdb ./backup/
```

### 8.4 定期维护任务

#### 每日任务
```bash
# 检查服务状态
docker-compose ps

# 检查磁盘空间
df -h

# 检查日志异常
docker-compose logs --since 24h | grep -i error
```

#### 每周任务
```bash
# 清理未使用的 Docker 资源
docker system prune -f

# 数据库备份
./backup_database.sh

# 检查更新
docker-compose pull
```

#### 每月任务
```bash
# 完整备份
tar -czf dify_full_backup_$(date +%Y%m).tar.gz \
  volumes/ .env docker-compose.yaml

# 检查 Dify 更新
# https://github.com/langgenius/dify/releases

# 更新文档
vim DIFY-SERVICE-GUIDE.md
```

---

## 9. 最佳实践

### 9.1 安全加固

#### 网络安全
```bash
# 1. 使用内部 IP 绑定端口
# docker-compose.yaml
ports:
  - "127.0.0.1:5001:5001"    # ✅ 只允许本地访问
  # - "0.0.0.0:5001:5001"    # ❌ 暴露到公网

# 2. 使用 Nginx 反向代理
# 在 Nginx 中配置 SSL、限流、防护

# 3. 配置防火墙
ufw allow 80/tcp
ufw allow 443/tcp
ufw deny 5001/tcp    # 拒绝直接访问 API
```

#### 密钥管理
```bash
# 1. 生成强密钥
openssl rand -base64 42

# 2. 定期轮换密钥
SECRET_KEY=<new-key>
PLUGIN_DAEMON_KEY=<new-key>

# 3. 不要在代码中硬编码密钥
# ❌ SECRET_KEY="my-secret-key"
# ✅ SECRET_KEY=${SECRET_KEY}
```

#### 权限控制
```bash
# 1. 限制文件权限
chmod 600 .env
chmod 600 volumes/app/storage

# 2. 使用非 root 用户运行 (在 docker-compose.yaml)
user: "1001:1001"

# 3. 只读挂载 (适用于配置文件)
volumes:
  - ./config.yaml:/app/config.yaml:ro
```

### 9.2 高可用部署

#### 多实例部署
```bash
# 使用 Docker Swarm 或 Kubernetes
# 示例: docker-compose.yaml
services:
  api:
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
```

#### 负载均衡
```nginx
# Nginx 负载均衡配置
upstream dify_api {
    least_conn;
    server 127.0.0.1:5001;
    server 127.0.0.1:5002;
    server 127.0.0.1:5003;
}
```

### 9.3 开发调试技巧

#### 本地开发环境
```bash
# 1. 使用开发配置
cp .env.example .env.dev
# 修改 DEBUG=true

# 2. 挂载代码目录 (热重载)
volumes:
  - ./api:/app/api:ro

# 3. 使用开发镜像
image: langgenius/dify-api:dev
```

#### 快速重启
```bash
# 只重启应用容器 (不重启数据库)
docker-compose restart api worker worker_beat web

# 重新构建并启动
docker-compose up -d --build api

# 查看构建日志
docker-compose logs -f api
```

### 9.4 故障恢复

#### 快速回滚
```bash
# 1. 保留旧版本镜像
docker tag langgenius/dify-api:1.11.4 langgenius/dify-api:1.11.4-backup

# 2. 备份配置
cp .env .env.backup
cp docker-compose.yaml docker-compose.yaml.backup

# 3. 回滚
docker-compose down
docker-compose up -d --force-recreate
```

#### 紧急修复流程
```bash
# 1. 确认问题
docker-compose ps
docker-compose logs --tail 100

# 2. 隔离问题服务
docker-compose stop problematic-service

# 3. 修复或回滚
# 修复配置 / 删除问题数据 / 回滚版本

# 4. 验证修复
docker-compose up -d problematic-service
docker logs -f problematic-service

# 5. 恢复服务
docker-compose up -d
```

---

## 附录 A: 快速命令参考

### 服务管理
```bash
# 启动所有服务
docker-compose --profile postgresql up -d

# 停止所有服务
docker-compose down

# 重启特定服务
docker-compose restart api worker

# 查看状态
docker-compose ps

# 查看日志
docker-compose logs -f

# 扩展服务实例
docker-compose up -d --scale worker=3
```

### 健康检查
```bash
# API
curl http://localhost:5001/health

# Web
curl -I http://localhost:3000

# PostgreSQL
docker exec docker-db_postgres-1 pg_isready

# Redis
redis-cli -h 10.0.0.5 -a 'password' ping
```

### 资源监控
```bash
# 容器状态
docker ps

# 资源使用
docker stats --no-stream

# 系统内存
free -h

# 磁盘空间
df -h
```

### 日志查询
```bash
# 实时日志
docker logs -f docker-api-1

# 错误日志
docker logs docker-api-1 2>&1 | grep -i error

# 最近 N 行
docker logs --tail 50 docker-api-1

# 时间范围
docker logs --since 10m docker-api-1
```

---

## 附录 B: 问题排查清单

### 启动失败
- [ ] 检查 `.env` 配置是否正确
- [ ] 检查 PostgreSQL 是否启动 (`--profile postgresql`)
- [ ] 检查端口是否被占用 (`netstat -tlnp | grep 5001`)
- [ ] 检查 Redis/Qdrant 地址配置 (使用内网 IP)
- [ ] 查看容器日志 (`docker logs`)

### 连接失败
- [ ] 检查网络配置 (Redis/Qdrant IP)
- [ ] 检查 Redis 绑定地址 (`CONFIG GET bind`)
- [ ] 测试网络连通性 (`telnet 10.0.0.5 6379`)
- [ ] 检查防火墙规则
- [ ] 验证密码配置

### 性能问题
- [ ] 检查内存使用 (`free -h`, `docker stats`)
- [ ] 检查 Redis 内存 (`INFO memory`)
- [ ] 检查 PostgreSQL 连接数
- [ ] 检查磁盘空间 (`df -h`)
- [ ] 查看 Worker 队列积压

### 插件问题
- [ ] 检查插件 SDK 版本兼容性
- [ ] 查看插件日志 (grep 'plugin')
- [ ] 删除不兼容插件
- [ ] 验证插件依赖
- [ ] 考虑使用 OpenAI 兼容接口替代

---

## 附录 C: 联系与资源

### 官方资源
- **官网**: https://dify.ai
- **GitHub**: https://github.com/langgenius/dify
- **文档**: https://docs.dify.ai
- **Discord**: https://discord.gg/dify

### 插件市场
- **Marketplace**: https://marketplace.dify.ai
- **插件开发文档**: https://docs.dify.ai/plugins

### 版本信息
- **当前版本**: 1.11.4
- **Plugin Daemon**: 0.5.3-local
- **dify_plugin SDK**: 0.7.1
- **Docker Compose**: v2.x

---

**文档维护**: 根据实际部署经验持续更新  
**最后更新**: 2026-01-31  
**维护者**: DevOps Team

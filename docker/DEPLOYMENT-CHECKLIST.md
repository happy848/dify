# Dify 部署检查清单

使用此清单确保所有步骤都正确完成。

## 📋 部署前检查

### 服务器环境

- [ ] 服务器操作系统：Ubuntu 20.04+ / Debian 11+
- [ ] CPU：至少 2 核（推荐 4 核）
- [ ] 内存：至少 4GB（推荐 8GB）
- [ ] 磁盘空间：至少 50GB 可用

### 必需软件

- [ ] Docker 已安装：`docker --version`
- [ ] Docker Compose 已安装：`docker-compose --version`
- [ ] Nginx 已安装并运行：`sudo systemctl status nginx`
- [ ] Certbot 已安装（用于 SSL）：`certbot --version`

### 外部服务

- [ ] Redis 正在运行
  ```bash
  redis-cli -h localhost -p 6379 -a 'sd@234#$dfSDwe23' ping
  # 应返回: PONG
  ```

- [ ] Qdrant 正在运行
  ```bash
  curl http://localhost:6333/
  # 应返回 JSON 响应
  ```

### DNS 解析

- [ ] `dify.agentsben.com` 指向服务器 IP
- [ ] `api.dify.agentsben.com` 指向服务器 IP
- [ ] `app.dify.agentsben.com` 指向服务器 IP
- [ ] `files.dify.agentsben.com` 指向服务器 IP

验证：
```bash
dig +short dify.agentsben.com
dig +short api.dify.agentsben.com
dig +short app.dify.agentsben.com
dig +short files.dify.agentsben.com
```

---

## 📝 配置文件检查

### 1. .env 文件

- [ ] 已复制 `.env.server-nginx` 到 `.env`
  ```bash
  cd /Users/chunlinle/dev/dify/docker
  cp .env.server-nginx .env
  ```

- [ ] 检查关键配置项：
  ```bash
  grep "REDIS_HOST" .env
  # 应显示: REDIS_HOST=host.docker.internal
  
  grep "REDIS_PASSWORD" .env
  # 应显示: REDIS_PASSWORD=sd@234#$dfSDwe23
  
  grep "VECTOR_STORE" .env
  # 应显示: VECTOR_STORE=qdrant
  
  grep "QDRANT_URL" .env
  # 应显示: QDRANT_URL=http://host.docker.internal:6333
  ```

### 2. Nginx 配置

- [ ] 已复制 Nginx 配置文件
  ```bash
  sudo cp dify-nginx.conf /etc/nginx/sites-available/dify
  ```

- [ ] 已创建软链接
  ```bash
  sudo ln -s /etc/nginx/sites-available/dify /etc/nginx/sites-enabled/
  ```

- [ ] Nginx 配置测试通过
  ```bash
  sudo nginx -t
  # 应显示: syntax is ok
  # 应显示: test is successful
  ```

### 3. SSL 证书

- [ ] 已获取 SSL 证书
  ```bash
  sudo ls -la /etc/letsencrypt/live/dify.agentsben.com/
  # 应看到: fullchain.pem, privkey.pem
  ```

或使用 Certbot 获取：
```bash
sudo certbot certonly --nginx \
  -d dify.agentsben.com \
  -d api.dify.agentsben.com \
  -d app.dify.agentsben.com \
  -d files.dify.agentsben.com
```

---

## 🚀 部署步骤检查

### 1. 数据目录准备

- [ ] 创建数据目录
  ```bash
  sudo mkdir -p /data/dify/storage
  sudo mkdir -p /data/dify/plugins
  ```

- [ ] 设置正确的权限
  ```bash
  sudo chown -R 1000:1000 /data/dify
  ```

- [ ] 验证权限
  ```bash
  ls -la /data/dify/
  # 应显示 owner 为 1000:1000
  ```

### 2. Docker 镜像

- [ ] 拉取最新镜像
  ```bash
  cd /Users/chunlinle/dev/dify/docker
  docker compose pull
  ```

- [ ] 验证镜像已下载
  ```bash
  docker images | grep dify
  # 应看到 dify-api, dify-web, dify-plugin-daemon 等
  ```

### 3. 启动服务

- [ ] 启动 Docker 服务
  ```bash
  docker-compose --profile postgresql up -d
  ```

或使用自动化脚本：
```bash
./start-dify.sh
```

- [ ] 检查服务状态
  ```bash
  docker-compose ps
  # 所有服务应显示 "Up" 状态
  ```

### 4. 重载 Nginx

- [ ] 重载 Nginx 配置
  ```bash
  sudo systemctl reload nginx
  ```

- [ ] 验证 Nginx 状态
  ```bash
  sudo systemctl status nginx
  # 应显示: active (running)
  ```

---

## ✅ 部署验证

### 本地服务测试

- [ ] API 健康检查
  ```bash
  curl http://localhost:5001/health
  # 应返回 JSON 响应
  ```

- [ ] Web 服务测试
  ```bash
  curl http://localhost:3000
  # 应返回 HTML 响应
  ```

- [ ] Plugin Daemon 测试
  ```bash
  curl http://localhost:5002/health
  # 应返回响应（如果有 health endpoint）
  ```

### HTTPS 访问测试

- [ ] 主站访问
  ```bash
  curl -I https://dify.agentsben.com
  # 应返回 200 OK
  ```

- [ ] API 访问
  ```bash
  curl -I https://api.dify.agentsben.com
  # 应返回 200 OK
  ```

- [ ] App 访问
  ```bash
  curl -I https://app.dify.agentsben.com
  # 应返回 200 OK
  ```

- [ ] Files 访问
  ```bash
  curl -I https://files.dify.agentsben.com
  # 应返回 200 OK
  ```

### 外部服务连接测试

- [ ] Redis 连接测试
  ```bash
  docker-compose exec api python -c "
  import redis
  r = redis.Redis(
      host='host.docker.internal',
      port=6379,
      password='sd@234#\$dfSDwe23',
      db=0
  )
  print('Redis PING:', r.ping())
  "
  # 应显示: Redis PING: True
  ```

- [ ] Qdrant 连接测试
  ```bash
  docker-compose exec api python -c "
  import requests
  r = requests.get('http://host.docker.internal:6333/')
  print('Qdrant Status:', r.status_code)
  "
  # 应显示: Qdrant Status: 200
  ```

### 数据库测试

- [ ] PostgreSQL 连接测试
  ```bash
  docker-compose exec api python -c "
  from extensions.ext_database import db
  try:
      db.session.execute('SELECT 1')
      print('Database connection OK')
  except Exception as e:
      print(f'Database connection failed: {e}')
  "
  # 应显示: Database connection OK
  ```

---

## 📊 服务监控检查

### 端口监听检查

- [ ] 验证所有端口正在监听
  ```bash
  sudo ss -tlnp | grep -E ':(80|443|3000|5001|5002|5432|6379|6333)'
  ```

应看到：
- `80` - Nginx HTTP
- `443` - Nginx HTTPS
- `127.0.0.1:3000` - Dify Web
- `127.0.0.1:5001` - Dify API
- `127.0.0.1:5002` - Plugin Daemon
- `127.0.0.1:5432` - PostgreSQL (Docker)
- `0.0.0.0:6379` 或 `127.0.0.1:6379` - Redis
- `0.0.0.0:6333` 或 `127.0.0.1:6333` - Qdrant

### 日志检查

- [ ] API 日志无严重错误
  ```bash
  docker-compose logs --tail=50 api | grep -i error
  # 应该没有重复的错误
  ```

- [ ] Worker 日志正常
  ```bash
  docker-compose logs --tail=50 worker | grep -i "started\|ready"
  ```

- [ ] Web 日志正常
  ```bash
  docker-compose logs --tail=50 web | grep -i "started\|ready"
  ```

- [ ] Nginx 日志无错误
  ```bash
  sudo tail -50 /var/log/nginx/error.log
  # 应该没有 502/504 错误
  ```

### 资源使用检查

- [ ] Docker 容器资源使用
  ```bash
  docker stats --no-stream
  ```

- [ ] 磁盘空间检查
  ```bash
  df -h /data/dify
  # 应有足够的可用空间
  ```

- [ ] 内存使用检查
  ```bash
  free -h
  # 应有足够的可用内存
  ```

---

## 🔒 安全检查

### 防火墙配置

- [ ] 开放必要端口
  ```bash
  sudo ufw status
  # 应看到:
  # 80/tcp ALLOW
  # 443/tcp ALLOW
  ```

- [ ] Docker 网络访问配置（如需要）
  ```bash
  sudo ufw allow from 172.17.0.0/16 to any port 6379
  sudo ufw allow from 172.17.0.0/16 to any port 6333
  ```

### 敏感信息检查

- [ ] .env 文件权限
  ```bash
  ls -la .env
  # 建议: -rw-r----- (640)
  chmod 640 .env
  ```

- [ ] 密码强度确认
  - [ ] DB_PASSWORD 是强密码
  - [ ] REDIS_PASSWORD 是强密码
  - [ ] SECRET_KEY 是随机生成的
  - [ ] API Keys 是随机生成的

### SSL/TLS 配置

- [ ] SSL 证书有效
  ```bash
  sudo certbot certificates
  # 检查过期时间
  ```

- [ ] TLS 版本配置正确（仅 TLSv1.2 和 TLSv1.3）
  ```bash
  curl -I --tlsv1.2 https://dify.agentsben.com
  # 应成功
  
  curl -I --tlsv1.0 https://dify.agentsben.com
  # 应失败
  ```

---

## 📈 性能优化检查

### 应用配置

- [ ] Worker 数量适当
  ```bash
  grep "SERVER_WORKER_AMOUNT" .env
  grep "CELERY_WORKER_AMOUNT" .env
  # 应根据 CPU 核心数配置
  ```

### 数据库优化

- [ ] PostgreSQL 配置优化
  ```bash
  grep "POSTGRES_SHARED_BUFFERS" .env
  grep "POSTGRES_EFFECTIVE_CACHE_SIZE" .env
  # 应根据内存大小配置
  ```

### Redis 配置

- [ ] Redis 持久化配置（在 Redis 服务器上）
  ```bash
  redis-cli -a 'sd@234#$dfSDwe23' CONFIG GET save
  redis-cli -a 'sd@234#$dfSDwe23' CONFIG GET appendonly
  ```

---

## 🔄 备份配置检查

### 自动备份脚本

- [ ] 创建备份脚本
  ```bash
  cat > /data/dify/backup.sh << 'EOF'
  #!/bin/bash
  DATE=$(date +%Y%m%d_%H%M%S)
  BACKUP_DIR="/data/dify/backups"
  mkdir -p $BACKUP_DIR
  
  # 备份数据库
  docker-compose exec -T db_postgres pg_dump -U dify -d dify | gzip > $BACKUP_DIR/db_$DATE.sql.gz
  
  # 备份文件存储
  tar -czf $BACKUP_DIR/storage_$DATE.tar.gz /data/dify/storage
  
  # 删除 30 天前的备份
  find $BACKUP_DIR -name "*.gz" -mtime +30 -delete
  
  echo "Backup completed: $DATE"
  EOF
  
  chmod +x /data/dify/backup.sh
  ```

- [ ] 配置定时备份
  ```bash
  sudo crontab -e
  # 添加: 0 2 * * * /data/dify/backup.sh >> /var/log/dify-backup.log 2>&1
  ```

### 备份测试

- [ ] 手动运行备份测试
  ```bash
  /data/dify/backup.sh
  ```

- [ ] 验证备份文件
  ```bash
  ls -lh /data/dify/backups/
  ```

---

## 📱 监控配置检查

### 日志轮转

- [ ] 配置 Docker 日志轮转
  ```bash
  # 在 docker-compose.yaml 中为每个服务添加:
  # logging:
  #   driver: "json-file"
  #   options:
  #     max-size: "100m"
  #     max-file: "3"
  ```

### 健康检查

- [ ] 创建健康检查脚本
  ```bash
  cat > /data/dify/health-check.sh << 'EOF'
  #!/bin/bash
  
  # 检查 API
  if ! curl -sf http://localhost:5001/health > /dev/null; then
      echo "API health check failed"
      exit 1
  fi
  
  # 检查 Web
  if ! curl -sf http://localhost:3000 > /dev/null; then
      echo "Web health check failed"
      exit 1
  fi
  
  echo "All services healthy"
  EOF
  
  chmod +x /data/dify/health-check.sh
  ```

---

## ✨ 最后检查

### 功能测试

- [ ] 访问主站并登录
  - 打开 https://dify.agentsben.com
  - 注册/登录账号
  - 创建测试应用

- [ ] 测试基本功能
  - [ ] 创建应用
  - [ ] 上传文件
  - [ ] 测试对话
  - [ ] 查看日志

### 文档检查

- [ ] 已阅读所有部署文档
  - [ ] CONFIGURATION-SUMMARY.md
  - [ ] NGINX-SETUP.md
  - [ ] EXTERNAL-SERVICES.md
  - [ ] DEPLOYMENT.md

- [ ] 保存了所有配置文件
  - [ ] .env 文件备份
  - [ ] Nginx 配置备份
  - [ ] SSL 证书路径记录

### 团队交接

- [ ] 记录所有密码和密钥
- [ ] 文档化自定义配置
- [ ] 记录故障联系人
- [ ] 准备运维文档

---

## 🎉 部署完成确认

当所有检查项都打勾后，您的 Dify 部署就完成了！

**最后验证：**

```bash
# 1. 所有服务运行正常
docker-compose ps

# 2. 可以通过 HTTPS 访问
curl -I https://dify.agentsben.com

# 3. API 响应正常
curl https://api.dify.agentsben.com/health

# 4. 日志没有错误
docker-compose logs --tail=100 | grep -i error
```

---

## 📞 问题报告

如果有任何检查项失败，请参考：

1. [CONFIGURATION-SUMMARY.md](./CONFIGURATION-SUMMARY.md) - 快速故障排查
2. [EXTERNAL-SERVICES.md](./EXTERNAL-SERVICES.md) - 外部服务问题
3. [NGINX-SETUP.md](./NGINX-SETUP.md) - Nginx 相关问题
4. [DEPLOYMENT.md](./DEPLOYMENT.md) - 详细部署问题

---

**部署日期：** _________________

**部署人员：** _________________

**备注：** 
```



```

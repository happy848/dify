# 🌐 使用自己的 Nginx 配置 Dify 知识库

## 📋 前提条件

- ✅ 已安装 Nginx
- ✅ 已部署 Dify 知识库精简版（使用 `docker-compose.kb-only.yaml`）
- ✅ API 服务运行在 `localhost:5001`

## 🚀 快速配置（3 步）

### 第 1 步：选择配置文件

我们提供了两个配置文件：

#### 选项 A：完整配置（推荐生产环境）
- 文件：`nginx-config/dify-api.conf`
- 包含：HTTP + HTTPS、日志、性能优化
- 适用：生产环境，需要 SSL

#### 选项 B：简化配置（开发/内网）
- 文件：`nginx-config/dify-api-simple.conf`
- 包含：仅 HTTP，基础配置
- 适用：开发环境或内网部署

### 第 2 步：复制配置文件到 Nginx

```bash
# 方式 1：使用符号链接（推荐）
sudo ln -s /Users/chunlinle/dev/dify/docker/nginx-config/dify-api.conf \
  /etc/nginx/sites-available/dify-api

sudo ln -s /etc/nginx/sites-available/dify-api \
  /etc/nginx/sites-enabled/dify-api

# 方式 2：直接复制
sudo cp /Users/chunlinle/dev/dify/docker/nginx-config/dify-api.conf \
  /etc/nginx/sites-available/dify-api

sudo ln -s /etc/nginx/sites-available/dify-api \
  /etc/nginx/sites-enabled/dify-api

# 如果使用简化配置，替换文件名为 dify-api-simple.conf
```

### 第 3 步：测试并重载 Nginx

```bash
# 测试配置
sudo nginx -t

# 如果测试通过，重载 Nginx
sudo nginx -s reload

# 或重启 Nginx
sudo systemctl reload nginx
```

## ✅ 验证配置

```bash
# 测试 API 访问
curl http://api.dify.agentsben.com/health

# 或使用 HTTPS（如果配置了）
curl https://api.dify.agentsben.com/health

# 预期返回
# {"status": "ok"}
```

## 🔧 配置说明

### 重要配置项

#### 1. 文件上传大小限制

```nginx
client_max_body_size 100M;
```

根据你的需求调整，知识库文档上传建议至少 50M。

#### 2. 超时时间

```nginx
proxy_read_timeout 300s;
proxy_connect_timeout 300s;
proxy_send_timeout 300s;
```

知识库文档处理可能需要较长时间，建议至少 300 秒。

#### 3. 后端服务地址

```nginx
proxy_pass http://127.0.0.1:5001;
```

确保这个地址与你的 API 服务端口一致。

## 🔐 HTTPS 配置

### 如果使用完整配置，需要配置 SSL 证书：

#### 方式 1：已有证书

```bash
# 将证书复制到 Nginx SSL 目录
sudo mkdir -p /etc/nginx/ssl
sudo cp your-cert.crt /etc/nginx/ssl/dify.agentsben.com.crt
sudo cp your-key.key /etc/nginx/ssl/dify.agentsben.com.key

# 设置权限
sudo chmod 600 /etc/nginx/ssl/dify.agentsben.com.key
sudo chmod 644 /etc/nginx/ssl/dify.agentsben.com.crt
```

#### 方式 2：使用 Let's Encrypt

```bash
# 安装 certbot
sudo apt install certbot python3-certbot-nginx  # Ubuntu/Debian
# 或
sudo yum install certbot python3-certbot-nginx  # CentOS/RHEL

# 获取证书
sudo certbot --nginx -d api.dify.agentsben.com

# Certbot 会自动配置 Nginx 的 SSL 设置
```

### 如果只需要 HTTP（内网环境）

使用简化配置 `dify-api-simple.conf`，不需要配置 SSL。

## 📝 配置模板自定义

### 修改域名

打开配置文件，修改 `server_name`：

```nginx
server_name your-domain.com;
```

### 修改后端端口

如果你的 API 服务不是运行在 5001 端口：

```nginx
# 修改 upstream 配置
upstream dify_api {
    server localhost:YOUR_PORT;  # 改成你的端口
}
```

### 添加多个域名

```nginx
server_name api.example.com api.example.net;
```

## 🎯 不同部署场景的配置

### 场景 1：生产环境（推荐配置）

```bash
# 使用完整配置 + HTTPS
sudo ln -s /Users/chunlinle/dev/dify/docker/nginx-config/dify-api.conf \
  /etc/nginx/sites-enabled/dify-api

# 配置 SSL 证书
sudo certbot --nginx -d api.dify.agentsben.com

# 重载 Nginx
sudo nginx -s reload
```

### 场景 2：开发环境（简化配置）

```bash
# 使用简化配置
sudo ln -s /Users/chunlinle/dev/dify/docker/nginx-config/dify-api-simple.conf \
  /etc/nginx/sites-enabled/dify-api

# 重载 Nginx
sudo nginx -s reload
```

### 场景 3：内网部署（使用 IP）

修改配置文件：

```nginx
server {
    listen 80;
    server_name 192.168.1.100;  # 改成你的服务器 IP
    
    location / {
        proxy_pass http://127.0.0.1:5001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 场景 4：子路径部署

如果你想将 API 部署在子路径（如 `example.com/dify-api/`）：

```nginx
server {
    listen 80;
    server_name example.com;
    
    location /dify-api/ {
        rewrite ^/dify-api/(.*)$ /$1 break;
        proxy_pass http://127.0.0.1:5001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## 🔍 日志查看

```bash
# 访问日志
sudo tail -f /var/log/nginx/dify-api-access.log

# 错误日志
sudo tail -f /var/log/nginx/dify-api-error.log

# 实时监控所有请求
sudo tail -f /var/log/nginx/dify-api-access.log | grep -v "/health"
```

## 🐛 常见问题

### 1. 502 Bad Gateway

**原因**：Nginx 无法连接到后端 API 服务

**解决方法**：

```bash
# 检查 API 服务是否运行
docker-compose -f docker-compose.kb-only.yaml ps api

# 检查端口是否正确
netstat -tlnp | grep 5001

# 测试本地访问
curl http://localhost:5001/health

# 检查防火墙
sudo ufw status
```

### 2. 413 Request Entity Too Large

**原因**：上传文件超过 Nginx 限制

**解决方法**：

```nginx
# 增加 client_max_body_size
client_max_body_size 200M;  # 根据需求调整
```

### 3. 504 Gateway Timeout

**原因**：请求处理时间超过超时限制

**解决方法**：

```nginx
# 增加超时时间
proxy_read_timeout 600s;
proxy_connect_timeout 600s;
proxy_send_timeout 600s;
```

### 4. CORS 错误

**原因**：跨域请求被拒绝

**解决方法 1** - 在 Nginx 中添加 CORS 头：

```nginx
location / {
    # 添加 CORS 头
    add_header 'Access-Control-Allow-Origin' '*' always;
    add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
    add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type' always;
    
    # 处理 OPTIONS 请求
    if ($request_method = 'OPTIONS') {
        return 204;
    }
    
    proxy_pass http://127.0.0.1:5001;
}
```

**解决方法 2** - 在 Dify 配置中设置：

```bash
# 编辑 .env.kb-only
WEB_API_CORS_ALLOW_ORIGINS=*
CONSOLE_CORS_ALLOW_ORIGINS=*
```

## 📊 性能优化

### 启用缓存（静态资源）

```nginx
location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
    proxy_pass http://127.0.0.1:5001;
    expires 1M;
    add_header Cache-Control "public";
}
```

### 启用 Gzip 压缩

```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
gzip_min_length 1000;
```

### 连接池优化

```nginx
upstream dify_api {
    server localhost:5001;
    keepalive 32;  # 保持连接数
    keepalive_timeout 60s;
    keepalive_requests 100;
}
```

## 🔄 与 Docker Nginx 的对比

| 特性 | 自己的 Nginx | Docker 内置 Nginx |
|------|--------------|-------------------|
| 配置灵活性 | ✅ 更灵活 | 有限 |
| 资源占用 | ✅ 更少（共享） | 额外容器 |
| 其他服务集成 | ✅ 可以代理其他服务 | 仅 Dify |
| 管理方式 | 系统服务 | Docker 容器 |
| SSL 证书 | 统一管理 | 单独管理 |
| 日志管理 | 系统日志 | 容器日志 |

## ✅ 完整部署流程

```bash
# 1. 部署 Dify 知识库精简版
cd /Users/chunlinle/dev/dify/docker
./deploy-kb-only.sh

# 2. 配置 Nginx
sudo ln -s /Users/chunlinle/dev/dify/docker/nginx-config/dify-api.conf \
  /etc/nginx/sites-enabled/dify-api

# 3. 配置 SSL（可选）
sudo certbot --nginx -d api.dify.agentsben.com

# 4. 测试配置
sudo nginx -t

# 5. 重载 Nginx
sudo nginx -s reload

# 6. 验证
curl https://api.dify.agentsben.com/health
```

## 📞 获取帮助

如果遇到问题：

1. 检查 Nginx 日志：`/var/log/nginx/dify-api-error.log`
2. 检查 API 日志：`docker-compose -f docker-compose.kb-only.yaml logs api`
3. 测试本地访问：`curl http://localhost:5001/health`
4. 查看防火墙：`sudo ufw status`

---

**最后更新**: 2026-01-31  
**适用版本**: Dify 知识库精简版  
**配置文件**: `nginx-config/dify-api.conf` 或 `dify-api-simple.conf`

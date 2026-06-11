# Ubuntu 22.04 部署文档

本文档说明 `ai-check-images` 在 Ubuntu 22.04 上的几种推荐部署方案。

## 服务信息

- 默认监听地址：`0.0.0.0:8767`
- 健康检查接口：`GET /v1/api/check_images/hello`
- 配置目录：`config/`
- 日志目录：`logs/`
- 可执行文件名：`ai-check-images`

运行时配置文件：

```text
config/
  apikey.txt
  gemini.toml
  prompt.txt
```

启动后验证：

```bash
curl http://127.0.0.1:8767/v1/api/check_images/hello
```

期望返回：

```json
{"message":"service is ok!!!"}
```

## 方案一：二进制 + systemd（最推荐）

适合公司内网单机部署。优点是简单、性能好、资源占用低、支持开机自启和自动重启。

### 1. 安装基础环境

```bash
sudo apt update
sudo apt install -y build-essential pkg-config git curl
```

安装 Rust：

```bash
curl https://sh.rustup.rs -sSf | sh
source "$HOME/.cargo/env"
```

### 2. 编译项目

```bash
git clone <你的仓库地址>
cd ai-check-images
cargo build --release
```

编译后的文件：

```text
target/release/ai-check-images
```

### 3. 安装到 `/opt`

```bash
sudo mkdir -p /opt/ai-check-images
sudo cp target/release/ai-check-images /opt/ai-check-images/
sudo cp -r config /opt/ai-check-images/
sudo mkdir -p /opt/ai-check-images/logs
```

创建系统用户：

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin ai-check-images
sudo chown -R ai-check-images:ai-check-images /opt/ai-check-images
```

### 4. 创建 systemd 服务

```bash
sudo nano /etc/systemd/system/ai-check-images.service
```

写入：

```ini
[Unit]
Description=AI Check Images Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=ai-check-images
Group=ai-check-images
WorkingDirectory=/opt/ai-check-images
ExecStart=/opt/ai-check-images/ai-check-images
Environment=AI_CHECK_IMAGES_HOST=0.0.0.0
Environment=AI_CHECK_IMAGES_PORT=8767
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 5. 启动服务

```bash
sudo systemctl daemon-reload
sudo systemctl enable ai-check-images
sudo systemctl start ai-check-images
sudo systemctl status ai-check-images
```

### 6. 查看日志

systemd 日志：

```bash
journalctl -u ai-check-images -f
```

应用文件日志：

```bash
tail -f /opt/ai-check-images/logs/ai-check-images.log.*
```

### 7. 更新部署

```bash
cd ai-check-images
git pull
cargo build --release
sudo systemctl stop ai-check-images
sudo cp target/release/ai-check-images /opt/ai-check-images/
sudo cp -r config /opt/ai-check-images/
sudo chown -R ai-check-images:ai-check-images /opt/ai-check-images
sudo systemctl start ai-check-images
sudo systemctl status ai-check-images
```

## 方案二：Docker 部署

适合已有 Docker 环境，或者希望部署环境一致、方便迁移和回滚。

### 1. Dockerfile

在项目根目录创建 `Dockerfile`：

```dockerfile
FROM rust:1-bookworm AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
RUN cargo build --release --locked

FROM debian:12-slim
WORKDIR /app
COPY --from=builder /app/target/release/ai-check-images /app/ai-check-images
COPY config ./config
RUN useradd --system --no-create-home --shell /usr/sbin/nologin appuser \
    && mkdir -p /app/logs \
    && chown -R appuser:appuser /app
USER appuser
EXPOSE 8767
ENV AI_CHECK_IMAGES_HOST=0.0.0.0
ENV AI_CHECK_IMAGES_PORT=8767
CMD ["/app/ai-check-images"]
```

### 2. 构建镜像

```bash
docker build -t ai-check-images:latest .
```

### 3. 运行容器

```bash
mkdir -p logs

docker run -d \
  --name ai-check-images \
  --restart unless-stopped \
  -p 8767:8767 \
  -v "$(pwd)/config:/app/config" \
  -v "$(pwd)/logs:/app/logs" \
  ai-check-images:latest
```

### 4. 查看状态和日志

```bash
docker ps
docker logs -f ai-check-images
```

## 方案三：Docker Compose

适合长期容器部署。后续如果要加 Nginx、监控、日志采集，Compose 更好维护。

### 1. `docker-compose.yml`

```yaml
services:
  ai-check-images:
    build: .
    container_name: ai-check-images
    restart: unless-stopped
    ports:
      - "8767:8767"
    environment:
      AI_CHECK_IMAGES_HOST: "0.0.0.0"
      AI_CHECK_IMAGES_PORT: "8767"
    volumes:
      - ./config:/app/config
      - ./logs:/app/logs
```

### 2. 启动

```bash
mkdir -p logs
chmod 777 logs
docker compose up -d --build
```

### 3. 查看日志

```bash
docker compose logs -f
```

### 4. 停止

```bash
docker compose down
```

### 5. 测试
```powershell
curl.exe http://127.0.0.1:8767/v1/api/check_images/hello
```

```bash
curl http://127.0.0.1:8767/v1/api/check_images/hello
```
## 方案四：Nginx 反向代理 + systemd

适合需要域名、HTTPS、统一入口或后续接入网关的场景。

服务本身建议只监听本机：

```ini
Environment=AI_CHECK_IMAGES_HOST=127.0.0.1
Environment=AI_CHECK_IMAGES_PORT=8767
```

Nginx 配置示例：

```nginx
server {
    listen 80;
    server_name your-domain.com;

    client_max_body_size 50m;

    location / {
        proxy_pass http://127.0.0.1:8767;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

启用配置：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

测试：

```bash
curl http://your-domain.com/v1/api/check_images/hello
```

如果需要 HTTPS，建议使用 `certbot` 配置证书。

## 方案选择建议

| 方案 | 推荐场景 | 优点 | 注意事项 |
| --- | --- | --- | --- |
| 二进制 + systemd | 内网单机、稳定服务 | 简单、性能好、资源占用低 | 需要手动更新二进制 |
| Docker | 已有容器环境 | 环境一致、迁移方便 | 需要维护镜像和挂载 |
| Docker Compose | 长期容器部署、多服务组合 | 配置清晰、易扩展 | 需要 Docker Compose |
| Nginx + systemd | 公网、域名、HTTPS | 可接入 TLS、限流、统一入口 | 多维护一个 Nginx |

当前项目最推荐：

1. 内网部署：`二进制 + systemd`
2. 容器化部署：`Docker Compose`
3. 公网访问：`Nginx + systemd`

## 防火墙配置

如果服务需要被其他机器访问，开放端口：

```bash
sudo ufw allow 8767/tcp
sudo ufw status
```

如果使用 Nginx 对外暴露，则通常只开放 `80` 和 `443`：

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

## 故障排查

### 端口被占用

```bash
sudo ss -ltnp | grep 8767
```

### 服务启动失败

```bash
sudo systemctl status ai-check-images
journalctl -u ai-check-images -n 100 --no-pager
```

### 配置文件权限问题

```bash
sudo chown -R ai-check-images:ai-check-images /opt/ai-check-images
sudo chmod 600 /opt/ai-check-images/config/apikey.txt
sudo chmod 600 /opt/ai-check-images/config/gemini.toml
```

### 健康检查失败

```bash
curl -v http://127.0.0.1:8767/v1/api/check_images/hello
```

确认：

- 服务是否已启动
- `AI_CHECK_IMAGES_PORT` 是否为 `8767`
- `config/gemini.toml` 是否存在
- 当前工作目录是否包含 `config/`

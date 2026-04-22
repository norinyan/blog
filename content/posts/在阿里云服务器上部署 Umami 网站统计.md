---
title: 在阿里云服务器上部署 Umami 网站统计
date: 2026-04-01T01:07:37+08:00
draft: false
---

## 1. 进入云服务器++++++++
```bash
ssh nor
```

## 2. 检查 docker & docker-compose 版本
```bash
docker --version
docker-compose version
```

## 3. 生成签名加密字符串
```bash
openssl rand -hex 32
```

复制输出备用。

## 4. 创建 umami 目录并编写 docker-compose 文件
```bash
mkdir ~/umami && cd ~/umami
nano docker-compose.yml
```

粘贴以下内容，将 `APP_SECRET` 替换为第 3 步的输出：
```yaml
version: '3'
services:
  umami:
    image: ghcr.io/umami-software/umami:postgresql-latest
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://umami:umami@db:5432/umami
      APP_SECRET: 替换为第3步的输出
    depends_on:
      - db
    restart: always
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: umami
      POSTGRES_USER: umami
      POSTGRES_PASSWORD: umami
    volumes:
      - ./data:/var/lib/postgresql/data
    restart: always
```

保存（Ctrl+O，回车，Ctrl+X）。

## 5. 启动
```bash
docker-compose up -d
```

## 6. 确认状态
```bash
docker-compose ps
```

两个容器均为 `Up` 即正常。

## 7. Nginx 转发
```bash
cd /etc/nginx/sites-available
sudo nano umami.conf
```

粘贴以下内容，将 `your.site.com` 替换为你的域名：
```nginx
server {
    listen 80;
    server_name your.site.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

保存（Ctrl+O，回车，Ctrl+X）。

## 8. 启用配置，检查语法，重启 Nginx
```bash
sudo ln -s /etc/nginx/sites-available/umami.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## 9. 添加 DNS 解析

去域名托管平台添加一条 A 记录：

- 类型：A
- 名称：`umami`
- IPv4 地址：你的服务器 IP
- 代理状态：已代理（橙色）

## 10. 访问
```
http://umami.your.site.com
```

> ⚠️ 国内服务器需备案才能通过域名正常访问 80/443 端口。

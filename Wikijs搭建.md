---
title: Wikijs搭建
description: 使用Docker快速搭建Wikijs
published: true
date: 2026-01-25T16:24:33.130Z
tags: 
editor: markdown
dateCreated: 2026-01-25T16:22:58.950Z
---

# Wiki.js搭建说明
`docker-compose.yml`
```yaml
version: "3.9"

networks:
  wikinet:
    driver: bridge

volumes:
  pgdata:

services:
  db:
    image: postgres:17
    container_name: db
    hostname: db
    restart: unless-stopped
    networks:
      - wikinet
    environment:
      POSTGRES_DB: wiki
      POSTGRES_USER: wiki
      POSTGRES_PASSWORD_FILE: /etc/wiki/.db-secret
    volumes:
      - /etc/wiki/.db-secret:/etc/wiki/.db-secret:ro
      - /etc/wiki/pgdata:/var/lib/postgresql/data

  wiki:
    image: ghcr.io/requarks/wiki:2
    container_name: wiki
    hostname: wiki
    restart: unless-stopped
    networks:
      - wikinet
    depends_on:
      - db
    ports:
      - "80:3000"
      - "443:3443"
    environment:
      DB_TYPE: postgres
      DB_HOST: db
      DB_PORT: 5432
      DB_USER: wiki
      DB_NAME: wiki
      DB_PASS_FILE: /etc/wiki/.db-secret
      UPGRADE_COMPANION: "1"
    volumes:
      - /etc/wiki/.db-secret:/etc/wiki/.db-secret:ro
      - /etc/wiki/keys:/etc/wiki/keys:ro
      - /etc/wiki/ssh/known_hosts:/etc/ssh/ssh_known_hosts:ro

  wiki-update-companion:
    image: ghcr.io/requarks/wiki-update-companion:latest
    container_name: wiki-update-companion
    hostname: wiki-update-companion
    restart: unless-stopped
    networks:
      - wikinet
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
```
# 1. 创建目录
```bash
sudo mkdir -p /etc/wiki
sudo mkdir -p /etc/wiki/keys
```

# 2. 生成数据库密码
```bash
openssl rand -base64 32 | sudo tee /etc/wiki/.db-secret > /dev/null
sudo chmod 644 /etc/wiki/.db-secret
```
# 3. 启动
```bash
docker compose up -d
```

# 4. 进行Git同步

1. 为Git仓库生成一个专用的无密码key

```bash
sudo ssh-keygen -t ed25519 -N "" -f /etc/wiki/keys/wikijs_git
sudo chmod 600 /etc/wiki/keys/wikijs_git
```
2. 为Git仓库添加key
GitHub：Repo → Settings → Deploy keys → Add deploy key（勾选 Allow write access）
```bash
sudo cat /etc/wiki/keys/wikijs_git.pub
```

3. 生成 known_hosts（避免 Host key verification failed）
```bash 
sudo mkdir -p /etc/wiki/ssh
sudo ssh-keyscan -H github.com | sudo tee /etc/wiki/ssh/known_hosts > /dev/null
sudo chmod 644 /etc/wiki/ssh/known_hosts
```
4. Wiki.js → Git 同步 → Authentication → SSH Private Key Path

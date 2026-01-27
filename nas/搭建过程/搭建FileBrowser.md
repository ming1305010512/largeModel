[TOC]

chuying:chuying_1234

lm:longming_123

admin:longming_1305010512

## 1. 写docker-compose.yml

```
version: "3.8"

services:
  filebrowser-chuyin:
    image: filebrowser/filebrowser:latest
    container_name: filebrowser-chuyin
    restart: unless-stopped

    user: "1000:1000"   # lm 的 uid:gid（后面我教你确认）
    volumes:
      - /data/vault/knowledge/chuyin:/srv
      - /data/apps/filebrowser/config:/config
      - /data/apps/filebrowser/db:/db

    environment:
      - FB_DATABASE=/db/chuyin.db
      - FB_CONFIG=/config/settings.json
      - FB_ROOT=/srv

    ports:
      - "8081:80"   # 只监听本机

```

## 2. 编写配置文件config/settings.json

```
{
  "port": 80,
  "address": "0.0.0.0",
  "log": "stdout",
  "database": "/db/chuyin.db",
  "root": "/srv"
}

```

## 3. 执行命令

```
## 启动了才执行
docker compose down
docker compose up -d
docker compose logs -n 30
curl -I http://127.0.0.1:8081/
```

## 4. 如果忘记看密码可以改密码

停掉服务docker compose down执行一下命令改密码

```
docker run --rm -it \
  -v /data/apps/filebrowser/db:/db \
  -v /data/apps/filebrowser/config:/config \
  filebrowser/filebrowser:latest \
  users update admin --password "longming_1305010512" --database /db/chuyin.db
```

## 5. 创建共享目录入口

```
# 在每个人目录里放一个 shared 的“入口”
sudo ln -sfn ../_shared /data/vault/knowledge/lm/_shared
sudo ln -sfn ../_shared /data/vault/knowledge/chuyin/_shared
```

### 1）`sudo`

用 root 权限执行。
 原因通常是：`/data/vault/...` 这些目录权限较严，普通用户可能没权限创建/覆盖链接。

我这个不需要

### 2）`ln`

创建“链接”。Linux 里有硬链接、软链接（符号链接），这里用的是软链接。

### 3）`-s`

创建**符号链接（软链接）**。
 它相当于 Windows 的“快捷方式”，指向另一个路径。

### 4）`-f`

**强制覆盖**：如果目标位置已经有同名文件/链接 `_shared`，就直接替换掉。

### 5）`-n`

配合 `-f` 用的：当目标 `_shared` 已经是一个 symlink 时，把它当作普通文件处理并替换，避免一些情况下“跟着链接进去覆盖里面内容”的风险。

> `-sfn` 这三个通常就被当作“安全覆盖符号链接”的固定组合。

### 6）`../_shared`

这是**链接指向的目标（被指向的真实位置）**，是一个相对路径。
 意思是：从 `/data/vault/knowledge/lm/` 这个目录往上一级到 `/data/vault/knowledge/`，再到 `/_shared`。

也就是最终指向：

- `/data/vault/knowledge/_shared`

### 7）`/data/vault/knowledge/lm/_shared`

这是**链接本身放在哪里**。
 也就是在 `lm` 的目录里创建一个叫 `_shared` 的入口。

# 二、第一步：安装 OnlyOffice 插件（很简单）

------

### 手动安装插件（万一商店不稳定）

```
docker exec -it nextcloud bash
cd /var/www/html/apps
```

下载插件（示例）：

```
wget https://github.com/ONLYOFFICE/onlyoffice-nextcloud/releases/latest/download/onlyoffice.tar.gz
tar -xzf onlyoffice.tar.gz
chown -R www-data:www-data onlyoffice
exit
```

然后：

- Nextcloud → 应用 → 已禁用的应用 → 启用 ONLYOFFICE

------

# 三、第二步：部署 OnlyOffice Document Server（关键）

这是**真正的编辑器服务**。

------

## 1️⃣ 在宿主机创建目录

```
mkdir -p /data/apps/onlyoffice
cd /data/apps/onlyoffice
```

------

## 2️⃣ 创建 docker-compose.yml（照抄）

```
version: "3.8"

services:
  onlyoffice:
    image: onlyoffice/documentserver:latest
    container_name: onlyoffice
    restart: always
    ports:
      - "8083:80"
    environment:
      JWT_ENABLED: "true"
      JWT_SECRET: "onlyoffice-secret-123"
    volumes:
      - ./data:/var/www/onlyoffice/Data
      - ./logs:/var/log/onlyoffice
```

📌 解释一句（不用记）：

- `8083`：OnlyOffice 访问端口
- `JWT_SECRET`：**等会要和 Nextcloud 对上**

------

## 3️⃣ 启动 Document Server

```
docker compose up -d
```

等 30～60 秒（第一次启动慢）

------

## 4️⃣ 验证是否成功

浏览器打开：

```
http://你的NASIP:8083
```

✅ 看到 OnlyOffice 欢迎页

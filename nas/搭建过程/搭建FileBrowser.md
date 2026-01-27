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


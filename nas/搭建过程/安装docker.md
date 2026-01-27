## 第一步：安装 Docker（官方推荐方式）

在 NAS 上执行（用 `lm`，带 sudo）：

```
sudo apt update
sudo apt install -y ca-certificates curl gnupg
```

### 添加 Docker 官方 GPG key

```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### 添加 Docker 官方仓库

```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 安装 Docker 本体 + compose 插件

```
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

------

## 第二步：把 Docker 的数据目录放到 /data/apps（非常关键）

默认 Docker 会把所有东西放在：

```
/var/lib/docker   ❌（系统盘）
```

我们要改成：

```
/data/apps/docker ✅
```

### 1️⃣ 停止 Docker

```
sudo systemctl stop docker
```

### 2️⃣ 创建数据目录

```
sudo mkdir -p /data/apps/docker
sudo chown -R root:root /data/apps/docker
```

### 3️⃣ 配置 Docker daemon

```
sudo nano /etc/docker/daemon.json
```

写入（如果原来有内容，整体替换）：

```
{
  "data-root": "/data/apps/docker"
}
```

保存退出。

------

### 4️⃣ 启动 Docker

```
sudo systemctl start docker
sudo systemctl enable docker
```

------

## 第三步：让 lm 不用 sudo 也能用 docker（强烈推荐）

```
sudo usermod -aG docker lm
```

⚠️ **这一步之后你需要重新登录一次（SSH 退出再进）**

------

## 第四步：验证（这一步会让你很安心）

重新登录后执行：

```
docker version
docker compose version
docker ps
```

再跑一个测试容器：

```
docker run --rm hello-world
```

如果看到：

```
Hello from Docker!
```

👉 **Docker 已经完美就位**



如果镜像下载太慢：

### 1️⃣ 编辑 Docker daemon 配置

```
sudo nano /etc/docker/daemon.json
```

把内容改成 **下面这样（完整覆盖）**：

```
{
  "data-root": "/data/apps/docker",
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://mirror.baidubce.com",
    "https://registry.cn-hangzhou.aliyuncs.com"
  ]
}
```

### 2️⃣ 重启 Docker

```
sudo systemctl daemon-reexec
sudo systemctl restart docker
```

等 3–5 秒。

------

### 3️⃣ 再试一次 hello-world（关键）

```
docker run --rm hello-world
```
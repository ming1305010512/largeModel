好，**Ubuntu 22.04（Jammy）安装 Docker** 我给你一套**官方标准 + 适配新版本 apt 的完整流程**。
你**一行一行照着执行**，不会再遇到刚才那种“没有 installation candidate”的问题。

------

# ✅ Ubuntu 22.04 安装 Docker（推荐 & 正确姿势）

> 适用系统：**Ubuntu 22.04 LTS**

------

## 一、卸载可能存在的旧 Docker（重要，避免冲突）

```bash
sudo apt remove -y docker docker-engine docker.io containerd runc
```

------

## 二、更新系统 & 安装依赖

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
```

------

## 三、添加 Docker 官方 GPG Key（22.04 新写法，关键）

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

设置权限（很重要）：

```bash
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

------

## 四、添加 Docker 官方仓库（适配 22.04）

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

------

## 五、刷新 apt 并安装 Docker

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

> ✅ Ubuntu 22.04 **官方推荐装 compose-plugin**（不用再装 docker-compose v1）

------

## 六、启动 Docker 并设置开机自启

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

------

## 七、验证是否安装成功（必做）

```bash
docker version
docker run hello-world
```

看到：

```
Hello from Docker!
```

说明 **Docker 已 100% 安装成功**

------

## 八、（强烈推荐）配置国内镜像加速

不然后面拉镜像会很慢。

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json >/dev/null <<'EOF'
{
    "registry-mirrors": [
    "https://docker.unsee.tech",
    "https://dockerpull.org",
    "https://docker.1panel.live",
    "https://dockerhub.icu",
    "https://docker.m.daocloud.io",
    "https://docker.nju.edu.cn",
    "https://registry.docker-cn.com",
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com",
    "https://5tqw56kt.mirror.aliyuncs.com",
    "https://docker.hpcloud.cloud",
    "http://mirrors.ustc.edu.cn",
    "https://docker.chenby.cn",
    "https://docker.ckyl.me",
    "http://mirror.azure.cn",
    "https://hub.rat.dev"]
}
EOF
```

重启 Docker：

```bash
sudo systemctl daemon-reexec
sudo systemctl restart docker
```

验证：

```bash
docker info | grep -i mirror
```

------

## 九、（可选）普通用户免 sudo（强烈推荐）

```bash
sudo usermod -aG docker $USER
```

⚠️ 然后 **退出 SSH，重新登录**

验证：

```bash
docker ps
```

------

# 🚑 如果你再遇到报错（快速自查）

### ❌ `Cannot connect to the Docker daemon`

```bash
sudo systemctl start docker
```

### ❌ `docker-ce has no installation candidate`

👉 一定是 **第 3 / 4 步没成功**
👉 用的是旧的 `apt-key` 或源写法

------

## ✅ 最稳的“确认三连”

你现在只需要跑这三条：

```bash
systemctl status docker --no-pager
docker version
docker run hello-world
```

------

如果你愿意，下一步我可以直接帮你：

- ✅ 用 Docker 跑 **nginx**
- ✅ 用 Docker 跑 **frp**
- ✅ 用 **docker compose** 一键部署项目（22.04 最佳实践）

你直接说一句：**“下一步做什么”** 就行。
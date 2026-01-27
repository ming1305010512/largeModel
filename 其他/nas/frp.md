## 一、客户端下载地址

https://github.com/fatedier/frp/releases/tag/v0.56.0

## 三、Step 2.1：在 ECS 上安装 frps（服务端）

### 1️⃣ 登录 ECS

```
ssh root@你的ECS公网IP
```

------

### 2️⃣ 下载 frp（官方版本）

去官方仓库，frp 是**单文件、非常干净**：

```
cd /opt
wget https://github.com/fatedier/frp/releases/download/v0.56.0/frp_0.56.0_linux_amd64.tar.gz
```

> 如果 ECS 是 ARM（少见），我再给你换包

解压：

```
tar -zxvf frp_0.56.0_linux_amd64.tar.gz
cd frp_0.56.0_linux_amd64
```

------

### 3️⃣ 配置 frps

创建配置文件：

```
vim frps.ini
```

内容（**最简可用版**）：

```
[common]
bind_port = 7000
```

📌 说明：

- `7000` 是 frp 控制通道端口
- **等下 frpc 会连这个端口**

------

### 4️⃣ 启动 frps（先前台跑，方便看日志）

```
./frps -c frps.ini
```

看到类似输出：

```
frps started successfully
```

👉 **frps 服务端已经跑起来了**

⚠️ 现在不要关这个窗口（或者你可以开新窗口）

------

## 四、⚠️ 非常重要：ECS 放行端口

你必须确认：

### ECS 安全组 / 防火墙 放行：

| 用途     | 端口                     |
| -------- | ------------------------ |
| frp 控制 | 7000                     |
| 映射端口 | 等下用的端口（如 6000+） |

如果你用的是：

- 阿里云 / 腾讯云：安全组里放行
- 服务器防火墙（ufw）：

```
ufw allow 7000
ufw allow 6006
```

（6006 是示例，等下会用）

------

## 五、Step 2.2：在本地电脑安装 frpc（客户端）

### 1️⃣ 下载 frp（和服务端同版本）

#### Windows

下载：

```
frp_0.56.0_windows_amd64.zip
```

解压后你会看到：

```
frpc.exe
frpc.ini
```

#### macOS / Linux

同理下载对应版本即可。

------

### 2️⃣ 配置 frpc.ini（关键）

编辑 `frpc.ini`：

```
[common]
server_addr = 你的ECS公网IP
server_port = 7000

[test_web]
type = tcp
local_ip = 127.0.0.1
local_port = 8080
remote_port = 6006
```

📌 解释（一定要看）：

- `local_port = 8080`
  - 👉 你本地要转发的服务端口
- `remote_port = 6006`
  - 👉 外网访问 ECS 用的端口

------

## 六、Step 2.3：本地先跑一个测试服务（很重要）

### 不要一上来就 Jellyfin

先用最简单的测试服务。

#### 方式一：Python（推荐）

```
python3 -m http.server 8080
```

终端显示：

```
Serving HTTP on 0.0.0.0 port 8080
```

------

## 七、Step 2.4：启动 frpc（客户端）

在本地：

```
./frpc -c frpc.ini
```

你应该看到：

```
login to server success
```

👉 说明 frpc 已经成功连上 ECS 的 frps。

------

## 八、Step 2.5：外网验证（关键时刻）

现在，用 **手机流量 / 另一台电脑** 打开浏览器：

```
http://你的ECS公网IP:6006
```

如果你看到：

👉 Python 的目录列表页面
 👉 或一个简单网页

🎉 **恭喜，这一步已经 100% 成功**
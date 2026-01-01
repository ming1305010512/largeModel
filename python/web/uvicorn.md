[TOC]

Uvicorn 是一个 **ASGI Web Server**（不是框架），你可以把它理解成“Python 世界里跑 FastAPI/Starlette 这类异步 Web 应用的**高性能服务器程序**”。它负责：监听端口、收 HTTP/WebSocket 连接、把请求按 ASGI 协议交给你的 `app` 处理、再把响应发回去。([uvicorn.dev](https://uvicorn.dev/deployment/))

------

## 1）Uvicorn 和 FastAPI/Starlette 的关系

- **FastAPI/Starlette**：写业务逻辑（路由、依赖注入、中间件、返回 JSON…）
- **Uvicorn**：把你的应用跑起来（网络 I/O、并发、协议解析、进程管理的一部分）

所以你写完：

```py
app = FastAPI()
```

还需要用 Uvicorn 来运行：

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

这里的 `main:app` 就是“模块:对象名”，等价于 `from main import app`。([fastapi.tiangolo.com](https://fastapi.tiangolo.com/deployment/manually/))

------

## 2）ASGI 是什么（Uvicorn为什么“快”）

ASGI（Asynchronous Server Gateway Interface）是一套 Python 服务器与应用之间的协议，支持：

- **异步请求处理（async/await）**
- **长连接**（WebSocket）
- **应用生命周期事件**（startup/shutdown，即 lifespan）

Uvicorn 作为 ASGI server，会把每个请求变成一个 ASGI scope + receive/send 调用，交给你的 app（FastAPI）去处理。

------

## 3）最常用的启动方式：命令行 & 关键参数

Uvicorn 官方文档里列了一堆 CLI 选项，最常用的是这些（也是你日常最容易踩坑的）：

### 3.1 监听地址：`--host` / `--port`

- `--host 127.0.0.1`：只允许本机访问（默认）
- `--host 0.0.0.0`：对外网卡开放（容器/服务器上常用）
- `--port 8000`：端口（默认 8000）

（注意：浏览器里访问要用 `localhost:8000` 或服务器 IP，不要用 `0.0.0.0:8000`。）

### 3.2 开发热重载：`--reload`

开发时常用：

```bash
uvicorn main:app --reload --port 5000
```

它会监控文件变化并重启进程，但**更耗资源、更不稳定，不建议生产用**。([fastapi.tiangolo.com](https://fastapi.tiangolo.com/deployment/manually/))

并且：**`--reload` 和 `--workers` 互斥**（不能同时用）。([uvicorn.dev](https://uvicorn.dev/deployment/))

### 3.3 多进程并发：`--workers`

想利用多核 CPU，可以：

```bash
uvicorn main:app --host 0.0.0.0 --port 8080 --workers 4
```

它会起一个父进程作为管理者，再 fork 多个 worker 进程并行处理请求。([fastapi.tiangolo.com](https://fastapi.tiangolo.com/deployment/server-workers/))

> 重要现实：Kubernetes 这类容器编排环境里，FastAPI 文档也提到通常更倾向于“每个容器只跑一个 Uvicorn 进程”，扩缩容交给 K8s 做。([fastapi.tiangolo.com](https://fastapi.tiangolo.com/deployment/server-workers/))

------

## 4）`uvicorn[standard]` 是啥？为什么经常建议装它

FastAPI 文档建议安装：

```bash
pip install "uvicorn[standard]"
```

因为 `standard` 会带上一些推荐依赖（例如 **uvloop** 等）来提升并发性能。([fastapi.tiangolo.com](https://fastapi.tiangolo.com/deployment/manually/))

------

## 5）协议/事件循环相关参数：你什么时候会用到

Uvicorn 允许你选择实现（一般保持默认 `auto` 就行）：

- `--loop [auto|asyncio|uvloop]`：事件循环实现
- `--http [auto|h11|httptools]`：HTTP 解析实现
- `--ws [auto|websockets|websockets-sansio|wsproto]`：WebSocket 实现
  这些选项都在官方 help 里列出。([uvicorn.dev](https://uvicorn.dev/deployment/))

经验上：

- 你遇到兼容性问题（比如某平台不支持 uvloop）才会强制 `--loop asyncio`
- 你要在 PyPy/极简环境里跑，可能会倾向纯 Python 的实现（性能略低但兼容更好）([uvicorn.dev](https://uvicorn.dev/deployment/))

------

## 6）生产部署：Uvicorn vs Gunicorn（以及一个“新变化”）

Uvicorn 文档给的通用建议是：

- 本地开发：`uvicorn --reload`
- 生产：用 **Gunicorn 管理进程** + Uvicorn worker（更强的进程管理、平滑重启等）([uvicorn.dev](https://uvicorn.dev/deployment/))

但要注意：Uvicorn 文档同时写了一个重要提醒——`uvicorn.workers` **已被标记为 deprecated**，未来会移除，建议改用 `uvicorn-worker` 包。([uvicorn.dev](https://uvicorn.dev/deployment/))

------

## 7）反向代理与真实客户端 IP：`--proxy-headers` / `--forwarded-allow-ips`

如果你在 Nginx/Ingress/负载均衡后面，真实客户端 IP、协议（http/https）可能会被代理“遮住”。
Uvicorn 支持读取：

- `X-Forwarded-For`
- `X-Forwarded-Proto`
  并强调：要正确配置“信任哪些代理来源”，因为这些头可以伪造。([uvicorn.dev](https://uvicorn.dev/deployment/))

------

## 8）你很可能会用到的“常见组合命令”

- **开发**（自动重载）

  ```bash
  uvicorn main:app --reload --host 127.0.0.1 --port 8000
  ```

- **单机多核（不在 K8s）**

  ```bash
  uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
  ```

- **生产更稳的建议方向**：Uvicorn 文档建议用进程管理器（比如 Gunicorn）来跑生产。([uvicorn.dev](https://uvicorn.dev/deployment/))


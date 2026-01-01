[TOC]

FastAPI 是一个用 Python 写 Web API 的框架，主打三件事：

1. **快**（基于 Starlette 的 ASGI，支持高并发/异步）
2. **开发爽**（用 Python 类型注解写参数校验、自动生成 OpenAPI 文档）
3. **规范**（输入输出模型清晰、可测试、可扩展）

下面按“你做项目时最常碰到的点”把 FastAPI 讲透。

------

## 1）FastAPI 的核心组成（你需要知道它靠谁实现的）

- **FastAPI**：把路由、依赖注入、参数解析、OpenAPI 文档这些“开发体验”做得很好。
- **Starlette**：底层 Web 框架（ASGI、路由、中间件、WebSocket、Streaming、BackgroundTasks 等）。
- **Pydantic**：数据校验与序列化（请求体/响应体模型、类型校验、字段约束、错误信息）。

一句话：FastAPI = “Starlette 的高性能 + Pydantic 的强校验 + 类型注解驱动的自动文档”。

------

## 2）最小可运行例子（路由 + 自动文档）

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/ping")
def ping():
    return {"msg": "pong"}
```

启动：

```bash
uvicorn main:app --reload
```

打开：

- `http://127.0.0.1:8000/docs`（Swagger UI）
- `http://127.0.0.1:8000/redoc`（ReDoc）

------

## 3）参数解析规则（FastAPI 最舒服的地方）

FastAPI 会根据函数签名自动判断参数来自哪里：

### 3.1 路径参数 Path

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/users/{user_id}")
def get_user(user_id: int):
    return {"user_id": user_id}
```

`user_id` 会自动转 int，转不了就返回 422（参数校验失败）。

### 3.2 查询参数 Query

```python
from typing import Optional
from fastapi import FastAPI

app = FastAPI()

@app.get("/items")
def list_items(q: Optional[str] = None, limit: int = 10):
    return {"q": q, "limit": limit}
```

访问 `/items?q=abc&limit=20`

### 3.3 请求体 Body（Pydantic Model）

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class ItemIn(BaseModel):
    name: str
    price: float
    tags: list[str] = []

@app.post("/items")
def create_item(item: ItemIn):
    return {"ok": True, "item": item.model_dump()}
```

> 你会发现：请求体的校验、报错信息、文档示例都是自动生成的。

------

## 4）响应模型（把输出也“规范化”）

你可以声明响应结构，FastAPI 会：

- 自动过滤多余字段
- 自动做类型转换/序列化
- 文档也会显示清楚

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class UserOut(BaseModel):
    id: int
    name: str

@app.get("/me", response_model=UserOut)
def me():
    return {"id": "1", "name": "Long Ming", "extra": "will be dropped"}
```

------

## 5）依赖注入 Depends（FastAPI 的“工程能力核心”）

依赖注入可以把“通用逻辑”抽出来：鉴权、DB Session、分页参数、公共校验等。

```python
from fastapi import FastAPI, Depends

app = FastAPI()

def get_current_user():
    # 真实项目里：解析 token -> 查用户
    return {"id": 1, "name": "admin"}

@app.get("/secure")
def secure_api(user=Depends(get_current_user)):
    return {"hello": user["name"]}
```

依赖还能“层层依赖”、还能在 `yield` 后做清理（非常适合 DB session）：

```python
from fastapi import Depends

def get_db():
    db = connect()
    try:
        yield db
    finally:
        db.close()
```

------

## 6）异步 async（什么时候用、什么时候别用）

FastAPI 支持同步和异步两种写法：

- `def`：同步视图（适合大多数“CPU 逻辑”或你使用的是同步库）
- `async def`：异步视图（适合 I/O 多：HTTP 调用、异步 DB、Redis、文件等）

关键原则：

- **如果你在 async 里调用同步阻塞函数（如 requests、同步 DB driver）**，会阻塞事件循环，反而降低并发。
- 走 async 要配套 async 生态：`httpx`（异步 HTTP）、`asyncpg`（PostgreSQL）、SQLAlchemy async 等。

------

## 7）中间件 Middleware（统一做日志、跨域、压缩、链路追踪）

例如 CORS：

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

------

## 8）异常处理与返回码

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

@app.get("/items/{item_id}")
def get_item(item_id: int):
    if item_id == 0:
        raise HTTPException(status_code=404, detail="Not found")
    return {"id": item_id}
```

你也可以自定义异常 handler，统一错误结构（工程里很常见）。

------

## 9）安全与鉴权（常见做法）

FastAPI 内置对 OpenAPI 的安全方案支持（Bearer/JWT、OAuth2、API Key），通常做法是：

- 定义一个依赖 `get_current_user`
- 从 `Authorization: Bearer <token>` 解析 token
- 校验并注入 user

------

## 10）测试（FastAPI 特别适合写单测）

```python
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_ping():
    r = client.get("/ping")
    assert r.status_code == 200
```

------

## 11）部署

常见组合：

- `uvicorn`：ASGI server（开发/轻量部署）
- `gunicorn + uvicorn workers`：生产常见（多进程）
- 反向代理：Nginx / Traefik


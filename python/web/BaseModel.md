[TOC]

`BaseModel` 是 Pydantic 里最核心的“数据模型基类”：你继承它、用**类型注解**声明字段，Pydantic 就会在创建/解析时帮你做 **校验（validation）+ 类型转换（parsing）+ 序列化（serialization）**，并且还能生成 JSON Schema（FastAPI 用它生成 OpenAPI 文档）。([Pydantic](https://docs.pydantic.dev/latest/api/base_model/?utm_source=chatgpt.com))

------

## 1) 基本用法：声明字段 = 注解属性

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int
    name: str
    age: int | None = None

u = User(id="1", name="Long Ming")  # id 会被转成 int
```

Pydantic 会按注解类型去验证/转换；不符合会抛 `ValidationError`（通常在 FastAPI 里会变成 422）。([Pydantic](https://docs.pydantic.dev/latest/concepts/models/?utm_source=chatgpt.com))

------

## 2) 常用“入口”和“出口”方法（Pydantic v2 命名）

### 2.1 入口：把外部数据变成模型

- `Model.model_validate(obj)`：校验并解析 Python 对象（dict/ORM 等）([Pydantic](https://docs.pydantic.dev/latest/concepts/models/?utm_source=chatgpt.com))
- `Model.model_validate_json(json_str)`：从 JSON 字符串校验解析([Pydantic](https://docs.pydantic.dev/latest/concepts/models/?utm_source=chatgpt.com))
- `Model.model_construct(...)`：**不校验**直接构造（很快但危险，适合你确信数据已干净时）([Pydantic](https://docs.pydantic.dev/latest/concepts/models/?utm_source=chatgpt.com))

### 2.2 出口：把模型变成 dict/JSON（用于返回/存库）

- `model.model_dump()`：转 dict（可配置是否用别名、排除 None、排除默认值等）([Pydantic](https://docs.pydantic.dev/latest/concepts/models/?utm_source=chatgpt.com))
  （v1 里类似的是 `.dict()` / `.json()`，v2 统一成 `model_dump()`）

------

## 3) Field：给字段加约束、默认值、描述、别名

```python
from pydantic import BaseModel, Field

class Item(BaseModel):
    name: str = Field(min_length=1, description="商品名")
    price: float = Field(gt=0)
```

`Field()` 主要用于：默认值、约束（min/max/gt/lt…）、元信息（title/description/examples…）。([Pydantic](https://docs.pydantic.dev/latest/concepts/fields/?utm_source=chatgpt.com))

------

## 4) Alias：字段别名（外部字段名 ≠ 内部字段名）

比如外部给你 `userName`，你内部想用 `user_name`：

```python
from pydantic import BaseModel, Field, ConfigDict

class User(BaseModel):
    model_config = ConfigDict(populate_by_name=True)  # 允许用字段名或别名都能输入
    user_name: str = Field(alias="userName")
```

Pydantic 支持在 `Field` 上指定 `alias/validation_alias/serialization_alias`，也支持 `alias_generator`（批量生成）。([Pydantic](https://docs.pydantic.dev/latest/concepts/alias/?utm_source=chatgpt.com))

------

## 5) model_config：控制模型行为（v2 的配置方式）

Pydantic v2 推荐用 `model_config = ConfigDict(...)`；v1 的 `class Config:` 仍支持但已不推荐。([Pydantic](https://docs.pydantic.dev/2.0/usage/model_config/?utm_source=chatgpt.com))

常见配置你会经常用到：

- `extra="forbid"/"ignore"/"allow"`：多余字段怎么处理（禁止/忽略/保留）
- `str_strip_whitespace=True`：自动去掉字符串两端空格
- `populate_by_name=True`：允许用字段名填充（配合 alias 很常用）
  这些都属于 `model_config` 的范畴。([Pydantic](https://docs.pydantic.dev/2.0/usage/model_config/?utm_source=chatgpt.com))

------

## 6) FastAPI 里 BaseModel 的实际价值

### 6.1 请求体（Request Body）

你把 `BaseModel` 写在参数位置，FastAPI 就把它当请求体并自动校验。([fastapi.tiangolo.com](https://fastapi.tiangolo.com/tutorial/body/?utm_source=chatgpt.com))

### 6.2 响应模型（response_model）

`response_model=...` 会用模型来**过滤/转换输出**，并生成 API 文档。([fastapi.tiangolo.com](https://fastapi.tiangolo.com/tutorial/response-model/?utm_source=chatgpt.com))

------

## 7) 你真正写业务时最常踩的 3 个点

1. **别名顺序/标签映射**：用 alias 或 ClassLabel 时要固定顺序，别依赖 `sorted(set(...))` 这种不可控排序。
2. **extra 字段**：对外接口建议 `extra="forbid"`（早点暴露脏字段），对兼容老接口可以先 `ignore`。([Pydantic](https://docs.pydantic.dev/2.0/usage/model_config/?utm_source=chatgpt.com))
3. **序列化**：返回给前端/存库用 `model_dump()`（配合 `exclude_none=True` 等），别直接 `__dict__`（会漏掉转换/别名规则）。


## 零、官网

https://docs.langchain.com/oss/python/integrations/graphs/neo4j_cypher



`GraphCypherQAChain` 是 LangChain 里一个“**让大模型写 Cypher → 去 Neo4j 查 → 再把结果组织成答案**”的链。你可以把它理解成：**“LLM 驱动的 Neo4j 查询代理”**。

下面我按你现在项目会真正用到的角度，把它讲透：**它做什么、内部流程、关键组件、常见坑、怎么更稳地用**。

------

## 它到底干嘛的

给它一个自然语言问题，比如：

> “Apple 旗下有哪些产品？”

它会做三步：

1. **读图谱 schema**（你有哪些节点/关系/属性）
2. **让 LLM 生成 Cypher**
3. **执行 Cypher 拿结果**，再**让 LLM 把结果写成自然语言答案**

所以它解决的是：

> 你不想手写 Cypher，但又希望答案来自真实图谱而不是模型瞎编。

------

## 内部工作流程（很重要，理解后你就能调稳）

典型流程是这样：

### 1) 获取/注入 schema

链会把图谱结构（节点 label、关系类型、常用属性等）提供给 LLM，用来约束生成。

- 你现在的 `self.graph.schema` 就是这块。

### 2) Cypher 生成（LLM）

LLM 输入大概长这样（概念上）：

- 用户问题：`question`
- 图谱结构：`schema`
- 规则：只生成 Cypher / 参数化 / 限制 label、关系类型等

输出：`cypher_query`

### 3) 执行 Cypher（Neo4j）

用 Neo4j driver 执行 `cypher_query`（以及参数）。

### 4) 结果生成（LLM）

把查询结果（表格/records）再喂给 LLM，总结成自然语言答案。

> 所以它其实是：**LLM → DB → LLM** 的典型“工具调用链”。

------

## 关键组件拆开讲

### A. Graph（Neo4jGraph）

负责连接 Neo4j、获取 schema、执行 query。

你会看到类似：

- `graph.schema`
- `graph.query(cypher, params)`

### B. cypher_llm

专门用来“写 Cypher”的 LLM（建议温度低、输出更稳定）。

### C. qa_llm

专门用来“读查询结果并回答”的 LLM（可比 cypher_llm 更强一些）。

### D. Prompt（Cypher 生成提示词）

这是成败关键。你现在自己手写 prompt，本质上就是在复刻/增强 chain 的提示词。

------

## 它的“优势”和“天然风险”

### 优势

- 上手快：问中文/英文都行
- Cypher 自动生成
- 能利用图谱关系推理（比如多跳查询）
- 比纯 RAG 更结构化、可解释（能看到 Cypher）

### 风险（你已经踩中两个）

1. **LLM 编 Cypher 幻觉**（关系名写错、属性名写错、方向错）
2. **schema 不完整/太长** 导致生成不稳
3. **安全风险**：LLM 可能生成写操作（DELETE / SET）或全图扫描
4. **性能风险**：没 LIMIT、全库扫、路径爆炸



------

## 让 GraphCypherQAChain 更稳的“工程做法”（强烈推荐）

### 1) 把 Cypher 生成和 QA 分开（双 LLM 或同 LLM 不同 prompt）

- `cypher_llm`: temperature=0，强约束输出
- `qa_llm`: 用更好的模型生成回答

### 2) 加“Cypher 安全守卫”（必做）

在执行前做规则校验：

- 禁止 `CREATE / MERGE / DELETE / SET / DROP / CALL dbms.*`
- 必须包含 `LIMIT`
- 必须在允许的 label/relationship 集合内

### 3) 做“schema 白名单”而不是全量 schema

把 schema 提炼成“对当前问题相关的子图”，会稳定很多。

### 4) 两段式：先实体对齐，再让 LLM 写结构化查询

你现在已经在做这个了（输出 entities_to_align），这其实是**最佳实践**：

- 先把“Apple”对齐到图谱里的标准实体名/ID
- 再生成 Cypher（参数化）
- 执行

这样大幅减少“LLM 猜错实体”的概率。

------

## 一段你可以直接对照理解的“最小链路伪代码”

```python
schema = graph.schema

# 1) LLM 生成 cypher（带 $param_0）
cypher, entities = cypher_llm.generate(question, schema)

# 2) 实体对齐（把 entities 变成真实值）
params = align_entities_to_params(entities)

# 3) 执行
rows = graph.query(cypher, params)

# 4) LLM 基于 rows 生成答案
answer = qa_llm.summarize(question, rows)

return answer
```

GraphCypherQAChain 就是把这套流程“封装成一个组件”。

------

## 常见配置点（你以后一定会调）

- `verbose=True`：打印生成的 Cypher（调试必开）
- `validate_cypher=True` / “自建校验器”：防止乱写
- `top_k`：限制返回条数，避免结果太大
- `return_direct=True`：只要查询结果，不要 LLM 总结（用于你自己后处理）
- `allow_dangerous_requests=False`：如果你的版本/实现支持，务必关写操作

（不同 LangChain 版本参数名会略有差异，但思路是这些。）

------

## 使用GraphCypherQAChain实现

```
from langchain_core.prompts.prompt import PromptTemplate

CYPHER_GENERATION_TEMPLATE = """Task:Generate Cypher statement to query a graph database.
Instructions:
Use only the provided relationship types and properties in the schema.
Do not use any other relationship types or properties that are not provided.
Schema:
{schema}
Note: Do not include any explanations or apologies in your responses.
Do not respond to any questions that might ask anything else than for you to construct a Cypher statement.
Do not include any text except the generated Cypher statement.
Examples: Here are a few examples of generated Cypher statements for particular questions:
# How many people played in Top Gun?
MATCH (m:Movie {{name:"Top Gun"}})<-[:ACTED_IN]-()
RETURN count(*) AS numberOfActors

The question is:
{question}"""

CYPHER_GENERATION_PROMPT = PromptTemplate(
    input_variables=["schema", "question"], template=CYPHER_GENERATION_TEMPLATE
)

chain = GraphCypherQAChain.from_llm(
    ChatOpenAI(temperature=0),
    graph=graph,
    verbose=True,
    cypher_prompt=CYPHER_GENERATION_PROMPT,
    allow_dangerous_requests=True,
)

chain.invoke({"query": "How many people played in Top Gun?"})
```

执行结果

```
> Entering new GraphCypherQAChain chain...
Generated Cypher:
MATCH (m:Movie {name:"Top Gun"})<-[:ACTED_IN]-()
RETURN count(*) AS numberOfActors
Full Context:
[{'numberOfActors': 4}]

> Finished chain.

{'query': 'How many people played in Top Gun?',
 'result': 'There were 4 actors in Top Gun.'}
```



## 自己编写的实现该功能的流程

```
# 核心聊天服务流程
def chat(self,question):
    # 1. 根据用户问题生成cypher以及需要对齐的实体
    result = self._generate_cypher(question)
    cypher = result['cypher_query']
    entities_to_align = result['entities_to_align']
    print(cypher)
    print("对齐之前的实体名称：",entities_to_align)
    # 2. 实体对齐（混合检索）
    aligned_entities = self._entity_align(entities_to_align)
    print("对齐之后的实体名称：",aligned_entities)
    # 3. 执行cypher得到查询结果
    query_result = self._execute_cypher(cypher, aligned_entities)
    print("查询结果：",query_result)
    # 4. 结合用户问题和查询结果生成答案
    answer = self._generate_answer(question,query_result)
    print("最终回答：",answer)
    return answer

# 1. 根据问题调用llm生成cypher
def _generate_cypher(self,question):
    # 提示词
    prompt = """
                    你是一个专业的Neo4j Cypher查询生成器。你的任务是根据用户问题生成一条Cypher查询语句，用于从知识图谱中获取回答用户问题所需的信息。

                    用户问题：{question}

                    知识图谱结构信息：{schema_info}

                    要求：
                    1. 生成参数化Cypher查询语句，用$param_0, $param_1等代替具体值
                    2. 识别需要对齐的实体
                    3. 必须严格使用以下JSON格式输出结果
                    {{
                      "cypher_query": "生成的Cypher语句",
                      "entities_to_align": [
                        {{
                          "param_name": "param_0",
                          "entity": "原始实体名称",
                          "label": "节点类型"
                        }}
                      ]
                    }}"""
    prompt = PromptTemplate.from_template(prompt)
    prompt = prompt.format(question=question,schema_info=self.graph.schema)
    # 得到模型输出
    output = self.llm.invoke(prompt)
    # 解析成json
    result = self.json_parser.invoke(output)
    return result

# 2. 实体对齐，混合检索
def _entity_align(self,entities_to_align):
    # 遍历所有的实体
    for index,entity_to_align in enumerate(entities_to_align):
        label = entity_to_align["label"]
        entity = entity_to_align["entity"]
        # 混合检索，得到对齐后的实体名称
        aligned_entity = self.neo4j_vectors[label].similarity_search(entity,k=1)[0].page_content
        # 覆盖原来的实体名称
        entities_to_align[index]['entity'] = aligned_entity
    return entities_to_align

# 3. 用对齐的实体名称替换param_0，执行cypher
def _execute_cypher(self,cypher,aligned_entities):
    # 提取对齐后的实体名称
    params = {aligned_entity['param_name']:aligned_entity['entity'] for aligned_entity in aligned_entities }
    return self.graph.query(cypher,params)

# 4. 生成回答
def _generate_answer(self,question,query_result):
    prompt = """
                    你是一个电商智能客服，根据用户问题，以及数据库查询结果生成一段简洁、准确的自然语言回答。
                    用户问题: {question}
                    数据库返回结果: {query_result}
            """
    prompt = prompt.format(question=question, query_result=query_result)
    output = self.llm.invoke(prompt)
    result = self.str_parser.invoke(output)
    return result
```
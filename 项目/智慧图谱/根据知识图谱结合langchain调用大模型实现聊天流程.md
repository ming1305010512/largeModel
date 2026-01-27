## 1. 给知识图谱Neo4j创建全文索引和向量索引

```
def __init__(self):
    self.graph = Neo4jGraph(
        url=NEO4J_CONFIG["uri"],
        username=NEO4J_CONFIG["auth"][0],
        password=NEO4J_CONFIG["auth"][1],
    )
    # 嵌入模型
    self.embedding_model = HuggingFaceEmbeddings(
        model_name="BAAI/bge-base-zh-v1.5",
        encode_kwargs={'normalize_embeddings': True},
    )

# 创建全文索引，传入索引名称，索引标签，属性
def create_fulltext_index(self,index_name,label,property):
    cypher = f"""
        create fulltext index {index_name} if not exists
        for (n: {label}) on each [n.{property}] 
    """
    self.graph.query(cypher)

# 创建向量索引,需要传入生成向量的原属性以及嵌入向量属性
def create_vector_index(self,index_name,label,source_property,embedding_property):

    # 生成嵌入向量并添加到节点属性中
    embedding_dim = self._add_embedding(label,source_property,embedding_property)

    cypher = f"""
        CREATE VECTOR INDEX {index_name} IF NOT EXISTS
        FOR (n:{label})
        ON n.{embedding_property}
        OPTIONS {{ indexConfig: {{
         `vector.dimensions`: {embedding_dim},
         `vector.similarity_function`: 'cosine'
        }}}}
    """
    self.graph.query(cypher)

# 生成嵌入向量并添加到节点属性中,返回向量维度
def _add_embedding(self,label,source_property,embedding_property):
    # 1. 查询所有节点对应的原属性值，作为模型的输入，还需要查出节点id
    cypher = f"""
        match (n:{label})
        return n.{source_property} as text,id(n) as id
    """
    results = self.graph.query(cypher)
    # 2. 获取查询结果中的文本内容
    docs = [result['text'] for result in results]

    # 3. 调用嵌入模型，得到嵌入向量
    embeddings = self.embedding_model.embed_documents(docs)

    # 4. 将id和嵌入向量组合成字典形式
    batch = []
    for result, embedding in zip(results,embeddings):
        batch.append({"id":result['id'], "embedding":embedding})

    # 5. 执行cypher,按id查节点，写入新的嵌入向量属性
    cypher = f"""
        unwind $batch as item
        match (n:{label})
        where id(n) = item.id
        set n.{embedding_property} = item.embedding
    """
    self.graph.query(cypher,params={'batch':batch})

    return len(embeddings[0])
```

通过HuggingFaceEmbeddings使用模型BAAI/bge-base-zh-v1.5，将文本生成向量，并将生成的向量当成向量属性

如

```
index.create_fulltext_index("spu_fulltext_index","SPU","name")
index.create_vector_index("spu_vector_index","SPU","name","embedding")
```

## 2. 根据问题调用llm生成cypher

```
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
```

## 3. 实体对齐

```
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
```

其中neo4j_vectors为

```
# 定义所有实体对应的混合检索Neo4jVector对象
self.neo4j_vectors = {
    'Trademark': Neo4jVector.from_existing_index(
        self.embedding_model,
        url=NEO4J_CONFIG["uri"],
        username=NEO4J_CONFIG["auth"][0],
        password=NEO4J_CONFIG["auth"][1],
        index_name='trademark_vector_index',
        keyword_index_name='trademark_fulltext_index',
        search_type=SearchType.HYBRID,
    ),
    'SPU': Neo4jVector.from_existing_index(
        self.embedding_model,
        url=NEO4J_CONFIG["uri"],
        username=NEO4J_CONFIG["auth"][0],
        password=NEO4J_CONFIG["auth"][1],
        index_name='spu_vector_index',
        keyword_index_name='spu_fulltext_index',
        search_type=SearchType.HYBRID,
    )}
```

使用Neo4jVector的similarity_search方法得到对齐后的实体名称

## 4. 用对齐的实体名称替换param_0，执行cypher，查询知识图谱中数据

```
# 3. 用对齐的实体名称替换param_0，执行cypher
def _execute_cypher(self,cypher,aligned_entities):
    # 提取对齐后的实体名称
    params = {aligned_entity['param_name']:aligned_entity['entity'] for aligned_entity in aligned_entities }
    return self.graph.query(cypher,params)
```

## 5. 调用大模型生成回答

```
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

其中步骤2、3、4、5可以直接使用[GraphCypherQAChain](Neo4j\langchain_neo4j\GraphCypherQAChain.md)实现

示例：

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


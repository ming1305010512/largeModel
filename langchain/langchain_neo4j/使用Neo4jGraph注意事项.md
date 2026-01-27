## 1. 实现

```
graph = Neo4jGraph(
    url=NEO4J_CONFIG["uri"],
    username=NEO4J_CONFIG["auth"][0],
    password=NEO4J_CONFIG["auth"][1],
)
```

## 2. 注意

1. 需要安装AOPC插件

   将labs下的包apoc-5.26.9-core.jar迁移到plugins下即可

2. 需要配置dbms.security.procedures.unrestricted=apoc.meta.data

需要解除限制的过程和函数可以通过数据库配置设置 [dbms.security.procedures.unrestricted](https://neo4j.ac.cn/docs/operations-manual/current/configuration/configuration-settings/#config_dbms.security.procedures.unrestricted) 指定，该设置位于 `conf/neo4j.conf` 文件中。例如，要解除对 `apoc.cypher.runFirstColumn` 以及 `apoc.cypher` 包中所有过程和函数的限制，请使用：

- `dbms.security.procedures.unrestricted=apoc.cypher.runFirstColumn,apoc.cypher.*`.
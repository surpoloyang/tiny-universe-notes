# GraphRAG技术学习笔记

## 一、GraphRAG简介与核心动机

### 1.1 传统RAG的痛点
传统RAG（检索增强生成）技术存在两个关键问题：
- **分块导致的语义断裂**：文档分块后，相关信息可能被拆分到不同块中。例如查询"小明认识的木匠叫什么名字"时，"小明爷爷叫老明"和"小明爷爷是木匠"可能在不同块，导致关键信息召回不全。
- **全局信息查询困难**：对于"查询文档中小明所有家人的信息"这类需要整合分散信息的问题，传统分块检索难以高效处理，且复杂推理能力弱。


### 1.2 GraphRAG的解决方案
GraphRAG通过将文档转化为知识图谱（三元组形式）解决上述问题：
- 实体及关系结构化存储，避免分块导致的语义割裂
- 利用图数据库的关联查询能力，高效整合分散信息
- 结合社区聚类技术，预处理全局信息，提升复杂查询响应能力


## 二、前置模块实现

### 2.1 LLM模块
- **作用**：负责文档清洗、实体提取、关系抽取等自然语言处理任务
- **实现方式**：
  - 定义抽象基类`BaseLLM`，统一接口（`predict`方法）
  - 基于智谱AI API实现`zhipuLLM`，通过`client.chat.completions.create`调用模型
- **关键代码**：
```python
# 核心调用逻辑
response = self.client.chat.completions.create(
    model=self.model_name,
    messages=[{"role": "user", "content": input}],
)
```


### 2.2 Embedding模块
- **作用**：将文本转化为向量，用于后续相似度计算
- **实现方式**：
  - 抽象基类`BaseEmb`定义`get_emb`接口
  - `zhipuEmb`调用智谱AI嵌入模型，返回向量列表
- **关键代码**：
```python
# 向量生成逻辑
emb = self.client.embeddings.create(
    model=self.model_name,
    input=text,
)
return emb.data[0].embedding
```


### 2.3 Neo4j交互
- **作用**：作为图数据库存储实体、关系及社区信息
- **核心操作**：
  - 通过`GraphDatabase.driver`创建连接
  - 使用Cypher语句执行查询（如`MATCH (n) RETURN n`查询所有节点）
  - 用`session.run`执行Cypher并处理结果


## 三、核心流程实现

### 3.1 数据预处理

#### 3.1.1 文本分块
- **策略**：滑动窗口分块（`segment_length`为块长度，`overlap_length`为重叠长度）
- **目的**：解决LLM输入长度限制，同时保留上下文关联
- **关键代码**：
```python
# 分块逻辑核心
while start_index + segment_length <= len(content):
    text_segments.append(content[start_index : start_index + segment_length])
    start_index += segment_length - overlap_length  # 滑动步长 = 块长 - 重叠长
```


#### 3.1.2 实体提取
- **实体结构**：
  ```python
  class Entity:
      name: str  # 实体名称（如"小明"）
      desc: str  # 实体描述（如"小明是北京大学的学生"）
      chunks_id: list  # 所属文本块ID
      entity_id: str  # 唯一标识（通过MD5哈希生成）
  ```
- **提取方式**：通过提示词引导LLM输出结构化实体（HTML标签包裹），再用正则解析
- **提示词设计**：明确任务（提取关键概念及描述）、提供示例、指定输出格式（`<concept><name></name><description></description></concept>`）


#### 3.1.3 三元组提取
- **目标**：从文本中提取`<subject, predicate, object>`关系（如"小明-的爷爷是-老明"）
- **约束**：主语和宾语必须来自已提取的实体
- **提取方式**：类似实体提取，用提示词引导LLM输出`<triplet>`标签包裹的结构化关系，正则解析获取`subject_id`、`predicate`、`object_id`等信息


### 3.2 实体消岐
- **问题**：同一实体可能有不同名称，或同名实体实为不同对象（如"苹果公司"和"苹果水果"）
- **解决方案**：
  1. 收集同名实体列表
  2. 用LLM判断是否可合并，生成实体ID映射（如`{"entity-2": "entity-1"}`表示entity-2合并到entity-1）
  3. 更新实体ID及三元组中的关联ID
- **关键**：确保合并后实体描述和所属块ID的整合


### 3.3 导入Neo4j
- **核心Cypher语句**：
```cypher
MERGE (a:Entity {name: $subject_name, description: $subject_desc, chunks_id: $subject_chunks_id, entity_id: $subject_entity_id})
MERGE (b:Entity {name: $object_name, description: $object_desc, chunks_id: $object_chunks_id, entity_id: $object_entity_id})
MERGE (a)-[r:Relationship {name: $predicate}]->(b)
```
- **作用**：`MERGE`确保节点/关系不存在则创建，存在则更新，避免重复


### 3.4 社区聚类
- **目的**：处理全局查询（如"小明家有几口人"），通过预聚类减少实时计算量
- **算法**：分层Leiden算法（基于Neo4j的GDS库）
  - 递归划分社区：全图→子社区→...→叶级社区
  - 用模块度衡量聚类质量，控制分辨率参数γ调整社区规模
- **实现步骤**：
  1. 投影图数据到GDS：`gds.graph.project`
  2. 运行Leiden算法并写入社区属性：`gds.leiden.write`
  3. 生成社区摘要：整合子社区信息，支持层级浏览


## 四、学习总结

### 4.1 GraphRAG核心流程
1. 文档分块→2. 实体提取→3. 三元组提取→4. 实体消岐→5. 导入Neo4j构建图谱→6. 社区聚类→7. 支持查询

### 4.2 关键技术点
- 用LLM实现非结构化文本到结构化图谱的转化（实体+关系提取）
- 实体消岐确保图谱准确性
- 社区聚类提升全局查询能力
- 图数据库（Neo4j）提供高效关联查询支持

### 4.3 待深入学习
- 社区聚类的模块度计算细节
- 实体消岐的准确率优化（如何减少LLM判断误差）
- 大规模文档处理时的性能优化（分块策略、并行计算等）
---
date: 2026-05-26
aliases:
  - my first note about Agentic Rag
tags:
---
# RAG的三种架构模式
## 1. Two Step RAG:可预测经典模式
### 工作流程：
Query → Retrieve → Generate
固定执行两个步骤：先检索相关文档，再生成答案。无论问题复杂度如何，系统都会执行完整的检索流程。
### 适用场景：
简单FAQ、客服问答等标准问答
``` python
```none
# 创建检索器
retriever = vector_store.as_retriever(k=3)

# 格式化检索到的文档
def format_docs(docs):
 return"

".join(doc.page_content for doc in docs)

# 构建 RAG 链
rag_chain = (
 {"context": retriever | format_docs, "question": RunnablePassthrough()}
 | prompt
 | chat_model
)

# 执行查询
answer = rag_chain.invoke("What were Nike's 2023 revenues?")
``````none
# 创建检索器
retriever = vector_store.as_retriever(k=3)

# 格式化检索到的文档
def format_docs(docs):
 return"

".join(doc.page_content for doc in docs)

# 构建 RAG 链
rag_chain = (
 {"context": retriever | format_docs, "question": RunnablePassthrough()}
 | prompt
 | chat_model
)

# 执行查询
answer = rag_chain.invoke("What were Nike's 2023 revenues?")
```

## 2. Agentic RAG
### 决策流程：
Agent判断——>执行工具——>执行0-n次——>生成答案
Agent负责判断：是否需要检索，检索几次，调用哪些工具

### 适用场景：
- 复杂问答
- 多步推理

``` python

from langchain_core.tools import tool

# 定义检索工具
@tool
def search_knowledge_base(query: str) -> str:
 """从知识库中检索相关信息"""
 results = vector_store.similarity_search(query, k=3)
 return"
".join([doc.page_content for doc in results])

# 创建 Agentic RAG
agent = create_react_agent(
 model="openai:gpt-4",
 tools=[search_knowledge_base],
 system_prompt="你是一个智能助手,可以根据需要查询知识库来回答问题。"
)

# 执行查询
result = agent.invoke({
 "messages": [{"role": "user", "content": "比较 Nike 和 Adidas 2023 年的收入"}]
})
```

## 2. Corrective RAG：质量优先的进阶

### **验证流程**：
查询增强——>检索——>相关性评估——>生成——>答案验证——>自纠正

整个RAG流程加入多个质量检查点。如果任何环节不达标系统就会自动回退并重试

### **特征**：
- 查询增强：对原始问题的优化
- 文档验证：评估检索文档是否真正相关
- 答案校验：检查答案的完整性和正确性
- 自纠正机制：发现问题自动重新执行
### **适用场景**：
对质量要求极高的法律、医疗、金融领域

### **优点**：         
- 答案质量高，自动纠错
- 减少幻觉和错误信息
- 多重验证保证准确性

### **缺点**：
- 耗时长
- LLM调用成本高
- 实现复杂度最高




# RAG Pipeline
## 1. 加载文档

文档加载的目标是从各种数据源提取文本内容，并规范统一为Document对象。每个Document包含page_content和metadata

``` python
from langchain_community.document_loaders import PyPDFLoader  
from langchain_core.documents import Document  
  
file_path=r"C:\Users\81361\Desktop\first_demo\MasteringRAG\data\2024全球经济金融展望报告.pdf"  
loader=PyPDFLoader(file_path)  
documents=loader.load()

```
### **常见数据源**：

| 数据源类型    | 加载器                         | 应用场景       |
| -------- | --------------------------- | ---------- |
| PDF      | PyPDFLoader                 | 财报、论文、技术文档 |
| 网页       | WebBaseLoader               | 在线文档、新闻文章  |
| Markdown | UnstructuredMarkdownLoader  | 技术博客、wiki  |
| CSV/JSON | CSVLoader/JSONLoader        | 结构化数据      |
| 企业系统     | NotionLoder/ObsidianLoader/ | 团队知识库      |

## 2. 文本切块
#### 切块三个原因：
- 上下文窗口限制：大模型一次只能处理有限的文本量
- 提高检索精度：小片段更容易精确匹配用户问题
- 优化存储效率：向量数据库对小片段的索引更高效 分片参数调优
``` python
from langchain_text_splitters import RecursiveCharacterTextSplitter  
text_splitter=RecursiveCharacterTextSplitter(chunk_size=1000,chunk_overlap=100,separators=['\n\n\n','\n\n',' '])  
chunks=text_splitter.split_documents(documents)  
print(len(chunks))

```

#### 分块参数调优：

| chunk size | 适用场景       | 特点            |
| ---------- | ---------- | ------------- |
| 200-500    | 精确问答、关键词匹配 | 检索精准但可能丢失上下文  |
| 500-1000   | 通用知识检索     | 平衡精度和上下文      |
| 1000-2000  | 复杂推理和上下文理解 | 保留语境但可能检索不够精准 |

## 3. 向量化
 向量化是把文本转变为数字的过程。计算机无法直接理解文本，但可以处理数字，向量化模型可以把文本转化为高维向量，而向量可以表示文本的语义信息。
 向量化的关键特性是：语义相近的文本会被转变为距离相近的向量，这使得系统可以实现跨语言的语义检索。

``` python
from langchain_community.embeddings import HuggingFaceBgeEmbeddings  
import torch  
device='gpu' if torch.cuda.is_available() else 'cpu'  
embeddings=HuggingFaceBgeEmbeddings(model_name="BAAI/bge-m3",model_kwargs={"device":device})  
embeddings.embed_query("hello world")

text=[chunk.page_content for chunk in chunks]  
doc_vectors=embeddings.embed_documents(text)  
print(len(doc_vectors))

```

#### 主流模型对比

| 模型                     | 维度   | 优势          | 适用场景   |
| ---------------------- | ---- | ----------- | ------ |
| BAAI/bge-m3            | 1024 | 开源、多语言、性能优秀 | 通用场景   |
| text-embedding-3-small | 1536 | API调用、速度快   | 快速原型开发 |
| text-embedding-3-large | 3072 | 精度最高        | 高精度场景  |

## 4. 向量检索与存储

### 技术选型： OceanBase
#### 为什么：
OceanBase v4.3.5 和 seekdb 提供了独特的 All-in-One 方案
- **混合存储能力**：单一平台同时支持向量存储、结构化数据、全文搜索，无需维护多个数据库
- **LangChain生态集成**：通过 langchain-oceanbase 组件无缝对接，使用标准 Vector Store API
- **生产级稳定性**：蚂蚁集团多年验证，经历双十一等极端场景考验
- **seekdb 赋能开发**：借助 OceanBase 开源的轻量嵌入式数据库 seekdb，开发者实现“本地 seekdb 开发，线上 OceanBase 部署”的无缝衔接。


``` python

from langchain_oceanbase.vectorstores import OceanBaseVectorStore
connection_args={
"host":"127.0.0.1",
"port":2881,
"user":"root@sys",
"password":"",
"db_name":"langchaindb"
}
vector_store=OceanBaseVectorStore(
embedding_function=embeddings,
table_name="knowledge_base",
connection_args=connection_args,
vidx_metric_type="cosine"，
index_type="HNSW"
)

#批量增加文档
ids=vetor_store.add_documents(chunks)
#相似度检索
result=vector_store.similarity_search(
query="2023年前三季度支撑美国经济增长的两个重要驱动力是什么？",
k=3#返回top3
)
```

#### **索引类型对比**：

| 索引类型 | 速度  | 精度   | 适用场景      |
| ---- | --- | ---- | --------- |
| HNSW | 极快  | 近似精确 | 大规模生产环境   |
| IVF  | 快   | 近似   | 中等规模数据    |
| FLAT | 慢   | 绝对精确 | 小规模或高精度要求 |

##### HNSW：
- 原理：把向量组织成“近邻图”
- 查询流程：
> query
    ⬇
    先在上层找到大概接近的位置
    ⬇
    进入下层找到更近的点
    ⬇
    最后到底层做精细搜索

##### IVF：
- 原理：先把向量分簇，再在相似的簇里进行搜索
- 查询流程：
query
⬇
先判断接近哪些相似的簇
⬇
在这些选择的簇里进行搜索

##### FLAT：
- 原理：暴力搜索，枚举遍历
- 查询流程：
query
⬇
每个向量逐个计算相似度
⬇
排序
⬇
top_k



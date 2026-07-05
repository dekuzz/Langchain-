---
date: 2026-05-27
aliases:
  - my second note about Agentic RAG
tags:
---
# 什么是Corrective RAG?

Corrective RAG是在检索过程中加入自纠正层的增强架构，通过验证检索质量并在文档不相关时启动备用方案，确保答案的准确性和可靠性。

# 基础RAG的问题
## 检索失败
### **具体表现**：
- 查询与文档语义不匹配
- 知识库缺少最新消息

### **对用户的影响**：
- 检索结果与问题无关


## 过时信息
### **具体表现**：
知识库的文档无法反映最新动态

### **对用户的影响**：
生成过时或错误的答案

## 缺乏恢复路径

### **具体表现**：
系统无法识别检索质量差的内容，盲目使用无关文档

### **对用户的影响**：
产生幻觉，降低信任度

## 总结：
基础RAG会盲目信任检索结果，即使检索结果和用户问题不相关，系统也会基于这些无关信息生成答案，导致答案质量无法保证

# CRAG核心机制

[[CRAG核心机制.canvas]]

## 主要步骤
### 文档评分：对检索到的文档进行相关性评估，判断是否与问题相关
### 智能托底：当检索失败，自动切换到备用数据源（如：Web search）
### 自适应流程： 根据评分结果动态调整执行路径

# Agentic RAG VS Corrective RAG

CRAG是Agentic RAG的一种特殊形式，在Agent执行过程中添加了质量检查机制，仍然遵循ReAct的推理-行动循环

Agent 保持主动权，Middleware在工具执行节点插入验证逻辑，不干预Agent决策过程。

# Seekdb

## 定义：
由Ocean Base 推出的AI原生搜索数据库，将向量存储，关系数据，全文搜索整合到统一平台。在构建RAG系统时，为避免额外的运维成本和系统复杂度，不选择专门的向量数据库而选择Seekdb

## 优势：
### **统一多模态引擎**：
- 说明：
	- 支持关系型，向量、全文、json、GIS等多种数据
- 优势
	- 无需维护多个数据库，降低运维成本
### **AI框架集成**：
- 说明：
	- 深度对接langchain,LlamaIndex,Dify,Mcp Server
- 优势：
	- 开箱即用，标准API
### **混合检索**：
- 说明：
	- 单次查询同时支持 向量相似度+全文匹配+元数据过滤
- 优势:
	- 提升检索精度和灵活性
### **ACID事务**：
- 说明：
	- 完整事务支持，保证数据一致性
- 优势：
	- 适合企业级应用

### **MySQL兼容**：
- 说明：
	- 使用MySQL驱动即可连接，代码迁移成本低




``` python
from langchain_oceanbase.vectorstores import OceanbaseVectorStore 
 
# 创建向量存储 
vector_store = OceanbaseVectorStore( 
 embedding_function=embeddings, 
 table_name="langchain_knowledge_base", 
 connection_args=connection_args, 
 vidx_metric_type="cosine", # 余弦相似度 
 index_type="HNSW" # 高性能向量索引 
) 
 
# 创建检索器 
retriever = vector_store.as_retriever(search_kwargs={"k": 3})

```

# CRAG使用的Middleware Hooks
## Middleware提供了6个Hook来拦截不同执行阶段：

| Hook            | 触发时机       | 返回类型            | 用途        |
| --------------- | ---------- | --------------- | --------- |
| before_agent    | agent开始运行前 | dict            | 初始化状态     |
| after_agent     | agent运行结束后 | dict            | 记录最终状态    |
| before_model    | 每次llm调用前   | dict            | 修改状态、触发托底 |
| after_model     | 每次调用llm后   | dict            | 记录响应，调试   |
| wrap_model_call | 包裹llm调用    | 必须调用handler()   | 修改可用工具列表  |
| wrao_tool_call  | 包裹工具调用     | 必须返回ToolMessage | 拦截工具输出并评分 |
## Middleware在CRAG的优势:
Middleware实现了关注点分离，将评分逻辑、托底逻辑、工具过滤逻辑分别封装到独立的中间件，提升了代码的可维护性和可测试性。
## CRAG使用的Middleware Hooks

### 1. `wrap_tool_call`：拦截RAG工具输出，对检索文档进行相关性评分
### 2. `before_model`:应用评分结果到状态；在文档不相关时触发Web_search托底
### 3. `wrap_model_call`：动态过滤可用工具（仅Wrap-Style使用）

# 构建CRAG ：从文档评分到智能托底

## 1. 构建RAG检索工具
``` python
from langchain.tools import tool  
@tool 
def search_nike_report(query: str) -> str:   
     """
     Search the 2024 Global Economic and Financial Outlook Report for macroeconomic information.  
     Use this tool when you need to find information about:  
     -Global economic growth trends and forecasts(2024-2024)
     - Major economies' performance (US,Europe,China,emerging markets) 
	- Monetary policy changes and central bank actions     
	- International trends,finance and capital flows     
	- Exchange rates,bond markets ,and financial risks     
	- Regional economic analysis and investment outlook           
	The report is published by Bank of China Research Institute and covers comprehensive analysis of global economic and financial conditions.     """# 执行向量检索   
results = retriever.invoke(query)   
     # 格式化结果   
formatted_results = []   
     for i, doc in enumerate(results, 1):   
         page=doc.metadata.get('page','Unknown')  
         formatted_results.append(f"Document{i} [Page: {page}]:{doc.page_content}")  
     return "\n".join(formatted_results)

```

## 2. 文档评分
``` python
from langchain_openai import ChatOpenAI  
chat_model=ChatOpenAI(model_name="qwen3.6-flash",temperature=0.2,top_p=0.9,max_retries=3,base_url=os.environ.get("BASE_URL"),api_key=os.environ['API_KEY'])

from pydantic import BaseModel,Field  
from langchain_core.prompts import ChatPromptTemplate  
#定义评分结果的数据结构  
class GradeDocument(BaseModel):  
    """  
    Binary score for relevant check on retrieved documents    """    binary_score:str=Field(description="Documents are relevant to the question ,'yes' or 'no'")  
    reasoning:str=Field(description="Brief explanation for the relevant decision")  
      
#评分 
promptgrade_prompt=ChatPromptTemplate.from_template(  
    """  
    You are a grader assessing relevance of retrieved documents to a user question.    Retrieved document:    {document}        User question:  
    {question}        If the document contains keywords or semantic meaning related to the question ,grade it as relevant.  
    Give a binary score 'yes' or 'no' to indicate weather the document is relevant to the question.    Also provide a brief reasoning for your decision.    """)  
# 创建评分器  
grader=grade_prompt |chat_model.with_structured_output(schema=GradeDocument)
```
### **评分原则**：
- 二元评分：只判断“相关”或不相关，避免复杂多级评分。
- 强制推理：要求LLM给出判断理由，降低随机性和幻觉
- 关键词+语义：同时考虑关键词匹配和语义相似度，提高准确性

## 3. 自定义State Schema

由于CRAG需要跟踪评分结果和托底状态，因此我们扩展Langchain的AgentState

``` python
from typing import Optional,List  
from typing_extensions import NotRequired  
from langchain.agents import AgentState  
class CorrectiveRAGState(AgentState):  
    """  
    Extended state for tracking corrective RAG flow.    
    Inherits 'messages' from AgentState.Adds fields for grading and fallback tracking.    """    
    last_rag_query:NotRequired[Optional[str]]#用于托底的查询  
    documents_relevant:NotRequired[Optional[bool]]#评分结果  
    grade_reasoning:NotRequired[Optional[str]]# 评分理由  
    used_web_fallback:NotRequired[Optional[bool]]# 是否已经使用web_search  
    grading_history:NotRequired[Optional[List[dict]]]#调试信息
```

## 4. 文档评分 Middleware

为什么不能在 wrap_tool_call 中直接修改 State？因为该Hook 必须返回 ToolMessage，不能返回 State 更新字典。因此我们：

1. 在 wrap_tool_call 中评分并暂存结果
2. 在 before_model（可以返回State更新）中应用结果
``` python
from langchain.agents.middleware import AgentMiddleware,ToolCallRequest,ModelRequest,ModelResponse  
from langchain.messages import ToolMessage  
from typing import Callable,Any  
from langgraph.runtime import Runtime  
class DocumentsGradingMiddleware(AgentMiddleware):  
    state_schema = CorrectiveRAGState  
    def __init__(self,grader,rag_tool_name:str='search_nike_report'):  
        self.grader=grader  
        self.rag_tool_name=rag_tool_name  
        self._pending_grading:dict|None=None  
    def wrap_tool_call(self,request:ToolCallRequest,handler:Callable)-> ToolMessage:  
        result=handler(request)  
        if request.tool_call.get('name')==self.rag_tool_name:  
            query=request.tool_call.get('args',{}).get('query')  
            doc_content=result.content  
        grade_result=self.grader.invoke(query=query,document=doc_content)  
        is_relevant=grade_result.binary_score.lower()=='yes'  
        self._pending_grading={  
            'last_rag_query':query,  
            'documents_relevant':is_relevant,  
            'grade_reasoning':grade_result.resoning,  
        }  
        print(f"文档评分:{is_relevant} |理由: {grade_result.resoning}")  
        return result  
    def before_model(self,state:CorrectiveRAGState,runtime:Runtime)->dict|None:  
        """应用评分结果到State"""  
        if self._pending_grading is not None:  
            updates=self._pending_grading  
            self._pending_grading=None  
            return updates  
            return None

```


## 5. Web Search 兜底Middleware

当文档评分为不相关时，启用备用数据源
``` python
from langchain.messages import AIMessage  
class WebSearchFallbackMiddleware(AgentMiddleware):  
    state_schema = CorrectiveRAGState  
    def __init__(self,web_search_tool):  
        self.web_search_tool=web_search_tool  
    def before_model(self,state:CorrectiveRAGState,runtime:Runtime)->dict|None:  
        """检测失败条件并进行托底"""  
        documents_relevant=state.get('documents_relevant')  
        used_web_fallback=state.get('used_web_fallback',False)  
        last_rag_query=state.get('last_rag_query')  
              
    #触发条件：文档不相关+未使用托底+存在查询  
        should_search=(  
            documents_relevant is False and not  
            used_web_fallback  and  
            last_rag_query  
        )  
        if should_search:  
            print(f"RAG失败-启动web搜索：{last_rag_query}")  
            #执行web搜索  
            web_result=self.web_search_tool.invoke(query=last_rag_query)  
              
            #将搜索结果注入到消息流  
            injection_messages=AIMessage(content=f'我找到了相关的web搜索结果：\n\n{str(web_result)[:1500]}\n\n让我为您总结这些信息。')  
            return {  
                "used_web_fallback":True,  
                "messages":injection_messages,  
            }  
        return None
```

## CRAG的两种实现风格
### 1. Node-Style: 完全依赖托底逻辑，文档不相关自动触发Web_Search，无需Agent决策
``` python

from langchain_tavily import TavilySearch  
os.environ["TAVILY_API_KEY"] ="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"  
tavily_search = TavilySearch(  
    max_results=3,  
    topic="general",  
)  
grading_middleware = DocumentsGradingMiddleware(  
    grader=grader,  
    rag_tool_name="search_nike_report",  
)  
fallback_middleware = WebSearchFallbackMiddleware(  
    web_search_tool=tavily_search,  
)

# 只注册 RAG 工具 from langchain.agents import create_agent  
SYSTEM_PROMPT = """  
You are a corrective RAG assistant.  
  
Use the local report search tool first.  
If the retrieved documents are relevant, answer based on them.  
If the retrieved documents are not relevant, use web search fallback results.  
Be concise and cite where the information comes from when possible.  
"""  
node_agent = create_agent(   
 model=chat_model,   
 tools=[search_nike_report], # 只有 RAG 工具   
system_prompt=SYSTEM_PROMPT,   
 middleware=[   
 grading_middleware, # 评分   
fallback_middleware, # 托底   
],   
 state_schema=CorrectiveRAGState,   
)
```

### 2. Wrap-Style:灵活智能的托底
通过`wrap_model_call`动态过滤工具列表，让Agent在文档不相关时主动选择Web_search

- 工具过滤Middleware
``` python
from langchain.agents.middleware import ModelRequest,ModelResponse,ModelCallResult
class ToolFilteringMiddleware(AgentMiddleware):
	def __init__(self.rag_tool_name:str="search_nike_report"):
		self.rag_tool_name=rag_tool_name
	def wrap_model_call(self,request:ModelRequest,handler:Callable[[ModelRequest],ModelResponse])->ModelCallResult:
	documents_relevant=requests.state.get("documents_relevant")
	used_web_fallback=requests.state.get("used_web_fallback",False)
	
	should_enable_web=(documents_relevant is False and not used_web_fallback)
	if should_enable_web:
		filtered_tools=request.tools
		print(f"enable all tools : {[t.name for t in filtered_tools]}")
	else；
		filtered_tools=[t for t in filtered_tools if t.name==self.rag_tool_name]
		print(f"Filtered to: {[t.name for t in filtered_tools]}")
	modified_request=request.override(tools=filtered_tools)
	return handler(modified_requests)	
```

## 6. 代码流程图
[[corrective rag 流程图.canvas]]

![[Corrective RAG 详解-1780381010285.webp]]

# 总结
## 1. CRAG的本质
- 在Agent执行过程中添加了质量检查机制，但仍遵循ReAct框架。
## 2. 三个核心机制
- 文档评分：使用LLM结构化输出，对检索结果进行二元评分
- 智能托底：文档不想关自动切换Web_search等备用数据源
- 自适应流程：通过Middleware动态调整执行路径，保持Agent自主性
## 3. 关键技术速查表

| 技术点           | 实现方式                                                | 作用                 |
| ------------- | --------------------------------------------------- | ------------------ |
| RAG Tool      | @tool 装饰器 + OceanBase Retriever                     | 将知识库检索包装为 Agent 工具 |
| 评分器           | Pydantic + with_structured_output()                 | 结构化输出评分结果          |
| State 扩展      | CorrectiveRAGState(AgentState)                      | 跟踪评分和托底状态          |
| 评分 Middleware | wrap_tool_call + before_model                       | 拦截工具输出并评分          |
| 托底 Middleware | before_model 注入消息（Node）或 wrap_model_call 过滤工具（Wrap） | 实现智能托底             |

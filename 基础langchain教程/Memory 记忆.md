---
date: 2026-04-26
aliases:
  - my fifth note about langchain
  - my sixth note about agent
tags:
  - 外部工具存储记忆
  - langchain入门
  - langgraph
  - 跨对话提取用户偏好
  - 短期记忆
  - 长期记忆
---
# 短期记忆


## StateGraph中的应用
``` python
import os  
from dotenv import load_dotenv  
from langchain_openai import ChatOpenAI  
from langgraph.graph import StateGraph, MessagesState, START, END  
from langgraph.checkpoint.memory import InMemorySaver  
  
# 加载模型配置  
_ = load_dotenv()  
  
# 加载模型  
model = ChatOpenAI(  
    api_key=os.getenv("DASHSCOPE_API_KEY"),  
    base_url=os.getenv("DASHSCOPE_BASE_URL"),  
    model="qwen3-coder-plus",  
    temperature=0.7,  
)  
  
# 创建助手节点  
def assistant(state: MessagesState):  
    return {'messages': [model.invoke(state['messages'])]}
# 创建短期记忆  
checkpointer = InMemorySaver()  
  
# 创建图  
builder = StateGraph(MessagesState)  
  
# 添加节点  
builder.add_node('assistant', assistant)  
  
# 添加边  
builder.add_edge(START, 'assistant')  
builder.add_edge('assistant', END)  
  
graph = builder.compile(checkpointer=checkpointer)  
  
# 告诉智能体我叫 luochangresult = graph.invoke(  
    {'messages': ['hi! i am luochang']},  
    {"configurable": {"thread_id": "1"}},  
)  
  
for message in result['messages']:  
    message.pretty_print()

# 让智能体说出我的名字  
result = graph.invoke(  
    {"messages": [{"role": "user", "content": "What is my name?"}]},  
    {"configurable": {"thread_id": "1"}},    
)  
  
for message in result['messages']:  
    message.pretty_print()
```

## create_agent()中的应用：
``` python
from langchain.agents import create_agent  
  
# 创建短期记忆  
checkpointer = InMemorySaver()  
  
agent = create_agent(  
    model=model,  
    checkpointer=checkpointer  
)  
  
# 告诉智能体我叫 luochangresult = agent.invoke(  
    {'messages': ['hi! i am luochang']},  
    {"configurable": {"thread_id": "2"}},  
)  
  
for message in result['messages']:  
    message.pretty_print()
# 让智能体说出我的名字  
result = agent.invoke(  
    {"messages": [{"role": "user", "content": "What is my name?"}]},  
    {"configurable": {"thread_id": "2"}},    
)  
  
for message in result['messages']:  
    message.pretty_print()
```

## 外部数据库支持短期记忆
``` python
# 删除SQLite数据库  
if os.path.exists("short-memory.db"):  
    os.remove("short-memory.db")
import os  
import sqlite3  
  
from dotenv import load_dotenv  
from langchain_openai import ChatOpenAI  
from langgraph.checkpoint.sqlite import SqliteSaver  
from langchain.agents import create_agent  
  
# 加载模型配置  
_ = load_dotenv()  
  
# 加载模型  
model = ChatOpenAI(  
    api_key=os.getenv("DASHSCOPE_API_KEY"),  
    base_url=os.getenv("DASHSCOPE_BASE_URL"),  
    model="qwen3-coder-plus",  
    temperature=0.7,  
)  
  
# 创建sqlite支持的短期记忆  
checkpointer = SqliteSaver(  
    sqlite3.connect("short-memory.db", check_same_thread=False)  
)  
  
# 创建Agent  
agent = create_agent(  
    model=model,  
    checkpointer=checkpointer,  
)  
  
# 告诉智能体我叫 luochangresult = agent.invoke(  
    {'messages': ['hi! i am luochang']},  
    {"configurable": {"thread_id": "3"}},  
)  
  
for message in result['messages']:  
    message.pretty_print()
    
[[重启kernel后运行下列代码：]]
import os  
import sqlite3  
  
from dotenv import load_dotenv  
from langchain_openai import ChatOpenAI  
from langgraph.checkpoint.sqlite import SqliteSaver  
from langchain.agents import create_agent  
  
# 加载模型配置  
_ = load_dotenv()  
  
# 加载模型  
model = ChatOpenAI(  
    api_key=os.getenv("DASHSCOPE_API_KEY"),  
    base_url=os.getenv("DASHSCOPE_BASE_URL"),  
    model="qwen3-coder-plus",  
    temperature=0.7,  
)  
  
# 创建sqlite支持的短期记忆  
checkpointer = SqliteSaver(  
    sqlite3.connect("short-memory.db", check_same_thread=False)  
)  
  
# 创建Agent  
agent = create_agent(  
    model=model,  
    checkpointer=checkpointer,  
)  
  
# 让智能体回忆我的名字  
result = agent.invoke(  
    {'messages': ['What is my name?']},  
    {"configurable": {"thread_id": "3"}},  
)  
  
for message in result['messages']:  
    message.pretty_print()

```

# 长期记忆

## 向量转换模块
``` python
import os  
from dotenv import load_dotenv  
from openai import OpenAI  
from langchain_core.runnables import RunnableConfig  
from langchain.agents import create_agent  
from langchain.tools import tool, ToolRuntime  
from langgraph.store.memory import InMemoryStore  
from dataclasses import dataclass  
  
EMBED_MODEL = "text-embedding-v4"  
EMBED_DIM = 1024  
  
# 加载模型配置  
_ = load_dotenv()  
  
# 用于获取text embedding的接口  
client = OpenAI(  
    api_key=os.getenv("DASHSCOPE_API_KEY"),  
    base_url=os.getenv("DASHSCOPE_BASE_URL"),  
)  
  
# 加载模型  
model = ChatOpenAI(  
    api_key=os.getenv("DASHSCOPE_API_KEY"),  
    base_url=os.getenv("DASHSCOPE_BASE_URL"),  
    model="qwen3-coder-plus",  
    temperature=0.7,  
)

# embedding生成函数  
def embed(texts: list[str]) -> list[list[float]]:  
    response = client.embeddings.create(  
        model=EMBED_MODEL,  
        input=texts,  
        dimensions=EMBED_DIM,  
    )  
  
    return [item.embedding for item in response.data]  
  
# 测试能否正常生成text embedding  
texts = [  
    "LangGraph的中间件非常强大",  
    "LangGraph的MCP也很好用",  
]  
vectors = embed(texts)  
  
len(vectors), len(vectors[0])

```
### `OpenAI()`
- **接口规范实用端点**：
		==他们的使用方式不同==
	- client.chat.completions.create: 用于聊天对话（生成文本）。
	- client.embeddings.create: 专门用于向量化。它不会返回对话文字，而是返回一个数学向量（List of floats）。
	- client.images.generate: 专门用于图像生成
	- **`client.audio.speech.create`**: 用于文字转语音（TTS）。它接收文本，返回一段自然人声的音频文件或高维音频数据流。
    
	- **`client.audio.transcriptions.create`**: 用于语音转文字（ASR）。它接收音频文件，并返回将其转录后的文本字符串。
	    
	- **`client.moderations.create`**: 专门用于内容安全审核。它不生成任何对话内容，而是返回输入文本是否违规的布尔值（True/False）以及各项敏感类别（如暴力、仇恨等）的具体评分。
	    
	- **`client.models.list`**: 用于查询可用模型。它返回当前 API Key 权限下支持的所有模型列表及其元数据（如创建时间、所有者）。
	    
	- **`client.files.create`**: 用于上传与管理文件。通常用于上传大型数据集（如 JSONL 文件），为后续的批量处理或模型微调做准备。
	    
	- **`client.fine_tuning.jobs.create`**: 专门用于创建模型微调任务。它不会立即返回生成内容，而是在后台使用你指定的数据集去训练基础模型，最终产出一个属于你的私有定制模型。


## 直接读写长期记忆
``` python
# InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production use.  
store = InMemoryStore(index={"embed": embed, "dims": EMBED_DIM})  
  
# 添加两条用户数据  
namespace = ("users", )  
key = "user_1"  
store.put(  
    namespace,  
    key,  
    {  
        "rules": [  
            "User likes short, direct language",  
            "User only speaks English & python",  
        ],  
        "rule_id": "3",  
    },  
)  
  
store.put(   
    ("users",),  # Namespace to group related data together (users namespace for user data)  
    "user_2",  # Key within the namespace (user ID as key)  
    {  
        "name": "John Smith",  
        "language": "English",  
    }  # Data to store for the given user  
)  
  
# get the "memory" by ID  
item = store.get(namespace, "a-memory")   
  
# search for "memories" within this namespace, filtering on content equivalence, sorted by vector similarity  
items = store.search(   
    namespace, filter={"rule_id": "3"}, query="language preferences"  
)  
  
items

```
### InMemoryStore(index=)
- **index的作用**：在原本 KV（键值对）存储上，加一层向量索引，让它支持语义检索
### `InMemoryStore.get()` &`InMemoryStore.search()`
- `InMemoryStore.get()`:
	- 作用：按 key 精确查一条
	- 特点：
		- 必须 key **完全匹配**
		- 只返回 **1 条**
		- **不用 embedding**
		- 查不到就是 `None`
	
- `InMemoryStore.search()`:
	- 作用：按条件 + 语义查多条
	- 流程：
		- namespace → 限定范围
		- filter → 实际存储在 value 里的字段键值对
		- query → 为了相似度搜索的语义描述
		- 返回“最相关”的多条数据
	- 特点：
	- 不需要知道 key
	- 返回 **多条**
	- 支持“按意思查”
	
## 使用工具读取长期记忆
``` python
@dataclass  
class Context:  
    user_id: str  
  
@tool  
def get_user_info(runtime: ToolRuntime[Context]) -> str:  
    """Look up user info."""  
    # Access the store - same as that provided to `create_agent`  
    store = runtime.store   
    user_id = runtime.context.user_id  
    # Retrieve data from store - returns StoreValue object with value and metadata  
    user_info = store.get(("users",), user_id)   
    return str(user_info.value) if user_info else "Unknown user"  
  
agent = create_agent(  
    model=model,  
    tools=[get_user_info],  
    # Pass store to agent - enables agent to access store when running tools  
    store=store,   
    context_schema=Context  
)  
  
# Run the agent  
result = agent.invoke(  
    {"messages": [{"role": "user", "content": "look up user information"}]},  
    context=Context(user_id="user_2")   
)  
  
for message in result['messages']:  
    message.pretty_print()

```

## 使用工具写入长期记忆
``` python
from typing_extensions import TypedDict  
  
# InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production.  
store = InMemoryStore()   
  
@dataclass  
class Context:  
    user_id: str  
  
# TypedDict defines the structure of user information for the LLM  
class UserInfo(TypedDict):  
    name: str  
  
# Tool that allows agent to update user information (useful for chat applications)  
@tool  
def save_user_info(user_info: UserInfo, runtime: ToolRuntime[Context]) -> str:  
    """Save user info."""  
    # Access the store - same as that provided to `create_agent`  
    store = runtime.store   
    user_id = runtime.context.user_id   
    # Store data in the store (namespace, key, data)  
    store.put(("users",), user_id, user_info)   
    return "Successfully saved user info."  
  
agent = create_agent(  
    model=model,  
    tools=[save_user_info],  
    store=store,  
    context_schema=Context  
)  
  
# Run the agent  
agent.invoke(  
    {"messages": [{"role": "user", "content": "My name is John Smith"}]},  
    # user_id passed in context to identify whose information is being updated  
    context=Context(user_id="user_123")   
)  
  
# You can access the store directly to get the value  
store.get(("users",), "user_123").value

```
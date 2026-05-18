---
date: 2026-04-16
aliases:
  - my second note about Agent
  - my first note about how to use langchain
tags:
  - ReActAgent
  - langchain入门
  - Agent搭建
  - Agent工具
---
# 一、 示例一 使用 `ToolRuntime` 控制工具权限

``` python
from typing import Literal,Any  
from pydantic import BaseModel  
from langchain.tools import tool,ToolRuntime  
  
class Context(BaseModel):  
    authority:Literal['admin','user']  
      
@tool  
def math_add(runtime:ToolRuntime[ Context, Any],a:int,b:int)->int:  
    authority=runtime.context.authority  
    if authority!='admin':  
        raise PermissionError("User does not have permission to add numbers")  
    return a+b  
  
tool_agent=create_agent(  
    model=llm,  
    tools=[get_weather,math_add],  
    system_prompt="You are a helpful assistant!"  
)  
response=tool_agent.invoke({'messages':[{'role':'user','content':'请计算 8234783 + 94123832 = ?'}]},  
                           config={"configurable":{"thread_id":"1"}},  
                           context=Context(authority='admin'))
```

## 1. 数据契约（Pydantic & Typing）

class Context(BaseModel):  
    authority: Literal["admin", "user"]

### 数据从哪来

- `Context`：后端创建（invoke 时传入）
- 工具参数（a, b）：模型生成

---

### 谁能看到

- `Context`：❌ 模型看不到
- `a, b`：✅ 模型可见

---

### 谁能改

- `Context`：只能后端改
- `a, b`：模型生成（可被用户间接影响）

---

### 在哪里校验

- `BaseModel`：创建 `Context` 时校验
- `Literal`：限制 authority 取值

---

### 学习侧重点

- 必须区分：**“模型输入数据” vs “系统数据”**
- 所有关键字段（权限/用户ID）必须建模，不用裸 dict
- 进阶：`field_validator`（复杂业务约束）

---

## 2. @tool（工具暴露机制）

@tool  
def math_add(runtime: ToolRuntime[Context, Any], a: int, b: int) -> int:

### 数据从哪来

- 函数定义：开发者写
- 参数 schema：由 `@tool` 从类型提示生成

---

### 谁能看到

模型能看到：

- 函数名
- 参数（a, b）
- docstring

模型看不到：

- 函数实现
- runtime/context

---

### 谁能改

- 模型只能生成 `a, b`
- 无法修改函数逻辑

---

### 在哪里校验

- 框架在调用前校验参数类型
- Python函数内部做最终校验

---

### 学习侧重点

- docstring = 给模型的“操作说明”
- 类型越严格，模型越不容易乱用
- 不要把 runtime 当作模型参数

---

## 3. ToolRuntime（运行时上下文）

authority = runtime.context.authority

### 数据从哪来

- 来自 `invoke(context=...)`
- 框架自动注入到 runtime

---

### 谁能看到

- 模型 ❌ 看不到
- 工具函数 ✅ 能访问

---

### 谁能改

- 只能后端改
- 模型无法篡改

---

### 在哪里校验

- 工具函数内部（核心位置）

if runtime.context.authority != "admin":  
    raise PermissionError

---

### 学习侧重点

- **权限必须走 runtime，不走模型参数**
- 区分：
    - 模型数据（不可信）
    - 系统数据（可信）
- 理解“依赖注入”（不是你手动传，是框架塞进去）

---

## 4. 权限控制（安全核心）

### 数据从哪来

- authority：后端 context 注入

---

### 谁能看到

- 模型 ❌ 看不到
- 工具函数 ✅ 能看到

---

### 谁能改

- 只能后端改

---

### 在哪里校验

- **必须在工具函数内部**

---

### 正确写法

if runtime.context.authority != "admin":  
    raise PermissionError

---

### 错误写法

def math_add(role: str, a: int, b: int):

（模型可以伪造 role）

---

### 学习侧重点

- 权限判断永远在“执行层”，不在“推理层”
- 模型不可信 → 必须二次校验

---

## 5. thread_id vs Context

### thread_id

config={"configurable": {"thread_id": "1"}}

#### 数据从哪来

- 后端指定

#### 谁能看到

- 模型间接看到（通过历史对话）

#### 谁能改

- 后端控制

#### 在哪里校验

- 框架内部（memory / checkpoint）

---

### Context

	context=Context(authority="admin")

#### 数据从哪来

- 后端（如登录态/JWT解析）

#### 谁能看到

- 模型 ❌ 看不到

#### 谁能改

- 后端

#### 在哪里校验

- 工具函数内部

---

### 核心区别

| 维度     | thread_id | Context |
| ------ | --------- | ------- |
| 数据来源   | 后端        | 后端      |
| 作用     | 对话记忆      | 权限控制    |
| 是否安全关键 | 否         | 是       |
| 校验位置   | 框架        | 工具函数    |

---

### 学习侧重点

- 不要用 thread_id 做权限判断
- Context 才是“真实身份”

---

## 6. 执行链（完整数据流）

用户输入  
 → messages  
 → 模型推理  
 → 决定调用 tool  
 → 框架解析参数(a,b)  
 → 注入 runtime.context  
 → 执行函数  
 → 返回结果

---

### 数据流拆解

| 数据       | 来源  | 是否可信 | 去向  |
| -------- | --- | ---- | --- |
| messages | 用户  | ❌    | 模型  |
| a,b      | 模型  | ❌    | 工具  |
| context  | 后端  | ✅    | 工具  |

---

### 学习侧重点

- 模型只负责“决定调用”
- 工具负责“真实执行”
- 安全逻辑只能在工具里

---

## 7. 错误处理（工程必须）

### 数据从哪来

- runtime/context/模型参数

---

### 谁能看到

- 错误会返回给 agent，再给模型

---

### 谁能改

- 只能开发者定义

---

### 在哪里校验

if runtime.context is None:  
    raise ValueError("Missing context")

---

### 常见错误

|错误|原因|
|---|---|
|AttributeError|context=None|
|PermissionError|权限不足|
|ValidationError|参数错误|

---

### 学习侧重点

- 工具函数必须“防御式编程”
- 所有外部输入都要怀疑

# 示例2 结构化输出
``` python
from pydantic import BaseModel,Field  
class CalcInfo(BaseModel):  
    output:int=Field(description="The calculation result")
    
structured_agent=create_agent(  
    model=llm,  
    tools=[get_weather,math_add],  
    system_prompt="You are a helpful assistant!",  
    response_format=CalcInfo  
)  

response=structured_agent.invoke({'messages':[{'role':'user','content':'请计算 8234783 + 94123832 = ?'}]},config={"configurable":{"thread_id":"1"}},context=Context(authority='admin'))

for message in response['messages']:  
    message.pretty_print()
```
## `response_format`

### 数据从哪里来
 `math_add`工具输出结果
 结构定义`CalcInfo`由开发者书写

### 谁能看到
AI模型能看到

### 谁能改
模型能填充output字段，开发者定义字段结构

### 在哪里校验
Pydantic在最终输出阶段校验
如果输出不符合结构根据框架看是否报错

### 学习侧重点
BaseModel不仅是输入校验，也是输出校验
response_format是强制模型按schema输出
真实数据必须来自tool，防止模型自己编


# 示例3：流式输出
``` python
agent=create_agent(  
    model=llm,  
    tools=[get_weather],  
    system_prompt="You are a helpful assistant!"  
)  
for chunk in agent.stream({'messages':[{'role':'user','content':'请查询上海的天气'}]},stream_mode='updates',):  
    for step,data in chunk.items():  
        print(f'step:{step}')  
        print(data['messages'][-1].content_blocks)
```
## 1. 数据从哪儿来

### A.用户输入
``` python
{'messages':[{'role':'user','content':'请查询上海的天气'}]}
```
- 来源：前端
### B.模型生成
``` python
AIMessageChunk(...)
```
### C.工具返回
``` python
ToolMessage(content='今天上海 的天气是晴天')
```
``` text
用户输入
  ↓
LLM（model节点）
  ↓（可能生成 tool_call）
Tool执行（tools节点）
  ↓
LLM总结（model节点）
  ↓
最终输出
```
==stream 本质：**把这整个流程拆成“实时事件流”一段一段吐出来**==


## 2.谁能看到数据
==取决于==`stream_mode`
### 2.1. `updates`模式
``` python
stream_mode="updates"
```
此时能看到的输出：
``` bash
step:model
[{'type': 'tool_call', 'name': 'get_weather', 'args': {'location': '上海'}, 'id': 'call_b94a89c9e205463c90ce3c3b'}]
step:tools
[{'type': 'text', 'text': '今天上海 的天气是晴天'}]
step:model
[{'type': 'text', 'text': '今天上海的天气是晴天。'}]

[[可概括为]]
{
  "model": {...},
  "tools": {...}
}
```
==**本质**==：
 - 按节点给数据
 - 看到的是流程状态
### 2.2. `messages`模式
``` python
for chunk in agent.stream({'messages':[{'role':'user','content':'请查询上海的天气'}]},stream_mode='messages',):  
    msg, meta = chunk  
      
    print(f'msg:{msg.content_blocks}')  
    for k,v in meta.items():  
        print(f'meta: {k}: {v}')
```
``` python
stream_mode="messages"
```
此时能看到的输出：
``` bash
msg:[{'type': 'tool_call_chunk', 'id': 'call_740c9d52de644f03b766df6d', 'name': 'get_weather', 'args': '', 'index': 0}]
meta: ls_integration: langchain_chat_model
meta: langgraph_step: 1
meta: langgraph_node: model
meta: langgraph_triggers: ('branch:to:model',)
meta: langgraph_path: ('__pregel_pull', 'model')
meta: langgraph_checkpoint_ns: model:1055ab6c-9cd0-396c-75b2-dcbaf8e7549e
meta: checkpoint_ns: model:1055ab6c-9cd0-396c-75b2-dcbaf8e7549e
meta: ls_provider: openai
meta: ls_model_name: qwen3-coder-plus
meta: ls_model_type: chat
meta: ls_temperature: None

[[可概括为]]
(msg, meta)
```
即：msg数据本体（模型/工具吐出来的内容）meta运行时状态（系统在干嘛）
==本质==：
- 按“token / chunk”给你数据
- 你看到的是 **生成过程**

## 3. 谁能改
- 用户：input messages
- AI: 生成
		text
		tool_calls
		tool_call_chunks（流式参数）
- 工具
	- 真正执行逻辑
	- 返回真实结果
- 框架（LangGraph / LangChain）
	- 拼接 chunk
	- 转换格式
	- 注入 runtime
## 4. 在哪里校验
runtime.context.authority

## 5. content vs content_blocks
### content

msg.content

👉 已经拼好的最终文本  
👉 适合最终输出

---

### content_blocks

msg.content_blocks

👉 原始结构（关键）

可能是：

[  
  {"type": "tool_call_chunk"},  
  {"type": "text"}  
]

👉 适合：

- 调试
- UI渲染
- 判断阶段

---

## 7. meta 是什么？

msg, meta = chunk

### meta 本质：

👉 **系统运行信息（不是业务数据）**

例如：

{  
  'langgraph_node': 'model',  
  'langgraph_step': 1,  
}

👉 用途：

- 判断当前阶段（model / tools）
- debug
- tracing
# 拓展：
[[Middleware中间件]]

# create_agent底层原理：
[[状态图的使用]]

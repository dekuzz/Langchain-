---
date: 2026-04-22
aliases:
  - my third note about langchain
  - my forth note about agent
tags:
  - Middleware
  - 预算控制
  - 消息截断
  - 敏感词过滤
  - PII检测
  - langchain入门
---
# 中间件的类型
Langchain可通过装饰器自定义中间件类型

| `@before_agent`    | 在 Agent 执行前执行逻辑 |
| ------------------ | --------------- |
| `@after_agent`     | 在 Agent 执行后执行逻辑 |
| `@before_model`    | 在每次模型调用前执行逻辑    |
| `@after_model`     | 在每次模型收到响应后执行逻辑  |
| `@wrap_model_call` | 控制模型的调用过程       |
| `@wrap_tool_call`  | 控制工具的调用过程       |
| `@dynamic_prompt`  | 动态生成系统提示词       |
| `@hook_config`     |  配置钩子行为         |
# Langchain官方插件：
- HumanInTheLoopMiddleware （）：
	- [[人机交互]]
- @dynamic_prompt
	- [[基础langchain教程/上下文工程]]
- SummarizationMiddleware():
	- [[基础langchain教程/上下文工程]]
### 预算控制 (`@wrap_model_call`的使用)
- **使用场景**：随着对话轮次增加，每次请求携带的对话记录也会越来越长，从而导致请求费用上升。为了控制预算，可以设定在对话轮次超过某个阈值后，切换到低费率模型。这个功能可以通过自定义中间件实现。
``` python
import os  
from dotenv import load_dotenv  
from langchain_openai import ChatOpenAI  
from langchain.agents import create_agent  
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse  
from langchain_core.messages import HumanMessage  
from langgraph.graph import MessagesState  
  
# 加载模型配置  
_ = load_dotenv()  
  
# 低费率模型  
basic_model = ChatOpenAI(  
    api_key=os.getenv("DASHSCOPE_API_KEY"),  
    base_url=os.getenv("DASHSCOPE_BASE_URL"),  
    model="qwen3-coder-plus",  
)  
  
# 高费率模型  
advanced_model = ChatOpenAI(  
    api_key=os.getenv("DASHSCOPE_API_KEY"),  
    base_url=os.getenv("DASHSCOPE_BASE_URL"),  
    model="qwen3-max",  
)

@wrap_model_call  
def dynamic_model_selection(request: ModelRequest, handler) -> ModelResponse:  
    """Choose model based on conversation complexity."""  
    message_count = len(request.state["messages"])  
  
    if message_count > 5:  
        # Use a basic model for longer conversations  
        model = basic_model  
    else:  
        model = advanced_model  
  
    request.model = model  
    print(f"message_count: {message_count}")  
    print(f"model_name: {model.model_name}")  
  
    return handler(request)  
  
agent = create_agent(  
    model=advanced_model,  # Default model  
    middleware=[dynamic_model_selection]  
)
state: MessagesState = {"messages": []}  
items = ['汽车', '飞机', '摩托车', '自行车']  
for idx, i in enumerate(items):  
    print(f"\n=== Round {idx+1} ===")  
    state["messages"] += [HumanMessage(content=f"{i}有几个轮子，请简单回答")]  
    result = agent.invoke(state)  
    state["messages"] = result["messages"]  
    print(f'content: {result["messages"][-1].content}')
```
#### ModelRequest &handler &ModelResponse
- **关系**：在 `@wrap_model_call` 的实现中，`ModelRequest`、`handler` 和 `ModelResponse` 共同构成了**标准中间件控制流**（Middleware/Decorator Pattern）。该架构用于在==执行大模型 API 调用前后，实施精准的拦截、路由或篡改==。
- **拆解**：
	- `ModelRequest`:
		- 本质：请求上下文
		- 组成：
			- `request.state`：当前对话状态（messages）
			- `request.model`：当前要用的模型（可被你改）
			- 可能还有工具信息、配置参数等
	- `handler`：
		- 本质：“继续执行默认模型调用流程的函数”
		- 在中间件控制流程充当角色：
				- request → middleware → handler(request) → model执行 → response
	- `ModelResponse`:
		- 本质：模型执行完之后的返回结果
		- 组成：
			- assistant 的回复内容
			- messages 更新后的状态
			- tool call 结果（如果有）
			- Token 消耗
**完整代码流程实质**：

> [!用户输入
   ↓
ModelRequest（封装当前 messages + model）
   ↓
middleware（dynamic_model_selection）
   ↓
判断 message_count
   ↓
修改 request.model（切换模型）
   ↓
handler(request)（真正执行模型调用）
   ↓
ModelResponse（模型输出结果）
   ↓
返回 agent] Title
> Contents




### 消息截断(`@before_model`的使用)
- **使用场景**：智能体的上下文存在长度限制。一旦超过限制，就需要对上下文进行压缩。在众多方法中，截断是最简单粗暴、易于实现的方法。消息截断功能可以通过 `@before_model` 装饰器实现。
- 我们尝试一种截断策略：在保留最近消息的同时，额外保留第一条消息。在下面的例子中，由于我们在第一条消息中告诉智能体「我是 bob」，因此它记得我是 bob.
``` python
from langchain.messages import RemoveMessage  
from langgraph.graph.message import REMOVE_ALL_MESSAGES  
from langgraph.checkpoint.memory import InMemorySaver  
from langchain.agents import create_agent, AgentState  
from langchain.agents.middleware import before_model  
from langgraph.runtime import Runtime  
from langchain_core.runnables import RunnableConfig  
from typing import Any

@before_model  
def trim_messages(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:  
    """Keep only the last few messages to fit context window."""  
    messages = state["messages"]  
  
    if len(messages) <= 3:  
        return None  # No changes needed  
  
    first_msg = messages[0]  
    recent_messages = messages[-3:] if len(messages) % 2 == 0 else messages[-4:]  
    new_messages = [first_msg] + recent_messages  
  
    return {  
        "messages": [  
            RemoveMessage(id=REMOVE_ALL_MESSAGES),  
            *new_messages  
        ]  
    }  
  
agent = create_agent(  
    basic_model,  
    middleware=[trim_messages],  
    checkpointer=InMemorySaver(),  
)  
  
config: RunnableConfig = {"configurable": {"thread_id": "1"}}  
  
def agent_invoke(agent):  
    agent.invoke({"messages": "hi, my name is bob"}, config)  
    agent.invoke({"messages": "write a short poem about cats"}, config)  
    agent.invoke({"messages": "now do the same but for dogs"}, config)  
    final_response = agent.invoke({"messages": "what's my name?"}, config)  
      
    final_response["messages"][-1].pretty_print()  
  
agent_invoke(agent)
```
#### RemoveMessage
- 本质：一个告诉系统删除某条id消息的开关
- 使用：当RemoveMessage(id=REMOVE_ALL_MESSAGES)时，就相当于告诉系统跳过寻找id消息这个逻辑，直接清空所有消息。
#### `*`在python中的用法
- **打包**：在函数定义中，使用`*args`可以将传入的任意数量的**位置参数**收集到一个元组（tuple）中
``` python
def sum_all(*args):
    return sum(args)

print(sum_all(1, 2, 3, 4)) # 输出 10

```
- **解包**：如果你有一个列表或元组，想将其元素作为独立的参数传给函数，可以在变量前加 `*`
``` python
nums = [1, 2, 3] 
print(*nums) # 等同于 print(1, 2, 3)

```
#### InMemorySaver()
- **本质**：一个**非持久化的“内存快照管理器”**。
		它利用 Python 字典在 **RAM** 中为每个 `thread_id` 开辟独立的**虚拟存档位**。其核心逻辑是将复杂的工作流状态简化为“时间线上的点”，每一步操作都会生成一个**不可变的快照**，从而让 AI 能够像查阅字典一样，根据线程 ID 瞬间回溯历史背景或从暂停的步骤精确恢复执行。
- **使用场景**：
	- **本地开发与快速原型设计**：无需配置数据库，仅用于验证Agent 的逻辑流、状态修剪（State Redacting）和多轮对话逻辑是否正确。
	- **自动化单元测试**：在持续集成（CI）流程中，你需要确保代码逻辑在模拟对话中保持一致，但测试运行完后不需要保留数据。
	- **短生命周期的临时任务**：某些 Agent 任务是一次性的，或者是针对单个会话的。比如：一个网页端的临时客服插件，用户关闭页面后，其对话上下文不再重要，甚至为了隐私安全，**不希望**将其持久化到硬盘。进程结束即刻销毁，自动符合某些敏感数据的即用即弃原则。
	- **教学演示与文档示例**
- **禁忌**：
	- **生产环境**：服务器重启或容器重新部署会导致所有用户对话进度丢失。
	- **横向扩展**：如果在多台服务器（Cluster）上运行，内存是互不相通的，用户请求落到 A 服务器有记录，落到 B 服务器就会变“失忆”。
	- **大数据量**：如果对话线程达到数万个，内存占用会迅速飙升。




当然，这个表现不足以说明截断中间件真的生效了。若这个中间件从未生效，也会有这样的结果。为了证明它真的生效了，我们再次修改截断策略。这次只保留最后两条对话记录。如果智能体不记得我是 bob，说明截断中间件确实起作用了。
``` python
@before_model  
def trim_without_first_message(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:  
    """Keep only the last few messages to fit context window."""  
    messages = state["messages"]  
  
    return {  
        "messages": [  
            RemoveMessage(id=REMOVE_ALL_MESSAGES),  
            *messages[-2:]  
        ]  
    }  
  
agent = create_agent(  
    basic_model,  
    middleware=[trim_without_first_message],  
    checkpointer=InMemorySaver(),  
)  
  
agent_invoke(agent)
```

## 敏感词过滤（`@before_agent`的使用）
- **适用场景**：
	- **护栏**（Guardrails）是智能体提供的一类内容安全能力的统称。大模型本身具备一定的内容风控能力，但很容易被突破。搜索「大模型破甲」就能找到此类教程。智能体可以在模型之外，提供额外的安全保护。这是通过工程上的强制检查实现的。
	- 在 LangGraph 中，护栏可以通过中间件轻松实现。下面我们实现一个简单的护栏：若用户的最新消息中包含某些敏感词，智能体将拒绝回答。
``` python
from typing import Any  
  
from langchain.agents.middleware import before_agent, AgentState  
from langgraph.runtime import Runtime  
  
banned_keywords = ["hack", "exploit", "malware"]  
  
@before_agent(can_jump_to=["end"])  
def content_filter(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:  
    """Deterministic guardrail: Block requests containing banned keywords."""  
    # Get the first user message  
    if not state["messages"]:  
        return None  
  
    last_message = state["messages"][-1]  
    if last_message.type != "human":  
        return None  
  
    content = last_message.content.lower()  
  
    # Check for banned keywords  
    for keyword in banned_keywords:  
        if keyword in content:  
            # Block execution before any processing  
            return {  
                "messages": [{  
                    "role": "assistant",  
                    "content": "I cannot process requests containing inappropriate content. Please rephrase your request."  
                }],  
                "jump_to": "end"  
            }  
  
    return None  
  
agent = create_agent(  
    model=basic_model,  
    middleware=[content_filter],  
)  
  
# This request will be blocked before any processing  
result = agent.invoke({  
    "messages": [{"role": "user", "content": "How do I hack into a database?"}]  
})
for message in result["messages"]:  
    message.pretty_print()
```

## PII 检测(@before_agent的使用)
- **使用场景**：
	- （Personally Identifiable Information）检测可以发现用户输入中的邮箱、IP、地址、银行卡等隐私信息，并做出处置。  
	- 下面的例子来源于生活。我们经常把报错复制给大模型，让它帮忙 debug。但报错中可能包含个人隐私信息。针对这种情况，采用以下两种方法进行处置：  
		1. 拒绝回答问题  
		2. 屏蔽隐私信息
``` python
from textwrap import dedent  
from pydantic import BaseModel, Field  
  
# 可信任的模型，一般是本地模型，为了方便，这里依然使用qwen  
trusted_model = ChatOpenAI(  
    api_key=os.getenv("DASHSCOPE_API_KEY"),  
    base_url=os.getenv("DASHSCOPE_BASE_URL"),  
    model="qwen3-coder-plus",  
)  
  
# 用于格式化智能体输出，若发现敏感信息返回True，没发现返回False  
class PiiCheck(BaseModel):  
    """Structured output indicating whether text contains PII."""  
    is_pii: bool = Field(description="Whether the text contains PII")  
  
def message_with_pii(pii_middleware):  
    agent = create_agent(  
        model=basic_model,  
        middleware=[pii_middleware],  
    )  
  
    # This request will be blocked before any processing  
    result = agent.invoke({  
        "messages": [{  
            "role": "user",  
            "content": dedent(  
                """  
                File "/home/luochang/proj/agent.py", line 53, in my_agent                    agent = create_react_agent(                            ==^^====^^====^^==^                File "/home/luochang/miniconda3/lib/python3.12/site-packages/typing_extensions.py", line 2950, in wrapper                    return arg(*args, **kwargs)                        ==^^====^^====^^==^^                File "/home/luochang/miniconda3/lib/python3.12/site-packages/langgraph/prebuilt/chat_agent_executor.py", line 566, in create_react_agent                    model = cast(BaseChatModel, model).bind_tools(                            ==^^====^^====^^====^^====^^====^^==^                AttributeError: 'RunnableLambda' object has no attribute 'bind_tools'                    ---  
    为啥报错  
                """).strip()  
        }]  
    })  
  
    return result
```
🍉 **处置方式一**：如遇隐私信息，拒绝回复。
``` python
@before_agent(can_jump_to=["end"])  
def content_blocker(state: AgentState,  runtime: Runtime) -> dict[str, Any] | None:  
    """Deterministic guardrail: Block requests containing banned keywords."""  
    # Get the first user message  
    if not state["messages"]:  
        return None  
  
    last_message = state["messages"][-1]  
    if last_message.type != "human":  
        return None  
  
    content = last_message.content.lower()  
    prompt = (  
        "你是一个隐私保护助手。请识别下面文本中涉及个人可识别信息（PII），"  
        "例如：姓名、身份证号、护照号、电话号码、邮箱、住址、银行卡号、社交账号、车牌等。"  
        "特别注意，若代码、文件路径中包含用户名，也应被视为敏感信息。"  
        "若包含敏感信息，请返回{\"is_pii\": True}，否则返回{\"is_pii\": False}。"  
        "请严格以 json 格式返回，并且只输出 json。文本如下：\n\n" + content  
    )  
  
    pii_agent = trusted_model.with_structured_output(PiiCheck)  
    result = pii_agent.invoke(prompt)  
  
    if result.is_pii is True:  
        # Block execution before any processing  
        return {  
            "messages": [{  
                "role": "assistant",  
                "content": "I cannot process requests containing inappropriate content. Please rephrase your request."  
            }],  
            "jump_to": "end"  
        }  
    else:  
        print("No PII found")  
  
    return None
    
result = message_with_pii(pii_middleware=content_blocker)  
  
for message in result["messages"]:  
    message.pretty_print()

```
🏀 **处置方式二**：如遇敏感信息，使用一串  `*****` 号屏蔽隐私信息。
``` python
@before_agent(can_jump_to=["end"])  
def content_filter(state: AgentState,  runtime: Runtime) -> dict[str, Any] | None:  
    """Deterministic guardrail: Block requests containing banned keywords."""  
    # Get the first user message  
    if not state["messages"]:  
        return None  
  
    last_message = state["messages"][-1]  
    if last_message.type != "human":  
        return None  
  
    content = last_message.content.lower()  
    prompt = (  
        "你是一个隐私保护助手。请识别下面文本中涉及个人可识别信息（PII），"  
        "例如：姓名、身份证号、护照号、电话号码、邮箱、住址、银行卡号、社交账号、车牌等。"  
        "特别注意，若代码、文件路径中包含用户名，也应被视为敏感信息。"  
        "若包含敏感信息，请返回{\"is_pii\": True}，否则返回{\"is_pii\": False}。"  
        "请严格以 json 格式返回，并且只输出 json。文本如下：\n\n" + content  
    )  
  
    pii_agent = trusted_model.with_structured_output(PiiCheck)  
    result = pii_agent.invoke(prompt)  
  
    if result.is_pii is True:  
        mask_prompt = (  
            "你是一个隐私保护助手。请将下面文本中的所有个人可识别信息（PII）用星号（*）替换。"  
            "仅替换敏感片段，其他文本保持不变。"  
            "只输出处理后的文本，不要任何解释或额外内容。文本如下：\n\n" + last_message.content  
        )  
        masked_message = basic_model.invoke(mask_prompt)  
        return {  
            "messages": [{  
                "role": "assistant",  
                "content": masked_message.content  
            }]  
        }  
    else:  
        print("No PII found")  
  
    return None
    
result = message_with_pii(pii_middleware=content_filter)  
  
for message in result["messages"]:  
    message.pretty_print()
```

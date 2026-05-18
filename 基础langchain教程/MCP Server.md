---
date: 2026-04-29
aliases:
  - my first note about mcp
  - my sixth note about langchain
tags:
  - 接入mcp服务
  - 封装mcp服务
---
# 封装MCP服务
## 步骤
### 文件组合

- 在mcp_server文件夹中添加要做的mcp服务文件夹,比如get_weather_mcp
- 在文件夹中添加`__init__.py`,`__main__.py` `server.py`
- `server.py`填写主要实现逻辑
- `__main__.py`填写调用方式
- 添加好所有mcp服务后
#### 代码示例：
`server.py`
``` python
from fastmcp import Fastmcp
mcp=Fastmcp(get_weather)

@mcp.tool
def get_weather(city:str)->str:
	"get weather for a certain city"
	return f"{city} is always sunny"

if __name__ == "__main__":  
    mcp.run()
```
`__main__.py`
``` python
from . import server
import asyncio
import os

host=os.getenv('HOST','127.0.0.1')
port=int(os.getenv('port',8000))

def stdio():
	"""stdio entry for the package"""
	asyncio.run(server.mcp.run(transport='stdio'))

def http():
	"""http entry for the package"""
	asyncio.run(server.mcp.run(host=host,port=port,path='/mcp',transport='http'))

def sse():
	"""sse entry for the package"""
	asyncio.run(server.mcp.run(host=host,port=port,transport='sse'))

if __name__ == "__main__":  
	# 写最终要实现的通信方式
    http()

```

### 启动准备

- 在父文件夹(mcp_server)下添加mcp_supervisor.conf（让ai根据mcp服务来写）
- 安装`supervisord`：
``` bash
pip install supervisor
```
### 启动命令

- 启动 `supervisord`：
``` bash
supervisord -c ./mcp_supervisor.conf
```
(使用supervisord来启动多个mcp服务)

- 检查端口状态:
``` bash  
	lsof -i :8000  
	lsof -i :8001  
```


# 接入MCP服务
使用 `MultiServerMCPClient` 接入 MCP Server.
``` python 
import os  
  
from dotenv import load_dotenv  
from langchain_openai import ChatOpenAI  
from langchain_mcp_adapters.client import MultiServerMCPClient    
from langchain.agents import create_agent  
  
# 加载模型配置  
_ = load_dotenv()  
  
# 加载模型  
llm = ChatOpenAI(  
    api_key=os.getenv("DASHSCOPE_API_KEY"),  
    base_url=os.getenv("DASHSCOPE_BASE_URL"),  
    model="qwen3-coder-plus",  
    temperature=0.7,  
)  
  
async def mcp_agent():  
    # 我们用两种方式启动 MCP Server：stdio 和 streamable_http    
    client = MultiServerMCPClient(  
    {
	    "math": { "command": "python",            
	    # Make sure to update to the full absolute path to your            
	    # math_server.py file           
	     "args": ["/path/to/math_server.py"],            
	     "transport": "stdio",
	     },        
	     "weather": {            
	     # Make sure you start your weather server on port 8000            
	     "url": "http://localhost:8000/mcp",           
	      "transport": "http",
	      }    
      })
      
    tools = await client.get_tools()  
    agent = create_agent(  
        llm,  
        tools=tools,  
    )  
  
    return agent  
  
async def use_mcp(messages):  
    agent = await mcp_agent()  
    response = await agent.ainvoke(messages)  
    return response
    
import asyncio  
  
async def main():  
    # 调用天气 MCP    
    messages = {"messages": [{"role": "user", "content": "福州天气怎么样？"}]}  
    response = await use_mcp(messages)    
    print(response["messages"][-1].content)  
if __name__ == "__main__":  
    asyncio.run(main())

```
## `MultiServerMCPClient`
- **功能**：主要用于连接已启动的多个MCP服务
### stdio:
#### **原理**：
``` Markdown
MCP Client
    │
    │ 启动子进程（python server.py）
    ▼
MCP Server（子进程）
    ▲
    │ stdout（返回结果）
    │ stdin（接收请求）
	
```
*stdio 模式本质不是“网络通信”，而是“父进程启动子进程，然后用标准输入输出管道交换 JSON 数据”。*
#### **特点**：
	- client启动进程
	- 没有网络
	- “管道通信”
#### `MultiServerMCPClient`中的连接写法：
``` json	
{  
"command": "python",  
"args": ["xxx.py"],  ___相对路径/绝对路径
"transport": "stdio"  
}
```
- **为什么这样写？
	- 因为stdio连接要做三件事：
	- 1. 启动流程（command+args）
	- 2. 拿到输入输出（stdin/stdout在运行时系统内部发生）
	- 3. 直接读写流（同上）
### streamble_http:
#### **本质**：
``` markdown
Client → HTTP request → MCP Server (FastAPI/Flask-like)
```
#### **特点**:
- server 是独立进程
- client 不负责启动
- 通过 URL 通信
- 类似 REST API
#### `MultiServerMCPClient`中的连接写法：
``` json
{
  "url": "http://localhost:8000/mcp",
  "transport": "http"
}

```
### SSE
#### 本质：
``` markdown
Client ← SSE stream ← MCP Server
Client → HTTP POST → MCP Server
```
	是 **“HTTP + 长连接推送”**
#### 特点：
- 单向流（server → client）
- 基于 HTTP
- 支持实时通知
- 比纯 HTTP 更“长连接化”
#### `MultiServerMCPClient`中的连接写法：
``` json
{
  "url": "http://localhost:8000/mcp",
  "transport": "sse"
}

```
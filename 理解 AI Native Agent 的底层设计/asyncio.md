---
date: 2026-06-11
tags:
  - asyncio
  - async
  - await
  - envent_loop
---
# asyncio、async def 、await 、create_task

``` python
import asyncio
import time

async def say_after(delay, text):
    await asyncio.sleep(delay)
    print(text)

async def main():
    start = time.time()

    task1 = asyncio.create_task(say_after(1, "hello"))
    task2 = asyncio.create_task(say_after(2, "world"))

    await task1
    await task2

    print("耗时:", time.time() - start)

asyncio.run(main())

```

## **在这份代码中的流程**：
asyncio.run(main())——启动事件循环：异步程序的入口
⬇
创建task1
创建task2
⬇
task1 执行到 await asyncio.sleep(1)，暂停  
task2 执行到 await asyncio.sleep(2)，暂停
⬇
1 秒：  
task1 恢复  
打印 hello  
task1 完成  
await task1 结束
⬇
2 秒：  
task2 恢复  
打印 world  
task2 完成  
await task2 结束
⬇
  
最后：  
打印耗时，大约 2 秒

**如果不用create_task**
``` python
async def main():
    start = time.time()

    await say_after(1, "hello")
    await say_after(2, "world")

    print("耗时:", time.time() - start)

```

输出：
``` bash
先等 1 秒，打印 hello
再等 2 秒，打印 world
总耗时约 3 秒

```

## 总结：
`async def` 定义协程，`create_task()` 把协程提交给 `asyncio` 事件循环并发运行，`await` 等待任务结果并在等待时让出控制权，`asyncio.run()` 启动整个异步调度过程。
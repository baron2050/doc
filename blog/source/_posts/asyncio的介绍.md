---
title: asyncio的介绍
tags: 
    - python
    - 异步编程
---

# asyncio的介绍

- **`asyncio.gather`**：
  
  - 用于并行运行多个协程，并等待它们全部完成。
    
  - 例如：
    
    Python
    
    ```python
    import asyncio
    
    async def task1():
       await asyncio.sleep(1)
       return 'Task 1 completed'
    
    async def task2():
       await asyncio.sleep(2)
       return 'Task 2 completed'
    
    async def main():
       results = await asyncio.gather(task1(), task2())
       print(results)
    
    asyncio.run(main())
    ```
    
    AI 生成的代码。仔细查看和使用。 [有关常见问题解答的详细信息](https://www.bing.com/new#faq).
    
- **`asyncio.wait`**：
  
  - 等待一组协程完成，可以选择等待所有协程完成或只等待第一个完成的协程。
    
  - 例如：
    
    Python
    
    ```python
    import asyncio
    
    async def task1():
       await asyncio.sleep(1)
       return 'Task 1 completed'
    
    async def task2():
       await asyncio.sleep(2)
       return 'Task 2 completed'
    
    async def main():
       tasks = [task1(), task2()]
       done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_COMPLETED)
       for task in done:
           print(task.result())
    
    asyncio.run(main())
    ```
    
    AI 生成的代码。仔细查看和使用。 [有关常见问题解答的详细信息](https://www.bing.com/new#faq).
    
- **`asyncio.create_task`**：
  
  - 创建一个任务（Task）来并行运行协程。
    
  - 例如：
    
    Python
    
    ```python
    import asyncio
    
    async def my_coroutine():
       await asyncio.sleep(1)
       print('Coroutine finished')
    
    async def main():
       task = asyncio.create_task(my_coroutine())
       await task
    
    asyncio.run(main())
    ```
    
    AI 生成的代码。仔细查看和使用。 [有关常见问题解答的详细信息](https://www.bing.com/new#faq).
    
- **`asyncio.run`**：
  
  - 运行一个顶层的协程，并管理事件循环的生命周期。
    
  - 例如：
    
    Python
    
    ```python
    import asyncio
    
    async def main():
       print('Hello')
       await asyncio.sleep(1)
       print('World')
    
    asyncio.run(main())
    ```
    
    AI 生成的代码。仔细查看和使用。 [有关常见问题解答的详细信息](https://www.bing.com/new#faq).
    
- **`asyncio.sleep`**：
  
  - 暂停协程的执行一段时间。
    
  - 例如：
    
    Python
    
    ```python
    import asyncio
    
    async def main():
       print('Start sleeping')
       await asyncio.sleep(2)
       print('Wake up')
    
    asyncio.run(main())
    ```
    
    AI 生成的代码。仔细查看和使用。 [有关常见问题解答的详细信息](https://www.bing.com/new#faq).
    
- **`asyncio.ensure_future`**：
  
  `asyncio.ensure_future` 是 Python 的 `asyncio` 库中的一个函数，用于调度协程（coroutine）在事件循环中运行。它的主要作用是将一个协程包装成一个 `Future` 对象，并立即开始执行该协程，即使没有显式地等待它完成。
  
  以下是 `asyncio.ensure_future` 的一些关键点：
  
  1. [**立即执行**：当你调用 `asyncio.ensure_future` 时，协程会立即开始执行，而不需要显式地 `await` 它](https://stackoverflow.com/questions/50960348/python-asyncio-ensure-future)[1](https://stackoverflow.com/questions/50960348/python-asyncio-ensure-future)。
    
  2. [**返回 Future 对象**：它返回一个 `Future` 对象，这个对象代表协程的最终结果。你可以稍后等待这个 `Future` 对象的完成](https://breadcrumbscollector.tech/what-is-asyncios-ensure_future/)[2](https://breadcrumbscollector.tech/what-is-asyncios-ensure_future/)。
    
  3. [**兼容性**：`asyncio.ensure_future` 可以接受任何可等待对象（awaitable），包括协程、任务（Task）和 `Future` 对象](https://stackoverflow.com/questions/50960348/python-asyncio-ensure-future)[1](https://stackoverflow.com/questions/50960348/python-asyncio-ensure-future)。
    
    例如：
    
    Python
    
    ```python
    import asyncio
    
    async def my_coroutine():
     print("Coroutine started")
     await asyncio.sleep(1)
     print("Coroutine finished")
    
    # 使用 ensure_future 调度协程
    future = asyncio.ensure_future(my_coroutine())
    
    # 运行事件循环直到协程完成
    asyncio.get_event_loop().run_until_complete(future)
    ```
    
    AI 生成的代码。仔细查看和使用。 [有关常见问题解答的详细信息](https://www.bing.com/new#faq).
    
    在这个例子中，`my_coroutine` 会立即开始执行，而不需要显式地 `await` 它。
    
- **asyncio.run_coroutine_threadsafe `和` asyncio.call_soon_threadsafe的区别**
  
  `asyncio.run_coroutine_threadsafe` 和 `asyncio.call_soon_threadsafe` 都是用于在不同线程中与事件循环交互的方法，但它们有不同的用途和行为：
  
  ### `asyncio.run_coroutine_threadsafe`
  
  - **用途**：用于从另一个线程中调度协程（coroutine）在事件循环中运行。
    
  - **返回值**：返回一个 `concurrent.futures.Future` 对象，你可以使用这个对象来等待协程的结果。
    
  - **线程安全**：确保协程在事件循环中安全地调度和执行。
    
  - **示例**：
    
    Python
    
    ```python
    import asyncio
    import threading
    
    async def my_coroutine():
        await asyncio.sleep(1)
        print("Coroutine finished")
    
    def run_in_thread(loop):
        asyncio.set_event_loop(loop)
        loop.run_forever()
    
    loop = asyncio.new_event_loop()
    t = threading.Thread(target=run_in_thread, args=(loop,))
    t.start()
    
    future = asyncio.run_coroutine_threadsafe(my_coroutine(), loop)
    result = future.result()  # 等待协程完成并获取结果
    
    loop.call_soon_threadsafe(loop.stop)
    t.join()
    ```
    
    AI 生成的代码。仔细查看和使用。 [有关常见问题解答的详细信息](https://www.bing.com/new#faq).
    
  
  ### `asyncio.call_soon_threadsafe`
  
  - **用途**：用于从另一个线程中调度一个回调函数在事件循环中运行。
    
  - **返回值**：返回一个 `asyncio.Handle` 对象，表示已调度的回调。
    
  - **线程安全**：确保回调函数在事件循环中安全地调度和执行。
    
  - **示例**：
    
    Python
    
    ```python
    import asyncio
    import threading
    
    def my_callback():
        print("Callback executed")
    
    def run_in_thread(loop):
        asyncio.set_event_loop(loop)
        loop.run_forever()
    
    loop = asyncio.new_event_loop()
    t = threading.Thread(target=run_in_thread, args=(loop,))
    t.start()
    
    loop.call_soon_threadsafe(my_callback)
    
    loop.call_soon_threadsafe(loop.stop)
    t.join()
    ```
    
    AI 生成的代码。仔细查看和使用。 [有关常见问题解答的详细信息](https://www.bing.com/new#faq).
    
  
  ### 区别总结
  
  - [**调度对象**：`run_coroutine_threadsafe` 用于调度协程，而 `call_soon_threadsafe` 用于调度普通的回调函数](https://stackoverflow.com/questions/60113143/how-to-properly-use-asyncio-run-coroutine-threadsafe-function)[1](https://stackoverflow.com/questions/60113143/how-to-properly-use-asyncio-run-coroutine-threadsafe-function)[2](https://stackoverflow.com/questions/66817841/difference-between-asyncio-call-soon-and-call-soon-threadsafe-why-is-call-soon)。
  - **返回值**：`run_coroutine_threadsafe` 返回一个 `Future` 对象，可以等待协程的结果；`call_soon_threadsafe` 返回一个 `Handle` 对象，表示已调度的回调。
  - **使用场景**：`run_coroutine_threadsafe` 适用于需要从另一个线程中运行协程的情况，而 `call_soon_threadsafe` 适用于需要从另一个线程中运行简单回调的情况。
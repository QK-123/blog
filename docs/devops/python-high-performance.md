#### Python 高性能编程<Badge type="tip" text="python" />

##### 异步使用

```bash
import asyncio

async def main():
    print("Start")
    await asyncio.sleep(2)  # 模拟 I/O 操作
    print("End")

# 创建并运行事件循环
asyncio.run(main())

```

1. **调用 `asyncio.run(main())`**：创建事件循环并运行 `main()`。
2. **打印 "Start"**：执行到第一条 `print` 语句。
3. **遇到 `await asyncio.sleep(2)`**：暂停当前协程并让事件循环去执行其他任务（此处没有其他任务）。
4. **2 秒后**：`asyncio.sleep(2)` 完成，事件循环恢复 `main()` 的执行。
5. **打印 "End"**：完成整个协程的执行。

##### 多线程使用

```bash
import socket
import threading

def handle_client(client_socket):
    request = client_socket.recv(1024)  # 接收请求数据
    print(f"Received request:\n{request.decode()}")

    # 生成简单的 HTTP 响应
    response = "HTTP/1.1 200 OK\n\nHello, World!"
    client_socket.send(response.encode())  # 发送响应
    client_socket.close()  # 关闭连接

def main():
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  # 创建 TCP 套接字
    server.bind(('0.0.0.0', 8080))  # 绑定到主机和端口
    server.listen(5)  # 开始监听连接
    print("Server is listening on port 8080...")

    while True:
        client_socket, addr = server.accept()  # 接受连接
        print(f"Accepted connection from {addr}")

        # 创建新线程来处理该连接
        client_handler = threading.Thread(target=handle_client, args=(client_socket,))
        client_handler.start()  # 启动线程

if __name__ == "__main__":
    main()
```

##### 最佳实践

###### 1. 明确选择异步或多线程

- **I/O 密集型任务**：优先考虑异步编程，使用 `asyncio` 和异步库（如 `aiohttp`）。
- **CPU 密集型任务**：考虑使用多线程（适合某些 I/O 操作）或多进程（使用 `multiprocessing`）以绕过 GIL。

###### 2. 使用 `asyncio` 的任务管理

- 使用 `asyncio.gather()` 来并发执行多个协程，确保在一个事件循环中管理任务。

###### 3. 避免阻塞操作

- 在异步函数中避免使用阻塞的 I/O 操作。对于阻塞任务，使用 `loop.run_in_executor()` 将其放入线程池或进程池。

###### 4.  **异常处理**

- 在异步函数中适当捕获异常，以防止未处理的异常导致程序崩溃。

###### 5. 使用线程池和进程池

- 对于需要并行执行的 CPU 密集型任务，使用 `concurrent.futures.ThreadPoolExecutor` 或 `ProcessPoolExecutor` 来管理线程和进程。
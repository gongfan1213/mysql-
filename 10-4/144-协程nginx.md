# Python 协程详细讲解

协程是Python中实现异步编程的核心概念，它允许函数在执行过程中暂停和恢复，从而实现高效的并发操作。

## 1. 协程基础概念

### 什么是协程
协程（Coroutine）是一种比线程更轻量级的并发执行单元，可以在单个线程内实现多个任务的交替执行。与线程不同，协程的切换由程序控制，而不是操作系统。

### 协程 vs 线程 vs 进程
- **进程**：操作系统资源分配的基本单位，有独立的内存空间
- **线程**：CPU调度的基本单位，共享进程内存空间
- **协程**：用户态的轻量级线程，切换开销小，由程序控制调度

## 2. 协程的发展历程

### 2.1 生成器实现协程（Python 2.5+）
```python
def simple_coroutine():
    print("协程开始")
    x = yield
    print("接收到数据:", x)

# 使用示例
coro = simple_coroutine()
next(coro)  # 启动协程，执行到第一个yield
coro.send(42)  # 向协程发送数据
```

### 2.2 @asyncio.coroutine 和 yield from（Python 3.4）
```python
import asyncio

@asyncio.coroutine
def old_style_coroutine():
    print("开始")
    yield from asyncio.sleep(1)
    print("结束")

# 使用
loop = asyncio.get_event_loop()
loop.run_until_complete(old_style_coroutine())
```

### 2.3 async/await 语法（Python 3.5+）
```python
import asyncio

async def modern_coroutine():
    print("开始")
    await asyncio.sleep(1)
    print("结束")

# 使用
asyncio.run(modern_coroutine())
```

## 3. 现代协程详解

### 3.1 定义协程函数
使用 `async def` 定义协程函数：
```python
async def hello():
    return "Hello, World!"
```

### 3.2 调用协程
协程函数不会立即执行，而是返回一个协程对象：
```python
async def main():
    # 正确调用方式
    result = await hello()
    print(result)
    
    # 或者使用 asyncio.create_task()
    task = asyncio.create_task(hello())
    result = await task
    print(result)
```

### 3.3 事件循环（Event Loop）
事件循环是协程的调度器：
```python
import asyncio

async def task1():
    await asyncio.sleep(1)
    print("任务1完成")

async def task2():
    await asyncio.sleep(2)
    print("任务2完成")

async def main():
    # 顺序执行
    await task1()
    await task2()
    
    # 并发执行
    await asyncio.gather(task1(), task2())
    
    # 或者
    task1_obj = asyncio.create_task(task1())
    task2_obj = asyncio.create_task(task2())
    await task1_obj
    await task2_obj

asyncio.run(main())
```

## 4. 常用的协程操作

### 4.1 asyncio.sleep()
```python
async def delayed_hello():
    await asyncio.sleep(1)
    print("Hello after 1 second")
```

### 4.2 并发执行多个协程
```python
async def concurrent_tasks():
    # 方法1: asyncio.gather()
    results = await asyncio.gather(
        task1(),
        task2(),
        task3()
    )
    
    # 方法2: asyncio.wait()
    tasks = [task1(), task2(), task3()]
    done, pending = await asyncio.wait(tasks)
    
    # 方法3: asyncio.as_completed()
    for coro in asyncio.as_completed(tasks):
        result = await coro
        print(result)
```

### 4.3 超时控制
```python
async def with_timeout():
    try:
        await asyncio.wait_for(slow_operation(), timeout=2.0)
    except asyncio.TimeoutError:
        print("操作超时")
```

### 4.4 协程间的通信
```python
async def producer(queue):
    for i in range(5):
        await asyncio.sleep(1)
        await queue.put(i)
        print(f"生产: {i}")
    await queue.put(None)  # 结束信号

async def consumer(queue):
    while True:
        item = await queue.get()
        if item is None:
            break
        print(f"消费: {item}")
        queue.task_done()

async def main():
    queue = asyncio.Queue()
    await asyncio.gather(
        producer(queue),
        consumer(queue)
    )

asyncio.run(main())
```

## 5. 实际应用示例

### 5.1 异步HTTP请求
```python
import aiohttp
import asyncio

async def fetch_url(session, url):
    async with session.get(url) as response:
        return await response.text()

async def main():
    urls = [
        'http://httpbin.org/get',
        'http://httpbin.org/delay/1',
        'http://httpbin.org/delay/2'
    ]
    
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_url(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
        
        for url, content in zip(urls, results):
            print(f"{url}: {len(content)} 字符")

asyncio.run(main())
```

### 5.2 异步文件操作
```python
import aiofiles
import asyncio

async def async_write_file():
    async with aiofiles.open('test.txt', 'w') as f:
        await f.write("Hello, Async World!")

async def async_read_file():
    async with aiofiles.open('test.txt', 'r') as f:
        content = await f.read()
        print(content)

async def main():
    await async_write_file()
    await async_read_file()

asyncio.run(main())
```

### 5.3 数据库异步操作
```python
import asyncpg
import asyncio

async def database_operations():
    # 连接数据库
    conn = await asyncpg.connect(
        'postgresql://user:password@localhost/db'
    )
    
    # 执行查询
    result = await conn.fetch('SELECT * FROM users WHERE age > $1', 18)
    
    for record in result:
        print(record['name'], record['age'])
    
    await conn.close()

asyncio.run(database_operations())
```

## 6. 高级协程特性

### 6.1 协程上下文
```python
import contextvars

request_id = contextvars.ContextVar('request_id')

async def handle_request():
    id = request_id.get()
    print(f"处理请求 {id}")
    await asyncio.sleep(1)
    print(f"完成请求 {id}")

async def main():
    # 为不同请求设置不同的上下文
    tasks = []
    for i in range(3):
        # 为每个任务创建新的上下文
        task = asyncio.create_task(handle_request())
        request_id.set(f"req-{i}")
        tasks.append(task)
    
    await asyncio.gather(*tasks)
```

### 6.2 异步生成器
```python
async def async_generator():
    for i in range(5):
        await asyncio.sleep(1)
        yield i

async def consume_async_gen():
    async for item in async_generator():
        print(f"收到: {item}")

asyncio.run(consume_async_gen())
```

### 6.3 异步上下文管理器
```python
class AsyncResource:
    async def __aenter__(self):
        print("获取资源")
        await asyncio.sleep(1)
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        print("释放资源")
        await asyncio.sleep(1)
    
    async def operation(self):
        print("执行操作")
        await asyncio.sleep(1)

async def use_async_context():
    async with AsyncResource() as resource:
        await resource.operation()

asyncio.run(use_async_context())
```

## 7. 错误处理

### 7.1 协程中的异常处理
```python
async def risky_operation():
    await asyncio.sleep(1)
    raise ValueError("出错了!")

async def handle_errors():
    try:
        await risky_operation()
    except ValueError as e:
        print(f"捕获到错误: {e}")
    
    # 在 gather 中处理错误
    tasks = [risky_operation() for _ in range(3)]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    for result in results:
        if isinstance(result, Exception):
            print(f"任务失败: {result}")
        else:
            print(f"任务成功: {result}")
```

## 8. 最佳实践

### 8.1 避免阻塞操作
```python
# 错误做法
async def bad_example():
    import time
    time.sleep(5)  # 这会阻塞整个事件循环

# 正确做法
async def good_example():
    await asyncio.sleep(5)  # 这不会阻塞事件循环
```

### 8.2 合理使用并发
```python
async def efficient_concurrency():
    # 限制并发数量
    semaphore = asyncio.Semaphore(10)  # 最多10个并发
    
    async def limited_task(url):
        async with semaphore:
            return await fetch_url(url)
    
    urls = [f"http://example.com/{i}" for i in range(100)]
    tasks = [limited_task(url) for url in urls]
    await asyncio.gather(*tasks)
```

## 总结

Python协程提供了强大的异步编程能力，主要特点包括：

1. **高效**：在单个线程内实现并发，切换开销小
2. **可读性好**：async/await语法让异步代码看起来像同步代码
3. **灵活**：支持复杂的并发模式和通信机制
4. **生态丰富**：有大量的异步库支持

掌握协程需要理解事件循环、任务调度、异步IO等概念，通过实践可以编写出高效、可维护的异步应用程序。

配置Nginx确实是个大话题，不过别担心，下面我会为你梳理出一个清晰的配置框架和常见场景的配置方法。为了让你对Nginx配置有个快速的整体印象，下面这个表格汇总了核心配置块及其作用。

| 配置块 | 层级关系 | 主要功能 |
| :--- | :--- | :--- |
| **全局块** | 配置文件最外层 | 设置影响Nginx整体运行的指令，如用户、工作进程数、PID路径等。 |
| **events块** | 与全局块、http块平级 | 设置影响Nginx服务器与用户网络连接的参数，如每个进程最大连接数。 |
| **http块** | 包含server块 | 代理、缓存、日志定义等绝大多数功能和第三方模块的配置都在此。 |
| **server块** | http块的内部 | 配置一个虚拟主机的相关参数，一个http块可以有多个server块。 |
| **location块** | server块的内部 | 根据用户请求的URI来匹配和处理特定请求，例如指定文件路径或进行代理。 |

### ⚙️ 核心配置详解

在了解了整体结构后，我们来看看每个部分具体可以如何配置。

1.  **全局块与Events块**
    这部分配置通常保持默认即可，但了解其含义有助于性能调优。
    
    ```nginx
    # 定义运行Nginx的用户和组，出于安全考虑，可以创建专用用户如`www`
    user www www;
    # 工作进程数，建议设置为等于CPU总核心数或设置为`auto`
    worker_processes auto;
    # 错误日志存放路径和日志级别
    error_log /var/log/nginx/error.log warn;
    # 存放Nginx主进程PID的文件路径
    pid /run/nginx.pid;
    
    events {
        # 每个工作进程的最大连接数，直接影响并发处理能力
        worker_connections 1024;
    }
    ```

2.  **HTTP块：配置的基石**
    HTTP块内含有很多重要配置，是配置的主要部分。
    
    ```nginx
    http {
        # 导入MIME类型定义文件，使Nginx能识别不同文件类型
        include       /etc/nginx/mime.types;
        # 默认文件类型，如果MIME文件未匹配，使用此类型
        default_type  application/octet-stream;
    
        # 定义日志格式，`main`是这个格式的名称
        log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';
        # 指定访问日志的存放路径和使用的格式
        access_log  /var/log/nginx/access.log  main;
    
        # 开启高效文件传输模式
        sendfile        on;
        # 仅在sendfile为on时有效，提升网络包传输效率
        # tcp_nopush     on;
    
        # 保持连接超时时间
        keepalive_timeout  65;
    
        # 开启Gzip压缩，有效减少网络传输数据量
        # gzip  on;
    
        # 包含其他配置文件，通常用于管理多个站点的配置
        include /etc/nginx/conf.d/*.conf;
        # 或者包含`servers`目录下的配置
        # include servers/*.conf;
    }
    ```
    
    通过`include`指令，你可以将不同网站的配置分散到单独的文件中，便于管理，这在配置多域名网站时非常有用。

### 🖥️ 配置一个网站（Server块）

在HTTP块内部，你需要使用`server`块来定义一个具体的网站（虚拟主机）。

```nginx
server {
    # 监听端口
    listen 80;
    # 服务器域名，多个用空格隔开，如果没有域名或测试可用_localhost_或下划线`_`
    server_name example.com www.example.com;

    # 网站根目录
    root /usr/share/nginx/html;
    # 默认首页文件，按顺序查找
    index index.html index.htm;

    # 配置具体的请求处理规则
    location / {
        # 尝试直接返回文件，如果未找到则返回404
        try_files $uri $uri/ =404;
    }

    # 配置错误页面
    error_page 404 /404.html;
    location = /404.html {
        # 内部location，指定404页面的根目录
        internal;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        # 内部location，指定50x错误页面的根目录
        internal;
    }
}
```

### 🎯 常见场景配置（Location块）

`location`块是Nginx配置中最灵活的部分，它使用匹配规则来处理不同的请求。

1.  **处理静态资源**
    为图片、CSS、JavaScript等静态资源设置缓存和跨域。
    
    ```nginx
    location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg)$ {
        # 资源文件所在目录，`alias`和`root`有区别，注意用法
        root /data/www/static;
        # 设置浏览器缓存时间
        expires 7d;
        # 添加跨域头，如果前端与资源在不同域名下
        add_header Access-Control-Allow-Origin "*";
    }
    ```

2.  **反向代理**
    将请求转发给后端应用服务器，这是前后端分离项目的常见配置。
    
    ```nginx
    location /api/ {
        # 注意：如果proxy_pass的URL带/，则去除location匹配的`/api/`后转发；
        # 如果不带/，则传递完整路径（包括`/api/`）。
        proxy_pass http://localhost:8080/;
        
        # 设置一些重要的代理头信息，传递客户端真实信息给后端
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    ```

3.  **单页应用（SPA）路由支持**
    对于Vue、React等框架开发的应用，需要将所有非文件请求指向入口HTML。
    
    ```nginx
    location / {
        try_files $uri $uri/ /index.html;
    }
    ```

### 🛠️ 配置实践与维护

- **检查与重启**：每次修改配置文件后，务必先测试语法，再重新加载配置。
  ```bash
  nginx -t              # 测试配置文件语法
  nginx -s reload       # 平滑重启，加载新配置而不中断服务
  ```

- **基础安全**：
  - 避免使用`autoindex on;`，以防目录结构被遍历。
  - 为Nginx创建专用的非登录系统用户来运行。

### 💎 进阶配置

当你熟悉基础配置后，可以探索更强大的功能：

- **配置HTTPS/SSL**：使用`listen 443 ssl`并配置证书路径，同时可以利用`rewrite`将HTTP请求强制跳转到HTTPS。
- **负载均衡**：在HTTP块中使用`upstream`块定义一组后端服务器，然后在`location`中通过`proxy_pass http://my_upstream;`进行调用。

希望这份指南能帮助你上手Nginx配置。如果你能告诉我你具体想用Nginx做什么（例如部署个人博客、代理后端API，还是配置负载均衡），我可以为你提供更针对性的配置片段。

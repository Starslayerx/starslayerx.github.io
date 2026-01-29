+++
date = '2026-01-29T8:00:00+08:00'
draft = false
title = 'Python Decorators: 从入门到实战'
tags = ['Python']
+++

这篇文章将通过一个**“为阿里云 Qwen API 添加缓存”**的例子，带你彻底理解 Python 装饰器。

首先，我们定义一个基础的调用函数：

```python
import time

def call_qwen(messages: list, model: str = "qwen-max", temperature: float = 0.7):
    """调用 Qwen API (模拟)"""
    print(f"正在请求 API (模型: {model})...")
    time.sleep(1) # 模拟网络耗时
    return {"content": "这是 AI 的回复", "usage": 100}
```

---

### 1. 函数是一等公民 (First-Class Citizen)

在 Python 中，函数可以像变量一样被传递和赋值。

```python
# 1. 赋值给变量
run_api = call_qwen

# 2. 作为参数传递
def logger(func, *args, **kwargs):
    print("准备开始调用...")
    return func(*args, **kwargs)

logger(call_qwen, [{"role": "user", "content": "你好"}])
```

---

### 2. 闭包 (Closure)

闭包是指函数内部定义了另一个函数，并且内部函数引用了外部函数的变量。它是实现装饰器的基石。

利用闭包，我们可以创建一个带缓存功能的函数：

```python
def make_cached_qwen():
    cache = {} # 外部函数的变量，会被内部函数“捕获”

    def wrapped(messages, **kwargs):
        # 简单起见，用最后一条消息的内容当 Key
        key = messages[-1]['content']
        if key in cache:
            print(" 命中缓存")
            return cache[key]

        result = call_qwen(messages, **kwargs)
        cache[key] = result
        return result

    return wrapped

# 此时 cached_call 就是一个带有自己 “私有缓存字典” 的函数
cached_call = make_cached_qwen()
```

---

### 3. 高阶函数 (High-Order Function)

如果我们想让缓存逻辑通用化，不只针对 `call_qwen`，我们可以写一个接收函数作为参数的高阶函数：

```python
def cache_wrapper(func):
    cache = {}

    def wrapper(*args, **kwargs):
        # 简单起见，取第一个参数（messages）的最后内容作为 key
        key = args[0][-1]['content']
        if key in cache:
            return cache[key]

        result = func(*args, **kwargs)
        cache[key] = result
        return result

    return wrapper

# 手动包装
call_qwen_with_cache = cache_wrapper(call_qwen)
```

---

### 4. 装饰器语法糖 (Decorator Syntax)

装饰器本质就是上述高阶函数的简化写法。
使用 `@` 符号，我们可以直接增强函数：

```python
@cache_wrapper
def call_qwen(messages, model="qwen-max", **kwargs):
    # ...
    pass
```

这一行 `@cache_wrapper` 等价于：`call_qwen = cache_wrapper(call_qwen)`

---

### 5. 通用装饰器 (wraps)

为了防止装饰器“弄丢”原函数的名称和文档字符串这类元数据，我们需要使用 `functools.wraps`

```python
from functools import wraps

def llm_cache(func):
    cache = {}

    @wraps(func) # 保持原函数的元数据，如 (__name__, __doc__)
    def wrapper(*args, **kwargs):
        key = args[0][-1]['content']
        if key in cache:
            return cache[key]
        return func(*args, **kwargs)

    return wrapper
```

---

### 6. 柯里化 (Currying)

柯里化是将一个多参数函数拆解为一系列单参数函数的过程。
例如：`add(x, y)` 变成 `add(x)(y)`。

```python
def add(x, y):
    return x + y

def add_config(x):
    def real_add(y):
        return x + y
    return real_add

print(add(5, 10)) # 15
print(add_config(5)(10)) # 15
```

**为什么讲这个**？因为“带参数的装饰器”本质上就是柯里化的应用。

---

### 7. 带参装饰器 (Parameterized Decorator)

如果我们需要设置缓存的有效期（TTL），装饰器就需要接收参数：`@llm_cache(ttl=60)`

这会产生一个三层结构：

1. **第一层**：接收配置参数 (如 `ttl`)。
2. **第二层**：接收目标函数 (如 `call_qwen`)。
3. **第三层**：包装逻辑。

```python
def llm_cache(ttl=3600):
    def decorator(func):
        cache = {}

        @wraps(func)
        def wrapper(*args, **kwargs):
            key = args[0][-1]['content']
            now = time.time()

            if key in cache:
                val, timestamp = cache[key]
                if now - timestamp < ttl:
                    print(" 命中有效缓存")
                    return val

            result = func(*args, **kwargs)
            cache[key] = (result, now)
            return result
        return wrapper
    return decorator

# 使用：@llm_cache(ttl=10) 实际上是先执行 llm_cache(10) 得到 decorator
# 然后执行 @decorator
@llm_cache(ttl=10)
def call_qwen(messages, **kwargs):
    return {"content": "..."}
```

---

### 8. 类装饰器 (Class Decorators)

当逻辑非常复杂时，用类来实现装饰器会让代码更整洁。
只要类实现了 `__call__` 方法，它就能作为装饰器。

```python
class LLMCache:
    def __init__(self, ttl=60):
        self.ttl = ttl
        self.cache = {}

    def __call__(self, func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            key = args[0][-1]['content']
            # ... 缓存逻辑 ...
            return func(*args, **kwargs)
        return wrapper

@LLMCache(ttl=300)
def call_qwen(messages, **kwargs):
    return {"content": "..."}
```

**总结**：
装饰器是 Python 的核心利器。
通过它，我们将 API 调用（业务逻辑）与 缓存/日志/耗时统计 彻底解耦，让代码既强大又优雅。


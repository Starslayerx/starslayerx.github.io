+++
date = '2026-01-29T8:00:00+08:00'
draft = false
title = 'Python Decorators Explained'
tags = ['Python']
+++

这篇文章将带你从 0 到 1 彻底理解 Python 装饰器。
为了避免枯燥的理论，将用一个**“调用 LLM API 并实现缓存”**的实战案例贯穿全文。

首先，我们定义一个模拟 LLM 调用的基础函数：

```python
import time

def call_llm_api(message: str, enable_thinking: bool = False) -> str:
    """模拟调用 LLM API，会有一定的网络延迟"""
    # 模拟网络延迟
    time.sleep(0.2)
    return f"LLM_Response(prompt='{message}', thinking={enable_thinking})"
```

---

### 1. 函数是一等公民 (First-Class Citizen)

在 Python 中，函数和整数、字符串一样，地位是平等的：

1. 可以赋值给变量。
2. 可以作为参数传递给另一个函数。
3. 可以作为函数的返回值。

```python
# 原始函数
def ask_llm(message: str, enable_thinking: bool = False) -> str:
    return call_llm_api(message, enable_thinking)

# 1. 赋值：fast_ask 变成了 ask_llm 的别名
fast_ask = ask_llm
print(fast_ask("Hi"))

# 2. 传参：将函数作为数据传递
def run_once(fn, message: str):
    print(f"Executing {fn.__name__}...")
    return fn(message)

print(run_once(ask_llm, "Hello"))
```

---

### 2. 闭包 (Closure)

当一个函数内部定义了另一个函数，且内部函数引用了外部函数的变量时，就形成了**闭包**。
闭包最强大的地方在于它能**捕获并保存状态**。

我们利用闭包来实现一个简单的缓存功能：

```python
def make_cached_asker():
    # 这是一个被“捕获”的自由变量，生命周期会跟随返回的 ask 函数
    cache = {}

    def ask(message: str, enable_thinking: bool = False) -> str:
        key = (message, enable_thinking)
        if key in cache:
            print("[Cache Hit]")
            return cache[key]

        result = call_llm_api(message, enable_thinking)
        cache[key] = result
        return result

    return ask

# 创建一个带记忆的函数实例
ask_llm_cached = make_cached_asker()

print(ask_llm_cached("Tell me a joke")) # 第一次调用，执行 API
print(ask_llm_cached("Tell me a joke")) # 第二次调用，命中缓存
```

这里的 `cache` 字典虽然是在 `make_cached_asker` 中定义的，但因为它被 `ask` 引用了，所以 `make_cached_asker` 执行完后它不会消失。

---

### 3. 高阶函数 (High-Order Function)

上面的例子有一个缺点：`make_cached_asker` 内部硬编码了 `call_llm_api` 的调用。
如果我想给其他函数也加缓存怎么办？

我们可以写一个**接收函数作为参数，并返回一个新函数**的高阶函数：

```python
def cache_wrapper(func):
    cache = {}

    def wrapper(message: str, enable_thinking: bool = False):
        key = (message, enable_thinking)
        if key in cache:
            return cache[key]

        # 这里不再硬编码，而是调用传入的 func
        result = func(message, enable_thinking)
        cache[key] = result
        return result

    return wrapper

# 手动包装
ask_llm_cached = cache_wrapper(ask_llm)
print(ask_llm_cached("Hi"))
```

这个 `cache_wrapper` 就是装饰器的原型。

---

### 4. 装饰器语法糖 (Decorator Syntax)

Python 提供了 `@` 语法糖，让代码更优雅。
下面的写法等价于 `ask_llm = cache_wrapper(ask_llm)`。

```python
@cache_wrapper
def ask_llm(message: str, enable_thinking: bool = False) -> str:
    return call_llm_api(message, enable_thinking)
```

**装饰器的本质**： 接收一个可调用对象（函数/类），并在运行时动态地修改或增强其行为，最后返回一个新的可调用对象。

---

### 5. 通用装饰器 (Generic Decorator)

生产环境的装饰器需要处理两个问题：

1. **参数通用性**：被装饰的函数可能有各种参数，需要用 `*args` 和 `**kwargs` 透传。
2. **元信息保留**：被装饰后，函数的 `__name__` 变成了 `wrapper`，这会影响调试。需要用 `functools.wraps` 修复。

```python
from functools import wraps

def cache_wrapper(func):
    cache = {}

    @wraps(func)  # 魔法：自动复制 func 的元信息（name, docstring 等）给 wrapper
    def wrapper(*args, **kwargs):
        # 简单粗暴地将参数转为 tuple 作为 key
        # 注意：kwargs 需要排序以保证 key 的唯一性
        key = (args, tuple(sorted(kwargs.items())))

        if key in cache:
            return cache[key]

        result = func(*args, **kwargs)
        cache[key] = result
        return result

    return wrapper

@cache_wrapper
def ask_llm(message: str, enable_thinking: bool = False) -> str:
    """调用 OpenAPI LLM"""
    return call_llm_api(message, enable_thinking)

print(f"Function name: {ask_llm.__name__}") # 输出 ask_llm，而不是 wrapper
print(f"Docstring: {ask_llm.__doc__}")
```

---

### 6. 带参装饰器 (Parameterized Decorator)

如果我们希望缓存的 TTL（过期时间）是可配置的，该怎么办？
比如：`@cache_llm(ttl_seconds=10)`。

本质上，这是对函数进行 **“柯里化” (Currying)** 的一种应用。我们通过多层函数调用，分步接收参数：先接收配置参数，再接收目标函数。

这需要我们再套一层函数。结构变成了：**`配置接收函数 -> 装饰器 -> 包装函数`**。

```python
from functools import wraps
import time

def cache_llm(ttl_seconds: int = 30):
    """最外层：接收装饰器参数"""

    def decorator(func):
        """中间层：真正的装饰器，接收目标函数"""
        cache = {}

        @wraps(func)
        def wrapper(*args, **kwargs):
            """最内层：实际执行逻辑"""
            key = (args, tuple(sorted(kwargs.items())))
            now = time.time()

            # 检查缓存是否存在且未过期
            if key in cache:
                value, expire_at = cache[key]
                if now < expire_at:
                    return value

            value = func(*args, **kwargs)
            # 写入缓存：(值, 过期时间戳)
            cache[key] = (value, now + ttl_seconds)
            return value

        return wrapper

    return decorator

# 调用过程：cache_llm(10) 返回 decorator -> @decorator 装饰 ask_llm
@cache_llm(ttl_seconds=10)
def ask_llm(message: str, enable_thinking: bool = False) -> str:
    return call_llm_api(message, enable_thinking)

print(ask_llm("Explain decorators", enable_thinking=True))
print(ask_llm("Explain decorators", enable_thinking=True)) # 命中缓存
```

---

### 7. 类装饰器 (Class-based Decorators)

到了这一步，你会发现带参装饰器的嵌套层级（3层）有点深，阅读起来比较费劲。而且闭包中的 `cache` 变量难以从外部访问或管理（例如想要手动清空缓存）。

这时候，**用类来实现装饰器**会更加清晰。只要一个类实现了 `__call__` 魔术方法，它的实例就可以像函数一样被调用。

我们把上面的 TTL 缓存逻辑重构为一个类：

```python
import time
from functools import update_wrapper

class TTLCache:
    def __init__(self, ttl_seconds: int = 60):
        self.ttl_seconds = ttl_seconds
        self.cache = {} # 状态保存在实例属性中，更加清晰

    def __call__(self, func):
        """
        当使用 @TTLCache(ttl=10) 时，__call__ 会被调用，
        接收被装饰的函数 func。
        """
        def wrapper(*args, **kwargs):
            key = (args, tuple(sorted(kwargs.items())))
            now = time.time()

            if key in self.cache:
                val, expire_at = self.cache[key]
                if now < expire_at:
                    print(f"[ClassCache Hit] key={key}")
                    return val
                else:
                    print(f"[ClassCache Expired] key={key}")

            val = func(*args, **kwargs)
            self.cache[key] = (val, now + self.ttl_seconds)
            return val

        # update_wrapper 等同于 @wraps(func)，用于修复元数据
        return update_wrapper(wrapper, func)

    def clear(self):
        """类装饰器的优势：可以轻松添加管理方法"""
        self.cache.clear()
        print("Cache cleared!")

# 使用类实例作为装饰器
# 1. TTLCache(ttl_seconds=5) 创建一个实例
# 2. 实例(ask_llm) 调用 __call__
@TTLCache(ttl_seconds=5)
def ask_llm_v2(msg: str):
    return call_llm_api(msg)

ask_llm_v2("Hello")      # Miss
ask_llm_v2("Hello")      # Hit
time.sleep(6)
ask_llm_v2("Hello")      # Expired -> Miss

# 进阶技巧：如果我们能访问到装饰器实例，甚至可以直接管理缓存
# 但注意：@语法糖会把 ask_llm_v2 替换为 wrapper 函数，
# 所以直接访问 ask_llm_v2.clear() 是行不通的（除非在 wrapper 上做手脚绑定回去）。
```

**总结：**

- **函数装饰器**：适合逻辑简单、无状态或状态简单的场景。
- **类装饰器**：适合逻辑复杂、需要参数配置、或者需要维护大量内部状态（如复杂的缓存策略、重试机制、速率限制）的场景。

装饰器是 Python 中极具魅力的特性，掌握它能让你写出更加 DRY (Don't Repeat Yourself) 且优雅的代码。

+++
date = '2025-11-11T8:00:00+01:00'
draft = false
title = 'Complete Python Logging Guide'
tags = ['Python']
+++

## Python logging 基础指南

实际项目中，`print()` 只能满足基本的输出要求，而 `logging` 模块提供了更灵活、分级别、可配置的日志系统。

### 核心概念

- Logger 记录器  
   记录器是拿来写日志的东西

  ```Python
  logger = logging.getLogger(__name__)
  logger.info("开始执行任务")
  ```

  接收日志消息，按级别判断是否要输出，并交给 Handler

- Handler 处理器  
   决定日志“去哪里”，有下面常见 Handler
  - `StreamHandler`: 输出到控制台
  - `FileHandler`: 写入文件
  - `RotatingFileHandler`: 自动滚动文件
  - `SMTHandler`: 发邮件
  - `SocketHandler`: 发送到日志服务器

  一个 Logger 可以挂多个 Handler

- Formatter 格式器  
  负责日志的格式

  ```Python
  '%(asctime)s - %(levelname)s - %(name)s - %(message)s'
  ```

  所有 Handler 都可以设置自己的 Formatter，不同输出渠道可以呈现不同格式

- LogRecord 日志记录对象  
  每次调用 `logging.info("hello")` 内部都会生成一个 LogRecord 对象

  LogRecord 是日志系统的“消息载体”，包括全部的元数据，例如：
  - 时间戳
  - 模块名
  - 文件名、行号
  - 日志级别
  - 写入消息 message
  - 线程 ID、进程 ID

- Filter 过滤器  
  Filter 是更细粒度的筛选工具，可以控制某个模块的日志，阻止某些关键字，基于上下文附加标签等。

### 简单使用

```Python
import logging

logging.basicConfig(
  # 输出 INFO 及以上几倍日志
  level=logging.INFO,
  # 时间 - 模块名 - 级别 - 消息内容
  format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

logging.info("程序已启动")
logging.warning("磁盘空间将不足")
logging.error("读取文件失败")
```

使用 `Logger` 对象：在较大的项目中，不会使用基础配置，而是为每个模块创建自己的 logger

```python
import logging

# __name__ 会使日志自动显示当前模块名称，便于定位来源
logger = logging.getLogger(__name__)

logger.debug("调试信息")
logger.info("开始处理任务")
logger.error("发生错误")
```

### 写入日志文件

将日志输出到文件，只需要在 `basicConfig` 中设置 `filename`

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    filename='app.log',
    filemode='a', # 写入模式, a 为追加
    format='%(asctime)s - %(levelname)s - %(message)s'
)

logging.info("日志将写入文件")
```

同时输出到控制台和文件

```Python
import logging

logger = logging.getLogger("app")
logger.setLevel(logging.INFO)

# 控制台
console_handler = logging.StreamHandler()
console_handler.setLevel(logging.INFO)

# 文件
file_handler = logging.FileHandler("app.log")
file_handler.setLevel(logging.INFO)

# 设置格式
formatter = logging.Formatter('%(asctime)s - %(levelname)s - %(message)s')
console_handler.setFormatter(formatter)
file_handler.setFormatter(formatter)

# 添加处理器
logger.addHandler(console_handler)
logger.addHandler(file_handler)

logger.info("同时输出到控制台和日志文件")
```

### 异步 logging

```Python
import logging
import asyncio

logger = logging.getLogger(__name_-)
logging.basicConfig(level=logging.INFO)

async def task():
    logger.info("在异步任务中输出日志")

asyncio.run(task())
```

## Advanced Python Logging: Mastering Configuration & Best Pratices for Production | Python 日志高级使用：掌握配置和生产环境的最佳实践

Python 的 logging 系统提供了监视、调试和维护应用的工具，这篇文章将介绍从基础的配置和高级的实现策略，帮助构建鲁棒性的 logging 解决方案。

### Why Proper Logging Matters 为什么日志很重要

日志的重要性有以下原因:

- 生产环境中的有效调试
- 为应用行为提供洞察
- 促进性能观测
- 帮助追踪安全事件
- 支持合规要求
- 提高维护效率

### Quick Start with Python Logging 快速开始

下面是一个 `logging.basicConfig` 的示例

```python
# Simple python logging example
import logging

# Basic logger in python
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

# Create a logger
logger = logging.getLogger(__name__)

# Logger in python example
logger.info("This is an information message")
logger.warning("This is a warning message")
```

### Getting Started with Python's Logging Module 日志模块基础

#### Basic Setup 基本设置

来开始一个简单的日志设置

```Python
import logging

# Basic configuration
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

# First logger
logger = logging.getLogger(__name__)

# Using the logger
logger.info("Application started")
logger.warning("Watch Out!")
logger.error("Something went wrong")
```

#### Understanding Log Levels 了解日志等级

Python 日志有 5 个标准等级

|  Level  | Numberic Value |    When to Use     |
| :-----: | :------------: | :----------------: |
|  DEBUG  |       10       |    细节调试信息    |
|  INFO   |       20       |    一般操作事件    |
| WARNING |       30       |   意料之外的事件   |
|  ERROR  |       40       |   更加严重的问题   |
| CRITIAL |       50       | 程序可能将无法运行 |

#### Beyond print() Statements 超越打印语句

为什么采用 logging 替代 print 语句呢？

- 事件严重等级分类
- 时间戳信息
- 源信息（文件、代码行数）
- 可配置的输出目的地
- 生产就绪的筛选
- 线程安全

### Configuring Your Logging System 设置你的日志系统

基础日志选项

```Python
logging.basicConfig(
    filename='app.log',
    filemode='w',
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.DEBUG,
    datefmt='%Y-%m-%d %H:%M:%S',
)
```

对于更加复杂的配置

```Python
config = {
    'version': 1,
    'formatters': {
        'detailed': {
            'format': '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        }
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'level': 'INFO',
            'formatter': 'detailed'
        },
        'file': {
            'class': 'logging.FileHandler',
            'filename': 'app.log',
            'level': 'DEBUG',
            'formatter': 'detailed'
        }
    },
    'loggers': {
        'myapp': {
            'handlers': ['console', 'file'],
            'level': 'DEBUG',
            'propagate': True
        }
    }
}

logging.config.dictConfig(config)
```

### Working with Advanced Logging 高级日志技巧

结构化日志

结构化日志提供了一个一致的、机器可读的格式，这对于日志分析和监控十分有必要。

```Python
import json
import logging
from datetime import datetime

class JSONFormatter(logging.Formatter):
    def __init__(self):
        super().__init__()

    def format(self, record):
        # Create base log record
        log_obj = {
            "timestamp": self.formatTime(record, self.detefmt),
            "name": record.name,
            "level": record.levelname,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno
        }

        # Add exception info if present
        if record.exc_info:
            lob_obj["exception"] = self.formatException(record.exc_info)

        # Add custom fields from extra
        is hasattr(record, "extra_fields"):
            log_obj.update(record.extra_fields)

        return json.dumps(log_obj)

# Usage Example
logger = logging.getLogger(__name__)
handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter)
logger.addHandler(handler)

# Log with extra fields
logger.info("User logged in", extra={"extra_fileds": {"user_id": "123", "ip": "192.168.1.1"}})
```

#### Error Management 错误管理

完善的错误日志对于调试生产环境的问题十分重要

```Python
import traceback
import sys
from contextlib import contextmanager

class ErrorLogger:
    def __init__(self, logger):
        self.logger = logger

    @contextmanager
    def error_context(self, operation_name, **context):
        """Context manager for error logging with additional context"""
        try:
            yield
        except Exception as e:
            # Capture the current stack here
            exc_type, exc_value, exc_trackback = sys.exc_info()

            # Format error details
            error_details = {
                "operation": operation_name,
                "error_type": exc_type.__name__,
                "error_message": str(exc_value),
                "context": context,
                "stack_trace": traceback.format_exception(exc_type, exc_value, exc_traceback)
            }

            # Log the error with full context
            self.logge.error(
                f"Error in {operation_name}: {str(exc_value)}",
                extra={"error_details": error_details}
            )

            raise

# Use example
logger = logging.getLogger(__name__)
error_logger = ErrorLogger(logger)

with error_logger.error_context("user_authentication", user_id="123", attempt=2):
    # code might raise an exception
    authenicate_user(user_id)
```

`traceback` 模块是专门用于获取、格式化、打印异常堆栈信息的模块，就是解释器出错的时候打印的那一大串东西

```arduino
Traceback (most recent call last):
  File "app.py", line 10, in <module>
    ...
```

#### Concurrent Logging 并发日志

在多线程进程中，需要确保线程安全

```Python
import threading
import logging
from queue import Queue
from logging.handlers import QueueHandler, QueueListener

def setup_thread_safe_logging():
    """Set up thread-safe logging with a queue"""
    # Create the queue
    log_queue = Queue()

    # Create queue handler and listener
    queue_handler = QueueHandler(log_queue)
    listener = QueueListener(
        log_queue,
        console_handler,
        file_handler,
        respect_handler_level=True,
    )

    # Configure root logger
    root_logger = logging.getLogger()
    root_logger.addHandler(queue_handler)

    # Start the listener in a sperate thread
    listener.start()

    return listener


# Usage
listener = setup_thread_safe_logging()

def worker_function():
    logger = logging.getLogger(__name__)
    logger.info(f"Wroker thread {threading.current_thread().name} starting")
    # Do work...
    logger.info(f"Worker thread {threading.current_thread().name} finished")

# Create and start threads
threads = [
    threading.Thread(target=worker_function)
    for _ in range(3)
]

for thread in threads:
    thread.start()
```

### Logging in Different Environments 不同环境中的日志

不同的应用环境需要不同的 logging 方式。
无论是 web 应用、微服务、或后台任务，每个环境都有独特的 logging 需求和最佳实践。
下面展示如何实现高效的跨部署环境日志系统。

#### Web Application Logging 浏览器应用日志

- **Django Logging Confiugration**

下面是一个 Django logging 配置

```python
# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {process:d} {thread:d} {message}',
            'style': '{',
        },
        'simple': {
            'format': '{levelname} {message}',
            'style': '{',
        },
    },
    'filters': {
        'require_debug_true': {
            '()': 'django.utils.log.RequireDebugTrue',
        },
    },
    'handlers': {
        'console': {
            'level': 'INFO',
            'filters': ['require_debug_true'],
            'class': 'logging.StreamHandler',
            'formatter': 'simple'
        },
        'file': {
            'level': 'ERROR',
            'class': 'logging.FileHandler',
            'filename': 'django-errors.log',
            'formatter': 'verbose'
        },
        'mail_admins': {
            'level': 'ERROR',
            'class': 'django.utils.log.AdminEmailHandler',
            'include_html': True,
        }
    },
    'loggers': {
        'django': {
            'handlers': ['console'],
            'propagate': True,
        },
        'django.request': {
            'handlers': ['file', 'mail_admins'],
            'level': 'ERROR',
            'propagate': False,
        },
        'myapp': {
            'handlers': ['console', 'file'],
            'level': 'INFO',
        }
    }
}
```

- **Flask Logging Setup**

Flask 提供可订制的日志系统

```python
import logging
from logging.handlers import RotatingFileHandler
from flask import Flask, request

app = Flask(__name__)

def setup_logger():
    create_formatter = RotatingFileHadnler(
        'flask_app.log',
        maxBytes=10485760, # 10 MB
        backupCount=10,
    )
    file_handler.setLevel(logging.INFO)
    file_handler.setFormatter(formatter)

    # Add request context
    class RequestFormatter(logging.Formatter):
        def format(self, record):
            record.url = request.url
            record.remote_addr = request.remote_addr
            return super().format(record)

    # Configure app logger
    app.logger.addHandler(file_handler)
    app.logger.setLevel(logging.INFO)

    return app.logger


# Usage in routes
@app.route('/app/endpoint')
def api_endpoint():
    app.logger.info(f"Request recieved from {request.remote_addr}")
    # coder here
    return jsonify({'status': 'success'})
```

- **Fastapi Logging Pratices**

FastAPI 可以通过一些中间件增强 middleware enhancements 来记录日志

```Python
from fastapi import FastAPI, Request
from typing import Callable
import logging
import time

app = FastAPI()

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

logger = logging.getLogger(__name__)

# Middleware for request logging
@app.middleware("http")
async def log_requests(request: Request, call_next: Callable):
    start_time = time.time()
    response = await call_next(request)
    duration = time.time() - start_time

    log_dict = {
        "url": str(request.url),
        "method": request.method,
        "client_ip": request.client.host,
        "duration": f"{duration:.2f}s",
        "status_code": response.status_code,
    }

    logger.info(f"Request processed: {log_dict}")
    return response

# Example endponit with logging
@app.get("/items/{item_id}")
async def read_item(item_id: int):
    logger.info(f"Retrieving item {item_id}")
    # Code here
    return {"item_id": item_id}
```

#### Microservices Logging 微服务日志

对于微服务而言，分布式追踪和关联性 ID 至关重要。

```Python
import logging
import contextvars
from uuid import uuid4

# Create context variable for trace ID
trace_id_var = contextvars.ContextVars('trace_id', default=None)

class TraceIDFilter(logging.Filter):
    def filter(self, record):
        trace_id = trace_id_var.get()
        record.trace_id = trace_id if trace_id else 'no_trace'
        return True

def setup_microservice_logging(service_name):
    logger = logging.getLogger(service_name)

    # Create formatter with trace ID
    formatter = logging.Formatter('%(asctime)s - %(name)s - [%(trace_id)s] - %(levelname)s - %(message)s')

    # Add handlers with trace ID filter
    handler = logging.StreamHandler()
    handler.setFormatter(formatter)
    handler.addFilter(TraceIDFilter)

    logger.addHandler(handler)
    logger.setLevel(logging.INFO)
    return logger

# Usage in microservice
logger = setup_microservice_logging('order_service')

def process_order(order_data):
    # Generate or get trace ID from request
    trace_id_var.set(str(uuid4()))

    logger.info("Starting order processing", extra={
        'order_id': order_data['id'],
        'custom_id': order_data['custom_id']
    })

    logger.info("Order processed successfully")
```

#### Background Task Logging 后台服务日志

对于后台任务而言，需要保持适当的日志处理和轮转

```Python
from logging.handlers import RotatingFileHandler
import logging
import threading
from datetime import datetime

class BackgroundTaskLogger:
    def __init__(self, task_name):
        self.logger = logging.getLogger(f'backgroudn_task.{task_name}')
        self.setup_logging()

    def setup_logging(self):
        # Create logs directory if it doesn't exist
        import os
        os.mkdirs('logs', exist_ok=True)

        # Setup rotating file handler
        handler = RotatingFileHandler(
            filename=f'logs/task_{datetime.now():%Y%m%d}.log',
            maxBytes=5*1024*1024, # 5 MB
            backupCount=5,
        )

        # Create formatter
        formatter = logging.Formatter(
            '%(asctime)s - [%(threadName)s] - %(levelname)s - %(message)s'
        )
        handler.setFormatter(formatter)

        self.logger.addHandler(handler)
        self.logger.setLevel(logging.INFO)

    def log_task_status(self, status, **kwargs):
        """Log task status with additional context"""
        extra = {
            'thread_id': threading.get_ident(),
            'timestamp': datetime.now().iosformat(),
            **kwargs
        }
        self.logger.info(f"Task status: {status}", extra=extra)

# Usage example
def background_job():
    logger = BackgroundTaskLogger('data_processing')
    try:
        logger.log_task_status('started', job_id=123)
        # Do some work ...
        logger.log_task_status('completed', records_processed=1000)
    except Exception as e:
        logger.logger.error(f"Task failed: {str(e)}", exc_info=True)
```

### Common Logging Patterns and Soultions 常见日志模式和解决方案

跟踪请求 ID

```Python
import logging
from contextlib import contextmanager
import threading
import uuid

# Store request ID in thread-local storage
_request_id = threading.local()

class RequestIDFilter(logging.Filter):
    def filter(self, record):
        record.request_id = getattr(_request_id, 'id', 'no_request_id')
        return True

@contextmanager
def request_context(request_id=None):
    """Context manager for request tracking"""
    if request_id is None:
        request_id = str(uuid.uuid4())

    old_id = getattr(_request_id, 'id', None)
    _request_id.id = request_id
    try:
        yield request_id
    finally:
        if old_id is None:
            del _request_id.id
        else:
            _request_id.id = old_id

# Setup logging with request ID
def setup_request_logging():
    logger = logging.getLogger()
    formatter = logging.Formatter(
        '%(asctime)s - [%(request_id)s] - %(levelname)s - %(message)s'
    )

    handler = logging.StreamHandler()
    handler.setFormatter(formatter)
    handler.addFilter(RequestIDFilter())

    logger.addHandler(handler)
    return logger

# Usage example
logger = setup_request_logging()

def process_request(data):
    with request_context() as request_id:
        logger.info("Processing request", extra={
            'data': data,
            'operation': 'process_request'
        })
        # Process the request...
        logger.info("Request processed successfully")
```

用户活动日志

```Python
import logging
from datetime import datetime
from typing import Dict, Any

class UserActivityLogger:
    def __init__(self):
        self.logger = logging.getLogger('user_activity')
        self.setup_logger()

    def setup_logger(self):
        formatter = logging.Formatter(
            '%(asctime)s - %(levelname)s - '
            '[User: %(user_id)s] %(message)s'
        )

        handler = logging.FileHandler('user_activity.log')
        handler.setFormatter(formatter)

        self.logger.addHandler(handler)
        self.logger.setLevel(logging.INFO)

    def log_activity(
        self,
        user_id: str,
        action: str,
        resource: str,
        details: Dict[str, Any] = None
    ):
        """Log user activity with context"""
        activity_data = {
            'timestamp': datetime.utcnow().isoformat(),
            'user_id': user_id,
            'action': action,
            'resource': resource,
            'details': details or {}
        }

        self.logger.info(
            f"User performed {action} on {resource}",
            extra={
                'user_id': user_id,
                'activity_data': activity_data
            }
        )

        return activity_data

# Usage example
activity_logger = UserActivityLogger()

def update_user_profile(user_id: str, profile_data: dict):
    try:
        # Update profile logic here...

        activity_logger.log_activity(
            user_id=user_id,
            action='update_profile',
            resource='user_profile',
            details={
                'updated_fields': list(profile_data.keys()),
                'source_ip': '192.168.1.1'
            }
        )

    except Exception as e:
        activity_logger.logger.error(
            f"Profile update failed for user {user_id}",
            extra={'user_id': user_id, 'error': str(e)},
            exc_info=True
        )
```

### Troubleshooting and Debugging 调试和排错

高效的日志排错要求懂得常见的问题和解决方案。

- **Missing Log Entries** 日志入口缺失

```python
# Common problem: Logs not appearing due to incorrect log level
import logging

# Wrong way
logger = logging.getLogger(__name__)
logger.debug("This won't appear") # No handler and wrong level

# Correct way
def setup_proper_logging():
    logger = logging.getLogger(__name__)

    # Set the base logger level
    logger.setLevel(logging.DEBUG)

    # Create the configure level
    handler = logging.StreamHandler()
    handler.setLevel(logging.DEBUG)

    # Add formatter
    formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    )
    handler.setFormatter(formatter)

    # Add handler to logger
    logger.addHandler(handler)

    return logger
```

- **Performance Bottlenecks** 性能瓶颈

```Python
import logging
import time
from functools import wraps

class PerformanceLoggingHandler(logging.Handler):
    def __init__(self):
        super().__init__()
        self.log_times = []

    def emit(self, record):
        self.log_times.append(time.time())
        if len(self.log_times) > 1000:
            time_diff = self.log_times[-1] - self.log_times[0]
            if time_diff < 1:  # More than 1000 logs per second
                print(f"Warning: High logging rate detected: {len(self.log_times)/time_diff} logs/second")
            self.log_times = self.log_times[-100:]

def log_performance(logger):
    """Decorator to measure and log function performance"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            start_time = time.perf_counter()
            result = func(*args, **kwargs)
            end_time = time.perf_counter()

            execution_time = end_time - start_time
            logger.info(
                f"Function {func.__name__} took {execution_time:.4f} seconds",
                extra={
                    'execution_time': execution_time,
                    'function_name': func.__name__
                }
            )
            return result
        return wrapper
    return decorator
```

### Common Logging Pitfalls and Soultions 常见日志陷阱和修复

- **Configuration Issues** 配置问题

```Python
# Common Mistake 1: Not setting the log level properly
logger = logging.getLogger(__name__)
logger.debug("This won't appera")

# Solution: Configure both logger and handler levels
logger = logging.getLogger(__name_-)
logger.setLevel(logging.DEBUG)
handler = logging.StreamHandler()
handler.setLevel(logging.DEBUG)
fromatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
handler.setFormatter(formattere)
logger.addHandler(handler)


# Common Mistack 2: Incorrect basicConfig usage
logging.basicConfig(levle=logging.INFO) # called after handler was add - won't work
logger.info("message")

# Solution: Configure logging before creating loggers
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name_-)
logger.info("message")
```

- **Memory and Resource Issues** 内存和资源问题

```Python
# Common Mistake 3: Creating handlers in loops
def process_item(item):  # DON'T DO THIS
    logger = logging.getLogger(__name__)
    handler = logging.FileHandler('app.log')  # Memory leak!
    logger.addHandler(handler)
    logger.info(f"Processing {item}")

# Solution: Create handlers outside loops
logger = logging.getLogger(__name__)
handler = logging.FileHandler('app.log')
logger.addHandler(handler)

def process_item(item):
    logger.info(f"Processing {item}")

# Common Mistake 4: Not closing file handlers
handler = logging.FileHandler('app.log')
logger.addHandler(handler)
# ... later
# handler.close()  # Forgotten!

# Solution: Use context manager
from contextlib import contextmanager

@contextmanager
def log_file_handler(filename):
    handler = logging.FileHandler(filename)
    logger.addHandler(handler)
    try:
        yield
    finally:
        handler.close()
        logger.removeHandler(handler)
```

+++
date = '2025-11-11T8:00:00+01:00'
draft = true
title = 'Complete Python Logging Guide'
tags = ['Python']
+++

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

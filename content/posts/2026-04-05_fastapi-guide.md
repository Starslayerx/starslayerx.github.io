+++
date = '2026-03-17T8:00:00+01:00'
draft = true
title = 'Fastapi - A Morden Dev Guide'
categories = ['Blog']
tags = ['Python', 'Fastapi']
+++

## 现代 Fastapi 项目开发实践指南

### 分层结构

这是社区里很常见的一种项目分层方式

```
project_root/
├── app/                        # 核心代码
│   ├── api/                    # 接口层 (Web Layer)
│   │   ├── v1/                 # 接口版本控制
│   │   │   ├── endpoints/      # 具体业务路由 (user.py, auth.py, etc.)
│   │   │   └── api.py          # 汇总所有 v1 路由
│   │   ├── dependencies/       # 依赖注入 (验证、Session 获取等)
│   │   │   └── db.py
│   │   ├── middlewares/        # 中间件 (Trace ID, 日志, 耗时统计)
│   │   │   └── logging.py
│   │   └── exception_handlers.py # 全局异常处理器 (捕获业务异常)
│   │
│   ├── core/                   # 核心配置与公共组件
│   │   ├── config/             # Pydantic Settings 配置管理
│   │   │   └── settings.py
│   │   ├── database/           # 数据库引擎与 Session 工厂
│   │   │   ├── base.py         # SQLAlchemy Base (供 alembic 使用)
│   │   │   └── session.py
│   │   ├── exceptions/         # 自定义业务异常类定义
│   │   │   └── base.py
│   │   └── logging/            # 日志格式化与 ContextVar (Trace ID)
│   │       ├── logger.py
│   │       └── context.py
│   │
│   ├── models/                 # 数据库 ORM 模型 (SQLAlchemy)
│   │   ├── __init__.py         # 统一导入所有模型供 Base 识别
│   │   └── user.py
│   │
│   ├── schemas/                # 数据校验模型 (Pydantic Models)
│   │   ├── base.py             # 基础基类 (含通用字段如 id, created_at)
│   │   └── user.py             # 继承基类的 Request/Response 模型
│   │
│   ├── services/               # 业务逻辑层 (Business Logic)
│   │   └── user.py             # 处理事务边界与核心业务
│   │
│   ├── repositories/           # 数据访问层 (Optional: 纯 CRUD 操作)
│   │   └── user.py
│   │
│   └── main.py                 # FastAPI App 实例化与组装入口
│
├── alembic/                    # 数据库迁移脚本目录
├── tests/                      # 测试用例 (Unit / Integration)
├── .env                        # 环境变量配置文件
├── .gitignore                  # Git 忽略文件
├── alembic.ini                 # Alembic 配置文件
├── main.py                     # Uvicorn 启动脚本入口
├── pyproject.toml              # 项目依赖管理 (uv)
└── README.md                   # 项目说明文档
```

下面逐个介绍各个模块

### 项目入口

首先 `app/main.py` 作为构建器，只负责构建 app 对象，不负责运行。

```Python
# app/main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager


@asynccontextmanager
async def lifespan():
    ...

def create_app() -> Fastapi:
    app = Fastapi(lifespan=lifespan, ...)

    # 注册路由
    app.include_router(router)

    # 注册中间件
    ...

    return app
```

然后外层的 `main.py` 作为启动器，负责导入并运行。

```Python
from app.main import create_app

app = create_app()
```

这样分工明确，并且方便进行测试，因为编写单元测试时，通常需要一个 app 实例，并且不想启动 uvicorn 服务器。

### 环境变量管理

使用 pydantic-settings 管理开发和部署环境下的环境变量

```Python
# core/config/settings.py
from functools import lru_cache
from pydantic import PostgresDsn, SecretStr
from pydantic-settings import BaseSettings, SettingsConfigDict


```

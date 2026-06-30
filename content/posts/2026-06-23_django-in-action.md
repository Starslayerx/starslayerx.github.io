+++
date = '2026-05-16T8:00:00+01:00'
draft = true
title = 'Django In Action'
categories = ['Note']
tags = ['Django', 'Web']
+++

## First Django Site

首先创建项目虚拟环境

```shell
mkdir PorjectName
uv init
uv venv -- python3.14
```

然后创建 django 环境

```shell
django-admin startproject ProjectName .
```

为了方便加载虚拟环境, 我还会使用 direnv, 配置 `.envrc` 如下

```bash
watch_file .python-version
watch_file pyproject.toml
watch_file uv.lock

if [ -f ".venv/bin/activate" ]; then
  source ".venv/bin/activate"
else
  log_error "Missing .venv. Run: uv sync"
fi
```

当进入目录的时候, 自动 source 该虚拟环境, 然后可以迁移运行服务器

```shell
uv run --env-file .env.dev python manage.py migrate
uv run --env-file .env.dev python manage.py runserver [port]
```

默认端口号 8000, 可以手动设置

```shell
complete -f -d -W "runserver createsuperuser test shell dbshell migrate makemigrations loaddata dumpdata" ./manage.py
```

使用上面命令可以为 `./manage.py` 提供智能补全

### First Django App

`app` 是 django 中的基础概念, 类似初始化一样, 创建 `app` 也有专门的命令, django 会自动创建下面结构的骨架:
- `apps.py`: app 配置
- `views.py`: 展示 web 页面内容的代码
- `models.py`: 定义数据库模型的代码
- `tests.py`: 该 app 的单元测试
- `admin.py`: 管理数据库对象的代码, 后台管理
- `migrations/`: 包含数据库迁移的脚本目录

`apps.py` 中包含默认的 app 配置, 会帮助 django 将该目录指定为 app, 一般不需要修改它.
另外四个 Python 文件是桩文件, 其中包含了类文件最常用的 django 模块的导入语句.

`manage.py` 脚本是 Django 管理命令 *management commands* 的入口.
*management command* 是希望使用 Django 在命令行来做的一些事情, 也可以编写自己的命令.

使用 `startapp` 管理命令来创建 app 目录和需要的桩文件:
```shell
uv run python manage.py startapp home
```
该命令是一个静态命令, 你会有任何返回, 创建后项目结构大概这样:

```text
$ tree . -I "__pycache_-"
.
├── DjangoAction
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── home
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── migrations
│   │   └── __init__.py
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── main.py
├── manage.py
├── pyproject.toml
├── README.md
└── uv.lock
```

上面的 home app 是一个 python 模块, 因为有 `__init__.py` 文件.
不幸的是, 这样创建 app 并不高效, 你仍然需要告诉 Django 它的信息.
在 `DjangoAction/DjangoAction` 下面的 `settings.py` 文件定义了 Django 的行为.

`settings.py` 文件有几十个变量配置, 并且可以通过 `django.config.settings` 模块访问.
如果要注册 app, 修改 `INSTALLED_APPS` 配置变量.

Django 开发者可以做出了选择: 让开发者手动注册, 而不是自动发现.
这样项目就不需要区分哪些是 app, 哪些是 python 模块.

Django 框架自带的也有一些 apps, 新的项目会将他们自动注册到 `INSTALLED_APPS` 里面.
为了注册新加的 app, 打开 `xx/xx/settings.py` 然后将 app 名称添加到列表中去.

```python
# Application definition

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'home',
]
```

### First Django View

然后就可以在 `home/views.py` 编写下面路由

```Python
from django.http import HttpRequest, HttpResponse

def credits(request: HttpRequest):
    content = 'Nicky\nYour Name'
    return HttpResponse(content, content_type='text/plain')
```

现在又路由了, 但还需要映射到 URL 上面, 可以通过编辑 `urls.py` 来实现


### Registering a route

当用户访问一个 Django 控制的 URL, web 服务器会携带 HTTP 请求信息，调用 `wsgi.py` 中定义的 WSGI 应用程序。
Django 会使用 `urls.py` 中定义的 URL 模式列表，并尝试寻找匹配项。
如果找到匹配项，就会调用与该 URL 模式相关联的视图。
上面已经在 `home` 应用的 `views.py` 文件中定义了 `credits()` 视图，现在需要为该视图注册一个 URL 路由。

在 `urls.py` 中定义的 URL 映射包含在一个叫 `urlpatterns` 的列表中。
使用 `path` 对象定义一个映射。
当需要 `startproject` 来创建项目的时候，会创建一个最基本的 *bare-bones* `url.py` 文件。

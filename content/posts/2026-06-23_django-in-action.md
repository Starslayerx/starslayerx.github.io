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

`manage.py` 脚本是 Django 管理命令 _management commands_ 的入口.
_management command_ 是希望使用 Django 在命令行来做的一些事情, 也可以编写自己的命令.

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
当需要 `startproject` 来创建项目的时候，会创建一个最基本的 _bare-bones_ `url.py` 文件。
生成的文件会包含 `path` 对象的导入，每个 `path` 对象要求至少两个参数：

- URL 模式 _pattern_
- 映射 _mapping_

这里的 mapping 是对 `credits()` 函数的引用，映射也可以是对其他映射文件的引用，从而能够创建层级结构的映射。

`DjangoAction/DjangoAction/urls.py` 是所有 URL 定义的中心。
所有的 apps 都通过这个文件注册路由，但这很容易让人混淆。
可以重命名模块，而不是都使用 views，例如：将 `from home import views` 修改为 `from home improt views as home_views`。

Django 有一个叫 DjangoAdmin 的管理工具，不要和 `django-admin` 命令混淆，这两者完全无关。
默认 `urlspatterns` 列表包含了一个到 Django Admin 的映射。
在注册 `credits` 视图时，需要将其与现有的映射一起添加。

```Python
# DjangoAction/DjangoAction/urls.py

from django.contrib import admin

from django.urls import path

from home import views as home_views

urlpatterns = [
    path('admin/', admin.site.urls),
    path('credits/', home_views.credits),
]
```

注意 `path` pattern 没有前导斜杠 /, django 会自动添加前导斜杠.
URL `/credits/` 使用 `path('credits/')`, 这是对该函数的引用, 并没有调用它, 因此不要加括号.

Django 会跟踪数据库模型的变化, 并会创建特殊的脚本, 叫做迁移 `migrations`.
运行数据库迁移的管理命令是 `migrate`, 在运行开发服务器之前, 运行 `python manage.py migrate` 来运行未应用的迁移.

```shell
uv run --env-file .env.dev python manage.py migrate
```

运行迁移会产生很多输出, 会出现一些需要迁移的 apps

```text
Apply all migrations: admin, auth, contenttypes, sessions
```

输出会包含 app 的完整名称, 例如:

```text
Applying admin.0001_initial... OK
```

表明脚本 `0001_initial` 迁移成功了.

`migrate` 命令应用的迁移脚本会跨多个模块.

## Templates

django 提供了命令来使用 REPL(Read, Eval, Print, Loop) 环境：

```shell
uv run --env-file .env.dev python manage.py shell
```

### Rendering a Template

首先导入需要的类

```Python
from django.template import Template, Context
```

创建一个模板对象 和 一个上下文对象，然后在模板对象上使用 `.render()` 方法渲染该上下文

```Python
t = Template('Hello {{ name }}')
c = Context({'name': 'World'})
t.render(c)
# 'Hello World'
```

模板语言可以使用 `.` 访问对象、字典、列表和元组的数据。
例如 `{{ person.name }}` 访问 `person` 对象或字典的 `name` 属性。
使用 `fruit.2` 访问 `fruit` 的第三个对象。
点引用还可以调用无参数的可调用对象，例如 `{{ person.show_name }}` 会调用 `person.show_name()` 方法并渲染结果。

除了使用双大括号来替换变量之外，还可以使用模板标签 _Tag_ 来控制最终显示的内容。
最常见的标签是条件块和循环，但也有一些用于直接替换的。

`{% lorem %}` 标签会被渲染成 lorem ipsum 文本，这是一段拉丁文，通常用于印刷行业表示占位文本。

示例如下：

```Python
t = Template('Cicero said: "{% lorem %}", amongst other things')
t.render(Context({}))
# 'Cicero said: "Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.", amongst other things'
```

和其他 tags 一样，`{% lorem %}` 有可选参数，前两个是数量和方法。
方法可以为 `w`, `p` 或 `b`，对应 `words`, `paragraphs` 或 `blocks`。

例如：

```Python
t = Template('{% lorem 5 w %}')
t.render(Context({}))
# lorem ipsum dolor sit amet'
```

### Common tags and filters

#### Conditional blocks

条件语句标签

```Python
{% if first_page %}
<h1>
    Welcome to <i> Django Action</i>
</h1>
{% endif %}
```

如果 `first_page` 是 `0`, `False`, 空容器（列表、集和、字典等），或者不存在于 context 中则不会渲染这块。

块标签通常不关心换行或空格，`{% if %}` 和 `{% endif %}` 标签可以放在同一行。
通常可以在 `<title>` 标签里放一个条件块，根据情况在默认的简短名称上加上页面名称：

```Python
<title>DjangoAction{% if title_suffix %} &mdash; {{ title_suffix }}
{% endif %}</title>
```

这里 `{% if %}` tag 同样支持 Python 的操作符，可以使用 `==`, `!=`, `<`, `>`, `<=`, `>=`, `in`, `not`, `is` 和 `is not`。例如：

```Python
{% if num_drummers > 1 %}
    <b>You only need one drummer!</b>
{% endif %}
```

当比较的变量在 context 中不存在时，条件就会变成 `False`，注意这里不会报错。

要构建更加复制的条件判断快，可以在条件语句加上 `{% elif %}` 和 `{% else %}` 标签。
这些标签的行为和 Python 里的对应标签一样。

#### Looping blocks

假如你有一系列乐器需要显示，这时候可以使用 `{% for %}` tag 来遍历数据。
一种常见的使用方式是用于构建 `<ul>`, `<ol>` 或 `<table>` HTML 结构。
在这些情况下， `{% for %}` 标签都放在外面的结构里。
`{% for %}` 块里面的内容就是需要重复的部分。

下面代码会输出一个 `<ul>` 标签，它的内部通过循环遍历 `instruments` 的值来构建，并渲染出 `<li>` 标签：

```html
<ul>
  {% for instrument in instruments %}
  <li>{{ instrument.name }}: {{ instrument.cost }}</li>
  {% endfor %}
</ul>
```

如果传递给 `{% for %}` 的可迭代对象是空的，那么块里面的内容就什么都不会显示。
这可能导致出现一个空列表或空表格，或者使用循环里可选的 `{% empty %}` 功能。
`{% for %}` 标签中 `{% empty %}` 部分里的内容，只有在被循环的可迭代对象里没有任何东西时才会渲染。

```html
<ul>
  {% for instrument in instruments %}
  <li>{{ instrument.name }}: {{ instrument.cost }}</li>
  {% empty %}
  <li><i>No instruments found</i></li>
  {% endfor %}
</ul>
```

除了在 `{% for %}` 中定义的循环变量外，还有一个隐式变量 `forloop`。
这个对象包括一些属性，记录了循环迭代过程中的状态信息。
其中，`forloop.counter` 属性表示当前是第几次循环（从 1 开始计数）。
布尔值 `forloop.first` 和 `forloop.last` 则分别表示当前是否是循环的第一次或最后一次迭代。
可以把前面例子中的 `<ul>` 列表改成一个句子：

```html
My favorite instruments are: {% for instrument in instruments %} {% if
forloop.first %} and {% endif %} {{ forloop.counter }}. {{instrument.name}} {%
if forloop.last %}. {% else %}, {% endif %} {% endfor %}
```

下面是所有的 forloop 属性：

- `forloop.counter` 迭代计数器，从 1 开始
- `forloop.counter0` 迭代计数器，从 0 开始
- `forloop.revcounter` 剩余的迭代次数，最有一个是 1
- `forloop.revcounter0` 剩余的迭代次数，最有一个是 0
- `forloop.first` 第一次迭代为 `True`
- `forloop.last` 最后一次迭代为 `True`
- `forloop.parentloop` 嵌套循环的情况下，表示外层循环的 `forloop` 对象

#### Comment blocks

Django 模板中有两种注释：inline 内嵌和多行 multiline tag。
Inline comment 使用 `{#  #}` 大括号，而 multiline comment 使用 `{% comment %}` tag。
多行形式对于在调试时，临时注释模板的部分很有用。

```html
<ul>
  <li>Guitar {# cool instrument #}</li>
  <li>Drums</li>
  {% comment "For orchestras only" %}
  <li>Oboe</li>
  <li>Basson</li>
  {% endcomment %}
</ul>
```

#### Verbatim blocks

> verbatim blocks 原样块

在代码中，会有许多 HTML 里有特殊含义的字符，这可能导致 HTML 混乱。
比如，要在网页中编写比较运算符的 Python if 语句，就得用 `&gt;` 和 `&lt;` 这样的字符实体才能正确显示。

这种情况可以使用 `{% verbatim %}` tag 解决，该块会告诉模板引擎保留块内不变。甚至可以嵌套定义。

```html
{% verbatim myblock %} Many JavaScript template use {{}} operators which overlap
with the Django Template Engine. A similar problem exits if you wish to show a
tag like {% verbatim %}. {% endverbatim myblock %}
```

#### Referencing URLs

有很多情况下 HTML 内会出现 URL。可能需要链接到其他页面，引用 CSS 文件，或者展示图片。
在大多数情况下，URL 会指向另一个 Django view 视图或 Django 控制的资产。

为了帮助解决这种问题，Django 提供了一种命名 URL mapping 映射的方法。
回想之前的 `django.urls.path` 对象在 `urls.py` 中将 URL 映射到一个 view 视图。
该对象有一个可选参数，允许用来命名 path。
并且可以在 URL 中通过 `{% url %}` tag 来引用该代码。

```Python
# DjangoAction/DjangoAction/urls.py
from home import views as home_views

urlpatterns = [
    ...,
    path('credits/', home_views.credits, name="credits"),
]
```

通过添加 `name="credits"` 属性，URL 映射可以在代码或模板中，使用字符串 `"credits"` 引用。

```html
<a href="{% url 'credits' %}">Credits Page</a>
```

前面的锚标签的 href 参数会使用 credits 页面的网址来渲染。
如果决定更改 `credits()` 这个视图的网址，只需要在 `urls.py` 文件里改一次，`{% url %}` 标签会自动处理好剩下的。

> ALWAYS NAME YOUR ROUTES

应该总是在代码中使用命名的 URL，否则就像在代码中不使用常量一样。

在本例中，HTML 使用双引号来标记标签的属性，在 `{% url %}` 标签里面就需要使用单引号。

#### Common filters

Tags 有时候用于渲染具体的值，例如 `{% now %}` 或 `{% url %}`。
但大多数都用于控制流，例如 `{% if %}` 和 `{% for %}`。
通常还需要修改数据的展示格式，例如 `datetime` 对象，它存储日期和时间，但需要通过 `strftime()` 的参数去修改展示。
Django 的过滤器是一种通用的，处理各种数据的实现方式。

过滤器在双花括号内的变量中生效，在变量被渲染之前，会先应用过滤器。
在双括号中，使用管道符 pipe operator `|`，过滤器 filter 的输入是变量的内容，而它的输出是对数据执行的一个操作。

两个常用的过滤器有 `| upper` 和 `| lower`，类似 `str.upper()` 和 `str.lower()` 调用。
还有其他一些常用的过滤器：

- `{{ num | pluralize }}`: 如果数量大于 1 则返回 s
- `{{ words | first }}`: 返回列表中第一项，也可以使用 `{{ words.0 }}` 来获取第一项
- `{{ words | last }}`: 返回列表中最后一项，注意模板并不支持负数索引

类似 tags ，filters 也可以通接受参数。可以使用 colon `:` 字符传递参数。
使用 `| join` 过滤器，类似 Python 的 `str.join()` 方法，接收一个列表并构造一个字符串。
这个过滤器需要一个参数来指定连接项目之间的分隔符，最常用的是逗号 comma `,`。
模板 `{{ words | join:"," }}` 类似 Python 中的 `",".join(words)`。

### Using render() in a view

`django.shortcuts.render(request, filename, data)` 函数封装了大部分渲染啊 HTML 需要的功能。
而手动实现需要以下流程：

- 获取模板引擎
- 从文件加载模板
- 创建对象上下文
- 渲染模板
- 返回 `HttpResponse` 对象

`render()` 函数有两个必要的参数：
一个是传入视图的 `HttpRequest` 对象，另一个是要从磁盘加载模板名称。

通过修改 `settings.py` 里面的 `TEMPLATES` 配置来告诉 django 哪里去加载模板, 它由 4 个键构成:

- `BACKEND`: 指明在项目中使用哪个模板引擎, 默认是 Django Template 但也有 Jinja2 支持, 或者使用其他第三方模板
- `DIRS`: django 寻找模板的目录列表
- `APP_DIRS`: 布尔值. 当为 True 时 Django 会在 app 目录下寻找模板目录. 会按照 INSTALLED_APPS 的顺序寻找.
- `OPTIONS`: 嵌套字典, 用来给模板引擎传递配置选项, 通常默认值就已经够用了.

默认情况下, DIRS 属性是空列表, 需要修改成自己放模板的地方.
常见的做法是, 在项目目录下创建一个模板目录, 格式类似下面这样:

```text
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
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── manage.py
└── templates
```

`TEMPLATES` 配置中的 `APP_DIRS` 允许告诉 Django 是否要从 apps 内部找模板.
当 `APP_DIRS` 为 `True` 时, Django 会在 apps 内部寻找一个叫 templates 的目录.
使用那种是一种风格选择, 一般来说将模板目录放在外层会更加简洁, 但如果要开发可复用的 app, 那么一般需要放在内层.

例如这种扁平的配置:

```Python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'APP_DIRS': False,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

Django 在 `settings.py` 顶部声明的配置 `BASE_DIR` 是项目的目录地址.

> APP_DIRS 写 True 会同时扫描根目录和 apps 目录下面的模板目录.
> 如果写 False 则只会扫描项目根目录下面的模板目录.


### Escaping special characters

HTML 使用尖括号 *angle brackets* 作为标签，可尖括号也能当大于号和小于号用。
因此，如果要编写大于号，就要使用 `&gt;`。

模板变量就是数据，这些数据来自各种地方，可能包含 HTML 里的特殊字符。
为了让数据在页面上正常显示，就需要进行转换。
Django 会自动完成这个转换，叫做转义字符 *escaping characters*。

加入在 `templates` 下面有 `expirment.html`

```html
<!DOCTYPE>
<html>
    <head>
    </head>
    <body>
        I love {{ instrument }}.
    </body>
</html>
```

然后不使用 `django.shortcuts.render` 渲染：
```Python
# uv run python manage.py shell
from django.template import Engine, Context

engine = Engine.get_default()
t = engine.get_template('./experiment.html')
data = {'instrument': 'trombone'}
t.render(Context(data))
# '<!DOCTYPE>\n<html>\n    <head>\n    </head>\n    <body>\n        I love trombone.\n    </body>\n</html>\n'

data = {'instrument': 'tuba > baritone'}
t.render(Context(data))
# '<!DOCTYPE>\n<html>\n    <head>\n    </head>\n    <body>\n        I love tuba &gt; baritone.\n    </body>\n</html>\n'
```

可以看到结果为一行，且换行符、比较符 `>` 都转义了。
Django 有很多控制自动转义的方式，`| safe` 过滤器标记为安全，不转义，`| escape` 过滤器则完全相反。
`{% autoescape %}` 块可以用于开关自动转义。

```html
I love the sound a {{ instrument }} makes.
A shiny {{ instrument | safe }} is good.

{% autoescape off %}
Don't put a {{ instrument }} too close to your ear.
{% endautoescape %}
```

```Python
from django.template import Engine, Context

engine = Engine.get_default()
t = engine.get_template('./experiment.html')

data = {'instrument': 'flute'}
t.render(Context(data))
# "I love the sound a flute makes.\nA shiny flute is good.\n\n\nDon't put a flute too close to your ear.\n\n\n"

data = {'instrument': '<b>sax</b>'}
t.render(Context(data))
# "I love the sound a &lt;b&gt;sax&lt;/b&gt; makes.\nA shiny <b>sax</b> is good.\n\n\nDon't put a <b>sax</b> too close to your ear.\n\n\n"
```

如果要在数据中显示 HTML 就可以使用这种方法。

此外，还可以通过改变数据来影响自动转义的行为。
Django 有一个 Python 字符串的子类叫 `SafeString`，通过对字符串调用 `mark_safe()` 来获得。
如果是动态地构建 HTML，可以使用 `format_html()`，该方法也返回一个 `SafeString` 类型。
任何标记为 safe 的字符串都不会被转义。

```Python
from django.utils.safestring import mark_safe

instrument = mark_safe('drum<br/>')
data = {'instrument': 'instrument'}
t.render(Context(data))
```

函数 `format_html()` 工作方式类似 `str.format()`, 使用花括号 braces 作为占位符, 并传参填充 *populate* 字符串.
如果要拼一段 HTML 代码,使用 `format_html()` 能省很多打字功夫.
不然就要自己拼凑, 还得对每一小块使用 `mark_safe()`.

```Python
from django.utils.html import format_html

instrument = 'bass'
instrument = format_html('big <i>{}</i>', instrument)
data = {'instrument': instrument}
t.render(Context(data))
# "I love the sound a big <i>bass</i> makes.\nA shiny big <i>bass</i> is good.\n\n\nDon't put a big <i>bass</i> too close to your ear.\n\n\n"
```

> User Input and the Safety of Strings

当开始处理用户输入时, 需要额外小心.
恶意用户可能故意输入一些数据来破坏网页, 使用任何关闭转义的工具都要小心.

在 `django.utils.html` 里还有更多字符串管理工具, 用来处理 JSON 渲染, 标签处理, 以及应付用户的烦人字符串.

---

转义完全是关于 HTML 的微观层面: 确保数据按照意图渲染.
在宏观层面, 会面临另一个问题: HTML 是重复的.
Django 模板语言包含了一种组合和继承区块的方法, 帮助写更少的 HTML.


### Composing templates out of blocks

现在 HTML 一般有很多移动的部分, 导航栏, 页脚, 侧边栏, 面包屑导航 *breadcrumbs* 等等.
除了有很多组成部分, 网页的结构也可能很复杂, 这样才能在不同平台和浏览器上做到自适应.
管理这种复杂性的很多代码, 在网站的大多数页面里都会重复出现.
Django Template Engine 允许定义可复用的组件, 叫做块 *blocks*.

一个模板块使用成对的 `{% block %}` 和 `{% endblock %}` 标签给页面的某一块区域取个名字.
各个模板页面可以通过继承层次组合在一起，子页面会继承父页面结构。
如果子页面有和父页面同名的块，那么子页面中的块会替换掉父页面中的块。

可以使用 `{% extends %}` tag 来进行继承，子页面会声明自己继承自一个父页面，并且继承父页面的结构。
最常见的用法就定义一个 base page 基础页面，该页面定义了网站的外观和内容。
然后，普通的页面就通过该页面扩展，并且只声明需要覆盖的块。
通常会叫 `base.html`，这个主要的页面包含了所有定义网站外观和感觉的 HTML 样板代码。

这个模板里，有一个叫 content 内容的区块，一开始是空的。
之后网站上的其他页面都会继承这个基础页面，各自再声明一个 content 区块，里面放的就是那个页面的具体内容。

对于更复制的网站，可能会弄好几个不同版本的基础页。
一直常见的模式是，有一个不含导航的主页面，一个包含导航的主页面，和实际的网站页面。

#### Writing a base template file

```html
<!doctype HTML>
<html lang="en">
    <head>
        <title>{% block title %}DjangoAction{% endblock %}</title>
    </head>
    <body>
        {% block content %}
        {% endblock content %}
    </body>
</html>
```

其中 `{% endblock %}` 可以可选的包含和 `{% block %}` tag 一样的名称。
如果不具体指定名称，最近的块会被关闭。
通过在结束标签中包含代码块的名称，在遇到嵌套代码块时，会更加清晰易懂。

如果要创建子页面，使用 `{% extends %}` tag，该 tag 只包含一个参数，即要继承的模板名称。

```html
<!-- news2.html -->
{% extends "base.html" %}

{% block title %}
    {{block.super}}: News
{% endblock %}

{% block content %}
<h1>Django Action News (2)</h1>
<ul>
    {% for item in items %}
        <li>{{ item }}</li>
    {% endfor %}
</ul>
{% endblock content %}
```

`{{ block.super }}` 变量包含父块的内容，如果要对父块进行增加而非替换时，会非常有用。

继承机制很适合创建公共基础文件，并在整个网站页面重复使用。
不过有时候可能只想在页面内部重复使用某一段 HTML，这时候可以在 HTML 中引入其他模板。

#### Including an HTML snippet

如果想在模板中包含其他模板, 可以使用 `{% include %}` 标签, 该标签会加载子模板到现在模板中.
有两种常见的使用方式: 组织模板文件和复用组件.
HTML 有越来越长的趋势, 如果想把这些内容分块组织, 可以通过一系列包含子模板来组成一个页面.
简单的 `{% include %}` 接受一个参数: 模板名称.
下面的例子展示了包含两个子模板的例子.

```html
<!-- Sample template using include -->
{% extends "base.html" %}

{% block content %}
<h1>Nicky's Blog</h1>
{% include "blog1.html" %}
{% include "musicans_list.html" %}
{% endblock content %}
```

`{% include %}` 还有一种使用关键字 `with` 的替代形式, 这种形式可以声明键值对作为子模板的上下文.
`{% include "xx.html" with key1=value2 key2=value2 %}`

考虑这样一种情况, 在网站上使用第三方评论区，例如 Disqus，或者加入了页面浏览器跟踪的 JS 代码。
这两种情况下，都需要加入一些针对当前页面进行参数化的 HTML 代码。
要实现可复用的效果，可以创建一个带变量的子模板，然后在引用 `{% include %}` 标签时，用 with 子句来实例化这个模板。

下面的子模板是分页页脚的一部分。渲染该模板的视图会接收一个指定页码的查询字符串。

```python
{% if prev %} # 1
<a href="?page={{ prev }}">Prev</a>
{% endif %}

{% if next %} # 2
    <a href="?page={{ next }}">Next</a>
{% endif %}
```

现在想象这样一种视图：它根据一个叫 `page` 的查询参数来渲染不同的内容。
子模板的每个实例都包含 “上一页” 和 “下一页” 的链接，分别基于名为 `prev` 和 `next` 的变量。
所有这些都包裹在 `{% if %}` 标签里，每个链接只有在对应变量非空时才会显示。
对于 `{% if %}` 标签来说，空值会被当作 `False` 来处理。

前面的子模板可以通过 `{% include %}` 标签在任何页面里重复使用。
要填上具体的数值时，这个标签会用到 `with` 格式，给 prev 和 next 提供数字。

```Python
<h3> More Content </h3>

{% include "pagination.html" with prev=4 next=6 %}
```

通常 `{% include %}` 的第一个参数是包含子模板的名称。
该标签同样支持使用变量来指定子模板名称。
如果在做一些非常动态的内容，可以从数据上下文 context 来获取子模板名称。

模板是一种强大的工具，可以用类似 Python 的方式来管理 HTML。
这样就不需要写一堆重复代码文件，而是构建可组合的部件。
不用把内容硬编码进去，而是构建包含条件渲染和循环的片段，根据视图传进去的上下文数据来改变页面。
这样写出来的代码更好维护，下一步就是将该能力延伸到视图层：
不再使用硬编码的数据字典，而是从数据库里获取内容。

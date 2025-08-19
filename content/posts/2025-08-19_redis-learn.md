+++
date = '2025-08-19T8:00:00+08:00'
draft = false
title = 'Redis String'
tags = ['Redis', 'Database']
+++
介绍Redis中的字符串键

## 字符串
字符串建是 Redis 最基本的键值对类型, 这种类型的键值对会在数据库中把单独的一个值关联起来, 被关联的键和值可以为文本, 也可以是图片, 视屏, 音频等二进制数据.

- SET: 为字符串键设置值 O(1)
> SET key value

    ```Redis
    SET number "10086"
    > OK

    SET book "Redis in action"
    > OK
    ```

    对于已经存在的 key, 再次赋值会覆盖原值, 若不想覆盖后面添加参数 NX, 相反, 默认 XX 允许覆盖
    ```Redis
    SET key "10086" NX
    > (nil)

    SET key "10086" XX
    > OK
    ```

- GET: 获取字符串键的值 O(1)
> GET key

    ```Redis
    GET number
    > "10086"
    ```

    对于不存在的值, 返回空
    ```Redis
    GET key_new
    > (nil)
    ```

- GETSET: 获取旧值并更新值 O(1)
> GETSET key new\_value

    ```Redis
    GETSET key "123456"
    > "10086"
    ```

### 示例: 缓存
对数据进行缓存是Redis最常见的用法之一, 将数据存储在内存比存储在硬盘要快得多
首先定义缓存
```Python
class Cache:
    def __init__(self, client):
        self.client = client

    def set(self, key, value):
        self.client.set(key, value)

    def get(self, key):
        return self.client.get(key)

    def update(self, new_value, key):
        return self.client.getset(key, new_value)

```

然后缓存文本数据
```Python
client = Redis(decode_responses=True) # 使用文本编码方式打开客户端
cache = Cache(client)

cache.set("web_page", "<html><p>hello world</p></html>")
print(cache.get("web_page"))

print(cache.update("web_page", "<html><p>update<p></html>"))
print(cache.get("web_page"))
```

下面是存储一个二进制图片的缓存示例
```Python
client = Redis() # 二进制编码打开客户端
cache = Cache(client)

image = open("DailyBing.jpg", "rb") # 二进制只读方式打开图片
data = image.read() # 读取文件内容
image.close() # 关闭文件

cache.set("daily_bing.jpg", data) # 将二进制图片缓存到键 daily_bing.jpg 中
print(cache.get("daily_bing.jpg")[:20]) # 读取二进制数据的前20字节
```
> b'\xff\xd8\xff\xe0\x00\x10JFIF\x00\x01\x01\x01\x00\x00\x00\x00\x00\x00'


### 示例: 锁
锁是一种同步机制, 用于保证一种资源任何时候只能被一个进程使用.
一个锁的实现通常有获取 (acquire) 和释放 (relase) 这两种操作.

- 获取操作用于获取资源的独占使用权, 任何时候只能有一个进程取得锁, 此时, 取得锁的进程称为锁的持有者.
- 释放操作用于放弃资源的独占使用权, 一般由持有者调用.

```Python
from redis import Redis

VALUE_OF_LOCK = "locking"
class Lock:
    def __init__(self, client, key):
        self.client = client
        self.key = key
    def acquire(self):
        result = self.client.set(self.key, VALUE_OF_LOCK, nx=True)
        return result is True
    def relase(self):
        return self.client.delete(self.key) == 1

client = Redis(decode_responses=True)
lock = Lock(client, 'test-lock')
print("第一次获取锁:", lock.acquire())

print("第二次获得锁:", lock.acquire())
print("取消锁:", lock.relase())

print("第三次获得锁:", lock.acquire())
```

> 第一次获取锁: True
> 第二次获得锁: False
> 取消锁: True
> 第三次获得锁: True

若要设置锁的时间 `SET key value NX EX time` 这样是原子性语法, 删除操作对应命令是 `DEL key`, 返回0表示 key 不存在,   返回1~N表示删除key的数量.

NX 确保锁只有在没有值时加锁成功, 若有值则返回 None, 通过检查 result 是否为 True 来判断是否获得了锁.


- MSET: 一次为多个字符串键设置值 O(N)
> MSET key value [key value ...]

同 SET 命令, MSET 执行成功后返回 OK, 并且会用新值覆盖旧值. 由于执行多条 SET 命令要客户端和服务端之间多次进行网络通讯, 因此 MSET 能减少程序执行操作的时间


- MGET: 一次获取多个字符串键的值 O(N)
> MGET key [key ...]

- MSETNX: 只在键不存在的情况下, 一次为多个键设置值
> MSETNX key value [key value ...]

若有任意一次键存在值, 则会取消所有操作, 并返回0. 只有所有键都没有值的时候, 执行才成功, 返回1.

### 示例: 存储文章信息
在构建应用程序的时候, 经常会需要批量设计和获取多项信息, 以博客为例:
- 当用户注册博客时, 程序将用户名字、帐号、密码、注册时间等存储起来, 并在登陆时查取这些信息.
- 当编写一篇博客文章时, 就要将博客标题、内容、作者、发表时间存储起来, 并在用户阅读的时候取出这些信息.

通过 MSET、MSETNX、MGET 命令, 可以实现上面提到的这些批量设置和批量获取操作

```Python
from redis import Redis
from datetime import datetime

class Article:
    def __init__(self, client, article_id):
        """根据id创建文章id"""
        self.client = client
        self.id = str(article_id)
        self.title_key = "article::" + self.id + "::title"
        self.content_key = "article::" + self.id + "::content"
        self.author_key = "article::" + self.id + "author"
        self.create_at_key = "article::" + self.id + datetime.now()

    def create(self, title, content, author):
        """创建文章"""
        article_data = {
            self.title_key: title,
            self.content_key: content,
            self.author_key: author,
            self.create_at_key: datetime.now(),
        }
        return self.client.msetnx(article_data)

    def get(self):
        """获取文章信息"""
        result = self.client.mget(
            self.title_key,
            self.content_key,
            self.author_key,
            self.create_at_key,
        )
        return {
            "id": self.id,
            "title": result[0],
            "content": result[1],
            "author": result[2],
            "create_at_key": result[3],
        }

    def update(self, title=None, content=None, author=None):
        """更新文章"""
        article_data = {}
        if title is not None:
            article_data[self.title_key] = title
        if content is not None:
            article_data[self.content_key] = content
        if author is not None:
            article_data[self.author_key] = author
        return self.client.mset(article_data)

client = Redis(decode_responses=True)
article = Article(client, 10086)

# 创建文章
print(article.create("message", "hello world", "sx"))

# 获取文章信息
print(article.get())

# 更新文章作者
print(article.update(author="join"))
```
上面程序使用了多个字符串键存储文章信息: `article::<id>::<attribute>`


- STRLEN: 获取字符串的字节长度 O(1)
> STRLEN key

对于存在的键, 返回字节长度信息. 对于不存在的键, 返回0


- GETRANGE: 获取字符串值指定索引范围上的内容 O(N)
> GETRANGE key start end

```Redis
SET message "hello world"
GETRANG message 0 4
> hello

GETRANGE message -5 -1
> world
```

- SETRANGE: 修改字符串索引范围的值 O(N)
> SETRANGE key index subsitute

```Redis
set message "hello world"
SETRANGE message 6 Redis
> (integer) 11

GET message
> hello Redis
```

当用户给定的新内容比被替换内容长的时候, SETRANGE 会自动扩展被修改的字符串值
```Redis
SETRANGE message 5 ", this is a message"
> (integer) 24

GET message
> "hello, this is a message"
```

当用户给出的索引长度超出被替换字符长度时, 字符串末尾到 index-1 之间部分将使用空字符串填充为0
```Redis
SET greeting "hello"
SETRANGE greeting 10 "hello"
> (integer) 15

GET greeting
> "hello\x00\x00\x00\x00\x00world"
```

### 示例: 给文章存储程序加上文章长度计数功能和文章御览功能给
- 文章长度计数功能: 显示文章长度, 用于估计阅读时长
- 文章预览功能: 显示文章开头一部分内容, 帮助读者快速了解文章

```Python
class Article:
    ...

    def get_content_len(self):
        return self.client.strlen(self.content_key)

    def get_content_perview(self, preview_len):
        start_index = 0
        end_index = preview_len - 1
        return self.client.getrange(self.content, start_index, end_index)
```


- APPEND: 追加新内容到值的末尾
> APPEND key suffix

若用户给定的 key 不存在, 则会先将键执行追加操作


### 示例: 存储日志
很多程序运行的时候会产生日志, 日志记录了程序的运行状态以及执行过的重要操作.
若每条日志存储一个键值对, 则会消耗很多资源, 且分散在数据库中, 需要额外的时间查找日志, 这里将不同日志拼接在同一个值里面.
```Python
from redis import Redis

LOG_SEPERATOR = "\n"

class Log:
    def __init__(self, client, key):
        self.client = client
        self.key = key

    def add(self, new_log):
        new_log += LOG_SEPERATOR
        self.client.append(self.key, new_log)

    def get_all(self):
        all_logs = self.client.get(self.key)
        if all_logs is not None:
            log_list = all_logs.split(LOG_SEPERATOR)
            log_list.remove("") # 删除默认多余的空字符串
            return log_list
        else:
            return []
```


----------

下面介绍使用字符串键存储数字值:

每当存储一个值到字符串键里面的时候, 有下面两种情况
1. C 语言 long long int 类型的整数, 取值范围为 -2^63 ~ 2^63-1 (超出范围会被当成字符串)
2. C 语言 long double 类型的浮点数, 正规数公式为 `value = (-1)^s x (1 + fraction) x 2^(exp-bias)`, s:1位 为符号位, fraction:52位 为尾数[0, 1), exp:11位 指数位, bias:2^10-1=1023 偏移量; 对于次正规数(指数全0)公式为 `value = (-1)^s x fraction x 2^(1-bias)`
    - 最大值: 指数全1, 除了最高位(都为1是无穷大NaN), 尾数全1, `MAX = (1 + (1 - 2^(-52))) x 2^(2046-1023)`
    - 最小正规值: 指数最小正规化=1, 尾数全0, `Normalized_min = 1 x 2^(1-1023)`
    - 次正规数: 当指数全为0表示次正规数, 尾数最小=0...01, `Subnormal_min = 2^(-1022) x 2^(-52)`, 次正规数用来表示比最小正规数还小的正数, 但精度降低


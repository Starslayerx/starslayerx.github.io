+++
date = '2025-08-19T8:00:00+08:00'
draft = true
title = 'Introduct Redis'
tags = ["Redis", "Database"]
+++


# 数据结构与应用

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


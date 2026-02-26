+++
date = '2025-09-08T8:00:00+08:00'
draft = false
title = 'Dealing With Grabage in Python'
categories = ['Blog']
tags = ['Python']
+++

## Grabage Collection In Python

本篇文章介绍 Python 中的 Grabage Collection (GC) 机制介绍

### What's Python Object?

Python 对象中有三样东西: 类型(Type)、值(value)和引用计数(reference count), 当给变量命名时, Python 会自动检测其类型, 值在定义对象时声明, 引用计数是指该对象名称的数量.

首先来看一个类

```Python
class Person:
    def __init__(self, name, unique_id, spouse):
        self.name = name
        self.unique_id = unique_id
        self.spouse = spouse

    def __del__(self):
        print(
            # !r: 调用 repr() 来获取该对象的字符串表达式
            # !s: str()
            # !a: ascii()
            f"Object {self.unique_id!r} is about to be removed from memory. Goodbye!"
        )
```

该 `Person` 类有以下3个属性:

- `name`: 人名
- `unique_id`: 唯一性 id
- `spouse`: 将为 `None` 或者将存储另一个 `Person` 对象

有一个特殊方法 `__del__()`, 这个特殊方法有一定的误导性. 该方法并不像 `__len__()` 与 `len()` 或者 `__iter__()` 与 `iter()` 那样与 `del` 关键字相关联. `__del__()` 特殊方法并不定义当对对象引用时 `del` 会发送什么, 相反, `__del__()` 是一个终结器 finaliser: 它在对象被消毁之前从内存中移除之间被调用.  
因此, `__del__()` 中 `print()` 调用的字符串仅在 Python 即将从内存中移除对象时显示.

- 注意: `del` 方法并不会直接删除该对象, 而是会删除该对象的引用

### Reference Counting 引用计数

在引用计数中, 引用总是被统计并存储在内存中, 如下示例:

```Python
a = 50
b = a
c = 50

print(id(a))
print(id(b))
print(id(c))

print(a is b)
print(c is b)
print(a is c)

# 输出:
134367443832424
134367443832424
134367443832424

True
True
True
```

该示例的 `id` 都是相同的, 即为同一个变量, 此时引用计数为3, 如果使用 `del` 删除 `a` 和 `b`, `c` 仍然会存在, 因此此时引用计数为 1

引用计数

- 优点: 易于实现, 无需手动管理内存 (之所以引用计数, 就是因为py中变量赋值是增加一个引用, 而不是 c++/java 那样去内存中复制一个相同的对象)
- 缺点: 引用计数的对象存储在内存中, 对内存管理不利; 此外, 无法处理循环引用的问题

例如下面这种最简单的循环引用:

```Python
a = []
a.append(a)
print(a)

# 输出
[[...]]
```

该对象 `a` 循环引用其自身, 无法靠引用计数法删除

### Generational Garbage Collection 分代回收

分代垃圾回收是一种基于追踪系统 trace-based 的垃圾回收, 它可以打破循环引用并删除未使用的对象.  
Python 跟踪内存中的每个对象, 程序运行时创建3个列表: 第0代、第1代和第2代.  
新创建的对象被放入第0代列表, 会创建一个待丢弃对象的列表, 检测循环引用. 如果一个对象没有外部引用, 就会被丢弃. 在此过程中, 存活下来的对象被放入第1代列表, 相同的步骤应用于第1代列表. 从第1代列表中存活下来的对象被放入第2代列表. 第2代列表中的对象会一直保留到程序执行结束.

Python 与其他编程语言的垃圾回收对比

垃圾回收在不同的编程语言中工作方式不同, 以下是 Python 的垃圾回收机制与其他常见编程语言的比较:

- Python: 自动, 通常基于内存中对象的引用计数, 当对象的引用计数达到零时, 对象会自动被垃圾回收
- Java: 自动, 当堆内存接近满时(即当老年代堆空间达到一定大小时)或经过一定时间后, 它会垃圾回收不再使用的对象
- JavaScript: 自动, 通常使用标记-清除算法进行自动垃圾回收, 该算法标记可达或正在使用的对象, 并自动清除未标记的对
- C++: 非自动, 必须通过手动分配和释放对象内存来完成垃圾回收
- Rust: 没有垃圾回收(或者说是编译时垃圾回收), Rust 确实实现了一种自动内存管理机制, 它通过一种独特的所有权系统在编译时就确保内存安全, 从而避免了在运行时进行垃圾回收的需要

+++
date = '2025-08-14T8:00:00+08:00'
draft = false
title = 'Python Generics'
+++
本篇文件介绍 Python 中的 泛型(Generics)

### Intro
在没有泛型的情况下, 会遇上一下几个问题:

1. 难以表达意图
    假设你编写了一个函数, 它接受一个列表, 并返回列表中的第一个元素.
    在不使用类型提示的情况下, 这个函数可以处理任何类型的列表, 但我们无法在函数签名中表达"返回的元素的类型与列表中的元素类型相同"这个意图
    ```Python
    def get_first_element(items):
        return items[0]
    ```

2. 丧失类型信息
    如果使用类型提示, 可能会像下面这样写, 但这样会丢失类型信息.
    `list[Any]` 表示可以接收任何类型的列表, 但 `-> Any` 意味着不知道返回的元素类型是什么, 这使得 mypy 这里静态类型检测工具无法追踪类型, 降低了代码的可读性和安全性
    ```Python
    from typing import Any
    def get_first_element(items: list[Any]) -> Any:
        return items[0]

    # 调用时, 类型检查工具无法得知 first_str 的类型
    first_str = get_first_element(["hello", "world"])
    ```

3. 代码重复
    如果为每种可能的类型都编写一个单独的函数, 则会导致代码重复
    ```Python
    def get_first_int(items: List[int]) -> int:
        return items[0]

    def get_first_str(items: List[str]) -> str:
        return items[0]
    ```

通过引入 **类型变量** (TypeVar) 来解决问题, 类型变量就像一个占位符, 代表在未来某时刻会被具体指定的类型
```Python
from typing import TypeVar

T = TypeVar("T")

def get_first_element(items: list[T]) -> T:
    reuturn items[0]

# 现在, 类型检查工具可以正确推断出类型
first_str: str = get_first_element(["hello", "world"])
first_int: int = get_first_element([1, 2, 3])
```
- `T = TypeVar('T')` 定义了一个名为 T 的类型变量, 这里 T 只是一个约定熟成的名字, 也可以使用其他字母
- `items: list[T]` 表示 `items` 是一个列表, 其内部元素类型是 T
- `-> T`: 返回类型也是 T
- 当使用 `["hello", "world"]` 调用函数时, 静态类型检查器会推断出 T 是 str, 返回类型为 str
- 当使用 `[1, 2, 3]` 调用函数时, T 被推断为 int

### Generic Class
除了函数, 泛型也常用于定义泛型类
```Python
from typing import TypeVar, Generic

T = TypeVar("T")

class Box(Generic[T]):
    def __init__(self, items: list[T]):
        self._items = items

    def get(self) -> T:
        return self._items[0]

    def add(self, item: T) -> None:
        self._items.append(item)

# 创建一个存储字符串的 Box
string_box = Box(["apple", "banana"])
item_str = string_box.get() # str
string_box.add("cherry")

# 创建一个存储整数的 Box
int_box = Box([10, 20])
item_int = int_box.get() # int
int_box.add(30)
```
- `TypeVar` 定义类型参数: 相当于一个占位符, 将来由使用者指定具体类型
- `Generic` 定义泛型类或泛型接口: 使这个类在类型检查器眼中变成一个模板


### Advanced Useage
简单介绍一下泛型的一些进阶用法

- 多类型参数
    ```Python
    K = TypeVar("K")
    V = TypeVar("V")

    class Pair(Generic[K, V]):
        def __init__(self, key: K, value: V):
            self.key = key
            self.value = value
    ```
    支持多个类型参数, 类似 `dict[K, V]` 的结构

- Constraints 类型约束
    有时候可能希望泛型只能是某些类型
    ```Python
    from typing import TypeVar

    Number = TypeVar('Number', int, float)
    
    def add(a: Number, b: Number) -> Number:
        return a + b
    ```
    `Number` 只能为 int 或 float, 传入其他类型, 类型检查工具会报错

- Convariant / Contravariant 协变于逆变
    在泛型类型中, 可以控制类型参数的变型关系
    ```Python
    from typing import Generic, TypeVar

    T_co = TypeVar("T_co", convariant=True) # 协变
    T_contra = TypeVar("T_contra", contravariant=True) # 逆变

    class ReadOnlyBox(Generic[T_co]):
        def __init__(self, value: T_co):
            self.value = value

    class Writer(Generic[T_contra]):
        def __init__(self, value, T_contra):
            ...
    ```
    协变: 面向产出(只读), 允许子类替代父类  
    逆变: 面向消费(只写), 允许父类替代子类  
    主要用于接口设计中(读/写分离)  

- 泛型与 Protocol
    `Protocol` 允许定义泛型接口 (duck typing)
    ```Python
    from typing import Protocol
    
    class SupportsLen(Protocol):
        def __len__(self) -> int: ...

    def total_length(items: list[SupportsLen]) -> int:
        return sum(len(x) for x in items)
    ```
    任何实现了`__len__`方法的对象都能接受, 比继承更加灵活

- 泛型在标准库中的使用
    - 集和类: `list[T]`, `dict[K, V]`, `set[T]`
    - 迭代器: `Iterator[T]`, `Iterable[T]`
    - 函数工具: `Callable[[T1, T2], R]`
    - 上下文管理器: `ContextManager[T]`

    ```Python
    from typing import Callable

    F = Callable[[int, int], int]

    def operate(a: int, b: int, func: F) -> int:
        return func(a, b)
    ```


### Wrapping Up
泛型是 Python 类型提示系统中一个非常强大的工具, 它通过类型变量帮助我们编写更加灵活、安全且可维护的代码.

它虽然不会影响程序的运行时行为, 但它为静态类型分析提供了必要的信息, 使得代码意图更加清晰, 并且能在早期发现类型错误.

Python 的泛型是类型提示系统的一部分, 和 C++/Java 的编译期泛型不同, 它的作用主要是: 帮助 IDE 和类型检查工具发现类型错误, 提升代码可读性和可维护性, 提供更精确的 API 类型签名等

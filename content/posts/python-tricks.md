+++
date = '2025-08-05T8:00:00+08:00'
draft = false
title = 'Python Tricks'
+++

### 1. The Self-Replicating Trick
将一个含有空列表的列表乘5, 得到有5个空列表的列表
```python
x = [[]] * 5
x
```
> [[], [], [], [], []]

当使用`.append("x")`方法时, 所有列表都被修改
```python
x[0].append("x")
x
```
> [["x"], ["x"], ["x"], ["x"], ["x"]]

打印其 id 可以看到, id 都相同
```python
for item in x:
    print(id(item))
```
> 4417579584  
> 4417579584  
> 4417579584  
> 4417579584  
> 4417579584  

或者使用`set()`发现 id 唯一
```python
set(id(item) for item in x)
```
> {4417579584}

也就是说, 当使用乘法的时候, 创建了5个内部列表的引用副本

使用反汇编发现, 只创建了两个列表, 并执行乘5
```python
dis.dis("[[]] * 5")

0           0 RESUME                   0     # 用于支持解释器恢复 (py3.11)

1           2 BUILD_LIST               0     # 构造一个空列表[], 压栈
            4 BUILD_LIST               1     # 从栈顶取一个对象, 构造列表[[]]
            6 LOAD_CONST               0 (5) # 加载常量 5
            8 BINARY_OP                5 (*) # 对栈顶两个元素执行乘法
           12 RETURN_VALUE                   # 返回栈顶结果
```

#### The alternative
如果要构造独立列表, 应改用列表推导式

```python
x = [[] for _ in range(5)]
x
```
> [[], [], [], [], []]

```python
for item in x:
    print(id(item))
```
> 4587832384  
> 4587818752  
> 4587831168  
> 4587839168  
> 4587809152  

```python
set(id(item) for item in x)
```
> {4587809152, 4587818752, 4587831168, 4587832384, 4587839168}


### 2. The Teleportation Trick
```python
def add_to_shopping_list(item, shopping_list=[]):
    shopping_list.append(item)
    return shopping_list
```
上面函数为一个空列表中添加一个 item, 期望每次创建一个新的空列表, 并添加一个item

```python
groceries = add_to_shopping_list("Bread")
groceries
```
> ['Bread']


```python
books = add_to_shopping_list("A Brief History of Time")
books
```
> ['Bread', 'A Brief History of Time']

然而, 'Bread' 被传送到 books 里面去了

下面不使用默认参数, 测试一下函数
```python
cakes = []
cakes = add_to_shopping_list("Chorolate Cake", cakes)
cakes
```
> ['Chorolate Cake']

```python
tools = []
tools = add_to_shopping_list("Snapper", tools)
tools
```
> ['Snapper']

当传入一个存在的列表时, 没有发生传送行为

回到函数定义: 默认参数的列表, 在函数定义的时候已经被创建了, 因此每次使用该函数而不传入列表参数的时候, 默认列表`shopping_list`就会被使用, 且 list 是一个可变类型, 因此每次会修改这个列表

使用下面方法, 每次打印出使用列表的 id, 会发现不传入列表参数时的 id 都相同
```python
def add_to_shopping_list(item, shopping_list=[]):
    print(id(shopping_list))
    shopping_list.append(item)
    return shopping_list
```

#### The alternative
这个 bug 在使用可变类型(mutable)作为默认参数的时候都会发生, 应该避免可变数据类型作为默认参数

如果想要默认值参数, 可以考虑使用 None 作为参数默认值
```python
def add_to_shopping_list(item, shopping_list=None):
    if shopping_list is None:
        shopping_list = []
    shopping_list.append(item)
    return shopping_list
```


### The Vanishing Trick
下面是最后一个 trick
```python
doubles = (number * 2 for number in range(10))

4 in doubles
# True

4 in doubles
# False
```
上面结果看起来很矛盾, 4 怎么一会儿在 doubles 中, 一会儿又不再 doubles 中?

再来看一个例子
```python
another_doubles = [number * 2 for number in range(10)]

4 in another_doubles
# True

4 in another_doubles
# True
```
上面这个例子中, 就都是 True

问题出在, 当使用括号()创建 doubles 的时候, 并不是创建了元组 tuple, 而是一个生成器 generator
```python
doubles = (number ** 2 for number in range(10))
doubles
```
> <generator object <genexpr> at 0x111718e10>

生成器并不会包含所有的值, 而是在使用的时候生成每个值  
例如, 调用 next() 会返回下一个值
```python
next(doubles)
# 0

next(doubles)
# 1

next(doubles)
# 2

next(doubles)
# 4

...

next(doubles)
# 18

next(doubles)
```
> ---------------------------------------------------------------------------  
> StopIteration                             Traceback (most recent call last)  

生成器是一次型的数据结构, 当生成下一个数据的时候, 之前的数据不会被保存, 也就是只能遍历数据一次

一但遍历完成, 就会报`StopIteration`的错误, 所以当运行`4 in doubles`的时候, 先得到0, 为 False, 生成器会继续遍历下一个, 直到得到4, 当再次调用的时候, 下一个返回6, 直到遍历结束也无法得到到4

同样的行为在迭代器 iterator 上也一样
```python
numbers = [2, 4, 6, 8]
numbers_rev = reversed(numbers)
numbers_rev
# <list_reverseiterator object at 0x......>

4 in numbers_rev
# True

4 in numbers_rev
# False
```

#### The alternative
如果使用生成器, 要知道只能遍历每个元素一次, 如果要获得一个有所有元素的数据结构, 应该使用元素 tuple 或列表 list
```python
doubles = (number * 2 for number in range(10)) # generator
more_doubles = tuple(number * 2 for number in range(10)) # tuple

4 in more_doubles
# True

4 in more_doubles
# True
```

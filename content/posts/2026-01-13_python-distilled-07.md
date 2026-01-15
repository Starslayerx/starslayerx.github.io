+++
date = '2026-01-13T8:00:00+08:00'
draft = true
title = 'Python Tricks Part 7: Classes and Object-Oriented Programming'
tags = ['Python']
+++

使用 `vars()` 方法返回对象的属性和值的字典对象，不带参数时，返回当前局部作用域的变量字典

```Python
vars()  # 相当于 locals()

class Person:
    name: str
    age: int

    def __init__(self, name, age):
        self.name = name
        self.age = age
        self.city = "Beijing"
p = Person("Alice", 25)
print(vars(p))
```

## Attribute Access

一个示例只有三种基础的方法：getting, setting 和 deleting 属性

```Python
class Attribute:
    owner: str
    blance: float

    def __init__(self, owner: str, balance: float):
        self.owner = owner
        self.balance = balance

    def __repr__(self):
        return f"Account({self.owner!r}, {self.balance!r})"

    def deposite(self, amount: float):
        self.balance += amount

    def withdraw(self, amount: float):
        self.balance -= amount

    def inquiry(self) -> float:
        return self.balance
```

例如

```Python
a = Account("Guido", 1000.0)
a.owner         # get
a.balance = 75  # set
del a.balance   # delete
```

Python 中的一切都是一个动态过程，几乎没有什么限制。
例如，可以给已创建的对象添加新属性：

```Python
a = Account("Guido", 1000.0)
a.creation_date = "2019-02-14"
a.nickname = "Fromer BDFL"
```

有时候不适用点 `.` 操作符来执行任务，而是通过将属性名传递给 `getattr()`, `setattr()` 和 `delattr()` 函数来实现。
`hasattr()` 函数允许你测试一个已存在的属性：

```Python
a = Account("Guido", 1000.0)
getattr(a, "owner")
setattr(a, "balance", 750.0)
delattr(a, "balance")
hasattr(a, "balance")
# False
getattr(a, "withdraw")(100)  # Method Call
# a = Account("Guido", 650.0)
```

`getattr()` 函数可以携带一个默认值，如果想要查看一个可能不存在的属性，可以这样实现：

```Python
a = Account("Guido", 1000.0)
getattr(s, "balance", "unknown")
# 1000.0
getattr(s, "crieation_date", "unknown")
# unknown
```

当向访问属性一样访问方法的时候，可以获取一个对象叫做 bound method 绑定方法。

```Python
a = Account("Guido", 1000.0)
w = a.withdraw
# “<bound method Account.withdraw of Account('Guido', 1000.0)>
```

绑定方法是一个对象，即包含示例 self，又包含实现该方法的函数。
当使用圆括号 parentheses 和参数调用 bound method 的时候，它会执行该方法，并将附加的实例作为第一个参数。

## Scoping Rules

尽管类为方法定义了一个独立的命名空间，但该命名空间并不作为解析方法内部所用的作用域。
因此，在实现类时，对属性和方法的引用必须完全限定。
例如，在方法中，总是通过 `self.` 来引用类属性。
缺少类级别作用域是 Python 与 C++ 或 Java 的一个不同之处，如果使用这些语言，`this` 指针就像 `self` 参数一样。
但在 Python 中需要显示地去调用。

## Inheritance

继承 inheritance 是一个创建新类的机制，该类专门修改或优化现有类的行为。
原始的类被称为 base class, superclass 或 parent class，新类被称为派生类 derived class, child class 或 subtype。
当一个类通过继承创建时，它会继承基类定义的属性。
但派生类可以重新定义其中任何属性，并添加新属性。

继承通过在类声明中以逗号分割的基类名称列表来指定。
若未指明基类，则会隐式继承 object 类，object 是 Python 对象的根类，它提供了一些常用方法的默认实现。

有时派生类型重新定义了某个方法，如果仍然想要调用基类方法，使用 `super()`：

```Python
class EvilAccount(Account):
    def inquiry(self):
        if random.randint(0, 4) == 1:
            return 1.10 * super().inquiry
        else:
            return super().inquiry
```

此外，类可以可以添加额外的属性：

```Python
class EvilAccount(Account):
    def __init__(self, owner, balance, factor):
        super().__init__(owner, balance)
        self.factor = factor
    def inquiry(self):
        if random.randint(0, 4) == 1:
            return self.factor * super().inquiry()
        else:
            return super().inquiry()
```

添加属性时一个棘手的问题在于处理已有的 `__init__()` 方法。
本例中，定义了一个新版本的 `__init__()` 方法，其中加入了额外的实例变量 `factor`。
然而，当重定义 `__init__()` 时，子类需要通过 `super().__init__()` 来初始化父类。
如果忘记这一步，将导致对象仅完成部分初始化，进而引发程序崩溃。
由于父类的初始化需要额外参数，这些参数仍需传递给子类的 `__init__()` 方法。

继承还可能通过下面方法导致代码出问题：

```Python
class Account:
    def __init__(self, owner, balance):
        self.owner = owner
        self.balance = balance

    def __repr__(self):
        return f"Account({self.owner!r}, {self.balance!r})"
```

该方法的目的是辅助调试输出信息，但这里缺硬编码了 Account，会导致继承对象调试时显示不符预期：

```Python
class EvilAccount(Account):
    pass

a = EvilAccount("Eval", 10.0)
a
# Account('Eva', 10)
type(a)
# <class 'EvilaAccount'>
```

解决方案是使用 `type()`：

```Python
class Account:
    ...
    def __repr__(self):
        return f"{type(self).__name}({self.owner!r}, {self.balance!r})"
```

继承建立在类型系统中建立了一种关系，使得任何子类都能通过类型检查：

```Python
a = EvilAccount("Eva", 10)
isinstance(a, Account)
# True
```

这就是所谓的 “是一个” 关系，有时 “是一个” 继承关系被用来定义对象类型的本体论或分类体系。

例如：

```Python
class Food:
    pass

class Sandwith(Food):
    pass

class RoastBeef(Sandwith):
    pass

class GrilledChess(Sandwith):
    pass

class Taco(Food):
    pass
```

在实践中，这种组织可能会相当困难且充满风险。

比如要添加一个 HotDog 类型，那是应该属于 Sandwith 还是 Taco，或者两者都是？

```Python
class HotDog(Sandwith, Taco):
    pass
```

## Avoiding Inheritance via Composition

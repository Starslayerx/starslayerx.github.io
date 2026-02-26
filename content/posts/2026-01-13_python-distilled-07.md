+++
date = '2026-01-13T8:00:00+08:00'
draft = false
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

关于继承的一个警示性用法即所谓的 implementation inheritance 实现继承。
假设你想实现一个栈数据结构，一个快速的方法是从列表继承并添加一个新方法：

```Python
class Stack(list):
    def push(self, item):
        self.append(item)

s = Stack()
s.push(1)
s.push(2)
s.push(3)
s.pop()  # 1
s.pop()  # 2
```

这就是 implementation inheritance，但这样会让用户感到困惑：为什么这个栈有个 `sort()` 方法？

一种通常更好的实现是 compoistion 组合。
并不是建立一个栈并直接继承列表，而是建立一个单独的栈类型，然后将列表作为其中一个元素添加进去。

```Python
class Stack:
    def __init__(self):
        self._items = list()

    def push(self, item);
        self._tiems.append(item)

    def pop(self):
        return self._items.pop()

    def __len__(self):
        return len(self._items)

s = Stack()
s.push(1)
s.push(2)
s.push(3)
s.pop()  # 3
s.pop()  # 2
```

该对象的工作方式与之前完全相同，但它的功能完全专注于栈的实现。
没有多余的列表方法，设计目的更加清晰明确。

对此实现的一个小扩展可能是将内部列表作为可选参数接受。

```Python
class Stack:
    def __init__(self, *, container=None):
        if container is None:
            container = list
        self._items = container

    def push(self, item):
        self._items.append(item)

    def pop(self):
        self._items.pop()

    def __len__(self):
        return len(self.container)
```

这种方法的优点之一是促进了组件间的松耦合。
例如，想要创建一个栈，可以将其元素存储在数组而非列表中：

```Python
import array

s = Stack(container=array.array("i"))
s.push(42)
s.push(23)
s.push("a lot")  # Type Error .
```

这也被称为 "dependency injection" 依赖注入。
并非手动硬编码使用 `list`，而是让其依赖任何用户决定使用的容器来实现接口。

## Avoiding Inheritance via Functions

有时候会编写只有一个方法需要修改的类，例如：

```Python
class DataParser:
    def parse(self, lines):
        records = []
        for line in lines:
            row = line.split(",")
            record = self.make_record(row)
            records.append(row)
        return records

    def make_record(self, row):
        raise NotImplementedError()

class PortfolioDataParser(DataParser):
    def make_record(self, row):
        return {
            "name": row[0],
            "shares": int(row[1]),
            "price": float(row[2]),
        }

parser = PortfolioDataParser()
data = parser.parse(open("portfolio.csv"))
```

如果只写了一个方法的类，考虑使用函数。例如：

```Python
def parse_data(lines, make_record):
    records = []
    for line in lines:
        row = line.split(',')
        record = make_record(row)
        records.append(row)
    return records

def make_dict(row):
    return {
        "name": row[0],
        "shares": int(row[1]),
        "price": float(row[2]),
    }

data = parse_data(open('portfolio.cs'), make_dict)
```

现在代码简洁多了，即使以后要增加更多方法，将他们再加入类也不迟。
Premature 过早抽象通常不是一件好事。

## Dynamic Binding and Duck Typing

动态绑定是 Python 在运行时查找对象属性的机制，它使得 Python 能处理实例而无需考虑类型。
在 Python 中，变量名没有关联的类型。
因此，属性绑定过程与对象 obj 的类型无关。
如果使用 `obj.name` 方法，则任何有 name 属性的对象都能工作。
这种行为有时候被称为 duck typing 鸭子类型，这来自一句格言：如果它看上去像鸭子，叫声像鸭子，走路和鸭子一样，那么就是一只鸭子。

Python 程序员经常依赖这个特性。
例如你想要制作一个自定义已存在版本的对象，要么继承该对象，要么创建一个有类似行为的新对象。
后一种方法通常用于保持程序组件之间的松耦合关系。
例如，只要某个对象具有特定的一系列方法，编写的代码就可以与之协同工作。
最常见的就是标准库中的 `iterable` 对象，有许多不同类型的对象可以与 `for` 循环工作并产生值。
然而，这些类并未继承任何特殊的可迭代基类。
他们只是实现了执行迭代所需要的方法，一切便可以运行。

## The Danger of Inheriting from Build-in Types

Python 允许继承内置类型，然而这样做会引发危险。
例如，假设你决定继承字典类，并强制将所有 key 转为大写。
为了实现这个功能，你可能需要重新定义 `__setitem__()`：

```Python
class udict(dict):
    def __setitem__(self, key, value):
        super().__setitem__(key.upper(), value)
```

这样确实能运行

```Python
u = udict()
u['name'] = 'Guido'
u['number'] = 37
u
# { 'NAME': 'Guido', 'NUMBER': 37 }
```

但进一步测试发现只是部分能正常工作：

```Python
u = udict(name='Guido', number=37)
u
# { 'name': 'Guido', 'number': 37 }

u.update(color='blue')
# { 'name': 'Guido', 'number': 37, 'color': 'blue'}
```

这是因为 Python 的内置类似是使用 c 实现的，而不是其他 Python 类型。

collections 模块有一些特殊的类型 UserDict, UserList 和 UserString，这些类型可以用于实现 dict, list 和 str 的安全子类型。
例如，这种解决方案能够正常工作：

```Python
from collections import UserDict

class udict(UserDict):
    def __setitem__(self, key, value):
        super().__setitem__(key.upper(), value)
```

大部分继承内置类型的需要都是可以避免的，如果真的要使用优先考虑 Stack 的实现方式。

## Class Variables and Methods

在类型定义中，所有函数都默认操作示例，第一个参数永远都是 `self`。
然而，该类本身也是一个对象，并可以被修改。
例如，你可以在类中追踪创建了多少个实例：

```Python
class Account:
    num_accounts = 0

    def __init__(self, owner, balance):
        self.owner = owner
        self.balance = balance
        Account.num_accounts += 1

    def __repr__(self):
        return f'{type(self).__name__ ({self.owner!r}, {self.balance!r})}'

    def deposit(self, amount):
        self.balance += amount

    def withdraw(self, amount):
        self.deposit(-amount)

    def inquiry(self):
        return self.balance
```

类变量在 `__init__()` 之外定义，当修改后使用该类，而不是 `self`：

```Python
a = Account('Guido', 1000.0)
a = Account('Eva', 10.0)
Account.num_accounts
# 2
```

有点不同寻常的是，类变量可以通过示例访问：

```Python
a.num_accounts
# 2
```

之所以这样操作，是因为当示例本身没有匹配的属性时，属性查找会转向检查其关联的类。
Python 查找方法字段也是一样的机制。

因此，也可以定义一个类方法，类方法应用于类本身，而非实例。
一个常见用途是定义替代的示例构造函数。
例如，加入存在一个需求，需要从一种传统的企业输出格式创建账户实例：

```Python
data = '''
<account>
    <owner>Guido</owner>
    <amount>1000.0</amount>
</account>
'''
```

为了实现通过类方法创建实例，可以使用 `@classmethod`：

```Python
class Account:
    def __init__(self, owner, balance):
        self.owner = owner
        self.balance = balance

    @classmethod
    def from_xml(cls, date):
        from xml.etree.ElementTree import XML
        doc = XML(data)
        return cls(doc.findtext('owner'), float(doc.findtext('amount')))

# example
data = '''
<account>
    <owner>Guido</owner>
    <amount>1000.0</amount>
</account>
'''
a = Account.from_xml(data)
```

类方法的第一个参数永远是类本身。
按照惯例，一般叫做 `cls`，在这个例子中 `cls` 设置在 Account 上。
如果一个类方法的目的是创建新实例，则必须采取明确的步骤来实现。
在类的最后一行 `cls(..., ...)` 和 `Account(..., ...)` 效果是一样的。

将类作为参数传递这一做法，解决了与继承相关的一个重要问题。
加入你要继承该类创建一个子类，然后想要创建一个实例，你会发现类方法仍然有用：

```Python
class EvilAccount(name):
    pass

e = EvilAccount.from_xml(data)  # Create an 'EvilAccount'
```

之所以仍然能运行是因为 `EvilAccount` 作为 `cls` 传入了方法。
因此最后一行 `from_xml` 创建的是 `EvilAccount` 对象。

类变量和方法有时候会结合使用，以匹配控制实例的运行方式。
另一例子，考虑下面的 `Date` 类型：

```Python
import time

class Date:
    datefmt = '{year}-{month:02d}-{day:02d}'
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day

    def __str__(self):
        return self.datefmt.format(
        year=self.year,
        month=self.month,
        day=self.day,
    )

    @classmethod
    def from_timestamp(cls, ts):
        tm = time.localtime(ts)
        return cls(tm.tm_year, tm.tm_mon, tm.tm_mday)

    @classmethod
    def today(cls):
        return cls.from_timestamp(time.time())
```

该类包含一个类变量 `datefmt` 用于调整 `__str__()` 方法的输出，这是一个可以通过继承进行自定义的功能。

```Python
class MDYDate(Date):
    datefmt = '{month}/{day}/{year}'

class DMYDate(Date):
    datefmt = '{day}/{month}/{year}'

# Example
a = Date(1967, 4, 9)
print(a)  # 1967-04-09

b = MDYDate(1967, 4, 9)
print(b)  # 4/9/1967

c = DMYDate(1967, 4, 9)
print(c)  # 9/4/1967
```

通过类型变量和继承配置，是调整实例的行为的常见方法。
类方法的使用对于实现该功能至关重要，因为他们确保了创建出正确类型的对象。

例如：

```Python
a = MDYDate.today()
b = DMYDate.today()
print(a)  # 2/13/2019
print(b)  # 13/2/2019
```

通过类方法进行实例构造是目前最常见的应用。
这类方法通常遵循一个命名惯例，即在方法名前加上 `from_` 前缀，例如 `from_timestamp()`。
在标准库和第三方库中，经常会看到这种命名惯例用于标识类方法。
例如，字典类九有一个类方法，用于根据一组创建预初始化的字典。

```Python
dict.from_keys(['a', 'b', 'c'], 0)
# {'a': 0, 'b': 0, 'c': 0}
```

关于类方法需要注意的一点是，Python 并未将其与实例方法置于独立的命名空间中管理。
结果是，仍然可以通过实例来访问该方法。
例如：

```Python
d = Date(1967, 4, 9)
b = d.today()  # Calls Date.now(Date)
```

## Static Methods

有时一个类仅被用作命名空间，用于存放通过 `@staticmethod` 声明的静态方法。
不同于普通的方法或类方法，静态方法不接受一个 `self` 或 `cls` 参数。
一个静态方法 static method 只是刚好在类里面定义的一个函数。

例如：

```Python
class Ops:
    @staticmethod
    def add(x, y):
        return x * y

    @staticmethod
    def sub(x, y):
        return x - y
```

通常不会去创建一个类实例，相反，会直接通过类来调用这些函数：

```Python
a = Ops.add(2, 3)  # 5
b = Ops.sub(2, 3)  # -1
```

有时，其他类会使用这样的静态方法集和来实现 "swappable" 可交换或 "configurable" 可配置的行为。
或者作为某种松散模拟导入模块功能的方式。
考虑在之前的 `Account` 类使用继承：

```Python
class Account:
    def __init__(self, owner, balance):
        self.owner = owner
        self.balance = balance

    def __repr__(self):
        return f'{}'

    def deposit(self, amount):
        self.balance += amount

    def withdraw(self, amount):
        self.balance -= amount

    def inquiry(self):
        return self.balance

# A speical "Evil" account
class EvilAccount(Account):
    def deposit(self, amount):
        self.balance += 0.95 * amount

    def inquiry(self):
        if random.randint(0, 4) == 1:
            return 1.10 * self.balance
        else:
            return self.balance
```

这里使用继承的方式有些奇怪。
它引入了两种不同类型的对象（`Account` 和 `EvilAccount`）。
而且没有明显的方法可以将现有的 `Account` 实例转换为 `EvilAccount`，或者反过来，因为这涉及到改变实例类型。

或许，让 evil 以某种账户策略的形式显现出来反而更好。
以下是使用静态方法重新构建 `Account` 的一种方案

```Python
class StandardPolicy:
    @staticmethod
    def deposit(account, amount):
        account.balance += amount

    @staticmethod
    def withdraw(account, amount):
        account.balance -= amount

    @staticmethod
    def inquiry(account):
        return account.balance

class EvilPolicy(StandardPolicy):
    @staticmethod
    def deposite(account, amount):
        account.balance += 0.95 * amount

    @staticmethod
    def inquiry(account):
        if random.randint(0, 4) == 1:
            return 1.10 * account.balance
        else:
            return account.balance

class Account:
    def __init__(self, owner, balance, *, policy=StandardPolicy):
        self.owner = owner
        self.balance = balance
        self.policy = policy

    def __repr__(self):
        return f'Account({self.policy}, {self.owner!r}, {self.balance!r})'

    def deposit(self, amount):
        self.policy.deposit(self, amount)

    def withdraw(self, amount):
        self.policy.withdraw(self, amount)

    def inquiry(self):
        return self.policy.inquiry(self)
```

在这个 reformulation 重构中，只创建一种 `Account` 类型。
但是，它有一个特殊的 `policy` 属性，提供了多种方法的实现。
如果需要，`Account` 实例的 policy 可以动态修改：

```Python
a = Account('Guido', 1000.0)
a.policy
# <class 'StandardPolicy'>
a.policy = EvilPolicy
a.policy
# <class 'EvilPolicy'>
```

在这里使用 `@staticmethod` 的一个原因是，无需创建 StandardPolicy 或 EvilPolicy 的实例。
这些类的的目标是组织一捆方法，而不是存储与 Account 相关的额外实例数据。

尽管 Python 的松耦合特性确实允许策略升级以持有自设数据。
将静态方法改为普通的实例方法。
例如：

```Python
class EvilPolicy(StandardPolicy):
    def __init__(self, deposit_factor, inquiry_factor):
        self.deposit_factor = deposit_factor
        self.inquiry_factor = inquiry_factor

    def deposite(self, account, amount):
        account.balance += self.deposit_factor * amount

    def inquiry(self, amount):
        if random.randint(0, 4) == 1:
            return self.inquiry_factor * account.balance
        else:
            return account.balance

# Example
a = Account('Guido', 1000.0, policy=EvilPolicy(0.95, 1.10))
```

这种将方法委托为支持类的做饭，是状态机及类似对象常用的实现策略。
每个允许状态都可以封装成独立的方法类（通常为静态类）。
如此例中的策略属性可变实例变量，便可用来存储与当前运行状态相关的具体实现细节。

## A World About Design Patterns

在编写面向对象程序时，有时会过于执着于实现特定的命名设计模式。
例如，策略模式 strategy pattern，享元模式 flyweight pattern，单例模式 singleton pattern 等。
这些多出于 Design Pattern 设计模式这本书。

如果你熟悉这些模式，这些内容当然可以应用到 Python 中。
然而，许多这些设计模式的设计目标是围绕着 C++ 或 Java 严格的类型系统设计的。
Python 的动态类型特性使得许多的这类模式变得过时或根本不再必要。

尽管如此，编写“优秀软件”有几个核心原则。
例如，致力于编写易于调试、可测试且可扩展的代码。
基本的特性包括为类编写 `__repr()__` 方法，组合优先于继承，并运行依赖注入，这些对实现目标很有帮助。
Python 程序员也喜欢编写被认为 Pythonic 的代码。
通常，这意味着创建遵循各种内置协议的对象，例如迭代、容器、上下文管理器等。
例如，Python 程序员不会视图从 Java 编程书中照搬某种复杂的数据遍历模式，而更可能通过生成器函数配合 for 循环来实现。
或者，干脆用几次字典查找就能替代整个模式。

## Data Encapsulation and Private Attributes

在 Python 中所有的属性和方法都是公共的，这意味着他们都将没有访问限制。
在面向对象应用中，这通常不是所期望的，因为人们希望隐藏或封装内部实现细节。
为了解决这个问题，Python 依赖命名约定来表示预期的使用方式。
一个管理是使用单下划线 `_` 表示内部实现。

例如，这里有个 `Account` 类，其中 `balance` 属性是 “私有” 的：

```Python
class Account:
    def __init__(self, owner, balance):
        self.owner = owner
        self._balance = balance

    def __repr__(self):
        return f'Account({self.owner!r}, {self._balance!r})'

    def deposit(self, amount):
        self._balance += amount

    def withdraw(self, amount):
        self._balance -= amount

    def inquiry(self):
        return self._balance
```

这段代码中，`_balance` 意味着内部细节。
没有任何阻止用户直接访问，但使用单下划线是一个强烈的信号。

一个灰色地带是，内部属性是否对子类可访问。
例如，之前的继承示例是否允许直接访问其父类的 `_balance` 属性？

```Python
class EvilAccount(Account):
    def inquiry(self):
        if random.randint(0, 4) == 1:
            return 1.10 * self._balance
        else:
            return self._balance
```

通常来说，这在 Python 中是可接受的。
此外，IDE 和其他工具可能会暴露这些属性。
如果是 C++、Java 类似的面向对象语言，这类似将 `_balance` 视为所谓的 “受保护” 属性。

如果希望使用一种更加 “私有” 的属性，名称前缀使用双下划线 `__`。
所有类似 `__name` 的名称都会自动重命名为 `_Classname__name` 的形式。
这确保了超类中使用的私有名称不会被子类中相同的名称所覆盖。

例如：

```Python
class A:
    def __init__(self):
        self.__x = 3

    def __spam(self):  # _A__spam()
        print('A.__spam', self.__x)

    def bar(self):
        self.__spam()  # Calls A.__spam()

class B(A):
    def __init__(self):
        A.__init__(self)
        self.__x = 37  # self._B__x

    def __spam(self):  # _B__spam()
        print('B.__spam', self.__x)

    def grok(self):
        self.__spam()    # Calls B.__spam()
```

在这个例子中，有两个不同的对 `__x` 属性的赋值。
此外，看起来 B 视图通过继承来重写 `__spam()` 方法。
然而事实并非如此，名称修饰机制回味每个定义生成唯一的名称。

使用 `vars()` 方法清楚显示对象 b 的属性：

```Python
vars(b)
# {'_A__x': 3, '_B__x': 37}

b._A_spam()
# 3

b._B__spame()
# 37
```

尽管这种方案营造了数据隐藏的假象，但实际上并没有严格的机制来真正阻止对类中 “私有” 属性的访问。
如果知道类名和对应的属性名称，仍然可以通过修改后的名称访问。
如果这样的私有属性访问对你仍然是个问题，那么应该考虑严格的代码审核。

要注意 name mangling 名称修饰在定义时就确定了，不会影响运行时。
它不会在方法执行期间发生，也不会给程序执行增加额外开销。

在实践中，最好不要过度思考私有名称。
使用单下划线就和很常见的做法，双下划线反而更少见。
即使你可以通过一些方法实现真正的私有属性，但额外的努力会增加复杂度，这并不值得。

## Type Hinting

用户自定义类的属性对其类型或值没有限制。
实际上，你可以使用任何想要的类型。
如果担心这方面的问题，那就不要这样做，也可以使用外部工具例如 linters 和 type checkers。
为此，类允许为特定属性指定可选的类型提示。

```Python
class Account:
    owner: str
    _balance: float

    def __init__(self, owner, balance):
        self.owner = owner
        self._balance = balance
```

包含类型提示对运行没有任何影响，但可以为编辑器提供额外的信息。

## Properties

如前文所述，Python 对运行时类型没有限制。
然而，强制类型是可能的，如果将属性放到叫做 "property" 的管理之下。
Property 属性是一种特殊类型的特性，通过用户定义的方法拦截特性访问，并处理它。
这些方法在管理属性时有完全的自由。

```Python
import string

class Account:
    def __init__(self, owner, balance):
        self.owner = owner  # 会调用 @owner.setter
        self._balance = balance

    @property
    # 将 owner() 变成只读属性，可以通过 instnce.owner 访问 self._owner
    # 实际上会调用 getter 方法
    def owner(self):
        return self._owner

    @owner.setter
    def owner(self, value):
        # 验证数据后存入 _owner, value 是 __init__() 里面的 owner
        if not isinstance(value, str):
            raise TypeError('Expected str')
        if not all(c in string.ascii_uppercase for c in value):
            raise ValueError('Must be uppercase ASCII')
        if len(value) > 10:
            raise ValueError('Must be 10 characters or less')
        self._owner = value
```

`@property` 装饰器用于将 attribute 变为一个 property。
在这个例子中，应用在 `owner` attribute 上。
该装饰器总是首先应用在一个获取 attribute 属性值的方法之上。
这个例子中，该方法会返回存储在私有属性 `_owner` 里的真实值。
`@owner.setter` 装饰器用于选择性的实现一个设置属性值的方法，这里的 `@owner.` 是前面的 property，而不是下面的函数。
该方法在将值存储到私有属性 `_owner` 之前，会执行各种类型和值的检查。

Properties 的一个关键特性是，相关联的名称变得 "magical"。
也就是说，对该属性任何访问都会通过 `getter/setter` 方法进行，而不需要修改任何已有的代码。
例如 `Account.__init__()` 无需任何改动，这看起来有些奇怪，
因为 `__init__()` 执行的是 `self.owner = owner` 的赋值，而不是直接通过私有属性 `self._owner`。
这是刻意为之，`owner` 属性的整个设计目的就是为了在设置属性时进行验证。
在创建示例时，会希望通过这种验证。

由于每次访问属性时，都会自动调用方法，因此实际值需要以不同名称存储。
这就是为什么 getter 和 setter 方法使用 `_owner`。
不能使用 `owner` 作为存储位置，因为那样会导致无限递归。

通常，属性允许拦截任何特定的属性名。
可以实现获取、设置或删除属性值的方法。

```Python
class SomeClass:
    @property
    def attr(self):
        print('Getting')

    @attr.setter
    def attr(self, value):
        print('Setting', value)

    @attr.deleter
    def attr(self):
        print('Deleting')

s = SomeClass()
s.attr       # Getting
s.attr = 13  # Setting
del s.attr   # Deleting
```

没有必要实现所有部分的 property，实际上，使用 properties 实现只读计算数据属性更常见。

```Python
class Box(object):
    def __int__(self, width, height):
        self.width = width
        self.height = height

    @property
    def area(self):
        return self.width * self.height

    @property
    def perimeter(self):
        return 2*self.width + 2*self.height

# use example
b = Box(4, 5)
print(b.area)
print(b.perimeter)
b.area = 5 # Error: can't set attribute
```

Python 程序员通常并不注意到，方法本身是一种隐式处理的 property:

```Python
class SomeClass:
    def yow(self):
        print('Yow!')
```

当用户创建一个实例，如 `s = SomeClass()` 然后访问 `s.yow`，原始的函数 `yow` 并没有返回。
反而是一个 bound method 绑定方法：

```Python
s.yow
# <bound method SomeClass.yow of <__main__.SomeClass object at 0x1079ce090>>
```

当函数放在类中的时候，其行为很像 property。
具体来说，函数会神奇地拦截属性访问，并在幕后创建绑定方法。
当使用 `@staticmethod` 和 `@classmethod` 定义 static 和 class 方法的时候，实际上是在修改这个过程。
`@staticmethod` 将方法函数直接返回，不进行任何特殊的包装和处理。

## Types, Interfaces, and Abstract Base Classes

当创建一个类的实例时，该实例的类型是类本身。
要测试一个对象是否属于某个类，使用内置函数 `isinstance(obj, cls)`。
返回 `True` 表示 obj 属于 cls 类型，或者任何继承 cls 的子类型。
此外，还有内置函数 `issubclass(A, B)` 内置函数来判断 A 是否是 B 的子类。

关于类型的一个常见用途是编程接口的规范。
例如，可以设计一个顶级基类来规定编程接口的具体要求。
该基类可能被用于类型提示 或 者防御类型检查 `isinstance()`

```Python
class Stream:
    def receive(self):
        return NotImplementedError()

    def send(self):
        return NotImplementedError()

    def close(self):
        return NotImplementedError()

# Example
def send_request(stream, request):
    if not isinstance(stream, Stream):
        raise TypeError('Excepted a Stream')
    stream.send(request)
    return stream.receive()
```

对于此类代码的预期，并非直接使用 Stream。
相反，不同的类会继承 Stream 并实现需要的功能。

```Python
class SocketStream(Stream):
    def receive(self):
        ...

    def send(self, value):
        ...

    def close(self):
        ...

class PipeStream(Stream):
    def receive(self):
        ...

    def send(self, value):
        ...

    def close(self):
        ...

s = SocketStream()
send_request(s, request)
```

在本例中，值得讨论的一点是 `send_request` 中的运行时的强制检查，是否应该使用类型提示代替呢？

```Python
def send_request(stream: Stream, request):
    stream.send(request)
    return stream.receive()
```

鉴于类型提示并非强制执行，如何根据某个接口来验证一个参数，实际上取决于你希望这种验证什么时候发生。

在大型代码框架和应用中，使用接口类型更加常见。
但这种方法的问题是如何确保子类实现要求的接口。
例如，如果一个子类没有现其中的某个方法，或者拼写错了，则该问题将难以被发现。
然而，当后面调用该方法的时候，程序就会崩溃，这往往发生在凌晨 3 点。

为了避免这个问题，使用 `abc` 模块将接口定义为抽象基类 "abstract base class" 是一种常见的做法。
该模块定义了一个基类 `ABC` 和 一个 `@abstractmethod` 装饰器来描述接口。

例如：

```Python
from abc import ABC, abstractmethod

class Stream(ABC):
    @abstractmethod
    def receive(self):
        pass

    @abstractmethod
    def send(self, msg):
        pass

    @abstractmethod
    def close(self):
        pass
```

一个抽象基类不应该被实例化，这样做会导致报错。

下面再编写类型继承该抽象基类：

```Python
class SocketStream(Stream):
    def read(self):  # 方法写错了
        ...

    def send(self, msg):
        ...

    def close(self):
        ...
```

一个抽象基类能够在实例化的时候就捕获到该错误，这种提早捕获结果的方法非常有用

```Python
s = SocketStream()
# Traceback (most recent call last):
#   File "<stdin>", line 1, in <module>
# TypeError: Can't instantiate abstract class SocketStream with abstract methods receive
```

虽然抽象基类无法实例化，但是它可以为子类创建方法与 properties。
更多的是，一个基类中的抽象方法仍然能被子类调用。
例如在子类中可以使用 `super().receive()` 调用基类方法。

## Multiple Inheritance, Interfaces, and Mixins

Python 支持多重继承 multiple inheritance (不是多层继承 multilevel inheritance)，如果子类继承多个父类，则子类会继承父类所有的特性：

```Python
class Duck:
    def walk(self):
        print('Widdle')

class Trombonist:
    def noise(self):
        print('Blat!')

class DuckBonist(Duck, Trombonist):
    pass

d = DuckBonist()
d.walk()
d.noise()
```

看上去这是一个整洁的想法，但实际会显露出很多问题。
例如，如果两个父类都定义了 `__int__()` 方法怎么办？
或这两个方法都有 `noise()` 方法呢？
突然，你意识到多重继承充满了危险。

多重继承应该被视为一种高度专业化的代码组织和复用工具，而非通用编程技术。
具体而言，随意选取一组互不相关的类，通过多重继承将他们组合成奇怪的 “鸭子音乐家” 式混合体，并非标准做法。
实际上，永远不应该这样做。

一个更常见的多重继承使用是组织类型和接口关系。
例如，上面介绍了抽象基类的概念。
抽象基类的目的是为了指定编程接口。
例如，你可能有多个抽象基类：

```Python
from abc import ABC, abstractmethod

class Stream(ABC):
    @abstractmethod
    def receive(self):
        pass

    @abstractmethod
    def send(self, msf):
        pass

    @abstractmethod
    def close(self):
        pass

class Iterable(ABC):
    @abstractmethod
    def __iter__(self):
        pass
```

对于这些类型，可以使用子类通过多重继承指定接口：

```Python
class MessageStream(Stream, Iterable):
    def receive(self):
        ...
    def send(self):
        ...
    def close(self):
        ...
    def __iter__(self):
        ...
```

多重继承的使用不是关于实施，而是类型关系。
没有代码复用，继承关系主要用于进行类型检查。

```Python
m = MessageStream()
isinstance(m, Stream)
isinstance(m, Iterable)
```

另一种多重继承的用法是定义 "mixin" 类型。
一个 minix 类用于修改或扩展其他类的功能。

例如：

```Python
class Duck:
    def noise(self):
        return 'Quark'

    def waddle(self):
        reutrn 'Waddle'

class Trombonist:
    def noise(self):
        return 'Blat!'

    def match(self):
        return 'Colmp'

class Cyclist:
    def noise(self):
        return 'On your left!'
    def pedal(self):
        return 'Pedaling'
```

这些类完全和彼此无关，他们之间没有继承关系，并实现了不同的方法。
然而，他们又一个共性，就是都实现了 `noise()` 方法。
一此为参考，定义如下修饰符 modifier 类：

```Python
class LoudMixin:
    def noise(self):
        return super().noise().upper()

class AnnoyingMixin:
    def noise(self):
        return super().noise() * 3
```

看上去这些类好像有问题，因为他们试图通过使用 `super()` 访问不存在的父类。
实际上，实例化调用 `noise()` 方法确实会报错。
这些是 minix 类型，正常的使用方式是将他们与其他实现了该方法的类结合起来：

```Python
class LoudDuck(LoudMixin, Duck):
    pass

class AnnoyingTrombonist(AnnoyingMixin, Trombonist):
    pass

class AnnoyingLoudCyclist(AnnoyingMixin, LoudMixin, Cyclist):
    pass

d = LoudDuck()
d.noise()  # 'QUACk'

t = AnnoyingTrombonist()
t.noise()  # 'Blat!Blat!Blat!'

c = AnnoyingLoudCyclist()
c.noise()  # 'ON YOUR LEFT!ON YOUR LEFT!ON YOUR LEFT!'
```

由于 mixin 类型和普通类的定义相同，因此建议在类名后面添加 "Mixin" 以表明该类的目的。

无论是否使用继承，Python 都会构建一个线性的类链称为 "Method Resolution Order"，即 MRO。
这个通过类中的 `__mro__` 属性可以访问，下面是一些单继承的例子：

```Python
class Base:
    pass

class A(Base):
    pass

class B(A):
    pass

Base.__mro__  #                                             (<class '__main__.Base'>, <class 'object'>)
A.__mro__     #                       (<class '__main__.A'>, <class '__main__.Base'>, <class 'object'>)
B.__mro__     # (<class '__main__.B'>, <class '__main__.A'>, <class '__main__.Base'>, <class 'object'>)
```

MRO 规定了属性查找的顺序，当访问一个类的属性和方法的时候，会按照 MRO 的顺序进行查找。

为了支持多重继承，Python 实现了 “协作式多重继承” cooperative multiple inheritance。
通过协作式继承，所有的类都根据下面两个规则放在 MRO 列表中：

1. 子类必须在父类之间被检查
2. 如果继承多个父类，则必须按照子类中父类的写入顺序进行检查

然而，实际实现的算法会十分复制，并不是简单的深度优先或广度优先遍历。
该算法称为 C3 linerization algorithm (C3 线性算法)，paper "A Monotonic Superclass Linerizaiton for Dylan" (K. Barrett, et al, presented at OOPSLA'96).
该算法的微妙之处在于某些类层次构造会被 Python 拒绝，抛出 `TypeError`，例如：

```Python
class X: pass
class Y(X): pass
class Z(X, Y): pass  # TypeError. Can't create consistent MRO
```

在这个例子中，X 在 Y 前面，因此 MRO 先检查 X 再检查 Y，但是 Y 继承自 X，导致 Y 又在 X 前面，这就产生了冲突。
这种问题一般很少出现，如果出现这说明有严重的设计问题。

`super()`函数的行为与底层的方法解析顺序（MRO）紧密相关。
具体而言，它的作用是将属性委托给 MRO 中的下一个类。
这一机制取决于使用`super()`的类所在的位置。
例如，当 `AnnoyingMixin` 类使用 `super()` 方法，他会查看 MRO 的实例来找到自己的位置。
随后，它将属性委托给下一个类。

在这个列子中，在 `AnnoyingLoudCyclist` 类使用 `super().noise()` 会触发 `LoudMixin.noise()`。

设计协作式多重继承和 mixin 十分有挑战性，但这仍有一些设计指导原则。
首先，在方法解析顺序 (MRO) 中，子类总是优先于任何基类被检查。
因此，minix 共用同一个父类实际上很常见，mixin 类通常共享一个共同的父类，且该父类会为方法提供空实现。
如果同时使用多重 mixin 类型，他们将会依次排列。
公共父类应置于最后，以便提供默认或错误检查。

例如：

```Python
class NoiseMixin:
    def noise(self):
        raise NotImplementedError('noise() not implemented')

class LoudMixin(NoiseMixin):
    def noise(self):
        return super().noise().upper()

class AnnoyingMixin(NoiseMixin):
    def noise(self):
        return 3 * suepr().noise()
```

第二个指导放针是所有的 mixin 方法的实现都应该相同的函数签名。
Mixin 的一个问题是他们是可选的，并通常以不可预测的顺序混合在一起。
为了让这个能正常工作，必须确保涉及 `super()` 的操作无论后续无论出现哪个类都能成功执行。
为此，调用链中的所有方法都必须具有兼容的调用签名。

最后，要确保在任何位置都使用 `super()`。
有时你会遇到一个类直接调用其父类：

```Python
class Base:
    def yow(self):
        print('Base.yow')

class A(Base):
    def yow(self):
        print('A.yow')
        Base.yow(self)

class B(Base):
    def yow(self):
        print('B.yow')
        Base.yow(self)
        super().yow(self)

class C(A, B):
    pass

c = C()
c.yow()
# A.yow
# Base.yow
```

这样的类型使用多重继承就不安全，这样会打破调用链并导致混乱。
这个例子中，从未调用过 `B.yow()` 即使它是继承的一部分。
如果要使用多重继承，应该考虑使用 `super()` 而不是直接调用父类方法。

## Type-based Dispatch

有时候需要编写根据特定类型进行分发的代码。

例如：

```Python
if isinstance(obj, Duck):
    handle_duck(obj)
elif isinstance(obj, Trombonist):
    handle_trombonist(obj)
elif isinstance(obj, Cyclist):
    handle_cyclist(obj)
else:
    raise RuntimeError('Unknown object')
```

编写一长串的 if-else 块脆弱又不优雅，一个解决方案是通过字典进行调度：

```Python
handlers = {
    Duck: handle_duck,
    Trombonist: handle_trombonist,
    Cyclist: handle_cyclist,
}

# Dispatch
def dispatch(obj):
    func = handlers.get(type(obj))
    if func:
        reutrn func(obj)
    else:
        raise RuntimeError(f'No handler of {obj}')
```

有时调度功能通过类接口 `getattr()` 实现：

```Python
class Dispatcher:
    def handle(self, obj):
        for ty in type(obj).__mro__:
            meth = getattr(self, f'handle_{ty.__name__}', None)
            if meth:
                return meth(objt)
        raise RuntimeError(f'No handler for {obj}')

    def handle_Duck(self, obj):
        ...

    def handle_Trombonist(self, obj):
        ...

    def handle_Cyclist(self, obj):
        ...

# Example
dispatcher = Dispatcher()
dispatcher.handle(Duck())     # handle_Duck()
dispatcher.handle(Cyclist())  # handle_Cyclist()
```

## Class Decorators

有时，在定义类之后，可能希望执行额外的处理步骤。
例如，将类添加到注册表中，或生成额外的支持代码。
解决此类问题的一种方法就是使用类装饰器。
一个类装饰器是接受一个类作为参数，并返回一个类的函数。

例如这个维护注册表的问题：

```Python
_registry = { }  # 注册表
def register_decorder(cls):
    for mt in cls.mimetypes:
        _registry[mt.mimetype] = cls
    return cls

# Factory function that uses the registry
def create_decoder(mimetype):
    return _registry[mimetype]()
```

在这个例子中，`register_decorder()` 函数会查找类内部的 `mimetypes` 属性。
如果找到该项，就会使用它将该类加入一个字典中，该字典用于将 MIME 类型映射到类对象。
为了使用这个函数，在类定义之前将其作为装饰器使用。

例如：

```Python
@register_decoder
class TextDecoder:
    mimetyeps = [ 'text/plain' ]
    def decode(self, data):
        ...

@regisger_docoder
class HTMLDecoder:
    mimetypes = [ 'text/html' ]
    def decode(self, data):
        ...

@register_decorder
class ImageDecoder:
    mimetype = [ 'image/png', 'image/jpg', 'image/gif' ]
    def decode(self, data):
        ...

# Example usage
decoder = create_decoder('image/jpg')
```

类装饰器能够自由修改类的内容。
例如，他们甚至可能重写现存的方法。
这是替代 mixin 类型和使用多重继承的一种方法。

```Python
def loud(cls):
    orig_noise = cls.noise
    def noise(self):
        return orig_noise(self).upper()
    cls.noise = noise
    return cls

def annoying(cls):
    orig_noise = cls.noise
    def noise(self):
        return orig_noise(self) * 3
    cls.noise = noise
    return cls


@annoying
@loud
class Cyclist(object):
    def noise(self):
        return 'One your left'

    def pedal(self):
        return 'Pedaling'
```

这个例子产生的结果和之前的 mixin 相同，但并没有使用多重继承和 `super()`。
在每个装饰器内，`cls.noise` 和 `spuer()` 的效果相同。
但是，由于装饰器仅在应用时执行一次，后续对 `noise()` 的调用会运行得更快一些。

类装饰器可以用于创建全新的代码。
例如，编写类的一个常见任务是编写有用的 `__repr__()` 方法以提升调试效率。

```Python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __repr__(self):
        return f'{type(self).__name__}({self.x!r}, {self.y!r})'
```

总是编写该方法挺麻烦的，或许可以实现一个类装饰器来完成：

```Python
import inspect
def with_repr(cls):
    args = list(inspect.signature(cls).parameters)  # 获取参数列表
    argvals = ', '.join('{self.%s!r}' % arg for arg in args)
    code = 'def __repr__(self):\n'
    code += f'  return f"{cls.__name__}({argvals})"\n'
    locs = { }
    exec(code, locs)  # 执行字符串代码，生成函数
    cls.__repr__ = locs['__repr__']
    return cls

# Example
@with_repr
def Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y
```

在这个例子中，`__repr__()` 方法是从 `__init__()` 方法的调用签名自动生成的。
该方法由一串文本字符创建，并传递给 `exec()` 函数，然后在绑定到类上。

类似的代码生成技术在标准库中也存在。
例如，定义数据结构的一种便捷方法是使用数据类：

```Python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int
```

dataclass 会自动从类的类型提示创建 `__init__()` 和 `__repr__()` 这样的方法，这些方法都使用 `exec()` 创建。
然而，这种方法的缺点之一是启动性能较差。
使用 `exec()` 动态创建的代码会绕过 Python 通常对模块应用的编译优化。
因此，以这种方式定义大量类会显著减慢代码的导入速度。

## Supervised Inheritance

有时候想要给类添加额外的功能，类装饰器是实现的方式之一。
然而，父类也可以代表其子类执行额外的操作。
这是通过实现一个 `__init_subclass__(cls)` 类方法来完成的。

```Python
class Base:
    @classmethod
    def __init_subclass__(cls):
        print('Initalizing', cls)

# Example
class A(Base):
    pass

class B(Base):
    pass
```

只要 `__init_subclass__()` 方法存在，无论子类继承有多深都会自动触发执行。
许多通常通过类装饰器完成的任务，现在可以通过 `__init_subclass__()` 来实现：

例如，类注册：

```Python
class DecoderBase:
    _registry = {}
    @classmethod
    def __init_subclass__(cls):
        for mt in cls.mimetypes:
            DecoderBase._registry[mt.mimetype] = cls

def create_decoder(mimetype):
    return DecoderBase._registry[mimetype]()

class TextDecoder(DecoderBase):
    mimetypes = [ 'text/plain' ]
    def decode(self, data):
        ...

class HTMLDecoder(DecoderBase):
    mimetypes = [ 'text/html' ]
    def decode(self, data):
        ...

class ImageDecoder(DecoderBase):
    mimetypes = [ 'image/png', 'image/jpg', 'image/gif' ]
    def decode(self, data):
        ...

decoder = create_decoder('image/jpg')
```

这里是一个类从 `__init__()` 方法自动创建 `__repr__()` 方法的例子：

```Python
import inspect

class Base:
    @classmethod
    def __init_subclass__(cls):
        # Create a __repr__ method
        args = list(inspect.signature(cls).parameters)
        argvals = ', '.join('{self.%s!r}' % arg for arg in args)
        code = 'def __repr__(self):\n'
        code += '  return f"{cls.__name__}({argvals})"\n'
        locs = { }
        exec(code, locs)
        cls.__repr__ = locs['__repr__']

class Point(Base):
    def __init__(self, x, y):
        self.x = x
        self.y = y
```

如果使用了多层继承，则应该使用 `super()` 来确保所有实现了 `__init_subclass__()` 的类都能被调用。

```Python
class A:
    @classmethod
    def __init_subclass__(cls):
        print('A.init_subclass')
        super().__init_subclass__()

class B:
    @classmethod
    def __init_subclass__(cls):
        print('B.init_subclass')
        super().__init_subclass__()

# 应该能看到两个类的输出
class C(A, B):
    pass
```

通过 `__init_subclass__` 来实现 supervising inheritance 监督继承是 Python 最强大的自定义特性之一。
大部分力量源于其隐含特性，一个顶级基类可以借此悄无声息地监督整个子类层次结构。
这种监督机制可以注册类、重写方法、执行验证等操作。

## The Object Life Cycle and Memmory Management

当定义一个类时，实现的类就是创建新实例的工厂。

```Python
class Amount:
    def __init__(self, owner, balance):
        self.owner = owner
        self.balance = balance

a = Amount('Guido', 1000.0)
b = Amount('Eva', 25.0)
```

实例的创建通过两个步骤完成:
首先使用特殊方法 `__new__()` 创建新实例，然后使用 `__init__()` 对其进行初始化。

例如 `a = Account('Guido', 1000.0)` 会执行以下步骤：

```Python
a = Account.__new__(Accound, 'Guido', 1000.0)
if isinstance(a, Account):
    Account.__init__('Guido', 1000.0)
```

除了第一个参数接收的类本身而非实例外，`__new__()` 通常接收与 `__init__()` 相同的参数。
有时候会看到 `__new__()` 仅传入单个参数调用。
然而，`__new__()` 的默认实现通常只是直接忽略它们。
有时候会看到 `__new__()` 仅被传入单个参数调用。

比如这样写也可以：

```Python
a = Account.__new__(Account)
Account.__init__('Guido', 1000.0)
```

直接使用 `__new__()` 方法并不常见，但有时会用它来创建实例，同时绕过对 `__init__()` 方法的调用。

类方法就有这样的，例如：

```Python
import time

class Date:
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day

    @classmethod
    def today(cls):
        t = time.localtime()
        self = cls.__new__(cls)  # Make instance
        self.year = t.tm_year
        self.month = t.tm_month
        self.day = t.tm_year
        return self
```

执行对象序列化的模块，如 pickle 在反序列化对象时也会利用 `__new__()` 方法来重新创建实例。
这是在不调用 `__init__()` 的情况下完成的。

有时，类会定义 `__new__()` 方法，以改变实例创建的某些方面。
典型的应用包括实例缓存 instance caching、单例模式 singleton 和不可变性 immutability。

假如你希望 Date 类实现日期驻留，缓存 Date 实例并复用有相同年月日的现有对象。
这是一种实现方式：

```Python
class Date:
    _cache = { }

    @staticmethod
    def __new__(cls, year, month, day):
        self = Date._cache.get((year, month, day))
        if not self:
            self = super().__new__(cls)
            self.year = year
            self.month = month
            self.day = day
            Date._cache[year, month, day] = self
        return self

    def __init__(self, year, month, day):
        pass

d = Date(2012, 12, 21)
e = Date(2012, 12, 21)

assert d is e  # same object
```

在这个例子中，类维护一个内部字典，用于存储先前创建的 Date 实例。
在创建新的 Date 对象时，会首先查询缓存。
如果找到匹配项，则返回该实例。
否则，会创建并初始化新的实例。

此解决方案的一个微秒之处在于其空的 `__init__()` 方法。
即使已经缓存了，每次调用 `Date()` 仍然会调用 `__init__()`。
为了避免这个情况，该方法仅执行空操作，实例的实际创建发生在首次创建实例时的 `__new__()` 方法中。

有方法可以避免对 `__init__()` 的额外调用，但这需要一些巧妙的技巧。
避免这种情况的一种方法是让 `__new__()` 返回一个完全不同的类型实例。
另一种解决方案是使用元类 metaclass。

实例一但创建就会通过引用计数管理，如果引用计数归零会立刻消毁。
当要消毁实例的时候，解释器首先寻找该对象上的 `__del__` 方法，然后调用它。

```Python
class Account(object):
    def __init__(self, owner, balance):
        self.owner = owner
        self.balance = balance

    def __del__(self):
        print('Deleting Account')

a = Account('Guido', 1000.0)
del a
# Deleting Account
```

有时，程序会使用 `del` 语句来删除对某个对象的引用。
如果这导致引用计数归零，则会调用 `__del__()` 方法。
但并不是总会调用该方法，因为该对象往往还有其他引用。
此外，还有许多其他导致对象被删除的方法。

例如，变量名的重新赋值或函数中变量超出作用域的情况：

```Python
a = Account('Guido', 1000.0)
a = 42  # Deleting Account

def func():
    a = Account('Guido', 1000.0)
func()  # Deleting Account
```

在实际中，很少需要定义 `__del__()` 方法。
唯一的例外是，当消毁对象需要进行额外清理操作时，例如关闭文件、断开网络连接或是否其他系统资源。
即使在这些情况下，依赖 `__del__()` 来实现正确的关闭也是危险的，因为无法保证该方法会在可能被调用的时候执行。
为了清晰的关闭资源，应该提供专门的 `close()` 方法。
同是也想要让类支持上下文管理器协议，这样就可以使用 `with` 语句。

下面是一个覆盖所有情况的例子：

```Python
class SomeClass:
    def __init__(self):
        self.resource = open_resource()

    def __del__(self):
        self.close()

    def close(self):
        self.resource.close()

    def __enter__(self):
        return self

    def __exit__(self, ty, val, tb):
        self.close()

# 通过 __del__() 关闭
s = SomeClass()
del s

# 显示关闭
s=  SomeClass()
s.close()

# 在 context 块结束后关闭
with SomeClass() as s:
    ...
```

再次强调，在大多数类中都没有必要编写 `__del__()` 方法。
Python 本身就有垃圾回收机制，没有自己实现的必要，除非在对象消毁过程中有额外的行为需要执行。
即使这样，你可能也不需要 `__del__()`。类可能早已编写为能自己合理地自我清除。

除了引用计数和对象消毁本身存在的诸多风险外，还有某些编程模式。
尤其设计父子关系、图结构或缓存的场景，还可能引发所谓的 “引用循环” 问题。

例如：

```Python
class SomeClass:
    def __del__(self):
        print('Deleting')

parent = SomeClass()
child = SomeClass()

# 创建一个父子循环引用
parent.child = child
child.parent = parent

# 尝试删除：没有输出 Deleting
del parent
del child
```

在这个例子中，变量名称被消毁了，但 `__del__()` 方法却没有被调用。
这两个对象每个内部都有另一个的引用。因此，引用计数无法降到 0。
为了应对这种情况，一个特殊的循环检测垃圾回收器会定期运行。
最终这些对象会被回收，但很难预测何时会发生。
如果想要强制垃圾回收，可以运行 `gc.collect()`。
`gc` 模块还包含多种与循环垃圾回收及内存监控相关的其他技能。

由于垃圾回收的不可预测性，`__del__()` 方法在使用上存在一些限制。
首先，任何从 `__del__()` 方法中传播出的异常都会被打印到 `sys.stderr`，除此之外将被忽略。
其次，`__del__()` 方法应避免涉及获取锁或其他资源的操作。
这样做可能导致死锁，当 `__del__()` 在信号处理和线程的第七层内部回调圈中执行无关函数时意外触发。
如果一定要使用 `__del__` 方法，请保持其简明。

## Weak References

有时候变量还存在，但你希望他们被回收。
例如上面的 `Date` 类型，该类型一但添加缓存就无法取消。

解决上面问题的一种方法是弱引用 `weakref` 模块。
弱引用是一种引用对象，但是不增加引用计数的方式。

```Python
import weakref

a = Account('Guido', 1000.0)
a_ref = weakref.ref(a)
# <weakref at 0x104617188; to 'Account' at 0x1046105c0>

del a
a_ref
# <weakref at 0x104617188; dead>
```

为了获取弱引用指向的对象，需要像无参函数那样调用它。
这将返回被指向的对象或 `None`：

```Python
acct = a_ref()
if acct is not None:
    acct.withdraw()

# 简洁写法
if acct := a_ref():
    acct.withdraw()
```

弱引用通常与缓存及其他高级内存管理技术结合使用。
下面是一个修改后的 `Data` 类型，能够自动移除不存在引用的缓存：

```Python
import weakref

class Date:
    _cache = { }

    def __new__(cls, year, month, day):
        selfref = Date._cache.get((year, month, day))
        if not selfref:
            self = super().__new__(cls)
            self.year = year
            self.month = month
            self.day = day
            Date._cache[year, month, day] = weakref.ref(self)
        else:
            self = selfref()
        return self

    def __init__(self, year, month, day):
        pass

    def __del__(self):
        del Date._cache[self.year, self.month, self.day]
```

支持弱引用要求实例具有可变的 `__weakref__` 属性。
用户定义的类通常默认有该属性。
然而，内部类型和特定的数据类型不支持。
如果想为这些类型创建弱引用，可以通过添加了 `__weakref__` 属性的变体来实现。

```Python
class wdict(dict):
    __slots__ = ('__weakref__', )

w = wdict()
w_ref = weakref.ref(w)
```

## Internal Object Representation and Attribute Binding

与实例关联的状态存储在一个字典中，该字典可通过实际的 `__dict__` 属性访问。
该字典包含每个实例独有的数据，例如这个例子：

```Python
a = Account('Guido', 1100.0)
a.__dict__
# {'owner': 'Guido', 'balance': 1100.0}
```

任何时间都可以添加新属性：

```Python
a.number = 123456  # Add attribute 'number' to a.__dict__
a.__dict__['number'] = 654321
```

对实例的修改总是会反应在其本地的 `__dict__` 属性中，除非该属性由 property 管理。
同样，如果直接修改 `__dict__` 则也会反应在实例属性上。

实例通过一个特殊属性 `__class__` 与类关联。
类本身也只是一个覆盖在字典上的薄层，这个字典可以在其自身的 `__dict__` 属性中找到。
类字段是存放方法的地方，例如：

```Python
a.__class__
# <class '__main__.Account'>

Account.__dict__.keys()
# dict_keys(['__module__', '__init__', '__repr__', 'deposit', 'withdraw', 'inquiry', '__dict__', '__weakref__', '__doc__'])

Account.__dict__['withdraw']
# <function Account.withdraw at 0x108204158>
```

类通过特殊属性 `__base__` 连接到他们的基类 base class，其为一个基类的元组。
`__base__` 属性只用于提供信息。
继承的实际运行实现依赖于 `__mro__` 属性，该属性是一个按搜索顺序排列的所有父类元组。
这一底层结构构成了实例属性进行获取、设置和删除等所有操作的基础。

当属性使用 `obj.name = value`时，会调用特殊方法 `obj.__setattr__('name', value)`。
如果同 `del obj.name` 删除属性，则会调用特殊方法 `obj.__delattr__('name')`。
这些默认方法用于从 obj 对象的 `__dict__` 本地变量中修改或删除对象，除非请求的对象是 property 或 descriptor。
这种情况下，set 和 delete 操作将由与该属性关联的函数执行。

对于属性查找，例如 `obj.name` 会调用特殊方法 `obj.__getattribute('name')`。
该方法执行查找属性的搜索过程，其通常会检查属性，查看本地 `__dict__` 属性，检查类型字典并搜索 MRO。
如果该搜索过程失败，最后会调用类的 `obj.__getattr('name')` 方法。
如果还失败则抛出异常。

用户定义的类型可以根据需要实现自己的属性访问函数。
例如，这里有一个表，它限制了可以设置的属性名称：

```Python
class Account:
    def __init__(self. owner, balance):
        self.owner = owner
        self.balance = balance

    def __setattr__(self, name, value):
        if name not in {'owner', 'balance'}:
            raise AttributeError(f'No attribute {name}')
        super().__setattr__(name, value)

# Example
a = Account('Guido', 1000.0)
a.balance  # OK
a.amount   # AttributeError
```

一个重新实现这些方法的类，应该依赖 `super()` 提供的默认实现来执行属性的实际工作。
这是因为默认实现已经处理了类中更高级的特性，如描述符和属性。
如果不使用 `super()`，则将需要自行处理这些细节。

## Proxies, Wrappers and Delegation

有时，类会围绕另一个对象实现一个包装层，以创建一种代理 proxy 对象。
代理对象保留暴露与另一个对象相同的接口，但由于各种原因，并不通过继承和原始对象关联。
这与组合有所不同，组合是从其他对象创建一个全新的对象，但新对象拥有自己独特的方法和属性。

这有许多种情况，例如，在分布式计算中，真实的对象实现可能在远程云服务器里。
和服务器交互的客户端可能使用和服务器对象一样的代理对象，但实际上其会将所有对象通过网络传递消息。
一个常见的实现技术涉及使用 `__getattr__()` 方法，下面是一个简单的示例：

```Python
class A:
    def spam(self):
        print('A.spam')
    def grok(self):
        print('A.grok')
    def yow(self):
        print('A.yow')

class LoggedA:
    def __init__(self):
        self._a = A()

    def __getattr__(self, name):
        print('Accessing', name)
        # Delegate to internal A instance
        return getattr(self._a, name)

a = LoggedA()
a.spam()  # Accessing spam\n A.spam
a.yow()   # Accessing yow\n A.yow()
```

有时候代理作为继承的替代，例如：

```Python
class A:
    def spam(self):
        print('A.spam')

    def grok(self):
        print('A.grok')

    def yow(self):
        print('A.yow')

class B:
    def __init__(self):
        self._a = A()

    def grok(self):
        print('B.grok')

    def __getattr__(self, name):
        return getattr(self._a, name)

# Example
b = B()
b.spam()  # A.spam
b.grok()  # B.grok
b.yow()   # A.yow
```

在这个例子中，B 看上去和继承了 A 一样，并重定义了一个方法。
这是观察到的行为，但并未使用继承，相反，B 内部持有一个对 A 的引用。
A 的某些方法可以被重新定义，例如，所有其他方法都通过 `__getattr__()` 方法进行委托。

通过 `__getattr__()` 转发属性查找是一种常见技术。
但注意，它不适用与映射到特殊方法的操作：

```Python
class ListLink:
    def __init__(self):
        self._item = list()

    def __getattr__(self, name):
        return getattr(self._items, name)

# Example
a = ListLink()
a.append(1)     # ✓
a.insert(0, 2)  # ✓
a.sort()        # ✓

len(a)  # Fails: No __len__() method
a[0]    # Fails: No __getitem__() method
```

在此例中，该类成功地将所有标准列表方法转发给内部列表。
然而，Python 的标准运算符均无法使用。
要使其生效，必须显示实现所需要的特殊方法：

```Python
class ListLink:
    def __init__(self):
        self._items = list()

    def __getattr__(self, name):
        return getattr(self._items, name)

    def __len__(self):
        return len(self._items)

    def __getitem__(self, index):
        return self._item[index]

    def __setitem__(self, index, value):
        return self._item[index] = value
```

## Reducing momory use with `__slots__`

前面提到过，实例存储在类字典中。
如果要创建大量实例，这会产生大量内存开销。
如果知道属性名称是固定的，可以在一个 `__slots__` 的特殊变量中指定这些名称。

```Python
class Account(object):
    __slots__ = ('owner', 'balance')
    ...
```

Slots 是一种定义提示，允许 Python 对内存和执行速度进行优化。
使用 `__slots__` 的类实例不再使用字典存储数据。
相反，会使用一个更加紧凑的数组类型。
在创建大量对象的程序中，使用 `__slots__` 可以显著减少内存占用，并略微提升执行效率。

1. 使用更加紧凑的数据结构，减少内存占用

2. 寻址方式不同，原来的 dict 需要计算哈希，现在的使用偏移量 offset，速度更快

`__slote__` 中仅包含实例数据属性。
无需列出方法、属性、类变量或其他任何类级属性。
本质上，这些名称与通常出现在实例 `__dict__` 字典中的键名称相同。

要注意在继承使用 `__slots__` 的基类的时候，子类也许要定义 `__slots__`，即使不定义任何自己的属性。
如果不这样会，解释器会默认认为需要动态特性，导致子类变得更慢。

`__slots__` 的存在不会影响诸如 `__getattribute__()`、`__getattr__()` 和 `__setattr__()` 等方法在类中被重新定义的调用。
然而，如果要实现这些方法，注意实例不再拥有 `__dict__` 属性。

## Descriptors

通常情况下，属性访问对应字典操作。
如果需要更加精细的控制，可通过用户自定义的获取、设置和删除函数路由属性访问。
属性的使用依赖于称为描述符 descriptors 的底层构造。
描述符 descriptors 是管理属性访问的类级对象，通过实现 `__get__()`、`__set__()` 或 `__delete__()` 等特殊方法，可以直接接入属性访问机制并定制相关操作。

下面是个例子：

```Python
class Typed:
    excepted_type = object

    def __set_name__(self, cls, name):
        self.key = name

    def __get__(self, instance, cls):
        if instance:
            return instance.__dict__[self.key]
        else:
            return self

    def __set__(self, instance, value):
        if not isinstance(value, self.excepted_type):
            raise TypeError(f'Excepted {self.excepted_type}')
        isinstance.__dict__[self.key] = value

    def __delete__(self, instance):
        raise AttrubuteError('Can\'t delete attribute')

class Integer(Typed):
    excepted_type = int

class Float(Typed):
    excepted_type = float

class String(Typed):
    excepted_type = str

# Example
class Account:
    owner = String()
    balance = Float()

    def __init__(self, owner, balance):
        self.owner = owner
        self.balance = balance
```

在这个例子中，`Typed` 类定义了一个描述符，当属性被赋值时进行类型检查。
如果尝试删除该属性则会引发错误。
`Inreger`、 `Float` 和 `String` 子类则特化了 `Typed` 以匹配特定类型。
在另一个类中使用这些类，会使这些属性在访问时自动调用相应的 `__get__()`、`__set__()` 或 `__delete__()` 方法。

例如：

```Python
a = Account('Guido', 1000.0)
b = a.owner      # Calls Account.owner.__get__(a, Account)
a.owner = 'Eva'  # Calls Account.owner.__set__(a, 'Eva')
del a.owner      # Calls Account.owner.__delete__(a)
```

描述符只能在类级别实例化。
在 `__init__()` 或其他方法内部创建描述符对象，从而基于每个实例创建描述符，这种做法是不合法的。

描述符的 `__set_name__()` 方法会在类定义完成后、任何实例创建之前被调用。
它的作用是告知描述符该属性在类中使用的名称。

例如，`balance = Float()` 定义会调用 `Float.__set_name__(Account, 'balance)` 并通知 descriptor 其所属的类及使用的名称。

---

这部分越来越难懂了，跳过 ing ...

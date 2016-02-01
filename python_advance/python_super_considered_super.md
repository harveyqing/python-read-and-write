# Python’s super() considered super! #

被项目中的一个多重继承坑了一个多小时，于是决定再翻翻[这篇](https://rhettinger.wordpress.com/2011/05/26/super-considered-super/)经典文章，按大致意思翻译了一下。翻译的不太好，英文的表现力有时比中文更强大！如果英语水平足够，建议直接读原文。

如果你没有被Python的super()惊愕过，那么要么是你不了解它的威力，要么就是你不知道如何高效地使用它。

有许多介绍super()的文章，这一篇与其它文章的不同之处在于:

* 提供了实例
* 阐述了它的工作模型
* 展示了任何场景都能使用它的手段
* 有关使用super()的类的具体建议
* 基于抽象ABCD[钻石模型](http://en.wikipedia.org/wiki/Diamond_problem)的实例

下面是一个使用Python 3语法，扩展了builtin类型`dict`中方法的子类：


```python
import pprint
import logging
import collections

class LoggingDict(dict):
    
    def __setitem__(self, key, value):
        logging.info('Setting %r to %r' % (key, value))
        super().__setitem__(key, value)
```

`LoggingDict`继承了父类*dict*的所有特性，同时其扩展了`__setitem__`方法来记录被设置的key；在记录日志之后，该方法用`super()`将真正的更新操作代理给其父类。

我们可以使用`dict.__setitem__(self, key, value)`来完成`super()`的功能，但是`super()`更优，因为它是一个计算出来的间接引用。

`间接`的一个好处是，我们不需要使用名字来指定代理类。如果你将基类换成其它映射(mapping)类型，`super()`引用将会自动调整。你只需要一份代码:


```python
class LoggingDict(someOtherMapping):                     # 新的基类
    
    def __setitem__(self, key, value):
        logging.info('Setting %r to %r' % (key, value))
        super().__setitem__(key, value)                  # 无需改变
```

对于计算出的间接引用，其除了隔离变化外，依赖于Python的动态性，可以在运行时改变其指向的class。

计算取决于类在何处被调用以及实例的继承树；super在何处调用取决于类的源码，在上例中`super()`是在`LoggingDict.__setitem__`方法中被调用的；实例的继承树在后文详述。

下面先构造一个有序的logging字典：


```python
class LoggingOD(LoggingDict, collections.OrderedDict):
    pass
```

新class的继承树是：`LoggingOD`, `LoggingDict`, `OrderedDict`, `dict`, `object`。出人意料的是`OrderedDict`竟然介于`LoggingDict`之后和`dict`之前，这意味着`LoggingDict.__setitem__`中`super()`调用会将键/值的更新委托给`OrderedDict`而不是`dict`。

在上例中，我们并没有修改`LoggingDict`的源码，只是创建了一个子类，这个子类的唯一逻辑是组合两个已有的类并控制它们的搜索顺序(search order)。

## Search Order ##

上面提到的`搜索顺序`或者`继承树`的官方称谓是`Method Resolution Order`（方法解析顺序）即`MRO`。可以用`__mro__`属性方便地打印出对象的MRO：


```python
pprint.pprint(LoggingOD.__mro__)
```

    (<class '__main__.LoggingOD'>,
     <class '__main__.LoggingDict'>,
     <class 'collections.OrderedDict'>,
     <class 'dict'>,
     <class 'object'>)


如果我们想创建出其MRO符合我们意愿的子类，就必须知道它是如何计算的。MRO的计算很简单，MRO序列包含类、类的基类以及基类们的基类......这个过程持续到到达`object`，`object`是所有类的根类；这个序列中，子类总是出现在其父类之前，如果一个子类有多个父类，父类按照子类定义中的基类元组的顺序排列。

上例中MRO是按照这些约束计算出来的:

* LoggingOD在其父类LoggingDict, OrderedDict之前
* LoggingDict在OrderedDict之前是因为LoggingOD.__bases__是(LoggingDict, OrderedDict)
* LoggingDict在它的父类dict之前
* OrderedDict在它的父类dict之前
* dict在它的父类object之前
    
解析这些约束的过程称为线性化(linearization)。创建出MRO符合我们期望的子类只需知道两个约束：子类在父类之前；符合`__bases__`里的顺序。

## Practical Advice ##

super()用来将方法调用委托给其继承树中的一些类。为了让super能正常作用，类需要协同设计。下面是三条简单的解决实践：

* 被调用的super()需存在
* 调用者和被调用者的参数签名需匹配
* 方法的任何出现都需要使用super()

1) 先来看一下使调用者和被调者参数签名匹配的策略。对于一般的方法调用而言，被调者在被调之前其信息是已经获知的；然而对于super()，直到运行时才能确定被调者（因为后面定义的子类可能会在MRO中引入新的类）。

一种方法是使用positional参数固定签名。这种方法对于像`__setitem__`这种只有两个参数的固定签名是适用的。`LoggingDict`例子中`__setitem__`的签名和`dict`中一致。

另一种更灵活的方法是规约继承树中的每一个方法都被设计成接受keyword参数和一个keyword参数字典，“截留住”自身需要的参数，然后将剩下的参数使用`**kwds`转发至父类中的方法，使得调用链中的最后一次调用中参数字典为空（即沿着继承树一层一层地将参数剥离，每层都留下自己需要的，将余下的参数传递给基类）。

每一层都会剥离其所需的参数，这样就能保证最终将空字典传递给不需要参数的方法（比如，`object.__init__`不需要参数）：


```python
class Shape:
    
    def __init__(self, shapename, **kwds):
        self.shapename = shapename
        super().__init__(**kwds)
        
        
class ColoredShape(Shape):
    
    def __init__(self, color, **kwargs):
        self.color = color
        super().__init__(**kwargs)
        
cs = ColoredShape(color='red', shapename='circle')

```

2) 现在来看一下如何保证目标方法存在。

上例仅展示了最简单的情形。我们知道*object*有一个`__init__`方法，它也总是MRO链中最后一个类，因此任意数量的`super().__init__`最终都会以调用`object.__init__`结束。换言之，我们可以保证调用继承树上任意对象的`super().__init__`方法都不会以产生*AttributeError*而失败。

对于*object*没有的方法（比如draw()方法），我们需要写一个根类并保证它在*object*对象之前被调用。根类的作用仅仅是将方法调用“截住”而不会再进一步调用super()。

*Root.draw*也可以使用[defensive programming](http://en.wikipedia.org/wiki/Defensive_programming)策略使用断言来保证*draw()*方法不会再被调用。


```python
class Root:
    
    def draw(self):
        #: 代理调用链止于此
        assert not hasattr(super(), 'draw')
        
        
class Shape(Root):
    
    def __init__(self, shapename, **kwds):
        self.shapename = shapename
        super().__init__(**kwds)
        
    def draw(self):
        print('Drawing. Setting shape to:', self.shapename)
        super().draw()
        
        
class ColoredShape(Shape):
    
    def __init__(self, color, **kwds):
        self.color = color
        super().__init__(**kwds)
        
    def draw(self):
        print('Drawing. Setting color to:', self.color)
        super().draw()
        
cs = ColoredShape(color='blue', shapename='square')
cs.draw()
```

    Drawing. Setting color to: blue
    Drawing. Setting shape to: square


如果子类想在MRO中注入其它类，那么这些类也需要继承自*Root*，这样在继承路径上的任何类调用*draw()*方法都不会最终代理至*object*而抛出*AttributeError*。这一规定需在文档中明确，这样别人在写新类的时候才知道需要继承*Root*。这一约束与Python中要求所有异常必须继承自*BaseException*并无不同。

3) 上面讨论的两点保证了方法的存在以及签名的正确，然而我们还必须保证在代理链上的每一步中都*super()*都被调用。这一目标很容易达成，只需要协同设计每一个相关类——在代理链上的每一步中增加一个*supper()*

## How to Incorporate a Non-cooperative Class ##

在某些场景下，子类可能希望使用多重继承，其大部分父类都是协同设计的，同时也需要继承自一个第三方类（可能将要使用的方法没有使用`super`或者该类没有继承自根类）。这种情形很容易通过使用[adapter class](http://en.wikipedia.org/wiki/Adapter_pattern)来解决。

例如，下面的*Moveable*类没有调用super()，*__init__()*函数签名与*object.__init__*也不兼容，而且它不集成自*Root*：


```python
class Moveable:
    
    def __init__(self, x, y):
        self.x = x
        self.y = y
        
    def draw(self):
        print('Drawing at position:', self.x, self.y)
```

如果我们想将这个类与之前协同设计的*ColoredShape*层级一起使用的话，我们需要创建一个适配器(adapter)，它调用了必须的*super()*方法。


```python
class MoveableAdapter(Root):
    
    def __init__(self, x, y, **kwds):
        self.moveable = Moveable(x, y)
        super().__init__(**kwds)
        
    def draw(self):
        self.moveable.draw()
        super().draw()

        
class MoveableColoredShape(ColoredShape, MoveableAdapter):
    pass


MoveableColoredShape(color='red', shapename='triangle', x=10, y=20).draw()
```

    Drawing. Setting color to: red
    Drawing. Setting shape to: triangle
    Drawing at position: 10 20


## Complete Example - Just for Fun

在Python 2.7和3.2中，`collections`模块有一个`Counter`类和一个`OrderedDict`类，可以将这两个类组合产生一个`OrderedCounter`类：


```python
from collections import Counter, OrderedDict


class OrderedCounter(Counter, OrderedDict):
    
    'Counter that remembers the order elements are first seen'
    def __repr__(self):
        return '%s(%r)' % (self.__class__.__name__,
                           OrderedDict(self))
    
    def __reduce__(self):
        return self.__class__, (OrderedDict(self),)
    
oc = OrderedCounter('abracadabra')
pprint.pprint(oc)
```

    OrderedCounter(OrderedDict([('a', 5), ('b', 2), ('r', 2), ('c', 1), ('d', 1)]))


## Notes and References ##

* 当子类化诸如*dict()*的builtin时，通常需要同时重载或者扩展多个方法。在上面的例子中，*\__setitem__*扩展不能用于诸如*dict.update*等其它方法，因此可能需要扩展这些方法。这一需求不止限于*super()*，子类化builtins时都需要。

* 在多继承中，如果要求父类按照指定顺序（例如，*LoggingOD*需要*LoggingDict*在*OrderedDict*前面，而*OrderedDict*在*dict*前面），可以利用断言来验证或者使用文档来表明方法解析顺序：


```python
position = LoggingOD.__mro__.index
assert position(LoggingDict) < position(collections.OrderedDict)
assert position(OrderedDict) < position(dict)
```

* 关于线性化算法可以参阅[Python MRO documentation](http://www.python.org/download/releases/2.3/mro/)和[Wikipedia entry for C3 Linearization](http://en.wikipedia.org/wiki/C3_linearization)。

* [Dylan编程语言](http://en.wikipedia.org/wiki/Dylan_(programming_language))有一个像Python的super()的*next-method*方法，可以参见[Dylan's class docs](http://www.opendylan.org/books/dpg/db_347.html)来了解它如何工作。

* 上面例子中使用的是Python 3的super()，Python 2的语法与之不同之处在于其`type`和`object`参数须是显式的，另外Python 2中的super()只适用于新式类。

【译】Python描述符指南
======================


| 作者       |    Raymond Hettinger     |
| -----------|:------------------------:|
| 联系       | <python at rcn dot com   |

**目录**

* Python描述符引导
 * <a href="#abstract" id="id2">摘要</a>
 * <a href="#definition-and-introduction" id="id3">定义和介绍</a>
 * <a href="#descriptor-protocol" id="id4">描述符协议</a>
 * <a href="#invoking-descriptors" id="id5">调用描述符</a>
 * <a href="#descriptor-example" id="id6">描述符实例</a>
 * <a href="#properties" id="id7">属性 (Properties)</a>
 * <a href="#functions-and-methods" id="id8">函数和方法</a>
 * <a href="#static-methods-and-class-methods" id="id9">静态方法和类方法</a>

<a href="#id2" id="#abstrace">摘要</a>
----

定义描述符，总结协议，并展示如何调用描述符。审查一个自定义描述符和包括函数、属性、静态方法以及类方法在内的一些Python内置描述符，通过给出一个纯Python实现和样例代码段来展示它们是如何运作的。

学习描述符不仅扩充了你的Python工具集，它还能加深你对Python的理解并体会到Python设计的优雅。

<a href="#id3" id="definition-and-introduction">定义和介绍</a>
---

一般来说，描述符是一个具有``绑定行为``的对象属性，其属性的访问被描述符协议方法覆写。这些方法是``__get__()``、 ``__set__()``和``__delete__()``，一个对象中只要包含了这三个方法（译者注：包含至少一个），就称它为描述符。

属性访问的默认行为是从一个对象的字典中获取 (get)、设置 (set)、删除 (delete) 属性。例如：``a.x`` 的查找链始于 ``a.__dict__['x']``，然后是 ``type(a).__dict__['x']``，然后是 ``type(a)`` 除元类之外的基类（译者注：如果继承树很深，可能会访问多个基类）。如果查找到的值是包含一个描述符方法的对象，那么Python可能会重写（该对象）的默认行为并调用那个描述符方法。注意只有在新式对象或者新式类（继承自``object``或者``type``）中描述符才会被调用。

描述符是一个功能强大、通用的协议。它们是属性、方法、静态方法、类方法、``super()``背后的实现机制。它们被广泛使用于Python 2.2中用来实现新式类。描述符简化了底层的C代码并为Python编程提供了一套灵活的新工具。

<a href="#id4" id="descriptor-protocol">描述符协议</a>
---

``descr.__get__(self, obj, type=None) --> value``
``descr.__set__(self, obj, value) --> None``
``descr.__delete__(self, obj) --> None``

这些就是描述符协议。定义这些方法中的任一个，对象就被认为是描述符并且在其被当做属性访问时调用相应的描述符方法。

如果一个对象同时定义了``__get__()``和``__set__()``方法，它就成为了一个**资料描述符 (data descriptor) **。只定义了``__get__()``方法的描述符成为非资料描述符（它们通常用于方法，但是也有其它用处）。

资料和非资料描述符的区别在于访问实例时获取结果的顺序。如果一个实例字典中有一个和资料描述符同名的项，那么访问时优先访问到资料描述符。反之，如果实例字典中有一个和非资料描述符同名的项，那么优先访问到的是字典项。

为了构造一个只读的资料描述符，需同时定义``__get__()``和``__set__()``方法并且在``__set__()``方法中抛出``AttributeError``异常。在``__set()__``方法中加上抛出异常的逻辑就足够让一个对象成为资料描述符。

<a href="#id5" id="invoking-descriptors">调用描述符</a>
---

可以通过直接调用方法名来调用描述符。例如：``d.__get__(obj)``。

另外，描述符更常见的调用场景是属性访问时描述符被自动调用。比如：``obj.d`` 在``obj``的字典中查找``obj.d``。如果``d``定义了``__get__()``方法，那么``d.__get__(obj)``会依据如下的优先规则被调用。

调用的细节取决于``obj``是一个对象还是类。

对于对象来说，关键在于``object.__getattribute__()``，它将``b.x``转换为``type(b).__dict__['x'].__get__(b, type(b))``。这种实现依据这样的一个优先链：资料描述符优先于实例变量，实例变量优先于非资料描述符，如果对象包含``__getattr__()``方法，那么这个方法 (``__getattr__()``)的访问优先级最低。完整的C实现在[Objects/object.c](https://hg.python.org/cpython/file/3.4/Objects/object.c)中[PyObject_GenericGetAttr()](https://docs.python.org/3/c-api/object.html#c.PyObject_GenericGetAttr)。

而对于类来讲，关键之处在于``type.__getattribute__()``方法，它将``B.x``转换为``B.__dict__['x'].__get__(None, B)``。如果用纯Python描述的话，它看起来像这样:

``` Python
def __getattribute__(self, key):
    "Emulate type_getattro() in Objects/typeobject.c"
    v = object.__getattribute__(self, key)
    if hasattr(v, '__get__'):
        return v.__get__(None, self)
    return v
```

这里应该记住的要点是：

* 描述符被``__getattribute__()``方法调用
* 重写``__getattribute__()``方法会阻止描述符的自动调用
* ``object.__getattribute__()``和``type.__getattribute__()``对``__get__()``方法的调用不一样
* 资料描述符总是覆写实例字典
* 非资料描述符可能被实例字典覆写
*

``super()``方法返回的对象也有一个定制的[``__getattribute__()``](https://docs.python.org/3/reference/datamodel.html#object.__getattribute__)方法来调用描述符。调用``super(B, obj).m()``时，会先在``obj.__class__.__mro__``中查找到``B``最邻近（译者注：即``B``的直接父类）的基类``A``，然后返回``A.__dict__['m'].__get__(obj, B)``。如果得到的不是一个描述符，``m``会被原封不动地返回。如果``m``不在字典中，会回溯并用``object.__getattribute__()``方法来查找``m''。

实现细节在[Objects/typeobject.c](https://hg.python.org/cpython/file/3.4/Objects/typeobject.c)中的``super_getattro()``中，纯Python的等价实现可以在[Guido's Tutorial](https://www.python.org/download/releases/2.2.3/descrintro/#cooperation)中找到。

上述细节展示了描述符的机制是在``object``、``type``和``super()``方法中嵌入``__getattribute__()``方法。当类派生自``object``或者其元类提供类似功能时它也继承这一机制。同样，类可以通过覆写``__getattribute__()``方法来关闭描述符调用。

<a href="#id6" id="descriptor-example">描述符实例</a>
---

下面的示例代码创建了一个类，它的对象是资料描述符，这些资料描述符在每次get和set的时候都会打印出一条信息。覆写每个对象属性的``__getattribute__()``方法是实现这种效果的另外一种方法，然而描述符可以用来选择性地监视特定的属性:

``` Python
class RevealAccess(object):
    """A data descriptor that sets and returns values
    normally and prints a message logging their access.
    """

    def __init__(self, initval=None, name='var'):
        self.val = initval
        self.name = name

    def __get__(self, obj, objtype):
        print('Retrieving', self.name)
        return self.val

    def __set__(self, obj, val):
        print('Updating', self.name)
        self.val = val

>>> class MyClass(object):
    x = RevealAccess(10, 'var "x"')
    y = 5

>>> m = MyClass()
>>> m.x
Retrieving var "x"
10
>>> m.x = 20
Updating var "x"
>>> m.x
Retrieving var "x"
20
>>> m.y
5
```

描述符协议非常简单，但是它却提供了令人惊喜的可能。一些非常常见的用例已经被集成到函数调用中。属性、绑定和非绑定方法、静态方法、类方法等全部都是基于描述符协议的。

<a href="#id7" id="properties">属性</a>
---

调用[property()](https://docs.python.org/3/library/functions.html#property)方法是创建资料描述符的一种简洁方法，它保证了访问属性时触发相应的函数调用。它的签名如下：

``` Python
property(fget=None, fset=None, fdel=None, doc=None) -> property attribute
```

文档显示了一个定义管理属性``x``的典型使用场景：

``` Python
class C(object):
    def getx(self): return self.__x
    def setx(self, value): self.__x = value
    def delx(self): del self.__x
    x = property(getx, setx, delx, "I'm the 'x' property.")
```

下面的纯Python实现展示了如何用描述符实现[property()](https://docs.python.org/3/library/functions.html#property)方法：

``` Python
class Property(object):
    "Emulate PyProperty_Type() in Objects/descrobject.c"

    def __init__(self, fget=None, fset=None, fdel=None, doc=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel
        if doc is None and fget is not None:
            doc = fget.__doc__
        self.__doc__ = doc

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        if self.fget is None:
            raise AttributeError("unreadable attribute")
        return self.fget(obj)

    def __set__(self, obj, value):
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(obj, value)

    def __delete__(self, obj):
        if self.fdel is None:
            raise AttributeError("can't delete attribute")
        self.fdel(obj)

    def getter(self, fget):
        return type(self)(fget, self.fset, self.fdel, self.__doc__)

    def setter(self, fset):
        return type(self)(self.fget, fset, self.fdel, self.__doc__)

    def deleter(self, fdel):
        return type(self)(self.fget, self.fset, fdel, self.__doc__)
```

[property()](https://docs.python.org/3/library/functions.html#property)内建方法能实现这样的一种诉求：当对一个用户接口授权了属性访问之后，需要一个能改变访问的方法。

例如：一个电子表格类可能通过``Cell('b10').value``来访问一个单元格。之后，程序需要改进，需要在每次访问时重新计算一次单元格的值；然而，码农们不想影响到客户端直接访问属性的代码。此时的解决方案是将对值属性的访问包装在一个属性资料描述符中:

``` Python
class Cell(object):
    ...
    def getvalue(self, obj):
        "Recalculate cell before returning value"
        self.recalc()
        return obj._value
    value = property(getvalue)
```

<a href="#id8" id="functions-and-methods">函数和方法</a>
---

Python面向对象特性的建立是基于一个函数环境的。利用非资料描述符，这两者（译者注：指OO和函数）无缝结合在一起了。

类字典将方法存作函数。在类定义中，方法是用``def``和``lambda``来定义的，这和定义函数是一样的。唯一不同的是，在类的方法定义中，第一个参数被保留用来指代对象实例。依据Python的惯例，实例引用被称为*self*，但是它可以是诸如*this*等的任意其它变量名。

为了支持方法调用，函数包括了[\_\_get\_\_()](https://docs.python.org/3/reference/datamodel.html#object.__get__)方法来在属性访问时绑定方法。这意味着所有的函数都是非资料描述符，它们依据其是从对象还是类中被调用来返回绑定(``bound``)或非绑定(``unbound``)方法。下述代码是一个纯Python实现：

``` Python
class Function(object):
    ...
    def __get__(self, obj, objtype=None):
        "Simulate func_descr_get() in Objects/funcobject.c"
        return types.MethodType(self, obj, objtype)
```

运行解释器展示实际情况下函数描述符是如何工作的：

``` Python
>>> class D(object):
        def f(self, x):
            return x

>>> d = D()
>>> D.__dict__['f']  #: 内部存储为一个函数
<function f at 0x00C45070>
>>> D.f  #: 从类中获取会变为非绑定方法
<unbound method D.f>
>>> d.f  #: 从实例中获取会变为绑定方法
<bound method D.f of <__main__.D object at 0x00B181C90>>
```

输出表明绑定和非绑定方法是两种不同的类型。它们可能像上述那样实现，[PyMethod_Type](https://docs.python.org/3/c-api/method.html#c.PyMethod_Type)的C实现在[Objects/classobject.c](https://hg.python.org/cpython/file/3.4/Objects/classobject.c)中，它是有两种不同表现形式的对象，根据``im_self``字段存在或*NULL*（C中等价于*None*的东西对象）而表现不同。

同样的，调用一个方法对象的效果取决于``im_self``字段。如果``im_self``被设定了（意味着绑定），原始函数（存储在``im_func``中）在被调用时其第一个参数必须被设置为一个实例（译者注：即原始函数必须得绑定到一个实例）。如果没有被绑定，所有参数都会原封不动地传递给原始函数（译者注：这意味着你可以通过直接调用类里面的方法，但是``self``这个参数可以是任意的已定义量）。实际上``instancemethode_call()``的C实现只是稍微复杂点儿，它包含了一些类型检查的逻辑。

<a href="#id9" id="static-methods-and-class-methods">静态方法和类方法</a>
---

非资料描述符为将函数绑定成方法这种常见模式提供了一个简单的机制。

回顾一下，函数有一个[\_\_get\_\_()](https://docs.python.org/3/reference/datamodel.html#object.__get__)方法，当函数被当做属性访问时它将函数转成了方法。非资料描述符将``obj.f(*args)``调用转换成``f(obj, *args)``；而调用``klass.f(*args)``将变成``f(*args)``。

下面的表格总结了绑定和它最有用的两种变种：

|   转换   |   从对象中调用    |   从类中调用   |
|:--------:|:-----------------:|:--------------:|
|   函数   |f(obj, \*args)     |f(\*args)       |
|静态方法  |f(\*args)          |f(\*args)       |
|类方法    |f(type(obj), *args)|f(klass, *args) |

静态方法会毫无更改地返回底层函数。调用``c.f``和``C.f``分别等价于直接调用``object.__getattribute__(c, "f")``和``object.__getattribute__(C, "f")``。这样，无论是从一个对象还是一个类中，这个函数都可以正确地访问到。

静态方法(``static methods``)最好的实现是利用不引用``self``变量的方法。

例如：一个统计包可能包含一个实验数据的容器来。该类提供了正常的方法来计算平均值、中位数以及其它的基于数据的描述性统计方法。然而，这个类可能有一些概念上相关却不依赖于类提供的数据的函数。比如：``erf(x)``是一个统计中方便的转换函数，它不直接依赖于特定的数据集，它可以从对象或者类中调用（对象调用``s.erf(1.5) --> .9332``，类调用``Sample.erf(1.5) --> .9332``)。

由于``staticmethod``将底层函数原封不动地返回，因此下面的示例调用也平淡无奇：

``` Python
>>> class E(object):
        def f(x):
            print(x)
        f = staticmethod(f)

>>> print(E.f(3))
3
>>> print(E().f(3))
3
```

利用非资料描述符协议，[staticmethod()](https://docs.python.org/3/library/functions.html#staticmethod)的纯Python实现看起来像这样：

```Python
class StaticMethod(object):
    "Emulate PyStaticMethod_Type() in Objects/funcobject.c"

    def __init__(self, f):
        self.f = f

    def __get__(self, obj, objtype=None):
        return self.f
```

与静态方法不一样的是，类方法在调用函数之前将到该类的引用至于参数列表之前（译者注：即将该类的引用作为函数的第一个参数）。无论调用者是对象还是类，其格式是一样的：

``` Python
>>> class E(object):
        def f(klass, x):
            return klass.__name__, x
        f = classmethod(f)

>>> print(E.f(3))
('E', 3)
>>> print(E().f(3))
('E', 3)
```

当函数只需要有一个类引用并不关心任何底层数据时，这种行为非常有用。类方法的一种作用是创建多种类构造器。在Python 2.3中，``dict.fromkeys()``类方法从一列键中创建了一个新字典。等价的纯Python实现是：

``` Python
class Dict(object):
    ...
    def fromkeys(klass, iterable, value=None):
        "Emulate dict_fromkeys() in Objects/dictobject.c"
        d = klass()
        for key in iterable:
            d[key] = value
        return d
    fromkeys = classmethod(fromkeys)
```

现在可以像这样构建一个唯一键的新字典：

``` Python
>>> Dict.fromkeys('abracadadra')
{'a': None, 'r': None, 'b': None, 'c': None, 'd': None}
```

利用非资料描述符协议，[classmethod](https://docs.python.org/3/library/functions.html#classmethod)的纯Python实现看起来像这样：

``` Python
class ClassMethod(object):
    "Emulate PyClassMethod_Type() in Objects/funcobject.c"

    def __init__(self, f):
        self.f = f

    def __get__(self, obj, klass=None):
        if klass is None:
            klass = type(obj)
        def newfunc(*args):
            return self.f(klass, *args)
        return newfunc
```

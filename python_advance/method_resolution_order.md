# Method Resolution Order #

本文整理自Python她爹Guido van Rossum所述的Python“正史”，戳[原文链接](http://python-history.blogspot.ru/2010/06/method-resolution-order.html)

在支持多继承(multiple inheritance)的语言中，查找方法时检索基类的顺序通常称为``方法解析顺序``，即``MRO``。（在Python中，这个定义也适用于其它属性）。对于仅支持单继承(single inheritance)的语言而言，讨论MRO并无意义；但讨论到多继承时，MRO算法的选择却是值得权衡的。Python历代版本中至少使用过三类不MRO算法：``classic``、Python 2.2的``new-style``以及Python 2.3的``new-style``（即``C3``）。在Python 3中仅使用了后者。

“古典类”(Classic classes)使用了一个简单的MRO策略：**在查找一个方法的时候，使用简单的``深度优先``，``自左向右``的查找策略；最先匹配到的对象将作为查找结果返回。**

考虑在下面这个例子中:

``` Python
class A:
    def save(self): pass


class B(A): pass


class C:
    def save(self): pass


class D(B, C): pass
```

现在实例化一个类D的对象x，那么classic的mro会按照``D -> B -> A -> C``的顺序排序这几个类。如此一来，查找方法x.save()将会返回A.save()，而不是C.save()。这种策略适用于比较简单的多继承情况，但是情形复杂时其问题就显现出来了。“钻石继承(diamond inheritance)”（注：继承关系构成一个菱形）就是不能用简单策略解决的情形之一：

``` Python
class A:
    def save(self): pass


class B(A): pass


class C(A):
    def save(self): pass


class D(B, C): pass
```

在这里，类D继承自类B和C，而类B和C均继承自类A。使用classic MRO时，将会以``D -> B -> A -> C -> A``的顺序来查找方法。这样调用x.save()时，会和前例一样，调用A.save()；然而，在这一场景下，该结果并不是你想要的。因为既然类B和C都继承自类A，有人会觉得在类C中重新定义的save方法才是其想调用的，因为这种情况下，类C中的save方法被视为更为合适（事实上，它可能都会调用A.save()）。例如，如果save()方法被用来保存某个对象的状态，没有调用C.save()可能会打乱程序执行结果，因为C的状态被忽略了。

在使用古典类的Python版本中，上述“钻石继承”的多继承情形并不多见。然而，在新式类(new-style classes)中，这种情形却是相当普遍；因为所有的新式类都会继承自``object``基类。这样，所有多继承的新式类总会创建前述的钻石关系。例如：

``` Python
class B(object): pass


class C(object):
    def __setattr__(self, name, value): pass


class D(B, C): pass
```

另外，由于对象经常会扩展一些诸如__setattr__()等方法，此时方法的解析顺序变得尤为重要。在上例中，方法C.__setattr__应当应用到类D的实例上。

在Python 2.2中，为了修复新式类的mro问题，在类定义的时候会将预先计算(pre-computed)好的的MRO设置为类的属性。官方文档显示MRO的计算同以往一样，**使用``深度优先、自左向右``的遍历方式；在遍历中遇到重复的类时，在MRO列表中只保留重复项的最后一项**。按照这一规则，前述例子中，解析顺序``D -> B -> A -> C -> A``将会变为``D -> B -> C -> A``。

事实上，MRO的计算比官方文档所述要复杂得多。在某些情况下，新的MRO算法并不起作用。
来看一个例子：

``` Python
class A(object): pass
class B(object): pass
class X(A, B): pass
class Y(B, A): pass
class Z(X, Y): pass
```

在这个例子中，类X和类Y都继承了类A和B，但是顺序并不一致，并且类A和B均继承自基类object。如果使用前述的MRO算法，那么这些类的MRO将会是``Z -> X -> Y -> B -> A -> object``（这里`object`是所有类的基类）。注意到此时类B和类A是逆序的，真实的MRO会交换类A和类B的顺序，所以MRO应该为``Z -> X -> Y -> A -> B -> object``。直观地来说，在搜索过程中，该MRO算法会为先出现的类的基类保留顺序。在上例的Z类中，类X会被先检查，因为类A是继承列表中的第一个基类，又因为类X继承自类A和B，MRO算法会尝试保留这一顺序。这就是Python 2.2的mro实现，但是文档却没有更新。

然而，在Python 2.2中引入新式类后不久，Samuele Pedroni发现了一处文档描述的MRO算法与实际代码中观察结果的不一致。另外，即使代码与上述描述的特殊案例不同，MRO规则与预期不一致的情况也会出现。经过讨论，Python社区决定舍弃Python 2.2中使用的MRO，而采用``C3 Linearization algorithm``，该算法出自``A Monotonic Superclass Linearization for Dylan`` (K. Barrett, et al, presented at OOPSLA'96)这篇论文。

实际上，在Python 2.2的MRO算法中，主要考虑了单调性(monotonicity)。在一个复杂的继承层级关系中(inheritance hierarchy)，每个继承关系都定义了一个类查找顺序的简单规则集合。具体来说，如果类A继承自类B，那么MRO应该先检查A，然后检查B；同样的，如果类B继承了类C和D，那么类B会先于类C被检查，类C会先于类D。

在一个复杂的继承层级中，你可能想以某种单调的方法来满足所有可能的规则。即，如果你已经规定了类A必须先于类B被检查，那么你就不会遇到需要先检查类B再检查类A的情形（否则，结果就是未定义的，这个基层关系会被拒绝）。这就是为什么使用C3算法而不用原来的MRO算法。粗略地讲，C3背后的原理是：当你写下一个复杂类层级的检查顺序规则时，C3算法会决定满足这些规则的单调顺序；如果不能找出这个顺序，那个该算法失败。

Python 2.3使用了C3算法。这一改变使得Python会拒绝任何与基类顺序不一致的继承层级。例如，在前面的例子中，类X和类Y的顺序规则有冲突。对类X来说，规则表明类A应该先于类B被检查；而对于类Y来讲，规则则要求类B先被检查。单独使用时，这些差异不会引起问题，但是如果在其它类（如示例中的类Z）中同时继承了类X和Y，C3算法就会拒绝这一行为。这与Python之禅中的``errors should never pass silently``吻合。

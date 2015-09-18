# 如何在Python中创建一个可迭代函数（或者可迭代对象）
问题 [链接](http://stackoverflow.com/questions/19151/build-a-basic-python-iterator)

Python里面的迭代对象遵从迭代器协议，这意味着它们都会提供两个方法:<code>__iter__()</code>和<code>next()</code>。<code>__iter__()</code>将返回迭代对象，且在循环开始时被隐式调用。<code>next()</code>方法返回下一个值，且在每次循环推进时被隐式调用。当迭代器对象中没有可返回值时，<code>next()</code>方法会抛出*StopIteration*异常，这个异常会被循环捕捉以终止循环。

下面是一个计数器的例子：

``` Python
class Counter:
    def __init__(self, low, high):
        self.current = low
        self.high = high

    def __iter__(self):
        return self

    def next(self):  # Python 3: def __next__(self)
        if self.current > self.high:
            raise StopIteration
        else:
            self.current += 1
            return self.current - 1

for c in Counter(3, 8):
    print c
```

这段代码将会输出：

``` Python
3
4
5
6
7
8
```

下面是一个更简单的方法:

``` Python
def counter(low, high):
    current = low
    while current <= high:
        yield current
        current += 1

for c in counter(3, 8):
    print c
```

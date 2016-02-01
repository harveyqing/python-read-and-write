Python线程同步机制
========


同步访问共享资源
---

在使用线程的时候，一个很重要的问题是要避免多个线程对同一变量或其它资源的访问冲突。一旦你稍不留神，重叠访问、在多个线程中修改（共享资源）等这些操作会导致各种各样的问题；更严重的是，这些问题一般只会在比较极端（比如高并发、生产服务器、甚至在性能更好的硬件设备上）的情况下才会出现。

比如有这样一个情况：需要追踪对一事件处理的次数

``` Python
counter = 0

def process_item(item):
    global counter
    ... do something with item ...
    counter += 1
```

如果你在多个线程中同时调用这个函数，你会发现``counter``的值不是那么准确。在大多数情况下它是对的，但有时它会比实际的少几个。

出现这种情况的原因是，``计数增加操作实际上分三步执行``:

* 解释器获取counter的当前值
* 计算新值
* 将计算的新值回写counter变量

考虑一下这种情况：在``当前线程``获取到counter值后，另一个线程抢占到了CPU，然后同样也获取到了counter值，并进一步将counter值重新计算并完成回写；之后时间片重新轮到``当前线程``（这里仅作标识区分，并非实际当前），此时``当前线程``获取到counter值还是原来的，完成后续两步操作后counter的值实际只加上1。

另一种常见情况是访问不完整或不一致状态。这类情况主要发生在一个线程正在初始化或更新数据时，另一个进程却尝试读取正在更改的数据。

原子操作
---

实现对共享变量或其它资源的同步访问最简单的方法是依靠解释器的原子操作。原子操作是在``一步``完成执行的操作，在这一步中其它线程无法获得该共享资源。

通常情况下，这种同步方法只对那些只由单个``核心数据类型``组成的共享资源有效，譬如，字符串变量、数字、列表或者字典等。下面是几个``线程安全``的操作：

* 读或者替换一个实例属性
* 读或者替换一个全局变量
* 从列表中获取一项元素
* 原位修改一个列表（例如：使用``append``增加一个列表项）
* 从字典中获取一项元素
* 原位修改一个字典（例如：增加一个字典项、调用``clear``方法）

注意，上面提到过，对一个变量或者属性进行读操作，然后修改它，最终将其回写不是线程安全的。因为另外一个线程会在这个线程读完却没有修改或回写完成之前更改这个共享变量/属性。

锁
---

锁是Python的``threading``模块提供的最基本的同步机制。在任一时刻，一个锁对象可能被一个线程获取，或者不被任何线程获取。如果一个线程尝试去获取一个已经被另一个线程获取到的锁对象，那么这个想要获取锁对象的线程只能暂时终止执行直到锁对象被另一个线程释放掉。

锁通常被用来实现对共享资源的同步访问。为每一个共享资源创建一个**Lock**对象，当你需要访问该资源时，调用**acquire**方法来获取锁对象（如果其它线程已经获得了该锁，则当前线程需等待其被释放），待资源访问完后，再调用**release**方法释放锁：

``` Python
lock = Lock()

lock.acquire()  #: will block if lock is already held
... access shared resource
lock.release()
```

注意，即使在访问共享资源的过程中出错了也应该释放锁，可以用**try-finally**来达到这一目的：

``` Python
lock.acquire()
try:
    ... access shared resource
finally:
    lock.release()  #: release lock, no matter what
```

在Python 2.5及以后的版本中，你可以使用**with**语句。在使用锁的时候，**with**语句会在进入语句块之前自动的获取到该锁对象，然后在语句块执行完成后自动释放掉锁：

``` Python
from __future__ import with_statement  #: 2.5 only

with lock:
    ... access shared resource
```


**acquire**方法带一个可选的等待标识，它可用于设定当有其它线程占有锁时是否阻塞。如果你将其值设为**False**，那么**acquire**方法将不再阻塞，只是如果该锁被占有时它会返回**False**:

``` Python
if not lock.acquire(False):
    ... 锁资源失败
else:
    try:
        ... access shared resource
    finally:
        lock.release()
```

你可以使用``locked``方法来检查一个锁对象是否已被获取，注意不能用该方法来判断调用**acquire**方法时是否会阻塞，因为在``locked``方法调用完成到下一条语句（比如**acquire**）执行之间该锁有可能被其它线程占有。

``` Python
if not lock.locked():
    #: 其它线程可能在下一条语句执行之前占有了该锁
    lock.acquire()  #: 可能会阻塞
```

简单锁的缺点
---

标准的锁对象并不关心当前是哪个线程占有了该锁；如果该锁已经被占有了，那么任何其它尝试获取该锁的线程都会被阻塞，即使是占有锁的这个线程。考虑一下下面这个例子：

``` Python
lock = threading.Lock()


def get_first_part():
    lock.acquire()
    try:
        ... 从共享对象中获取第一部分数据
    finally:
        lock.release()
    return data
    
    
def get_second_part():
    lock.acquire()
    try:
        ... 从共享对象中获取第二部分数据
    finally:
        lock.release()
    return data
```

示例中，我们有一个共享资源，有两个分别取这个共享资源第一部分和第二部分的函数。两个访问函数都使用了锁来确保在获取数据时没有其它线程修改对应的共享数据。

现在，如果我们想添加第三个函数来获取两个部分的数据，我们将会陷入泥潭。一个简单的方法是依次调用这两个函数，然后返回结合的结果：

``` Python
def get_both_parts():
    first = get_first_part()
    seconde = get_second_part()
    return first, second
```

这里的问题是，如有某个线程在两个函数调用之间修改了共享资源，那么我们最终会得到不一致的数据。最明显的解决方法是在这个函数中也使用lock:

``` Python
    def get_both_parts():
        lock.acquire()
        try:
            first = get_first_part()
            seconde = get_second_part()
        finally:
            lock.release()
        return first, second
```

然而，这是不可行的。里面的两个访问函数将会阻塞，因为外层语句已经占有了该锁。为了解决这个问题，你可以通过使用标记在访问函数中让外层语句释放锁，但这样容易失去控制并导致出错。幸运的是，**threading**模块包含了一个更加实用的锁实现：**``re-entrant锁``**。

Re-Entrant Locks (RLock)
---

**RLock**类是简单锁的另一个版本，它的特点在于，同一个锁对象只有在被``其它的``线程占有时尝试获取才会发生阻塞；而简单锁在同一个线程中同时只能被占有一次。如果当前线程已经占有了某个**RLock**锁对象，那么当前线程仍能再次获取到该**RLock**锁对象。

``` Python
lock = threading.Lock()
lock.acquire()
lock.acquire()  #: 这里将会阻塞

lock = threading.RLock()
lock.acquire()
lock.acquire()  #: 这里不会发生阻塞
```

**RLock**的主要作用是解决嵌套访问共享资源的问题，就像前面描述的示例。要想解决前面示例中的问题，我们只需要将**Lock**换为**RLock**对象，这样嵌套调用也会OK.

``` Python
lock = threading.RLock()


def get_first_part():
    ... see above
    
    
def get_second_part():
    ... see above
    
    
def get_both_parts():
    ... see above
```

这样既可以单独访问两部分数据也可以一次访问两部分数据而不会被锁阻塞或者获得不一致的数据。

注意**RLock**会追踪递归层级，因此记得在**acquire**后进行**release**操作。

Semaphores
---

信号量是一个更高级的锁机制。信号量内部有一个计数器而不像锁对象内部有锁标识，而且只有当占用信号量的线程数超过信号量时线程才阻塞。这允许了多个线程可以同时访问相同的代码区。

``` Python
semaphore = threading.BoundedSemaphore()
semaphore.acquire()  #: counter减小
... 访问共享资源
semaphore.release()  #: counter增大
```

当信号量被获取的时候，计数器减小；当信号量被释放的时候，计数器增大。当获取信号量的时候，如果计数器值为0，则该进程将阻塞。当某一信号量被释放，counter值增加为1时，被阻塞的线程（如果有的话）中会有一个得以继续运行。

信号量通常被用来限制对容量有限的资源的访问，比如一个网络连接或者数据库服务器。在这类场景中，只需要将计数器初始化为最大值，信号量的实现将为你完成剩下的事情。

``` Python
max_connections = 10

semaphore = threading.BoundedSemaphore(max_connections)
```

如果你不传任何初始化参数，计数器的值会被初始化为1.

Python的**threading**模块提供了两种信号量实现。**Semaphore**类提供了一个无限大小的信号量，你可以调用**release**任意次来增大计数器的值。为了避免错误出现，最好使用**BoundedSemaphore**类，这样当你调用**release**的次数大于**acquire**次数时程序会出错提醒。

线程同步
---

锁可以用在线程间的同步上。**threading**模块包含了一些用于线程间同步的类。

Events
---

一个事件是一个简单的同步对象，事件表示为一个内部标识(internal flag)，线程等待这个标识被其它线程设定，或者自己设定、清除这个标识。

``` Python
event = threading.Event()

#: 一个客户端线程等待flag被设定
event.wait()

#: 服务端线程设置或者清除flag
event.set()
event.clear()
```

一旦标识被设定，**wait**方法就不做任何处理（不会阻塞），当标识被清除时，**wait**将被阻塞直至其被重新设定。任意数量的线程可能会等待同一个事件。

Conditions
---

条件是事件对象的高级版本。条件表现为程序中的某种状态改变，线程可以等待给定条件或者条件发生的信号。

下面是一个简单的生产者/消费者实例。首先你需要创建一个条件对象：

``` Python
#: 表示一个资源的附属项
condition = threading.Condition()
```

生产者线程在通知消费者线程有新生成资源之前需要获得条件：

``` Python
#: 生产者线程
... 生产资源项
condition.acquire()
... 将资源项添加到资源中
condition.notify()  #: 发出有可用资源的信号
condition.release()
```

消费者必须获取条件（以及相关联的锁），然后尝试从资源中获取资源项：

``` Python
#: 消费者线程
condition.acquire()
while True:
    ...从资源中获取资源项
    if item:
        break
    condition.wait()  #: 休眠，直至有新的资源
condition.release()
... 处理资源
```

**wait**方法释放了锁，然后将当前线程阻塞，直到有其它线程调用了同一条件对象的**notify**或者**notifyAll**方法，然后又重新拿到锁。如果同时有多个线程在等待，那么**notify**方法只会唤醒其中的一个线程，而**notifyAll**则会唤醒全部线程。

为了避免在**wait**方法处阻塞，你可以传入一个超时参数，一个以秒为单位的浮点数。如果设置了超时参数，**wait**将会在指定时间返回，即使**notify**没被调用。一旦使用了超时，你必须检查资源来确定发生了什么。

注意，条件对象关联着一个锁，你必须在访问条件之前获取这个锁；同样的，你必须在完成对条件的访问时释放这个锁。在生产代码中，你应该使用**try-finally**或者**with**.

可以通过将锁对象作为条件构造函数的参数来让条件关联一个已经存在的锁，这可以实现多个条件公用一个资源：

``` Python
lock = threading.RLock()
condition_1 = threading.Condition(lock)
condition_2 = threading.Condition(lock)
```
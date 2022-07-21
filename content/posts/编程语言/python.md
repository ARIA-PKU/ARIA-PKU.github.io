---
title: Python
date: 2022-06-15T23:21:39+08:00
lastmod: 2022-06-15T23:21:39+08:00

cover: https://aria9766.oss-cn-beijing.aliyuncs.com/blog-pictures/6_15/python.jpg
# images:
#   - /img/cover.jpg
categories:
  - 编程语言
tags:
  - python
# nolastmod: true
draft: false
---

Python语言的总结归纳

<!--more-->

实习、写项目一直都在用python，发现和其他编程语言相比还是有很多特性平时没有注意到，这里总结整理一下，记录遇到的一些问题。

python作为一门解释型语言，以代码简洁易懂著称。我们可以直接对名称赋值，而不必声明类型。名称类型的确定、内存空间的分配与释放都是由python解释器在运行时进行的。当然，简洁的代价带来的就是执行效率的降低，但是其实Python的很多库函数都是利用C++实现的，在真正使用的时候，可以链接库函数，解决性能的问题。

# 一、Python的底层原理

## 1、垃圾回收机制

**结论：**在Python中，**主要通过引用计数进行垃圾回收**；**通过 “标记-清除” 解决容器对象可能产生的循环引用问题**；**通过 “分代回收” 以空间换时间的方法提高垃圾回收效率。**



**引用计数**

Python中，主要通过**引用计数（Reference Counting）**进行垃圾回收。

```text
typedef struct_object {
 int ob_refcnt;
 struct_typeobject *ob_type;
} PyObject;
```

在Python中每一个对象的核心就是一个结构体PyObject，它的内部有一个引用计数器（ob_refcnt）。程序在运行的过程中会实时的更新ob_refcnt的值，来反映引用当前对象的名称数量。当某对象的引用计数值为0,那么它的内存就会被立即释放掉。

引用计数法有其明显的优点，如高效、实现逻辑简单、具备实时性，一旦一个对象的引用计数归零，内存就直接释放了。不用像其他机制等到特定时机。将垃圾回收随机分配到运行的阶段，处理回收内存的时间分摊到了平时，正常程序的运行比较平稳。但是，引用计数也存在着一些缺点，通常的缺点有：

- 逻辑简单，但**实现有些麻烦**。每个对象需要分配单独的空间来统计引用计数，这无形中加大的空间的负担，并且需要对引用计数进行维护，在维护的时候很容易会出错。
- 在一些场景下，可能会比较慢。正常来说垃圾回收会比较平稳运行，**但是当需要释放一个大的对象时，比如字典，需要对引用的所有对象循环嵌套调用，从而可能会花费比较长的时间**。
- **循环引用**。这将是引用计数的致命伤，引用计数对此是无解的，因此必须要使用其它的垃圾回收算法对其进行补充。

也就是说，Python 的垃圾回收机制，很大一部分是**为了处理可能产生的循环引用**，是对引用计数的补充。



**标记清除解决循环引用**

Python采用了**“标记-清除”(Mark and Sweep)**算法，解决容器对象可能产生的循环引用问题。(注意，只有容器对象才会产生循环引用的情况，比如列表、字典、用户自定义类的对象、元组等。而像数字，字符串这类简单类型不会出现循环引用。作为一种优化策略，对于只包含简单类型的元组也不在标记清除算法的考虑之列)

但是，**标记清除算法会暂停整个应用程序，等待标记清除结束后才会恢复应用程序的运行。**



**分代回收算法**

在循环引用对象的回收中，整个应用程序会被暂停，为了减少应用程序暂停的时间，Python 通过**“分代回收”(Generational Collection)**以空间换时间的方法提高垃圾回收效率。

分代回收是基于这样的一个统计事实，**对于程序，存在一定比例的内存块的生存周期比较短；而剩下的内存块，生存周期会比较长，甚至会从程序开始一直持续到程序结束。生存期较短对象的比例通常在 80%～90% 之间，这种思想简单点说就是：对象存在时间越长，越可能不是垃圾，应该越少去收集。这样在执行标记-清除算法时可以有效减小遍历的对象数，从而提高垃圾回收的速度。**

## 2、GIL锁(Global Interpreter Lock)

### 2.1 GIL是什么

**GIL并不是Python的特性**，它是在实现Python解析器(CPython)时所引入的一个概念。就好比C++是一套语言（语法）标准，但是可以用不同的编译器来编译成可执行代码。有名的编译器例如GCC，INTEL C++，Visual C++等。Python也一样，同样一段代码可以通过CPython，PyPy，Psyco等不同的Python执行环境来执行。像其中的JPython就没有GIL。然而因为CPython是大部分环境下默认的Python执行环境。所以在很多人的概念里CPython就是Python，也就想当然的把GIL归结为Python语言的缺陷。所以这里要先明确一点：GIL并不是Python的特性，Python完全可以不依赖于GIL。

而且，GIL锁锁的其实是进程。

### 2.2 GIL是否保证了线程安全

结论是**有GIL并不意味着python一定是线程安全的**。

一个线程有两种情况下会释放全局解释器锁，一种情况是在该线程**进入IO操作**之前，会主动释放GIL，另一种情况是解释器**不间断运行了1000字节码（Py2）或运行15毫秒（Py3）后**，该线程也会放弃GIL。既然一个线程可能随时会失去GIL，那么这就一定会涉及到线程安全的问题。GIL虽然从设计的出发点就是考虑到线程安全，但这种线程安全是粗粒度的线程安全，即不需要程序员自己对线程进行加锁处理（同理，所谓细粒度就是指程序员需要自行加、解锁来保证线程安全，典型代表是 Java , 而 CPython 中是粗粒度的锁，即语言层面本身维护着一个全局的锁机制,用来保证线程安全）。

首先，线程因为进入IO操作而主动释放了GIL，容易理解这样是安全的。

其次，另一种方式放弃则不一定安全，即线程A是因为解释器不间断执行了1000字节码的指令或不间断运行了15毫秒而放弃了GIL，那么此时实际上线程A和线程B将同时竞争GIL锁。在同时竞争的情况下，实际上谁会竞争成功是不确定的一个结果，所以一般被称为“抢占式多任务处理”，这种情况下当然就看谁抢得厉害了。当然，在python3上由于对GIL做了优化，并且会动态调整线程的优先级，所以线程B的优先级会比较高，但**仍然无法肯定线程B就一定会拿到GIL**。那么在这种情况下，线程可能就会出现不安全的状态。这种情况，只要在代码上加锁即可，在 Python 中获取一个 **threading.Lock** 也就是一行代码的事**。**

还有一种特殊情况是**原子操作**，原子操作对应的一定是线程安全的，典型的例子比如sort方法。

### 2.3 如何避免GIL锁影响

1）在以IO操作为主的IO密集型应用中，多线程和多进程的性能区别并不大，原因在于即使在Python中有GIL锁的存在，由于线程中的IO操作会使得线程立即释放GIL，切换到其他非IO线程继续操作，提高程序执行效率。相比进程操作，线程操作更加轻量级，线程之间的通讯复杂度更低，建议使用多线程。

2）如果是计算密集型的应用，尽量使用多进程或者协程来代替多线程。

# 二、Python的高级特性

## 1、列表推导式

列表推导式不只是看起来更简洁，其效率也比一般的for循环要更高，具体原因是for循环需要每次载入append属性再CALL_FUNCTION’，因此还是要多用列表推导式的。

写列表生成式时，把要生成的元素`x * x`放到前面，后面跟`for`循环，就可以把list创建出来，十分有用，多写几次，很快就可以熟悉这种语法。

for循环后面还可以加上if判断，这样我们就可以筛选出仅偶数的平方：

```
>>> [x * x for x in range(1, 11) if x % 2 == 0]
[4, 16, 36, 64, 100]
```

## 2、生成器

生成器在机器学习中也经常使用，确实是一个很方便实用的方法。

通过列表生成式，我们可以直接创建一个列表。但是，受到内存限制，列表容量肯定是有限的。而且，创建一个包含100万个元素的列表，不仅占用很大的存储空间，如果我们仅仅需要访问前面几个元素，那后面绝大多数元素占用的空间都白白浪费了。

所以，如果列表元素可以按照某种算法推算出来，那我们是否可以在循环的过程中不断推算出后续的元素呢？这样就不必创建完整的list，从而节省大量的空间。在Python中，这种一边循环一边计算的机制，称为生成器：generator。

### 2.1 使用方法一

第一种方法很简单，**只要把一个列表生成式的`[]`改成`()`，就创建了一个generator**：

```
>>> L = [x * x for x in range(10)]
>>> L
[0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
>>> g = (x * x for x in range(10))
>>> g
<generator object <genexpr> at 0x1022ef630>
```

可以通过`next()`函数获得generator的下一个返回值

```
>>> next(g)
0
>>> next(g)
1
>>> next(g)
4
```

### 2.2 使用方法二

第二种方法，**通过yield返回结果。**

generator函数和普通函数的执行流程不一样。普通函数是顺序执行，遇到`return`语句或者最后一行函数语句就返回。而变成generator的函数，在每次调用`next()`的时候执行，遇到`yield`语句返回，再次执行时从上次返回的`yield`语句处继续执行。

举个简单的例子，定义一个generator函数，依次返回数字1，3，5：

```
def odd():
    print('step 1')
    yield 1
    print('step 2')
    yield(3)
    print('step 3')
    yield(5)
```

调用该generator函数时，首先要生成一个generator对象，然后用`next()`函数不断获得下一个返回值：

```
>>> o = odd()
>>> next(o)
step 1
1
>>> next(o)
step 2
3
>>> next(o)
step 3
5
>>> next(o)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```

## 3、迭代器

可以被`next()`函数调用并不断返回下一个值的对象称为迭代器：`Iterator`。

我们已经知道，可以直接作用于`for`循环的数据类型有以下几种：

一类是集合数据类型，如`list`、`tuple`、`dict`、`set`、`str`等；

一类是`generator`，包括生成器和带`yield`的generator function。

这些可以直接作用于`for`循环的对象统称为可迭代对象：`Iterable`。

可以使用`isinstance()`判断一个对象是否是`Iterable`对象：

```
>>> from collections.abc import Iterable
>>> isinstance([], Iterable)
True
>>> isinstance({}, Iterable)
True
>>> isinstance('abc', Iterable)
True
>>> isinstance((x for x in range(10)), Iterable)
True
>>> isinstance(100, Iterable)
False
```

生成器都是`Iterator`对象，但`list`、`dict`、`str`虽然是`Iterable`，却不是`Iterator`。

把`list`、`dict`、`str`等`Iterable`变成`Iterator`可以使用`iter()`函数：

```
>>> isinstance(iter([]), Iterator)
True
>>> isinstance(iter('abc'), Iterator)
True
```

**小结：**

凡是可作用于`for`循环的对象都是`Iterable`类型；

凡是可作用于`next()`函数的对象都是`Iterator`类型，它们表示一个惰性计算的序列；

集合数据类型如`list`、`dict`、`str`等是`Iterable`但不是`Iterator`，不过可以通过`iter()`函数获得一个`Iterator`对象；

Python的`for`循环本质上就是通过不断调用`next()`函数实现的。

## 4、装饰器

**在代码运行期间动态增加功能的方式，称之为“装饰器”（Decorator）。**

这是一个非常实用的功能，结合在廖雪峰官网上的例子总结一下使用方法。

由于函数也是一个对象，而且函数对象可以被赋值给变量，所以，通过变量也能调用该函数。

```
>>> def now():
...     print('2015-3-25')
...
>>> f = now
>>> f()
2015-3-25
```

现在，假设我们要增强`now()`函数的功能，比如，在函数调用前后自动打印日志。

本质上，decorator就是一个返回函数的高阶函数。所以，我们要定义一个能打印日志的decorator，可以定义如下：

```
def log(func):
    def wrapper(*args, **kw):
        print('call %s():' % func.__name__)
        return func(*args, **kw)
    return wrapper
```

观察上面的`log`，因为它是一个decorator，所以接受一个函数作为参数，并返回一个函数。我们要借助Python的@语法，把decorator置于函数的定义处：

```
@log
def now():
    print('2015-3-25')
```

调用`now()`函数，不仅会运行`now()`函数本身，还会在运行`now()`函数前打印一行日志：

```
>>> now()
call now():
2015-3-25
```

把`@log`放到`now()`函数的定义处，相当于执行了语句：

```
now = log(now)
```

由于`log()`是一个decorator，返回一个函数，所以，原来的`now()`函数仍然存在，只是现在同名的`now`变量指向了新的函数，于是调用`now()`将执行新函数，即在`log()`函数中返回的`wrapper()`函数。

`wrapper()`函数的参数定义是`(*args, **kw)`，因此，`wrapper()`函数可以接受任意参数的调用。在`wrapper()`函数内，首先打印日志，再紧接着调用原始函数。

如果decorator本身需要传入参数，那就需要编写一个返回decorator的高阶函数，写出来会更复杂。比如，要自定义log的文本：

```
def log(text):
    def decorator(func):
        def wrapper(*args, **kw):
            print('%s %s():' % (text, func.__name__))
            return func(*args, **kw)
        return wrapper
    return decorator
```

这个3层嵌套的decorator用法如下：

```
@log('execute')
def now():
    print('2015-3-25')
```

执行结果如下：

```
>>> now()
execute now():
2015-3-25
```

和两层嵌套的decorator相比，3层嵌套的效果是这样的：

```
>>> now = log('execute')(now)
```

我们来剖析上面的语句，首先执行`log('execute')`，返回的是`decorator`函数，再调用返回的函数，参数是`now`函数，返回值最终是`wrapper`函数。

以上两种decorator的定义都没有问题，但还差最后一步。因为我们讲了函数也是对象，它有`__name__`等属性，但你去看经过decorator装饰之后的函数，它们的`__name__`已经从原来的`'now'`变成了`'wrapper'`：

```
>>> now.__name__
'wrapper'
```

因为返回的那个`wrapper()`函数名字就是`'wrapper'`，所以，需要把原始函数的`__name__`等属性复制到`wrapper()`函数中，否则，有些依赖函数签名的代码执行就会出错。

不需要编写`wrapper.__name__ = func.__name__`这样的代码，Python内置的`functools.wraps`就是干这个事的，所以，一个完整的decorator的写法如下：

```
import functools

def log(func):
    @functools.wraps(func)
    def wrapper(*args, **kw):
        print('call %s():' % func.__name__)
        return func(*args, **kw)
    return wrapper
```

或者针对带参数的decorator：

```
import functools

def log(text):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kw):
            print('%s %s():' % (text, func.__name__))
            return func(*args, **kw)
        return wrapper
    return decorator
```

## 5、异步编程

### 5.1 python的协程

协程，又称微线程，纤程。英文名Coroutine。

子程序，或者称为函数，在所有语言中都是层级调用，比如A调用B，B在执行过程中又调用了C，C执行完毕返回，B执行完毕返回，最后是A执行完毕。

所以子程序调用是通过栈实现的，一个线程就是执行一个子程序。

子程序调用总是一个入口，一次返回，调用顺序是明确的。而协程的调用和子程序不同。

协程看上去也是子程序，但执行过程中，在**子程序内部可中断**，然后转而执行别的子程序，在适当的时候再返回来接着执行。

**优势：**

最大的优势就是协程极高的执行效率。因为子程序切换不是线程切换，而是由程序自身控制，因此，没有线程切换的开销，和多线程比，线程数量越多，协程的性能优势就越明显。这里子程序的切换其实就是进程的切换。

第二大优势就是不需要多线程的锁机制，因为只有一个线程，也不存在同时写变量冲突，在协程中控制共享资源不加锁，只需要判断状态就好了，所以执行效率比多线程高很多。

因为协程是一个线程执行，那怎么利用多核CPU呢？最简单的方法是多进程+协程，既充分利用多核，又充分发挥协程的高效率，可获得极高的性能。

Python对协程的支持是通过generator实现的。

在generator中，我们不但可以通过`for`循环来迭代，还可以不断调用`next()`函数获取由`yield`语句返回的下一个值。

但是Python的`yield`不但可以返回一个值，它还可以接收调用者发出的参数。

### 5.2 async/await

这个在web游戏开发的项目中使用过，也总结过。async就是声明这个函数是异步的，await则是函数的中断条件，只有await的条件满足了，函数才会继续执行下去。

用`asyncio`提供的`@asyncio.coroutine`可以把一个generator标记为coroutine类型，然后在coroutine内部用`yield from`调用另一个coroutine实现异步操作。

为了简化并更好地标识异步IO，从Python 3.5开始引入了新的语法`async`和`await`，可以让coroutine的代码更简洁易读。

请注意，`async`和`await`是针对coroutine的新语法，要使用新的语法，只需要做两步简单的替换：

1. 把`@asyncio.coroutine`替换为`async`；
2. 把`yield from`替换为`await`。

让我们对比一下使用asynio的代码：

```
@asyncio.coroutine
def hello():
    print("Hello world!")
    r = yield from asyncio.sleep(1)
    print("Hello again!")
```

用新语法重新编写如下：

```
async def hello():
    print("Hello world!")
    r = await asyncio.sleep(1)
    print("Hello again!")
```

# 三、Python的其他特性

## 1、== 和 is 有什么区别？

```==```比较的是两个变量的 value，只要值相等就会返回True

```is```比较的是两个变量的 id，即```id(a) == id(b)```，只有两个变量指向同一个对象的时候，才会返回True

但是需要注意的是，比如以下代码：

```
a = 2
b = 2
print(a is b)
```

按照上面的解释，应该会输出False，但是事实上会输出True，这是因为Python中对小数据有缓存机制，-5~256之间的数据都会被缓存。这一点和java中的一致。

## 2、python的单例模式的使用

单例模式在菜鸟教程上找到的解释是：这种模式涉及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。

简单理解，就是为了避免某个实例被频繁创建，而使用的创建方式，保证全局只会创建一次。它的主要目的就是确保只有一个实例对象的存在。换句话说，当一个类的功能比较单一，只需要一个实例对象就可以完成需求的时，就可以使用单例模式来节省内存资源。

在实习的项目中， 实现单例的方式使用的是利用python的模块引入会生产.pyc文件。

python 模块是天然的单例模式：需要使用实例的时候，在文件中导入即可使用，从而减少相关实例的创建；

模块在第一次导入时，会生成 .pyc 文件，当第二次导入时，就会直接加载 .pyc 文件，而不会再次执行模块代码;

因此，我们只需把相关的函数和数据定义在一个模块中，就可以获得一个单例对象了。

# 四、参考文章

1、[Python垃圾回收机制！非常实用](https://zhuanlan.zhihu.com/p/83251959)

2、[深入理解Python中的GIL（全局解释器锁）](https://zhuanlan.zhihu.com/p/75780308)

3、[廖雪峰的官方网站](https://www.liaoxuefeng.com/)


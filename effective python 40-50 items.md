## **Item 41: Consider** **concurrent.futures** **for True** Parallelism (多进程手拉手才是真正的并行)

多次提到了python原生解释器带的GIL实现不了真正的多线程,所以只能使用别的方法.

第一种是可以编写c拓展模块,将python优化成c语言的速度

> But rewriting your code in C has a high cost. Code that is short and understandable in Python can become verbose and complicated in C.

所以..还有别的解决办法吗?

> The multiprocessing built-in module, easily accessed via the concurrent.futures built-in module, may be exactly what you need. It enables Python to utilize multiple CPU cores in parallel by running additional interpreters as child processes第二种是使用多进程的方式来达到真正的并行

> .These child processes are separate from the main interpreter, so their globalinterpreter locks are also separate. Each child can fully utilize one CPU core. Each child has a link to the main process where it receives instructions to do computation and returns results. 

进程是相互之间独立的 不像线程一样处于同个上下文中 受互相制约

来看一下简单的例子

```
def gcd(pair):
    """
    算法里很常见的函数 用来求最大公约数
    Greatest Common Divisor(GCD)
    :param pair:
    :return:
    """
    a, b = pair
    low = min(a, b)
    for i in range(low, 0, -1):
        if a % i == 0 and b % i == 0:
            return i
numbers = [(1963309, 2265973), (2030677, 3814172), (1551645, 2229620), (2039045, 2020802)] 
start = time.time()
results = list(map(gcd, numbers)) 
end = time.time()
print('Took %.3f seconds' % (end - start))
```

```
Took 0.462 seconds
```

```
Took 0.435 seconds
```

```
Took 0.436 seconds
```

如果使用多线程,很明显会自己卡自己 还导致了互斥导致的一小点开销

```

start = time.time()
pool = concurrent.futures.ThreadPoolExecutor(max_workers=4)
results = list(pool.map(gcd, numbers)) 
end = time.time()
print('Took %.3f seconds' % (end - start))

```

```
Took 0.444 seconds
```

```
Took 0.445 seconds
```

所以真正靠得住的是多进程

```
def gcd(pair):
    """
    算法里很常见的函数 用来求最大公约数
    Greatest Common Divisor(GCD)
    :param pair:
    :return:
    """
    a, b = pair
    low = min(a, b)
    for i in range(low, 0, -1):
        if a % i == 0 and b % i == 0:
            return i
numbers = [(1963309, 2265973), (2030677, 3814172), (1551645, 2229620), (2039045, 2020802)]

if __name__ == '__main__':
    start = time.time()
    results = list(map(gcd, numbers))
    end = time.time()
    print('Took %.3f seconds' % (end - start))

    start = time.time()
    pool = concurrent.futures.ThreadPoolExecutor(max_workers=4)
    results = list(pool.map(gcd, numbers))
    end = time.time()
    print('Took %.3f seconds' % (end - start))

    start = time.time()
    pool = concurrent.futures.ProcessPoolExecutor(max_workers=2) # The one change
    results = list(pool.map(gcd, numbers))
    end = time.time()
    print('Took %.3f seconds' % (end - start))

```

```
Took 0.434 seconds
Took 0.443 seconds
Took 0.343 seconds
```

奇怪的是,他并没有带来成倍的提升,是为什么呢?

当我把最大的进程翻倍后才有成倍的提升

```

    pool = concurrent.futures.ProcessPoolExecutor(max_workers=4) # The one change
```

```
Took 0.433 seconds
Took 0.443 seconds
Took 0.231 seconds
```

这是多进程提速的原理

1. 它从数字输入数据中获取要映射的每个项目。
2.  它使用pickle 模块将其序列化为二进制数据（参见条款44：“使用copyreg 使pickle 变得可靠”）。
3. 它通过本地套接字将序列化数据从主解释器进程复制到子解释器进程。 
4. 接下来，它在子进程中使用 pickle 将数据反序列化回 Python 对象。 
5. 然后导入包含 gcd 函数的 Python 模块。 
6. 它与其他子进程并行运行对输入数据的函数。 
7. 将结果序列化回字节。 
8. 它通过套接字将这些字节复制回。 
9. 它将字节反序列化回父进程中的 Python 对象。 
10.  最后，它将多个子节点的结果合并到一个列表中返回。

听上去还挺复杂的实现,不过相对于多线程带来的毛病还是九牛一毛而已.

> The overhead of using multiprocessing is high because of all of the serialization and deserialization that must happen between the parent and child processes. 

多进程并行适合执行一些具有少量参数,专门设计于计算函数的情况,别忘了还有 mutilateprocess你可以用呢 虽然那个更难而且更复杂..



**Things to Remember** 

- Moving CPU bottlenecks to C-extension modules can be an effective way to improve performance while maximizing your investment in Python code. However, the cost of doing so is high and may introduce bugs. 
- The multiprocessing module provides powerful tools that can parallelize certain types of Python computation with minimal effort. 
- The power of multiprocessing is best accessed through the concurrent.futures built-in module and its simple ProcessPoolExecutor class. 
- The advanced parts of the multiprocessing module should be avoided because they are so complex.

# **6.** **Built-in Modules**

python的内置模块强大到没人用,我也搞不懂 因为平时好像接触不到 或者说想用的时候想不起来了,就很难受



## **Item 42: Define Function Decorators with** **functools.wraps**  (装饰后的函数别失其本心)

这玩意老久之前就见到了,但是我觉得我自己可能一直用不上..

无非就是这几种情况

```
def trace(func): 
    def wrapper(*args, **kwargs): 
        result = func(*args, **kwargs) 
        print('%s(%r, %r) -> %r' % (func.__name__, args, kwargs, result)) 
        return result 
    return wrapper

@trace 
def fibonacci(n): 
    '''Return the n-th Fibonacci number'''
    if n in (0, 1): 
        return n 
    return (fibonacci(n - 2) + fibonacci(n - 1))

fibonacci(3)
```

```
print(fibonacci) >>> <function trace.<locals>.wrapper at 0x107f7ed08>
```

直接打印的话会失去函数的原信息

使用help也会毛也看不到

```
Help on function wrapper in module __main__:

wrapper(*args, **kwargs)

None

```

只要使用这个装饰器就能将信息给还原了

```
def trace(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs): 
        result = func(*args, **kwargs) 
        print('%s(%r, %r) -> %r' % (func.__name__, args, kwargs, result)) 
        return result 
    return wrapper
```

```
Help on function fibonacci in module __main__:

fibonacci(n)
    Return the n-th Fibonacci number

None
```

多用装饰器语法糖

Things to Remember** 

- Decorators are Python syntax for allowing one function to modify another function at runtime. 
- Using decorators can cause strange behaviors in tools that do introspection, such as debuggers. 
- Use the wraps decorator from the functools built-in module when you define your own decorators to avoid any issues. 

## **Item 43: Consider** **contextlib** **and** **with** **Statements for** Reusable** **try****/****finally** **Behavior** (使用上下文管理器替代异常四大金刚)

上下文管理器自带异常处理,这个我们早就知道了

但是怎么用好呢?

下段代码是等价的

```
from threading import Lock

lock = Lock()
with lock:
    print('Lock is held')

lock.acquire() 
try:
    print('Lock is held') 
finally: 
    lock.release()
```

当然with块是更好的,因为可以写少点,而且表意更好

你可以重写类方法,也可以使用上下文管理器

```
from contextlib import contextmanager


def my_function():
    logging.debug('Some debug data') 
    logging.error('Error log here')
    logging.debug('More debug data')

# my_function()


@contextmanager
def debug_logging(level):
    logger = logging.getLogger()
    old_level = logger.getEffectiveLevel()
    logger.setLevel(level)
    try:
        yield
    finally:
        logger.setLevel(old_level)

with debug_logging(logging.DEBUG):
    print('Inside:')
    my_function()

print('After:')
my_function()
```

```
>>> 

Inside: 

Some debug data 

Error log here 

More debug data 

After: 

Error log here 
```

你可以把对象return出去 就像你用 with open …  as f一样

```

@contextmanager 
def log_level(level, name): 
    logger = logging.getLogger(name) 
    old_level = logger.getEffectiveLevel() 
    logger.setLevel(level) 
    try:
        yield logger 
    finally: 
        logger.setLevel(old_level)
```

```
with log_level(logging.DEBUG, 'my-log') as logger:
    logger.debug('This is my message!') 
    logging.debug('This will not print')
```

**Things to Remember** 

- The with statement allows you to reuse logic from try/finally blocks and reduce visual noise.
- The contextlib built-in module provides a contextmanager decorator that makes it easy to use your own functions in with statements. 
- The value yielded by context managers is supplied to the as part of the with statement. It’s useful for letting your code directly access the cause of the special context

## **Item 44: Make** **pickle** **Reliable with** **copyreg** (相信pickle的序列化)

pass

## **Item 45: Use** **datetime** **Instead of** **time** **for Local Clocks** (人类能看到那白驹过隙吗?)

> 协调世界时（UTC）是标准的、与时区无关的时间表示。协调世界时对于那些以UNIX纪元以来的秒数来表示时间的计算机来说非常有效。但UTC对人类来说并不理想。人类参考的时间是相对于他们目前所处的位置。人们说 "中午 "或 "上午8点"，而不是 "UTC 15:00减去7小时"。
>
> 如果你的程序要处理时间，你可能会发现自己要在UTC和本地时钟之间转换时间，以便让人类更容易理解。Python 提供了两种完成时区转换的方法。
>
> 旧的方法，使用时间内置模块，是灾难性的容易出错。
>
> 新的方法，使用内置的数据时间模块，在社区建立的名为 pytz 的包的帮助下，效果很好。你应该对时间和数据时间都有所了解，以彻底理解为什么数据时间是最好的选择，而时间应该被避免。

datetime大火肯定用过,直接来看代码

先来看 time 模块 我们也很熟悉

首先是根据时间戳转化为 人类可读的刻度

```
from time import localtime, strftime 
now = 1407694710
local_tuple = localtime(now) 
time_format = '%Y-%m-%d %H:%M:%S' 
time_str = strftime(time_format, local_tuple) 
print(time_str)
```

```
2014-08-11 02:18:30
```

还是很强大的,有了这个功能.

反向变化很容易

```
from time import localtime, strftime 
now = 1407694710
local_tuple = localtime(now) 
time_format = '%Y-%m-%d %H:%M:%S' 
time_str = strftime(time_format, local_tuple) 
print(time_str)

from time import mktime, strptime
time_tuple = strptime(time_str, time_format)
utc_now = mktime(time_tuple)
print(utc_now)

now = time.time()
local_tuple = localtime(now)
time_format = '%Y-%m-%d %H:%M:%S'
time_str = strftime(time_format, local_tuple)
print(time_str)

```

> 如何将一个时区的当地时间转换为另一个时区的当地时间？例如，假设你在旧金山和纽约之间坐飞机，想知道你到达纽约后旧金山的时间是多少。
>
> 直接操作time、localtime和strptime函数的返回值来进行时区转换是一个坏主意。由于当地法律，时区一直在变化。自己管理太复杂了，特别是如果你想处理全球每个城市的航班起飞和到达。
>
> 许多操作系统有配置文件，可以自动跟上时区的变化。Python 让你通过时间模块使用这些时区。例如，这里我解析了太平洋夏令时的旧金山时区的出发时间。

这些就比较没啥用处了 没意思



> 在 Python 中表示时间的第二个选择是来自 datetime 内置模块的 datetime 类。
>
> 像时间模块一样，datetime可以用来将当前的UTC时间转换为本地时间。在这里，我把当前的UTC时间转换为我的计算机的本地时间（太平洋夏令时）。

```
from datetime import datetime, timezone
now = datetime(2014, 8, 10, 18, 18, 30)
now_utc= now.replace(tzinfo=timezone.utc)
now_local = now_utc.astimezone()
print(now_local)
```

有了这个模块可以调整地区和很方便就能显示出可读的代码

```
2014-08-11 02:18:30+08:00
```

> 与时间模块不同，datetime模块具有将一个本地时间可靠地转换为另一个本地时间的功能。
>
> 然而，datetime只提供了时区操作的机制，包括tzinfo类和相关方法。
>
> 缺少的是除了UTC之外的时区定义。幸运的是，Python 社区已经通过 pytz 模块解决了这个问题，该模块可以从 Python Package Index (https://pypi.python.org/pypi/pytz/) 下载。
>
> pytz 包含了你可能需要的所有时区定义的完整数据库。

这个对我来说没什么用啊啊啊啊.寄

### **Things to Remember** 

- Avoid using the time module for translating between different time zones. 
- Use the datetime built-in module along with the pytz module to reliably convert between times in different time zones. 
- Always represent time in UTC and do conversions to local time as the final step before presentation





## **Item 46: Use Built-in Algorithms and Data Structures** (你写的代码速度慢一般都不是python的锅)

### 只是你没用对内置的数据结构罢了

> 当你实现处理非微不足道的数据的Python程序时，你最终会看到由你的代码的算法复杂性引起的速度下降。
>
> 这通常不是Python作为一种语言的速度造成的 (如果是这样的话，请看第41项："考虑concurrent.futures的真正并行性")。
>
> 更可能的问题是，你没有为你的问题使用最好的算法和数据结构。幸运的是，Python 标准库中内置了许多你需要使用的算法和数据结构。
>
> 除了速度之外，使用这些常用的算法和数据结构可以使你的生活更轻松。一些你可能想要使用的最有价值的工具在正确实现上很棘手。避免重新实现常见的功能将节省你的时间和头痛。

嗯..你理解了吗?

一个个来吧

### 双端队列

```
fifo = deque() 
fifo.append(1) # Producer  
x =fifo.popleft() # Consumer
```

作为生产者消费者模型还是很经典的一个结构

> 列表内置类型也包含像队列一样的有序项目序列。
>
> 你可以在恒定的时间内从一个列表的末尾插入或删除项目。但是从列表的头部插入或删除项目需要线性时间，这比 deque 的恒定时间慢得多

适用于在头尾增加数据和弹出的情况

### 有序字典

这个大家都用过吧

由于字典是无序的 所以会出现顺序不定但是值的情况

```
a={1:1,2:2}
b={2:2,1:1}
print(a==b)
```

```
True
```

不过只要这样子 就可以了

```
a={1:1,2:2}
b={2:2,1:1}
c=collections.OrderedDict(a)
d=collections.OrderedDict(b)

assert c==d
```

```
AssertionError
```

值得注意的的是

有序字典的顺序有一个链表确定,所以占用的空间更大..能不用就不要用了

### 缺省字典

这个大家都用过了,都没啥好说的

况且之前还提到过

> see Item 23: “Accept Functions for Simple Interfaces Instead of Classes” for another example

```
defaultDict=collections.defaultdict(list)
defaultDict["asd"].append(213213)

def get_ILoveYou():
    return "ILOVEYOU"
defaultDict=collections.defaultdict(get_ILoveYou)
print(defaultDict["123"])

def get_ILoveYou():
    return "ILOVEYOU"
defaultDict=collections.defaultdict(get_ILoveYou,{"123":"我不爱你"})
print(defaultDict["1234"])
print(defaultDict)
```

### 堆队列

这个没用过额

不过我大概知道是根据优先级来排队列

```
a = []
heappush(a, 5)
heappush(a, 3)
heappush(a, 7)
heappush(a, 4)

print(heappop(a), heappop(a), heappop(a), heappop(a))
```

```
[3, 4, 7, 5]
```

> 这些 heapq 操作中的每一个都需要对数时间，与列表的长度成比例。
>
> 用一个标准的 Python 列表做同样的工作将是线性扩展。



### **Bisection**

这玩意 就是用来插入的

> bisect模块的函数，如bisect_left，提供了一个有效的二进制搜索，通过一个排序的项目序列。它返回的索引是该值在序列中的插入点。

这玩意我在实现折半查找的时候了解过,但是没什么机会用到

```
x = list(range(10**6))
i = x.index(991234)
i = bisect_left(x, 991234)

```

### **Iterator Tools** 

这玩意很好用 我经常用额

The itertools functions fall into three main categories: 

- Linking iterators together 
    - chain: Combines multiple iterators into a single sequential iterator. 
    - cycle: Repeats an iterator’s items forever. 
    - tee: Splits a single iterator into multiple parallel iterators. 
    - zip_longest: A variant of the zip built-in function that works well with iterators of different lengths. 
- Filtering items from an iterator
    - islice: Slices an iterator by numerical indexes without copying.
    - takewhile: Returns items from an iterator while a predicate function returns True. 
    - dropwhile: Returns items from an iterator once the predicate function returns False for the first time. 
    - filterfalse: Returns all items from an iterator where a predicate function returns False. The opposite of the filter built-in function.
- Combinations of items from iterators 
    - product: Returns the Cartesian product of items from an iterator, which is a nice alternative to deeply nested list comprehensions. 
    - permutations: Returns ordered permutations of length N with items from an iterator. 
    - combination: Returns the unordered combinations of length N with unrepeated items from an iterator.

我经常用到的是 是chain 真的舒服

```
print(list(itertools.chain(*[[123],[12312,312,321,31,23]])))
```

```
[123, 12312, 312, 321, 31, 23]
```

> There are even more functions and recipes available in the itertools module that I don’t mention here. Whenever you find yourself dealing with some tricky iteration code, it’s worth looking at the itertools documentation again to see whether there’s anything there for you to use.

简而言之 去了解一下 你会发现你用的上的

尽量不要自己去复现,毕竟你只是个菜鸡pythoner,比不上别人的(bushi

### **Things to Remember** 

- Use Python’s built-in modules for algorithms and data structures. 

- Don’t reimplement this functionality yourself. It’s hard to get right.



## **Item 47: Use** **decimal** **When Precision Is Paramount** (我需要高精度浮点运算)

听过没用过,怎么克服浮点计算的误差呢?

用这个就行了..

### **Things to Remember** 

- Python has built-in types and classes in modules that can represent practically every 
- type of numerical value.The Decimal class is ideal for situations that require high precision and exact rounding behavior, such as computations of monetary values. 





## **Item 48: Know Where to Find Community-Built Modules** (人生苦短,我用pip)

来都来了 我就介绍一下几个常用的pip命令

### 更换国内镜像

> 可以在使用pip的时候，加上参数-i和镜像地址(如
> https://pypi.tuna.tsinghua.edu.cn/simple)，
> 例如：pip install -i https://pypi.tuna.tsinghua.edu.cn/simple pandas，这样就会从清华镜像安装pandas库。
>
> ------------------------------------------------
> 版权声明：本文为CSDN博主「Chaser_LittleBee」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
> 原文链接：https://blog.csdn.net/sinat_21591675/article/details/82770360

### 导出python环境依赖txt

```
pip freeze > <目录>/requirements.txt
```

```
pip install <包名> 或 pip install -r requirements.txt
```

列出当前包

```
pip list
```

```
pip        21.2.4
ply        3.11
PySide2    5.15.2
setuptools 57.4.0
shiboken2  5.15.2
WARNING: You are using pip version 21.2.4; however, version 21.3.1 is available.
You should consider upgrading via the 'd:\ccrepository\yjcl\venv\scripts\python.exe -m pip install --upgrade pip' command.

```

### **升级 pip**

```
pip install -U pip
```



### **Things to Remember** 

- The Python Package Index (PyPI) contains a wealth of common packages that are built and maintained by the Python community. 
- pip is the command-line tool to use for installing packages from PyPI. 
- pip is installed by default in Python 3.4 and above; you must install it yourself for older versions.The majority of PyPI modules are free and open source software.

# **7.** **Collaboration** 

怎么让你的代码简单易懂呢?

合作合作 多写点注释和文档不会错的!!

> 在Python中，有一些语言特性可以帮助你构建具有清晰界面边界的定义明确的API。Python社区已经建立了最佳实践，使代码的可维护性随着时间的推移而最大化。
>
> 还有一些与Python一起提供的标准工具，使大型团队能够在不同的环境中一起工作。在Python程序上与他人合作需要慎重考虑如何编写代码。
>
> 即使你在自己的工作中，你也有可能使用别人通过标准库或开放源码包写的代码。了解使与其他 Python 程序员合作变得容易的机制是很重要的。

## **Item 49: Write Docstrings for Every Function, Class, and** **Module** (你懂不懂文档怎么装逼啊)



```
def 函数():
    """
    懂不懂有多重要啊
    :return:
    """

print(函数.__doc__)
```

```

    懂不懂有多重要啊
    :return:
    
```

> Docstrings 可以连接到函数、类和模块。这种连接是编译和运行 Python 程序过程的一部分。
>
> 对 docstrings 和 __doc__ 属性的支持有三个结果
>
> 文档的可及性使交互式开发更容易。你可以通过使用内置的帮助功能来检查函数、类和模块，查看它们的文档。这使得Python交互式解释器（Python "shell"）和像IPython Notebook (http://ipython.org)这样的工具在你开发算法、测试API和编写代码片断时使用起来非常愉快。
>
> 定义文档的标准方法使得建立将文本转换成更有吸引力的格式（如HTML）的工具变得容易。这已经导致了优秀的文档生成工具，如Sphinx (http://sphinx-doc.org)。这也使社区资助的网站成为可能，比如Read the Docs (https://readthedocs.org)，它为开源的Python项目提供了免费的精美文档托管。
>
> Python一流的、可访问的、漂亮的文档鼓励人们编写更多的文档。Python 社区的成员对文档的重要性有着强烈的信念。有一个假设，即 "好的代码 "也意味着有良好文档的代码。这意味着你可以期望大多数开源的Python库都有体面的文档。

所以啊 写文档注释很重要噻

毕竟pycharm天天要叫你写 不然就黄线警告了

### **Documenting Modules**  (在开头装逼

你可以在文件最上面写上最牛逼的话

> 每个模块都应该有一个顶级的文档串。这是一个字符串字面，是源文件中的第一个语句。它应该使用三个双引号（""）。这个文档串的目的是介绍该模块及其内容。文
>
> 档串的第一行应该是一个描述该模块目的的单句。接下来的段落应该包含所有用户应该知道的关于模块操作的细节。模块文档串也是一个跳转点，你可以在这里强调模块中的重要类和功能。
>
> 下面是一个模块文档串的例子
>
> ```
> \# words.py 
> \#!/usr/bin/env python3 
> “““Library for testing words for various linguistic patterns. 
> Testing how words relate to each other can be tricky sometimes! 
> This module provides easy ways to determine when words you’ve 
> found have special properties. 
> Available functions: 
> \- palindrome: Determine if a word is a palindrome. 
> \- check_anagram: Determine if two words are anagrams. 
> …
> ”””
> ```
>
> 

不过是懒得写而已 谁又不喜欢装逼呢

### **Documenting Classes**  (类里硬装)

```
class Player(object): 

    “““Represents a player of the game. 

    Subclasses may override the ‘tick’ method to provide 

    custom animations for the player’s movement depending 

    on their power level, etc. 

    Public attributes: 

    \- power: Unused power-ups (float between 0 and 1). 

    \- coins: Coins found during the level (integer). 

    ””” 
```

开头没装够 可以在你想象的任何地方装逼

### **Documenting Functions** (函数才是你应该装的地方

```
def find_anagrams(word, dictionary): 

    “““Find all anagrams for a word. 

    This function only runs as fast as the test for 

    membership in the ‘dictionary’ container. It will 

    be slow if the dictionary is a list and fast if 

    it’s a set. 

    Args:

    word: String of the target word. 

    dictionary: Container with all strings that 

    are known to be actual words. 

    Returns: 

    List of anagrams that were found. Empty if 

    none were found. 

    ”””
```

> 在为函数编写文档时，还有一些特殊情况需要了解。
>
> - 如果你的函数没有参数，只有一个简单的返回值，那么一句话的描述可能就够了。
> - 如果你的函数不返回任何东西，最好不提任何返回值，而说 "返回无"。
> - 如果你不希望你的函数在正常运行时引发异常，就不要提及这一事实。
> - 如果你的函数接受可变数量的参数（见项目18："用可变位置的参数减少视觉噪音"）或关键字参数（见项目19："用关键字参数提供可选的行为"），在文档中的参数列表中使用*args和**kwargs来描述其目的。
> - 如果你的函数有默认值的参数，应该提到这些默认值（参见第20项："使用无和文档字符串来指定动态默认参数"）。
> - 如果你的函数是一个生成器（参见第 16 项："考虑生成器而不是返回列表"），那么你的文档串应该描述生成器在迭代时产生了什么。
> - 如果你的函数是一个 COROUTINE（参见第 40 项："考虑用 COROUTINE 来同时运行许多函数"），那么你的文档串应该包含 COROUTINE 产生的东西，它期望从产量表达式中得到的东西，以及它何时停止迭代。

### **Things to Remember** 

- Write documentation for every module, class, and function using docstrings. Keep them up to date as your code changes. 
- For modules: Introduce the contents of the module and any important classes or functions all users should know about. 
- For classes: Document behavior, important attributes, and subclass behavior in the docstring following the class statement. 
- For functions and methods: Document every argument, returned value, raised exception, and other behaviors in the docstring following the def statement.

## **Item 50: Use Packages to Organize Modules and Provide** **Stable APIs** (大哥背起了行囊)

pass




# effective python 30-40 items

## **Item 31: Use Descriptors for Reusable** **@property** **Methods** (使用描述符协议替代很多的property)

考虑这种情况

```
class Homework(object):
    def __init__(self):
        self._grade = 0
    @property
    def grade(self):
        return self._grade
    @grade.setter
    def grade(self, value):
        if not (0 <= value <= 100):
            raise ValueError('Grade must be between 0 and 100')
        self._grade = value

hw=Homework()
hw.grade=1
hw.grade=123

```

现在是一切正常的情况,但是你想给学生加一个考试成绩的检查

就会变得复杂且漫长

```
class Exam:
    def __init__(self):
        self._grade=0
        self._math_grade=0
    @staticmethod
    def _check_grade(v):
        if not (0<=v<=100):
            raise ValueError
    @property
    def _grade(self):
        return self._grade
    
    @_grade.setter
    def _grade(self, value):
        self._check_grade(value)
        self._grade=value
        
    @property
    def _math_grade(self):
        return self._math_grade

    @_math_grade.setter
    def _math_grade(self, value):
        self._check_grade(value)
        self._math_grade=value
```

可以看到有重复的代码出现,非常的不pythonic 每次出现这种情况 我们观察到 其实可以通过几种方式解决

一是直接使用 内置获得属性的 `__setattr__` 方法 专门写一个grade类 每次获得属性时都做一个判断

```
class Exam:
    def __setattr__(self, key, value):
        if 0 <= value <= 100:
            self.__dict__[key] = value
        else:
            raise ValueError

exam=Exam()
exam.chinese_grade=100
exam.math_grade=88
print(exam.__dict__)
```

```
{'chinese_grade': 100, 'math_grade': 88}
```

这样可以正常运行.但是表意不够丰富 python提供了描述符协议,像这样

```
class Grade:
    def __set__(self,k,v):
        print(k,v)
        self.__dict__[k]=v
    def __get__(self,k):
        return self.__dict__[k]


class Exam:
    math_grade=Grade()
    chinese_grade=Grade()
e=Exam()
e.math_grade=1
e.chinese_grade=2
```

```
<__main__.Exam object at 0x0000023C09A75388> 1
<__main__.Exam object at 0x0000023C09A75388> 2
```

如果一个类支持描述符协议 那么他在被访问时 会触发 `__set__` 和 `__get__` 方法

这时候就达到类似的效果了

```
class Grade:
    def __set__(self, instance, value):
        print(instance,value)
        if 0<=value<=100:
            self.__dict__[instance]=value
        else:
            raise AttributeError
    def __get__(self,instance,instance_type):

        return self.__dict__[instance]


class Exam:
    math_grade=Grade()
    chinese_grade=Grade()

e=Exam()
e.math_grade=100
e.chinese_grade=100
print(e.math_grade)

e2=Exam()
e2.math_grade=12
e2.chinese_grade=11
print(e.math_grade)
```

```
<__main__.Exam object at 0x000001BCDA4C5408> 100
<__main__.Exam object at 0x000001BCDA4C5408> 100
100
<__main__.Exam object at 0x000001BCDA70F688> 12
<__main__.Exam object at 0x000001BCDA70F688> 11
100
```

如果不以字典的方式存储,像这样,会出现静态变量共享的问题

```
class Grade:
    def __init__(self):
        self._value=None
    def __set__(self, instance, value):
        if 0<=value<=100:
            self._value=value
        else:
            raise AttributeError
    def __get__(self,instance,instance_type):
        return self._value


class Exam:
    math_grade=Grade()
    chinese_grade=Grade()

e=Exam()
e.math_grade=100
e.chinese_grade=100
print(e.math_grade)

e2=Exam()
e2.math_grade=12
e2.chinese_grade=11
print(e.math_grade)
```

```
100
12
```

可以看到 第二个实例改变了第一个实例的值 因为这个变量他在Exam里面是静态的 .访问了同一个变量并且改变了他

但是,如果用字典来处理这个,将会造成一个很严重的问题就是 内存泄露

Grade中的 字典始终有一个对Exam的引用 这就导致了引用计数器不会清零,那么导致GC释放不了他

```
{<__main__.Exam object at 0x000001DF821E5408>: 100, <__main__.Exam object at 0x000001DF8242F648>: 12}
{<__main__.Exam object at 0x000001DF821E5408>: 100, <__main__.Exam object at 0x000001DF8242F648>: 11}
```

对于这种情况 python提供了弱引用机制

```
class Grade:
    def __init__(self):
        self._values=weakref.WeakKeyDictionary()
        ...
```

现在一切都能正常工作了

弱引用会被特殊处理,当他发现字典里的类引用是程序的最后一个时,将会释放该类 从而释放外部的类内存

> To fix this, I can use Python’s weakref built-in module. This module provides a special class called WeakKeyDictionary that can take the place of the simple dictionary used for _values. The unique behavior of WeakKeyDictionary is that it will remove Exam instances from its set of keys when the runtime knows it’s holding the instance’s last remaining reference in the program. Python will do the bookkeeping for you and ensure that the _values dictionary will be empty when all Exam instances are no longer in use.

### **Things to Remember** 

- Reuse the behavior and validation of @property methods by defining your own descriptor classes. 
- Use WeakKeyDictionary to ensure that your descriptor classes don’t cause memory leaks. 
- Don’t get bogged down trying to understand exactly how __getattribute__ uses the descriptor protocol for getting and setting attributes. 



## **Item 32: Use** **__getattr__****,** **__getattribute__****, and** __setattr __for Lazy Attributes** (懒属性访问处理)

对于类属性 的操作 我们之前也搞了很多 就是用于一个类的懒赋值

```
class Exam:
    def __setattr__(self, key, value):
        if 0 <= value <= 100:
            self.__dict__[key] = value
        else:
            raise ValueError
    def __getattribute__(self, item):
        return super(Exam, self).__getattribute__(item)
    def __getattr__(self, item):
        if item in self.__dict__:
            return self.__dict__[item]
        else:
            setattr(self,item,0)
            return 0

exam=Exam()
exam.chinese_grade=100
exam.math_grade=88
exam.english_grade
print(exam.__dict__)
```

```
{'chinese_grade': 100, 'math_grade': 88, 'english_grade': 0}
```

可以看到访问没有的属性的时候 首选访问getattribute 之后 会去访问 __getattr__ 那么 在这里就可以操作一下 让没有记录的成绩初始化为0

值得注意的是 如果你需要在 getattribute里 访问自己类的属性 你需要这么操作

```
class Exam:
    _data={}
    def __setattr__(self, key, value):
        Exam._data[key]=value
        super(Exam, self).__setattr__(key,value)

    def __getattribute__(self, item):
        _data=super(Exam, self).__getattribute__("_data")
        return _data[item]


exam=Exam()
exam.math=100
exam.chinese=100
print(exam.math)
```

这样写 外部只能访问_data 变量里的内容 否则就会报错

### **Things to Remember** 

- Use __getattr__ and __setattr__ to lazily load and save attributes for an object.
- Understand that __getattr__ only gets called once when accessing a missing attribute, whereas __getattribute__ gets called every time an attribute is accessed. 
- Avoid infinite recursion in __getattribute__ and __setattr__ by using methods from super() (i.e., the object class) to access instance attributes directly

## **Item 33: Validate Subclasses with Metaclasses** (声明类定义时,使用元类可以提早发现错误)

我们知道 一生二 二生三 三生万物.

对应的类的关系 其实是这样的

一是 元类 二是类 三是类的属性和方法 万物则是各种衍生的对象

所以说 元类是很重要的基石  只要我们学会了元类的使用 那么后面的行为都可以被元类限定

先看下面这个栗子 它指定了该类声明时使用了元类

```
class Meta(type):
    def __new__(meta, name, bases, class_dict):
        print((meta, name, bases, class_dict))
        return type.__new__(meta, name, bases, class_dict)

class MyClass(object, metaclass=Meta):
    stuff = 123
    def foo(self):
        pass
```

这个栗子会打印出

```
(<class '__main__.Meta'>,
 'MyClass',
 (<class 'object'>,),
 {'__module__': '__main__',
  '__qualname__': 'MyClass',
  'foo': <function MyClass.foo at 0x000001F81941DDC8>,
  'stuff': 123})
```

第一个是元类类型 meta 他本身也是类 第二个是定义的类名字 第三是它的父类 

大火都知道所有类默认继承object

第四个则是这个类的属性和方法字典 那么这意味着你可以像这样创建一个类

```
    def __init__(cls, what, bases=None, dict=None): # known special case of type.__init__
        """
        type(object_or_name, bases, dict)
        type(object) -> the object's type
        type(name, bases, dict) -> a new type
        # (copied from class doc)
        """
        pass
```

```
CustomClass = type( "CustomClass",(object,), {"stuff": 124})
cc=CustomClass()
```

但是对自动补全不友好 所以挺难受的用着

python2有一些细小的区别 

```
Python 2 
class Meta(type): 
    def __new__(meta, name, bases, class_dict): 
		...
class MyClassInPython2(object): 
    __metaclass__ = Meta # …
```

那么元类可以干嘛呢?  当然 越底层能做的事越多

继承该元类的子类 都会被检查是否属性有错误

```
class ValidatePolygon(type):
    def __new__(meta, name, bases, class_dict):
        # Don’t validate the abstract Polygon class
        if bases != (object,):
            if class_dict['sides'] < 3:
                raise ValueError('Polygons need 3+ sides')
        return type.__new__(meta, name, bases, class_dict)


class Polygon(object, metaclass=ValidatePolygon):
    sides = None  # Specified by subclasses

    @classmethod
    def interior_angles(cls):
        return (cls.sides - 2) * 180


class Triangle(Polygon):
    sides = 3


class Line(Polygon):
    sides = 1

```

这会在声明的时候抛出

```
Traceback (most recent call last):
  File "G:/workplace/main.py", line 851, in <module>
    class Line(Polygon):
  File "G:/workplace/main.py", line 835, in __new__
    raise ValueError('Polygons need 3+ sides')
ValueError: Polygons need 3+ sides
```

还是那句话 越早发现错误越好 所以 元类的用处就是在声明类的时候就可以帮你做检查 和各种纠错

### **Things to Remember** 

- Use metaclasses to ensure that subclasses are well formed at the time they are defined, before objects of their type are constructed. 
- Metaclasses have slightly different syntax in Python 2 vs. Python 3. 
- The __new__ method of metaclasses is run after the class statement’s entire body has been processed.



## **Item 34: Register Class Existence with Metaclasses** (元类可以在声明类时注册这个类)

另一个元类常用的用途就是 自动注册 你声明的类

来看下面这个栗子

```
class Serializable(object):
    def __init__(self, *args):
        self.args = args
    def serialize(self):
        return json.dumps({'args': self.args})

class Deserializable(Serializable):
    @classmethod
    def deserialize(cls, json_data):
        params = json.loads(json_data)
        return cls(*params['args'])

class DePoint2D(Deserializable):
    def __init__(self, x, y):
        super().__init__(x, y)
        self.x = x
        self.y = y
    def __repr__(self):
        return 'DePoint2D(%d, %d)' % (self.x, self.y)

point = DePoint2D(5, 3)
print(point.serialize())
print(DePoint2D.deserialize(point.serialize()))
```

```
{"args": [5, 3]}
DePoint2D(5, 3)
```

该例子使用了类似插件的功能 使 Point2D 被支持去 序列化自身

但目前的插件类json只能支持这个 Point2D  当我想去支持更多类型时,我们需要这样

我们使用一个辅助的函数去把这个类注册到可以被序列化的队列里

```

class BetterSerializable(object):
    def __init__(self, *args):
        self.args = args

    def serialize(self):
        return json.dumps({"class": self.__class__.__name__, "args": self.args, })


registry = {}


def register_class(target_class):
    registry[target_class.__name__] = target_class


def deserialize(data):
    params = json.loads(data)
    name = params["class"]
    target_class = registry[name]
    return target_class(*params['args'])


class EvenBetterPoint2D(BetterSerializable):
    def __init__(self, x, y):
        super().__init__(x, y)
        self.x = x
        self.y = y


register_class(EvenBetterPoint2D)

```

这样其实还是很麻烦,因为在遇到新的类时需要显式调用一次注册

```
class Point3D(BetterSerializable): 
    def __init__(self, x, y, z): 
        super().__init__(x, y, z) 
        self.x = x 
        self.y = y 
        self.z = z
```

或者你在这个时候忘记了注册

```
point = Point3D(5, 9, -4) 
data = point.serialize() 
deserialize(data) 
>> KeyError: ‘Point3D’
```

直接GG

更好的办法是为所有继承该类的子类直接被继承 那么这时候就可以用元类

```
class Meta(type):
    def __new__(meta, name, bases, class_dict):
        cls = type.__new__(meta, name, bases, class_dict)
        register_class(cls)
        return cls


class RegisteredSerializable(BetterSerializable, metaclass=Meta):
    pass
```

只要继承了注册序列化类 那么后面的事已经不需要开发者担心了

> Using metaclasses for class registration ensures that you’ll never miss a class as long as the inheritance tree is right. This works well for serialization, as I’ve shown, and also applies to database object-relationship mappings (ORMs), plug-in systems, and system hooks. 

### **Things to Remember** 

- Class registration is a helpful pattern for building modular Python programs. 
- Metaclasses let you run registration code automatically each time your base class is subclassed in a program. 
- Using metaclasses for class registration avoids errors by ensuring that you never miss a registration call.

## **Item 35: Annotate Class Attributes with Metaclasses** (元类可以捕获定义类时的属性名并且预先赋值)

考虑这样的情况

```
class Field(object):
    def __init__(self, name):
        self.name = name
        self.internal_name = "_" + self.name
    def __get__(self, instance, instance_type):
        if instance is None: return self
        return getattr(instance, self.internal_name,'')
    def __set__(self, instance, value):
        setattr(instance, self.internal_name, value)

class Customer(object):
    #Class attributes 
    first_name = Field('first_name') 
    last_name = Field('last_name') 
    prefix = Field('prefix') 
    suffix = Field('suffix')
```

Customer类 的属性 你是不是看着很不爽 

明明左边的属性名已经有了 却还要给Field 传入一个属性名 为什么要多这一步呢?

直接在定义属性的时候给他传入不好吗?

用原文来说就是

> But it seems redundant.

你看,定义属性的时候传入 那么就元类做的事了

 

```
class Meta(type):
    def __new__(meta, name, bases, class_dict):
        print(name, bases, class_dict)
        for key, value in class_dict.items():
            if isinstance(value, Field):
                value.name = key
                value.internal_name = "_" + key
        cls = type.__new__(meta, name, bases, class_dict)
        return cls


class Field(object):
    def __init__(self):
        self.name = None
        self.internal_name = None

    def __get__(self, instance, instance_type):
        if instance is None: return self
        return getattr(instance, self.internal_name, '')

    def __set__(self, instance, value):
        setattr(instance, self.internal_name, value)


class BetterCustomer(metaclass=Meta):
    first_name = Field()
    last_name = Field()
    prefix = Field()
    suffix = Field()
```

我们在声明类的时候 就去检查 BetterCustomer定义的类属性 , 如果是 Field类 那么就将 他里面的值赋值 这样就避免了 还要显示传入 

那么这样就做到了一个 左边变量名字被右边类捕获的为字符串的操作

### **Things to Remember** 

- Metaclasses enable you to modify a class’s attributes before the class is fully defined. 
- Descriptors and metaclasses make a powerful combination for declarative behavior and runtime introspection. 
- You can avoid both memory leaks and the weakref module by using metaclasses along with descriptors

# **5.** **Concurrency and Parallelism** 

并发和并行

## **Item 36: Use** **subprocess** **to Manage Child Processes**

>  In Windows , to use echo in subprocess, you would need to use shell=True . This is because echo is not a separate executable, but rather a built-in command for the windows command line. Example -
>
> 在windows 在子进程中使用echo，需要设置 shell =True，因为 echo 不是单独的命令，而是window CMD 内置的命令 

```
proc = subprocess.Popen( ['echo', 'Hello from the child!'], stdout=subprocess.PIPE,shell=True)
out, err = proc.communicate()
print(out.decode('utf-8'))
```

现在作为一个cmd进程 他成功打印出了信息

```
"Hello from the child!"
```

现在想做一个子程序运行时 去做其他事的逻辑 也就是说等另一个进程结束之前 一直等待 或者做别的事

```
proc = subprocess.Popen(['sleep', '0.3'],shell=True)
while proc.poll() is None:
    print('Working…') # Some time-consuming work here # …
print('Exit status', proc.poll())
```

该代码会打印很多的working 因为在等待 进程结束

> 将子进程与父进程解耦意味着父进程可以自由地并行运行多个子进程。 您可以通过预先启动所有子进程来做到这一点

启动所有子进程后,轮询子进程 也就是说和子进程进行通讯,然后获得他们的信息

```
def run_sleep(period):
    proc = subprocess.Popen(['sleep', str(period)],shell=True)
    return proc
start = time.time()
procs = []
for _ in range(10):
    proc = run_sleep(1000)
    procs.append(proc)

for proc in procs:
    proc.communicate()
    end = time.time()
    print('Finished in %.3f seconds' % (end - start))
```

这种多进程并发可以用来解决很多情景下的问题

由于原文使用的环境是linux 这里就很难给出相似的例子了

值得注意的是  进程的返回可以用超时参数指定

```

proc = run_sleep(10) 
try:
    proc.communicate(timeout=0.1) 
except subprocess.TimeoutExpired: 
    proc.terminate() 
    proc.wait() 

```

> Unfortunately, the timeout parameter is only available in Python 3.3 and later. In earlier versions of Python, you’ll need to use the select built-in module on proc.stdin

不过要3.3以后的版本了

**Things to Remember** 

- Use the subprocess module to run child processes and manage their input and output streams. 
- Child processes run in parallel with the Python interpreter, enabling you to maximize your CPU usage. 
- Use the timeout parameter with communicate to avoid deadlocks and hanging child processes

## **Item 37: Use Threads for Blocking I/O, Avoid for** **Parallelism** (使用多线程处理密集I/O,避免并行)

大火都知道python的GIL不能真正意义上利用多线程,存在一个互斥锁来对线程进行管理

>  The GIL prevents these interruptions and ensures that every bytecode instruction works correctly with the CPython implementation and its C- extension modules. 

**This means that when you reach for threads to do parallel computation and speed up your Python programs, you will be sorely disappointed**

来看一个简单的例子

```
def factorize(number):
    for i in range(1, number + 1):
        if number % i == 0:
            yield i

numbers = [10000000,10000000]
start = time.time()
for number in numbers:
    list(factorize(number))
end = time.time()
print('Took %.3f seconds' % (end - start))
```



```
Took 1.181 seconds
```

那你想以多线程运行呢?

```
def factorize(number):
    for i in range(1, number + 1):
        if number % i == 0:
            yield i

numbers = [10000000,10000000]

from threading import Thread
class FactorizeThread(Thread):
    def __init__(self, number):
        super().__init__()
        self.number = number
    def run(self):
        self.factors = list(factorize(self.number))

start = time.time()
for number in numbers:
    FactorizeThread(number).start()
while threading.activeCount()!=1:

    pass
end = time.time()
print('Took %.3f seconds' % (end - start))

```

```
Took 1.790 seconds
```

特么竟然慢了这么多 我要你有何用?

原文这里举了一个串口的例子 去访问linux下的串口通信 然后就有了五倍速 这里也不方便举这个例子

> The parallel time is 5× less than the serial time. This shows that the system calls will all run in parallel from multiple Python threads even though they’re limited by the GIL. 
>
> The GIL prevents my Python code from running in parallel, but it has no negative effect on system calls. This works because Python threads release the GIL just before they make system calls and reacquire the GIL as soon as the system calls are done

如果是系统级别的调用 那么GIL是管不了的 所以一般都建议使用多进程和调用系统api

一般多线程都是放在等待网络请求 或者 密集型IO时 去做别的事.这是我以前写的爬虫的一个简单的例子

```
	def start(self):
		self.page = 20
		self.threads = []
		for _ in range(1, self.page):
			self.threads.append(threading.Thread(target=self.next_page, args=[_]))
		for t in self.threads:
			t.start()
		while (threading.active_count() != 1):
			time.sleep(.1)
		self.data_Analysis()
```

因为其他线程都在等待 GIL就可以一直切换线程来实现假的并行,就是这个道理 但是这样的开销是有的,所以新版的都是建议用协程

> There are many other ways to deal with blocking I/O besides threads, such as the asyncio built-in module, and these alternatives have important benefits. But these options also require extra work in refactoring your code to fit a different model of execution (see Item 40: “Consider Coroutines to Run Many Functions Concurrently”). Using threads is the simplest way to do blocking I/O in parallel with minimal changes to your program. 

不过线程还是最简单的方式了.虽然切换上下文会有cost

**Things to Remember** 

- Python threads can’t run bytecode in parallel on multiple CPU cores because of the global interpreter lock (GIL). 
- Python threads are still useful despite the GIL because they provide an easy way to do multiple things at seemingly the same time. 
- Use Python threads to make multiple system calls in parallel. This allows you to do blocking I/O at the same time as computation. 



## **Item 38: Use** **Lock** **to Prevent Data Races in Threads** (防止多线程数据互相打架)

这里就是多线程的最大的坑了 就是数据掐架

这里是一个简单的例子

```
class Counter(object):
    def __init__(self):
        self.count = 0

    def increment(self, offset):
        self.count += offset


def worker(sensor_index, how_many, counter):
    for _ in range(how_many):  # Read from the sensor # …
        counter.increment(1)


def run_threads(func, how_many, counter):
    threads = []
    for i in range(5):
        args = (i, how_many, counter)
        thread = threading.Thread(target=func, args=args)
        threads.append(thread)
        thread.start()
    for thread in threads:
        thread.join()


how_many = 10 ** 5
counter = Counter()
run_threads(worker, how_many, counter)
print('Counter should be %d, found %d' % (5 * how_many, counter.count))
```

这5个线程互相自增10万 但结果却是随机的

```
Counter should be 500000, found 463955
```

```
Counter should be 500000, found 500000
```

```
Counter should be 500000, found 410684
```

速度大概很快

```
Counter should be 500000, found 441309
took 0.08100652694702148
```

这就涉及到很多很多底层的东西了 我也说不清 大概就是

有些线程会被翻译成以下的语句

```
# Running in Thread A 

value_a = getattr(counter, ‘count’) 

\# Context switch to Thread B 

value_b = getattr(counter, ‘count’) 

result_b = value_b + 1 

setattr(counter, ‘count’, result_b) 

\# Context switch back to Thread A 

result_a = value_a + 1 

setattr(counter, ‘count’, result_a) 
```

> Thread A stomped on thread B, erasing all of its progress incrementing the counter. This is exactly what happened in the light sensor example above

对于这种打架的 只能劝架了

稍微改动一下即可

```
class Counter(object):
    def __init__(self):
        self.lock=threading.Lock()
        self.count = 0

    def increment(self, offset):
        with self.lock:
            self.count += offset
```

但实际上 这种速度也很慢

```
took 1.1145908832550049
```

比线性还慢很多的

```
def run_threads(func, how_many, counter):
    threads = []
    for i in range(5):
        args = (i, how_many, counter)
        func(*args)
```

```
Counter should be 500000, found 500000
took 0.0800321102142334
```

所以 加锁负担是很重的!

**Things to Remember** 

- Even though Python has a global interpreter lock, you’re still responsible for \protecting against data races between the threads in your programs. 
- Your programs will corrupt their data structures if you allow multiple threads to modify the same objects without locks. 
- The Lock class in the threading built-in module is Python’s standard mutual exclusion lock implementation

## **Item 39: Use** **Queue** **to Coordinate Work Between Threads** (线程队列可以解决生产者消费者问题)

pass

## **Item 40: Consider Coroutines to Run Many Functions** **Concurrently** (协程并发)



我们知道 python原生解释器的线程上面讨论过反而是一个缺点,他会由于GIL的存在反而还要等待,无法实现并行.

那么python3.5后面内置了一个协程的库,他可以解决GIL所带来的一些痛点.

首先线程需要开销,大概每一个线程要消耗8MB的内存,在计算机上可能不算什么,但是如果数千个线程并行,那么你可能需要一个专门的服务器去处理这种情况了

其次就是由于锁的情况,线程的需要做很多额外的处理,会使得程序的开始和结束都要附带一些操作,减缓了速度.

那么可以使用协程解决这个问题.

协程原来是通过一个 yield 关键字来实现的,这里做一个简单的演示.send作为生成器的方法给生成器传递数据

```
def my_coroutine():
    while True:
        received = yield
        print('Received:', received)
it = my_coroutine()
next(it)
# Prime the coroutine
it.send('First')
it.send('Second')
```

对于一开始来说,可能会难以理解这段代码在干嘛,实际上他将这个my_coroutine变成了一个生成器,需要使用 一次next方法激活这个生成器,他会来到 生成器的yield 处等待外部输入

否则就会报错

```
TypeError: can't send non-None value to a just-started generator
```

使用send方法向这个生成器里传递数据,在遇到下一个yield前 会将控制权返回给外部调用,直到遇到下一个send为止.

可以看到这种你叫他,他就调用,调完之后又会回来给你的行为有一点像回调函数.

那么再来看一个栗子

```
def minimize():
    current = yield
    while True:
        value = yield current

        _min = min(value, current)
        print("value=%s current=%s min=%s"%(
            value,current,_min
        ))

it=minimize()
next(it)
it.send(1)
it.send(3)
it.send(0)
```

该生成器在激活后可记住第一个值,类似闭包,然后根据后面的输入值一直比较该值

```
value=3 current=1 min=1
value=0 current=1 min=0
```

机翻还挺有意思的↓

> 生成器函数似乎会永远运行，随着每个新的 send 调用向前推进。
>
> 与线程一样，协程是独立的函数，可以使用来自其环境的输入并产生结果输出。
>
>  不同之处在于协程在生成器函数中的每个 yield 表达式处暂停，并在每次从外部调用 send 后恢复。 这就是协程的神奇机制。

### 康威生命游戏

具体可以百度查找是什么,是一个基于规则的进化游戏

> 生命游戏(Game of Life)没有游戏玩家各方之间的竞争，也谈不上输赢，可以把它归类为仿真游戏。事实上，也是因为它模拟和显示的图像看起来颇似生命的出生和繁衍过程而得名为“生命游戏”。在游戏进行中，杂乱无序的细胞会逐渐演化出各种精致、有形的结构；这些结构往往有很好的对称性，而且每一代都在变化形状。一些形状一经锁定就不会逐代变化。有时，一些已经成形的结构会因为一些无序细胞的“入侵”而被破坏。但是形状和秩序经常能从杂乱中产生出来。
>
> 生命游戏是一个二维网格游戏，这个网格中每个方格居住着一个活着或死了的细胞。一个细胞在下一个时刻的生死取决于相邻8个方格中活着或死了的细胞的数量。如果相邻方格活着的细胞数量过多，这个细胞会因为资源匮乏而在下一个时刻死去；相反，如果周围活细胞过少，这个细胞会因为孤单而死去。在游戏初始阶段，玩家可以设定周围活细胞(邻居)的数目和位置。如果邻居细胞数目设定过高，网格中大部分细胞会因为找不到资源而死去，直到整个[网格](https://baike.baidu.com/item/网格/265734)都没有生命；如果邻居细胞数目设定过低，世界中又会因为生命稀少而得不到繁衍。实际中，邻居细胞数目一般选取2或者3；这样整个生命世界才不至于太过荒凉或拥挤，而是一种动态平衡。游戏规则是：当一个方格周围有两个或3个活细胞时，方格中的活细胞在下一个时刻继续存活；即使这个时刻方格中没有活细胞，在下一个时刻也会“诞生”活细胞。在这个[游戏](https://baike.baidu.com/item/游戏/33581)中，还可以设定一些更加复杂的规则，例如当前方格的状态不仅由父一代决定，而且还考虑到祖父一代的情况。
>
> 每个方格中都可放置一个生命细胞，每个生命细胞只有两种状态：
>
> “生”或“死”。用黑色方格表示该细胞为“生”，空格(白色)表示该细胞为“死”。或者说方格网中黑色部分表示某个时候某种“生命”的分布图。生命游戏想要模拟的是：随着时间的流逝，这个分布图将如何一代一代地变化

> Let me demonstrate the simultaneous behavior of coroutines with an example. Say you want to use coroutines to implement Conway’s Game of Life. The rules of the game are simple. You have a two-dimensional grid of an arbitrary size. Each cell in the grid can either be alive or empty. 
>
> ALIVE = ‘*’ 
>
> EMPTY = ‘-‘ 
>
> The game progresses one tick of the clock at a time. At each tick, each cell counts how many of its neighboring eight cells are still alive. Based on its neighbor count, each cell decides if it will keep living, die, or regenerate. Here’s an example of a 5×5 Game of Life grid after four generations with time going to the right. I’ll explain the specific rules further below

我们用协程实现一下这个游戏

首先来实现一下查询某个格子的邻居 全部使用生成器来完成,最后返回一个生存的数量

```

"查询格子类"
Query = namedtuple("Query", ('y', 'x'))
ALIVE = "*"
EMPTY = "-"
"该函数可以知道某个位置的邻居类,然后返回邻居生存的数量"
def count_neighbors(y, x):
    n_ = yield Query(y + 1, x ) # North
    ne = yield Query(y + 1, x + 1) # Northeast
    nw = yield Query(y + 1, x-1)

    e_ = yield Query(y,x+1)
    w_ = yield Query(y, x-1)

    se = yield Query(y-1, x+1)
    s_ = yield Query(y-1, x)
    sw = yield Query(y-1, x-1)


    # Define e_, se, s_, sw, w_, nw … # …
    neighbor_states = [n_, ne, e_, se, s_, sw, w_, nw]
    count = 0
    for state in neighbor_states:
        if state == ALIVE:
            count += 1
    return count
```

```
"看看 第10行第5列的格子的邻居"
it = count_neighbors(10, 5)
q1 = next(it)
"第一个返回的是北方的"
# Get the first query
print('First yield:', q1)
q2 = it.send(ALIVE) # Send q1 state, get q2
print('Second yield:', q2)
q3 = it.send(ALIVE) # Send q2 state, get q3 # …
try:
    count = it.send(EMPTY) # Send q8 state, retrieve count
except StopIteration as e:
    print("Count: ", e.value) # Value from return statement
```

接下来来定义这个格子

```
"一个格子的位置 和状态"
Transition = namedtuple("Transition", ('y', 'x', 'state'))
```

然后根据游戏的逻辑来设定生存规则

```
def game_logic(state, neighbors):
    if state == ALIVE:
        if neighbors < 2:
            return EMPTY # Die: Too few
        elif neighbors > 3:
            return EMPTY # Die: Too many
    else:
        if neighbors == 3:
            return ALIVE # Regenerate
    return state

```

对于某个格子,如果少人就去世,如果有3个人就存活

然后基于这个写出每一个格子下一步的状态

```
def step_cell(y, x):
    """
    获得 某个xy格子的状态和邻居
    把下一个状态传给逻辑判断
    然后返回该格子的生存与否
    :param y:
    :param x:
    :return:
    """
    "某个格子"
    state = yield Query(y, x)
    '使用yield from 把每一个迭代对象的元素 给生产 出去 '
    neighbors = yield from count_neighbors(y, x)
    next_state = game_logic(state, neighbors)
    yield Transition(y, x, next_state)
```

这里是一个栗子

```
it = step_cell(10, 5)
q0 = next(it) # Initial location query
print('Me: ', q0)
"设置邻居他们的状态"
q1 = it.send(ALIVE) # Send my status, get neighbor query
print('Q1: ', q1) # …
q1 = it.send(ALIVE) # Send my status, get neighbor query
print('Q1: ', q1) # …
q1 = it.send(ALIVE) # Send my status, get neighbor query
print('Q1: ', q1) # …
q1 = it.send(ALIVE) # Send my status, get neighbor query
print('Q1: ', q1) # …
q1 = it.send(ALIVE) # Send my status, get neighbor query
print('Q1: ', q1) # …
q1 = it.send(ALIVE) # Send my status, get neighbor query
print('Q1: ', q1) # …
q1 = it.send(ALIVE) # Send my status, get neighbor query
print('Q1: ', q1) # …
q1 = it.send(ALIVE) # Send my status, get neighbor query
print('Q1: ', q1) # …
t1 = it.send(EMPTY) # Send for q8, get game decision
print('Outcome: ', t1)

```

```
Me:  Query(y=10, x=5)
Q1:  Query(y=11, x=5)
Q1:  Query(y=11, x=6)
Q1:  Query(y=11, x=4)
Q1:  Query(y=10, x=6)
Q1:  Query(y=10, x=4)
Q1:  Query(y=9, x=6)
Q1:  Query(y=9, x=5)
Q1:  Query(y=9, x=4)
Outcome:  Transition(y=10, x=5, state='-')
```

接着来仿真一下这个康威游戏 对每一个格子都判断下一步的状态

```
TICK = object()
def simulate(height, width):
    while True:
        for y in range(height):
            for x in range(width):
                yield from step_cell(y, x)
        yield TICK
```

对于这个格子区域定义为

```
class Grid(object):
    """
    整个区域的大小
    """
    def __init__(self, height, width):
        self.height = height
        self.width = width
        self.rows = []
        for _ in range(self.height):
            self.rows.append([EMPTY] * self.width)
    def __str__(self):
        return '\n'.join(map(lambda s:''.join(s),self.rows))

    def query(self, y, x):
        return self.rows[y % self.height][x % self.width]

    def assign(self, y, x, state):
        self.rows[y % self.height][x % self.width] = state
```

对于每次迭代的下一代函数为

```
def live_a_generation(grid, sim):
    progeny = Grid(grid.height, grid.width)
    item = next(sim)

    while item is not TICK:
        if isinstance(item, Query):
            state = grid.query(item.y, item.x)
            item = sim.send(state)
        else:  # Must be a Transition
            progeny.assign(item.y, item.x, item.state)
            item = next(sim)
    return progeny
```

初始化一下状态

```
rows=5
cols=9
grid = Grid(rows, cols)
for row in range(rows):
    for col in range(cols):
        grid.assign(row,col,random.choice([ALIVE,EMPTY]))
```

最后开始游戏过程

```

sim = simulate(grid.height, grid.width)
i=0
while 1:
    print("epoch %s" % i)
    grid = live_a_generation(grid, sim)
    print(grid)
    time.sleep(1)
    i += 1
```

这里给几个demo

```

epoch 0
*-**-*--*
-**---*--
**-------
**--**---
*--*-----
epoch 1
*--**---*
---*----*
-----*---
--*-*---*
---*-*---
epoch 2
*-**----*
*--*----*
---**----
---***---
*-*--*--*
epoch 3
--***--*-
**------*
--*--*---
--*--*---
*-*--*--*
epoch 4
--***--*-
**--*---*
*-*------
--*****--
--*--**-*
epoch 5
--*-*-**-
*---*---*
*-*-----*
--*-*-**-
-*-------
epoch 6
**-*-*-**
*----*---
*----*---
*-**---**
-**------
epoch 7
----*-*-*
-----*---
*---*-*--
*-**----*
----*-*--
epoch 8
----*-**-
----*-**-
**-***--*
**-**--**
*---*---*
epoch 9
---**-*--
*--------
-*-------
-------*-
-*--*-*--


```

这样看还是比较丑的 换种表达

```

epoch 0
🖤🖤🖤🖤🖤🖤🖤🖤💝
🖤💝🖤💝🖤🖤🖤💝💝
🖤🖤💝🖤🖤🖤🖤🖤💝
🖤🖤🖤🖤💝🖤💝💝💝
🖤🖤🖤🖤🖤🖤🖤🖤🖤
epoch 1
💝🖤🖤🖤🖤🖤🖤💝💝
🖤🖤💝🖤🖤🖤🖤💝💝
🖤🖤💝💝🖤🖤💝🖤🖤
🖤🖤🖤🖤🖤🖤🖤💝💝
🖤🖤🖤🖤🖤🖤🖤🖤💝
epoch 2
💝🖤🖤🖤🖤🖤🖤🖤🖤
💝💝💝💝🖤🖤💝🖤🖤
🖤🖤💝💝🖤🖤💝🖤🖤
🖤🖤🖤🖤🖤🖤🖤💝💝
🖤🖤🖤🖤🖤🖤🖤🖤🖤
epoch 3
💝🖤💝🖤🖤🖤🖤🖤🖤
💝🖤🖤💝🖤🖤🖤🖤🖤
💝🖤🖤💝🖤🖤💝🖤💝
🖤🖤🖤🖤🖤🖤🖤💝🖤
🖤🖤🖤🖤🖤🖤🖤🖤💝
epoch 4
💝💝🖤🖤🖤🖤🖤🖤💝
💝🖤💝💝🖤🖤🖤🖤🖤
💝🖤🖤🖤🖤🖤🖤💝💝
💝🖤🖤🖤🖤🖤🖤💝🖤
🖤🖤🖤🖤🖤🖤🖤🖤💝
epoch 5
🖤💝💝🖤🖤🖤🖤🖤💝
🖤🖤💝🖤🖤🖤🖤💝🖤
💝🖤🖤🖤🖤🖤🖤💝🖤
💝🖤🖤🖤🖤🖤🖤💝🖤
🖤💝🖤🖤🖤🖤🖤💝🖤
epoch 6
💝💝💝🖤🖤🖤🖤💝💝
💝🖤💝🖤🖤🖤🖤💝🖤
🖤💝🖤🖤🖤🖤💝💝🖤
💝💝🖤🖤🖤🖤💝💝🖤
🖤💝💝🖤🖤🖤🖤💝🖤
epoch 7
🖤🖤🖤💝🖤🖤💝💝🖤
🖤🖤💝🖤🖤🖤🖤🖤🖤
🖤🖤💝🖤🖤🖤🖤🖤🖤
💝🖤🖤🖤🖤🖤🖤🖤🖤
🖤🖤🖤🖤🖤🖤🖤🖤🖤
epoch 8
🖤🖤🖤🖤🖤🖤🖤🖤🖤
🖤🖤💝💝🖤🖤🖤🖤🖤
🖤💝🖤🖤🖤🖤🖤🖤🖤
🖤🖤🖤🖤🖤🖤🖤🖤🖤
🖤🖤🖤🖤🖤🖤🖤🖤🖤
epoch 9
🖤🖤🖤🖤🖤🖤🖤🖤🖤
🖤🖤💝🖤🖤🖤🖤🖤🖤
🖤🖤💝🖤🖤🖤🖤🖤🖤
🖤🖤🖤🖤🖤🖤🖤🖤🖤
🖤🖤🖤🖤🖤🖤🖤🖤🖤
epoch 10
🖤🖤🖤🖤🖤🖤🖤🖤🖤
🖤🖤🖤🖤🖤🖤🖤🖤🖤
🖤🖤🖤🖤🖤🖤🖤🖤🖤
🖤🖤🖤🖤🖤🖤🖤🖤🖤
🖤🖤🖤🖤🖤🖤🖤🖤🖤
epoch 11
🖤🖤🖤🖤🖤🖤🖤🖤🖤
🖤🖤🖤🖤🖤🖤🖤🖤🖤
🖤🖤🖤🖤🖤🖤🖤🖤🖤
🖤🖤🖤🖤🖤🖤🖤🖤🖤
🖤🖤🖤🖤🖤🖤🖤🖤🖤

Process finished with exit code -1

```

这里可以清除的看到每一代是怎么消亡和繁殖的,比较厉害了



这也是协程的好处

> The best part about this approach is that I can change the game_logic function without having to update the code that surrounds it. I can change the rules or add larger spheres of influence with the existing mechanics of Query, Transition, and TICK. This demonstrates how coroutines enable the separation of concerns, which is an important design principle.

我们可以关注 游戏逻辑函数而不需要关注其他代码 用协程就决定了我们可以将游戏逻辑和游戏组成分离,这是一个很舒服的东西.

不过遗憾的是 python2 没有 yield from的语法 也就代表着只能多一层循环了 

```
# Python 2 

def delegated(): 
​	yield 1 
​	yield 2 

def composed(): 
​	yield ‘A’ 

for value in delegated(): # yield from in Python 3 
​	yield value 
​	yield ‘B’ 

print list(composed()) 

\>>> 

[‘A’, 1, 2, ‘B’]
```

还有一件事 python2 不支持在生成器里返回值 这样不会中止生成器

\# Python 2 

```
class MyReturn(Exception): 

def __init__(self, value): 
​	self.value = value 

def delegated(): 
​	yield 1 
​	raise MyReturn(2) # return 2 in Python 3 
​	yield ‘Not reached’ 

def composed(): 
​	try:
​		for value in delegated(): 
​			yield value 
	except MyReturn as e: 
​		output = e.value 

yield output * 4 
print list(composed()) 
```

### **Things to Remember** 

- Coroutines provide an efficient way to run tens of thousands of functions seemingly at the same time. 
- Within a generator, the value of the yield expression will be whatever value was passed to the generator’s send method from the exterior code. 
- Coroutines give you a powerful tool for separating the core logic of your program from its interaction with the surrounding environment. 
- Python 2 doesn’t support yield from or returning values from generators
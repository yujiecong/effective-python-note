# effective python 20-30 items 

[TOC]



## Item 21: Enforce Clarity with Keyword-Only Arguments (使用关键字参数丰富函数表意)

对于很多参数的函数,你怕是记不住几个参数的

```
def exmaple(a,b,c,d,e,f,g,h):
    pass
```

你只能像这样传入,那就没有任何代码可读性了

```
exmaple(1, 2, 3, 5, 6, 7, 8, 1)
```

你可以使用关键字参数传入

```
exmaple(a=1, b=2,...)
```

 但是你不能管控调用你函数的人是怎么调的 为了减少这种烦恼,你可以在python3里使用这种语法

```
def dump(obj, fp, *, skipkeys=False, ensure_ascii=True, check_circular=True,
        allow_nan=True, cls=None, indent=None, separators=None,
        default=None, sort_keys=False, **kw):
```

强制用户调用函数时使用关键字,那么对双方都是一种妥协.

但是python2 没有类似的语法 你可以使用另一种方式来达到一样的效果

如果不传关键字,就会报错

```
def example(number,**kwargs):
    "如果传了需要的关键字 就拿出来 如果没传就不理会"
    a=    kwargs.pop("a",False)
    b=    kwargs.pop("b",False)
    if kwargs:
        raise TypeError("检测到别的参数传入了 %s"%kwargs)

example(1,1,2)
```

```
TypeError: example() takes 1 positional argument but 3 were given
```

需要传入哦

```
example(1,a=1,b=2)
```

这样就能正常工作了

### **Things to Remember** 

- Keyword arguments make the intention of a function call more clear. 
- Use keyword-only arguments to force callers to supply keyword arguments for potentially confusing functions, especially those that accept multiple Boolean flags. 
- Python 3 supports explicit syntax for keyword-only arguments in functions. 
- Python 2 can emulate keyword-only arguments for functions by using **kwargs and manually raising TypeError exceptions

# **3.** **Classes and Inheritance** 

## **Item 22: Prefer Helper Classes Over Bookkeeping with** Dictionaries and Tuples (字典套迭代对象通常来说难以理解)

使用命名元组 工厂类 来产生一堆 不需要实例方法的对象

```
studentFactory = collections.namedtuple("Student", ["name","height", "weight"])
letian=studentFactory("啊哈吊事后的",123,321)
print(letian)
```

```
Student(name='啊哈吊事后的', height=123, weight=321)
```

如果是用字典的话可能是这样

```
def getStudentFactory(name,height,weight):
    return {
        "name":name,
        "height": height,
        "weight": weight
    }
```

如果你想要对象方法,那就麻烦你重构一下了

```
class Student:
    def __init__(self):
        pass
    def get_grade(self):
        ...
```

舒服

总之就是 如果你的数据结构很多参数或者变的更加复杂了,要用类重构一下数据结构

而不是只是一个字典列表的嵌套啦

### **Things to Remember** 

- Avoid making dictionaries with values that are other dictionaries or long tuples. 
- Use namedtuple for lightweight, immutable data containers before you need the flexibility of a full class. 
- Move your bookkeeping code to use multiple helper classes when your internal state dictionaries get complicated.

## **Item 23: Accept Functions for Simple Interfaces Instead of** **Classes** (函数对象作为函数参数)

defaultdict大家都用过,但是你想没想过他也可以传一个函数进去呢?

我们平时只会想这样用..

```
defaultDict=collections.defaultdict(list)
defaultDict["asd"].append(213213)
```

来看看别的用法

```
    def __init__(self, default_factory=None, **kwargs): # known case of _collections.defaultdict.__init__
        """
        defaultdict(default_factory[, ...]) --> dict with default factory
        
        The default factory is called without arguments to produce
        a new value when a key is not present, in __getitem__ only.
        A defaultdict compares equal to a dict with the same items.
        All remaining arguments are treated the same as if they were
        passed to the dict constructor, including keyword arguments.
        
        # (copied from class doc)
        """
```

他其实会返回一个工厂 在missing的时候调用这个函数(或者是一个构造的类函数) 我们可以自定义这种行为

主要是看对象有没有实现 `__call__` 方法

```
def get_ILoveYou():
    return "ILOVEYOU"
defaultDict=collections.defaultdict(get_ILoveYou)
print(defaultDict["123"])

```

每次缺省值都会变成我爱你,属于是很浪漫了

```
ILOVEYOU
```

但是还不只这些功能,你还可以为该字典指定一个初始值

```
def get_ILoveYou():
    return "ILOVEYOU"
defaultDict=collections.defaultdict(get_ILoveYou,{"123":"我不爱你"})
print(defaultDict["1234"])
print(defaultDict)
```

```
ILOVEYOU
defaultdict(<function get_ILoveYou at 0x0000012C436B51F8>, {'123': '我不爱你', '1234': 'ILOVEYOU'})
```

基于此你可以做出很多的工厂函数,像这样

```
class ILOVEYOU:
    def __new__(cls, *args, **kwargs):
        return "ILOVEYOU"

defaultDict=collections.defaultdict(ILOVEYOU)
print(defaultDict["1234"])
print(defaultDict)
```

```
ILOVEYOU
defaultdict(<class '__main__.ILOVEYOU'>, {'1234': 'ILOVEYOU'})
```

以及像这样

```
class ILOVEYOU:
    def __call__(self, *args, **kwargs):
        return "ILOVEYOU"
iloveyou=ILOVEYOU()
defaultDict=collections.defaultdict(iloveyou)
print(defaultDict["1234"])
print(defaultDict)

```

```
ILOVEYOU
defaultdict(<__main__.ILOVEYOU object at 0x000002A11D155048>, {'1234': 'ILOVEYOU'})
```

不过就不展开了

记住  函数是简单的接口对于一个繁重的类 

### **Things to Remember** 

- Instead of defining and instantiating classes, functions are often all you need for simple interfaces between components in Python. 
- References to functions and methods in Python are first class, meaning they can beused in expressions like any other type. 
- The __call__ special method enables instances of a class to be called like plain Python functions. 
- When you need a function to maintain state, consider defining a class that provides the __call__ method instead of defining a stateful closure (see Item 15: “Know How Closures Interact with Variable Scope”)

## **Item 24: Use** **@classmethod** **Polymorphism to Construct** **Objects Generically**  (使用类方法来达到多态)

python是没有重载构造函数的说法的,我们都知道用类方法

```
class Student:
    def __init__(self,name):
        self.name = name

    @classmethod
    def 我是普通人(cls,name):
        print('我只能是个普通人')
        return cls(name)
    @classmethod
    def 我是人上人(cls,name):
        print("我是人上人没想到吧")
        return cls(name)

这一辈子=Student.我是普通人("yjc")
print(这一辈子)
```

```
我只能是个普通人
<__main__.Student object at 0x0000022A59651408>
```

原文还提到了用类方法来实现抽象类接口,作为子类的构造方法

```
class Animals:
    def walk(self):
        raise Exception("我特么是抽象的,怎么走啊")
    @classmethod
    def gen_animal(cls):
        raise Exception("我特么抽象的")
class Dog(Animals):
    def walk(self):
        print("我直接起飞")
    @classmethod
    def gen_animal(cls):
        print("我是真的狗")
        raise cls()
```

不方便,还不如用abc

### **Things to Remember** 

- Python only supports a single constructor per class, the __init__ method. 
- Use @classmethod to define alternative constructors for your classes. 
- Use class method polymorphism to provide generic ways to build and connect concrete subclasses. 

## **Item 25: Initialize Parent Classes with** **super** (想调用父类方法 无脑用super)

```
class A:
    def __init__(self,name):
        self.name = name
class B(A):
    def __init__(self,name):
        A.__init__(self,name)

b=B(123)
print(b.name)
```

这种老旧的写法就不要再出现了

全部换成super

```
class A:
    def __init__(self,name):
        self.name = name
class B(A):
    def __init__(self,name):
        # A.__init__(self,name)
        super(B, self).__init__(name)
```

为了解决什么 钻石继承的问题 多继承是个大坑,就不要掺和了  MRO的问题还不够格触碰

**Things to Remember** 

- Python’s standard method resolution order (MRO) solves the problems of superclass initialization order and diamond inheritance. 
- Always use the super built-in function to initialize parent classes.



## **Item 26: Use Multiple Inheritance Only for Mix-in Utility** Classes (使用插件类拓展已有类的功能)

这个就比较复杂了 这个 mix-in类 我先翻译成插件类 会好理解  即插即用的类 (多继承插入)

这里定义了一个插件类 `ToDictMixin` 该插件可以将子类的属性递归序列化 

遇到字典就继续深入一层递归 遇到列表就对每一个元素递归,还遇到相同类就继续递归 递归到无法解析这个类型(比如整型或者字符串)为止



```

class ToDictMixin(object):
    def to_dict(self):
        return self._traverse_dict(self.__dict__)

    def _traverse_dict(self, instance_dict):
        output = {}
        for key, value in instance_dict.items():
            output[key] = self._traverse(key, value)
        return output

    def _traverse(self, key, value):
        if isinstance(value, ToDictMixin):
            return value.to_dict()
        elif isinstance(value, dict):
            return self._traverse_dict(value)
        elif isinstance(value, list):
            return [self._traverse(key, i) for i in value]
        elif hasattr(value, '__dict__'):
            return self._traverse_dict(value.__dict__)
        else:
            return value
class BinaryTree(ToDictMixin):
    def __init__(self, value, left=None, right=None):
        self.value = value
        self.left = left
        self.right = right

lleftChild=BinaryTree(4)
leftChild=BinaryTree(2,left=lleftChild)
rightChild=BinaryTree(3)
root=BinaryTree(1,leftChild,rightChild)
pprint.pprint(root.to_dict())
```

这里会输出

```
{'left': {'left': {'left': None, 'right': None, 'value': 4},
          'right': None,
          'value': 2},
 'right': {'left': None, 'right': None, 'value': 3},
 'value': 1}
```

插件类的好处就是即插即用,还能够根据实际情况改变行为

例如这里新增一个二叉树类,该类会记住它的父亲

```
class BinaryTreeWithParent(BinaryTree):
    def __init__(self, value, left=None, right=None, parent=None):
        super().__init__(value, left=left, right=right)
        self.parent = parent
```

如果你想在序列化的时候输出他的父亲,直接用是不行的

像这样

```
lleftChild=BinaryTreeWithParent(4)
leftChild=BinaryTreeWithParent(2,left=lleftChild)
rightChild=BinaryTreeWithParent(3)
root=BinaryTreeWithParent(1,leftChild,rightChild)
leftChild.parent=root
pprint.pprint(root.to_dict())
```

该代码会无限递归

```
RecursionError: maximum recursion depth exceeded while calling a Python object
```

因为每次遇到了左子树的parent都会递归这个parent,然后这个parent遇到了左子树,然后递归他的parent..然后..

为了解决这个问题,需要重写一下这个函数, 不需要遍历父亲 只返回父亲即可

```
class BinaryTreeWithParent(BinaryTree):
    def __init__(self, value, left=None, right=None, parent=None):
        super().__init__(value, left=left, right=right)
        self.parent = parent

    def _traverse(self, key, value):
        if isinstance(value, BinaryTreeWithParent) and key == 'parent':
            return value.value  # Prevent cycles
        else:
            return super()._traverse(key, value)
```

```
lleftChild = BinaryTreeWithParent(4)
leftChild = BinaryTreeWithParent(2, left=lleftChild)
rightChild = BinaryTreeWithParent(3)
root = BinaryTreeWithParent(1, leftChild, rightChild)
leftChild.parent = root
pprint.pprint(root.to_dict())
```

```
{'left': {'left': {'left': None, 'parent': None, 'right': None, 'value': 4},
          'parent': <__main__.BinaryTreeWithParent object at 0x000002911F1CE488>,
          'right': None,
          'value': 2},
 'parent': None,
 'right': {'left': None, 'parent': None, 'right': None, 'value': 3},
 'value': 1}
```

你甚至能再写一个 json插件类混入到一个类中

```
class JsonMixin(object):
    @classmethod
    def from_json(cls, data):
        kwargs = json.loads(data)
        return cls(**kwargs)

    def to_json(self):
        return json.dumps(self.to_dict())
```

这是一个json插件类 他能 读取json并实例化 将序列化的参数转变成json后反实例化.

```

class Switch(ToDictMixin, JsonMixin):
    def __init__(self, **kwargs):
        for k,v in kwargs.items():
            setattr(self,k,v)

class Machine(ToDictMixin, JsonMixin):
    def __init__(self, **kwargs):
        for k,v in kwargs.items():
            setattr(self,k,v)


class DatacenterRack(ToDictMixin, JsonMixin):
    def __init__(self, switch=None, machines=None):
        self.switch = Switch(**switch)
        self.machines = [Machine(**kwargs) for kwargs in machines]
   
```

序列化的数据为

```
serialized = """{ "switch": {"ports": 5, "speed": 1e9}, "machines": [ {"cores": 8, "ram": 32e9, "disk": 5e12}, 
{"cores": 4, "ram": 16e9, "disk": 1e12}, {"cores": 2, "ram": 4e9, "disk": 500e9} ] } """
```

可以看到 现在准备反序列化这段数据到实例

```
deserialized = DatacenterRack.from_json(serialized)
roundtrip = deserialized.to_json()
print(deserialized.to_dict())
print(roundtrip)

```

```
{'switch': {'ports': 5, 'speed': 1000000000.0}, 'machines': [{'cores': 8, 'ram': 32000000000.0, 'disk': 5000000000000.0}, {'cores': 4, 'ram': 16000000000.0, 'disk': 1000000000000.0}, {'cores': 2, 'ram': 4000000000.0, 'disk': 500000000000.0}]}
{"switch": {"ports": 5, "speed": 1000000000.0}, "machines": [{"cores": 8, "ram": 32000000000.0, "disk": 5000000000000.0}, {"cores": 4, "ram": 16000000000.0, "disk": 1000000000000.0}, {"cores": 2, "ram": 4000000000.0, "disk": 500000000000.0}]}
```

结果是一样的

基于此 我自己也写了一个插件类

该插件若是混入,会将子类所有的赋值更改成下划线命名

```
class NameFormatPlugin:
    _lower_alpha={chr(lower_word) for lower_word in range(ord("A"),ord("Z")+1)}
    def __setattr__(self, key:str, value):
        if not key.islower():
            strList=list(key)
            for i,word  in enumerate(strList):
                if word in self._lower_alpha:
                    strList[i]="_%s"%word
            key=''.join(strList)
        self.__dict__[key]=value

class Example(NameFormatPlugin):
    pass

e=Example()
e.tuoFengMingMingFaGeiYePa=1
e.woYaoXiaHuaXianAAAAA=2
pprint.pprint(e.__dict__)

```

```
{'tuo_Feng_Ming_Ming_Fa_Gei_Ye_Pa': 1, 'wo_Yao_Xia_Hua_Xian_A_A_A_A_A': 2}
```



### **Things to Remember** 

- Avoid using multiple inheritance if mix-in classes can achieve the same outcome. Use pluggable behaviors at the instance level to provide per-class customization 
- when mix-in classes may require it. Compose mix-ins to create complex functionality from simple behaviors. 

## **Item 27: Prefer Public Attributes Over Private Ones** (大火都是成年人了,变量怎么访问不用我多说了吧?)

说实话,写python就不要在乎强类型那一套变量访问权限(公有,继承,保护)了 大家都喜欢浅显易懂的 不需要访问修饰符.

大家都知道 python dunder变量 (self.__abc=1) 会被转译成这样

```
class MyClass:
    __abc=1

print(MyClass._MyClass__abc)
```

这样仅仅是防止外部意外访问而已,ide也会帮你在代码补全的时候隐藏这个变量.

你可以做一个接口像这样

```
class MyClass:
    __abc=1
    @staticmethod
    def get_private_abc():
        return getattr(MyClass,"_%s__abc" % MyClass.__name__)

print(MyClass.get_private_abc())
```

为什么要这么做呢?

python之父这么说

> Why doesn’t the syntax for private attributes actually enforce strict visibility? The simplest answer is one often-quoted motto of Python: “**We are all consenting adults here**.” 
>
> Python programmers believe that the benefits of being open outweigh the downsides of being closed.

别的不用管太多 只记住单下划线是 该类的和子类的 内部变量 双下划线是该类的私有变量就好了.

### **Things to Remember** 

- Private attributes aren’t rigorously enforced by the Python compiler. 
- Plan from the beginning to allow subclasses to do more with your internal APIs and attributes instead of locking them out by default. 
- Use documentation of protected fields to guide subclasses instead of trying to force access control with private attributes. 
- Only consider using private attributes to avoid naming conflicts with subclasses that are out of your control.

## **Item 28: Inherit from** **collections.abc** **for Custom** **Container Types** (继承抽象类编写自己的类)

这玩意升过级其实.有过变动,如果直接继承以前的东西会被告知废弃

```
class MyList(collections.Iterable):
    pass
```

```
DeprecationWarning: Using or importing the ABCs from 'collections' instead of from 'collections.abc' is deprecated since Python 3.3,and in 3.9 it will stop working
  class MyList(collections.Iterable):
```

新东西来了,我们要这么继承

```
class MyList(collections.abc.Iterable):
    pass
```

我们写一个简单的列表类来实现emoji列表

首先得实现它给的迭代协议,否则会报错

```
TypeError: Can't instantiate abstract class EmojiList with abstract methods __iter__
```



```
class EmojiList(collections.abc.Iterable):
    def __init__(self):
        super(EmojiList, self).__init__()
        pass
    def __iter__(self):
        for i in range(10):
            yield "😀"
for emoji in EmojiList():
    print(emoji)
```

这样我们就能实现一个emoji 列表了

如果嫌麻烦你可以直接继承列表

```
class EmojiList(list):
    def __init__(self):
        super(EmojiList, self).__init__()
        pass
    def __iter__(self):
        for i in range(10):
            yield "😀"
for emoji in EmojiList():
    print(emoji)
```

那么这两个的区别在哪里呢?

在fluent python说

> 在 Python 2.2 之前，内置类型（如 list 或 dict）不能子类化。在 Python 2.2 之后，内置类型可以子类化了，但是有个重要的注意事项： 内置类型（使用 C 语言编写）不会调用用户定义的类覆盖的特殊方法

> 至于内置类型的子类覆盖的方法会不会隐式调用，CPython 没有制 定官方规则。基本上，内置类型的方法不会调用子类覆盖的方法。 例如，dict 的子类覆盖的 __getitem__() 方法不会被内置类型的 get() 方法调用。

```
>>> class DoppelDict(dict): 
    ... def __setitem__(self, key, value): 
    ... 	super().__setitem__(key, [value] * 2) # ➊ 
... 

\>>> dd = DoppelDict(one=1) # ➋ 

\>>> dd 

{'one': 1} 

\>>> dd['two'] = 2 # ➌ 

\>>> dd 

{'one': 1, 'two': [2, 2]} 

\>>> dd.update(three=3) # ➍ 

\>>> dd 

{'three': 3, 'one': 1, 'two': [2, 2]} 
```

> 原生类型的这种行为违背了面向对象编程的一个基本原则：始终应该从 实例（self）所属的类开始搜索方法，即使在超类实现的类中调用也是 如此。在这种糟糕的局面中，__missing__ 方法（参见 3.4.2 节）却能 按预期方式工作，不过这只是特例。 不只实例内部的调用有这个问题（self.get() 不调用 self.__getitem__()），内置类型的方法调用的其他类的方法，如果 被覆盖了，也不会被调用。

所以说了这么多 直接继承内置类型是有问题的,所以用抽象基类才能避免这个问题

**Things to Remember** 

- Inherit directly from Python’s container types (like list or dict) for simple use cases. 
- Beware of the large number of methods required to implement custom container types correctly. 
- Have your custom container types inherit from the interfaces defined in collections.abc to ensure that your classes match required interfaces and behaviors.

# **4.** **Metaclasses and Attributes**

python最难的地方就是元对象了,玩得好能特么出花❀来

## **Item 29: Use Plain Attributes Instead of Get and Set** **Methods** (别学java的get/set接口)

对于内部变量,你想在外部获取,你可能会这么写

```
class OldResistor(object): 
    def __init__(self, ohms): 
        self._ohms = ohms 
    def get_ohms(self): 
        return self._ohms 
    def set_ohms(self, ohms): 
        self._ohms = ohms
```

他们在一定程度上确实有用

> These utility methods do help define the interface for your class, making it easier to encapsulate functionality, validate usage, and define boundaries.

然而在python,建议不要使用这种方法

> you almost never need to implement explicit setter or getter methods. 
>
> Instead, you should always start your implementations with simple public attributes

```
class Resistor(object): 
    def __init__(self, ohms): 
        self.ohms = ohms 
        self.voltage = 0 
        self.current = 0 
r1 = Resistor(50e3) 
r1.ohms = 10e3
```

如果非要用,请使用装饰器property

在pycharm里可以快速生成模板

![1631890438082](effective%20python%2020-30%20items.assets/1631890438082.png)

```
class Resistor(object):
    def __init__(self, ohms):
        self._ohms = ohms
        self.voltage = 0
        self.current = 0

    @property
    def ohms(self):
        return self._ohms
    @ohms.setter
    def ohms(self,ohms):
        self._ohms=ohms
        print('ohms setter')


r1 = Resistor(50e3)
r1.ohms = 10e3
```

你可以在外部操作该属性的时候进行判断,例如非法输入的判断

```
    @ohms.setter
    def ohms(self,ohms):
        if ohms<0:
            raise ValueError
        self._ohms=ohms
        print('ohms setter')
```

定义属性有个好处,就是在构造函数的时候就能够判断出来

```
class Resistor(object):
    def __init__(self, ohms):
        self.ohms = ohms
        self.voltage = 0
        self.current = 0

    @property
    def ohms(self):
        return self._ohms
    @ohms.setter
    def ohms(self,ohms):
        if ohms<0:
            raise ValueError
        self._ohms=ohms
        print('ohms setter')

class Resistor2(Resistor):
    def __init__(self,ohms):
        super(Resistor2, self).__init__(ohms)

r1 = Resistor2(-50e3)

```

```
Traceback (most recent call last):
  File "G:/workplace/main.py", line 659, in <module>
    r1 = Resistor2(-50e3)
  File "G:/workplace/main.py", line 657, in __init__
    super(Resistor2, self).__init__(ohms)
  File "G:/workplace/main.py", line 641, in __init__
    self.ohms = ohms
  File "G:/workplace/main.py", line 651, in ohms
    raise ValueError
ValueError

```

基于此 你可以做一个只能对一个属性设置一次的变量

```
class MyClass:
    def __init__(self):
        self._a=1
    @property
    def a(self):
        return self._a

    @a.setter
    def a(self, value):
        if hasattr(self,"_a"):
            raise AttributeError
        self._a=value

c=MyClass()
c.a=1
```

```
  File "G:/workplace/main.py", line 675, in <module>
    c.a=1
  File "G:/workplace/main.py", line 671, in a
    raise AttributeError
```

还有一个特别的忠告,如果使用getter和setter方法 千万不要混入其他的属性

> Finally, when you use @property methods to implement setters and getters, be sure that the behavior you implement is not surprising. For example, don’t set other attributes in getter property methods. 

```
class MysteriousResistor(Resistor): 
    @property 
    def ohms(self): 
        self.voltage = self._ohms * self.current 
        return self._ohms
```

很多时候这不是你想要的结果

```
r7 = MysteriousResistor(10) 
r7.current = 0.01 
print(‘Before: %5r’ % r7.voltage) 
r7.ohms 
print(‘After: %5r’ % r7.voltage) 

Before: 0 
After: 0.1 
```

### **Things to Remember** 

- Define new class interfaces using simple public attributes, and avoid set and get methods. 
- Use @property to define special behavior when attributes are accessed on your objects, if necessary. 
- Follow the rule of least surprise and avoid weird side effects in your @property methods. 
- Ensure that @property methods are fast; do slow or complex work using normal methods.

## **Item 30: Consider** **@property** **Instead of Refactoring** Attributes ( 使用property装饰器为已有属性添加新功能)

> The built-in @property decorator makes it easy for simple accesses of an instance’s attributes to act smarter

同29

### **Things to Remember** 

- Use @property to give existing instance attributes new functionality. Make incremental progress toward better data models by using @property. 
- Consider refactoring a class and all call sites when you find yourself using @property too heavily.






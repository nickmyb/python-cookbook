类与对象
=======

## 8.1 改变对象的字符串显示

```python
class Coordinate:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __repr__(self):
        # 标准是 eval(repr(x)) == x 或 <>标识的有用文本
        return f'Coordinate(x={self.x}, y={self.y})'

    def __str__(self):
        # 如果没有定义会自动调用__repr__
        return self.__repr__()

```

## 8.2 自定义字符串的格式化

```python
class Date:
    def __init__(self, y, m, d):
        self.y = y
        self.m = m
        self.d = d

    def __format__(self, spec):
        if spec == '':
            # format在调用这个方法的时候如果不传入参数默认是传入空字符串
            spec = 'ymd'
        _format = {
            'ymd': '{0.y}-{0.m}-{0.d}',
            'mdy': '{0.m}/{0.d}/{0.y}'
        }
        return _format[spec].format(self)


date = Date(2020, 7, 1)
format(date)
format(date, 'mdy')

```

## 8.3 让对象支持上下文管理协议

```python
from functools import partial
from socket import socket, AF_INET, SOCK_STREAM


class LazyConnection:
    def __init__(self, address, family=AF_INET, type_=SOCK_STREAM):
        self.address = address
        self.family = family
        self.type_ = type_
        self.connections = []

    def __enter__(self):
        # with语句触发__enter__, 返回值赋给as的变量
        s = socket(self.family, self.type_)
        s.connect(self.address)
        self.connections.append(s)
        return s

    def __exit__(self, exp_type, exp_value, exp_tb):
        # with语句结束时触发,返回值为True的话异常被清空
        self.connections.pop().close()


conn = LazyConnection(('www.python.org', 80))

with conn as s1:
    s1.send(b'GET /headers HTTP/1.0\r\n')
    s1.send(b'Host: www.httpbin.org\r\n')
    s1.send(b'\r\n')
    httpbin_response = b''.join(iter(partial(s1.recv, 8192), b''))
    print('httpbin response:', httpbin_response)

    with conn as s2:
        s2.send(b'GET /index.html HTTP/1.0\r\n')
        s2.send(b'Host: www.python.org\r\n')
        s2.send(b'\r\n')
        python_response = b''.join(iter(partial(s2.recv, 8192), b''))
        print('python response:', python_response)

```

## 8.4 创建大量对象时节省内存方法

```python
class Date:
    # __slots__初衷是作为内存优化的工具,占用内存大小类似tuple,并且不能再添加额外属性,对于类的一些操作也不再支持
    # __slots__ 和 __getitem__是不同的,不会相互覆盖影响
    # __slots__: 对应 obj.attr
    # __getitem__: 对应 obj[attr]
    __slots__ = ['year', 'month', 'day']
    
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day

```

## 8.5 在类中封装属性名

```python
# type_: 通过后缀下划线避免保留字冲突

class A:
    def __init__(self):
        # _attr就是私有属性
        self._m = 1
        # __attr在继承中无法覆盖,_classname__attrname
        self.__n = 10

class B(A):
    def __init__(self):
        super().__init__()
        self._m = 2
        self.__n = 20
    
    def get__n(self):
        # 在类外部读取需要用._B__n
        return self.__n


a = A()
b = B()
print(b._m)
print(b._B__n)

```

## 8.6 创建可管理的属性

```python
# 有除访问与修改之外的其他处理逻辑才考虑使用property(少用property)
class Person:
    def __init__(self, name):
        self.name = name

    # 属性需要被定义后才能使用setter,deleter
    # name = property(get_func, set_func, del_func)也可以定义property
    # Person.name.fget/fset/fdel是实际绑定的方法,class.attr是一个property
    @property
    def name(self):
        # 可以用于动态计算属性的方法,只有调用时才生成,不实际存储
        return self._name * 2

    @name.setter
    def name(self, name):
        if isinstance(name, str):
            self._name = name
        else:
            raise TypeError('name must be a str!')

    @name.deleter
    def name(self):
        raise AttributeError("name can't be deleted!")

```

## 8.7 调用父类方法

```python
# 子类中调用父类的已经被覆盖的方法
class Base:
    def __init__(self):
        print('Base.__init__')

class A(Base):
    def __init__(self):
        super().__init__()
        print('A.__init__')

class B(Base):
    def __init__(self):
        super().__init__()
        print('B.__init__')

class C(A,B):
    def __init__(self):
        super().__init__()
        print('C.__init__')


# MRO就是深度优先,相同的取最后一个
# C A B -> C A Base B Base -> C A B Base
# super就是根据MRO层层递进,感觉是会默认记录当前所在MRO的位置
# 继承体系中所有相同名字的方法拥有可兼容的参数签名,最好确保最顶层的类提供了这个方法的实现
# TODO: https://rhettinger.wordpress.com/2011/05/26/super-considered-super/
C.__mro__

```

## 8.8 子类中扩展property

```python
class Person:
    def __init__(self, name):
        self.name = name

    @property
    def name(self):
        return self._name

    @name.setter
    def name(self, value):
        if not isinstance(value, str):
            raise TypeError('Expected a string')
        self._name = value

    @name.deleter
    def name(self):
        raise AttributeError("Can't delete attribute")

# 拓展整个property
class SubPerson(Person):
    @property
    def name(self):
        print('Getting name')
        return super().name

    @name.setter
    def name(self, value):
        print('Setting name to', value)
        # 深入理解super
        # https://docs.python.org/zh-cn/3/library/functions.html?highlight=super#super
        # super可以简单理解为返回MRO的一个指定位置
        # If the second argument is omitted, the super object returned is unbound: 暂时理解为如果是忽略第二个参数就是绑定到对象(self)的,否则绑定到类
        super(SubPerson, SubPerson).name.__set__(self, value)

    @name.deleter
    def name(self):
        print('Deleting name')
        super(SubPerson, SubPerson).name.__delete__(self)

# 拓展property的单个方法
class SubPersonSingle(Person):
    # 如果同时定义getter/setter/deleter,只有最后定义的那个方法生效
    # 这里必须使用定义setter方法的父类名Person,如果不知道具体哪个父类实现这个方法就需要重定义整个property,并使用super()来将控制权传递
    @Person.name.setter
    def name(self, value):
        print('Setting name to', value)
        super(SubPersonSingle, SubPersonSingle).name.__set__(self, value)


# 对于super的深入理解
# super可以简单理解为返回MRO的一个指定位置,并且绑定到实例或者类
class A:
    def f(self):
        print('A f')

class B(A):
    def f(self):
        # 绑定到类
        A_f = super(B, B).f
        # 绑定到实例
        instance_f = super().f
        print('A_f:', A_f)
        print('instance_f:', instance_f)
        A_f(self)
        instance_f()

```

## 8.9 创建新的类或实例属性

`descriptor`: 实现了__get__, __set__, __delete__的类,必须是类变量,当有很多重复代码的时候描述器就很有用

```python
class TypeChecked:
    def __init__(self, name, legal_type):
        self.name = name
        self.legal_type = legal_type

    def __get__(self, instance, cls):
        # instance: descriptor作为类变量访问为None,作为实例属性访问为descriptor所在类的实例
        # cls: descriptor所在类
        if instance is None:
            return self
        return instance.__dict__[self.name]

    def __set__(self, instance, value):
        if isinstance(value, self.legal_type):
            instance.__dict__[self.name] = value
        else:
            raise TypeError(f'{self.name} must be {self.legal_type}!')

    def __delete__(self, instance):
        del instance.__dict__[self.name]


def class_attribute_type_check(**kwargs):
    def decorator(cls):
        for name, legal_type in kwargs.items():
            # 只能设置类的name变量
            # cls.name = TypeChecked(name, legal_type)
            # TypeError: 'type' object does not support item assignment
            # cls[name] = TypeChecked(name, legal_type)
            setattr(cls, name, TypeChecked(name, legal_type))
        return cls
    return decorator


@class_attribute_type_check(name=str, age=int)
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

```

## 8.10 使用延迟计算属性

```python
# descriptor实现懒加载
class lazyproperty:
    def __init__(self, method):
        # 传入的method是unbound
        self.method = method

    # 如果一个描述器只定义__get__的话,只有当被访问属性不在实例底层的字典中时__get__才会被触发
    def __get__(self, instance, cls):
        if instance is None:
            return self
        else:
            value = self.method(instance)
            setattr(instance, self.method.__name__, value)
            return value

class Square:
    def __init__(self, edge):
        self.edge = edge

    @lazyproperty
    def perimeter(self):
        return 4 * self.edge

    @lazyproperty
    def area(self):
        return self.edge * self.edge


# property实现懒加载并且无法修改属性值(property不设置setter方法)
def lazyproperty(method):
    name = '_lazy_' + method.__name__

    @property
    def lazy(self):
        if hasattr(self, name):
            return getattr(self, name)
        else:
            value = method(self)
            setattr(self, name, value)
            return value
    # 返回一个只有getter方法的property
    return lazy


class Square:
    def __init__(self, edge):
        self.edge = edge

    @lazyproperty
    def perimeter(self):
        return 4 * self.edge

    @lazyproperty
    def area(self):
        return self.edge * self.edge

```

## 8.11 简化数据结构的初始化

```python
class Structure:
    _fields = []

    def __init__(self, *args, **kwargs):
        if len(args) > len(self._fields):
            raise AttributeError(f'Structure allows {len(self._fields)} at most!')

        for k, v in zip(self._fields, args):
            # 其实就是通过setattr设置属性
            # __slots__ / property 无法通过 __dict__访问
            setattr(self, k, v)

        for k in self._fields[len(args):]:
            setattr(self, k, kwargs.pop(k))

        if kwargs:
            illegal_keys = ', '.join(kwargs)
            raise AttributeError(f'[{illegal_keys}] are not allowed!')

```

## 8.12 定义接口或者抽象基类

```python
from abc import ABCMeta, abstractmethod


# 抽象基类主要用于配合isinstance做类型检查
# @abstractmethod可以添加staticmethod/classmethod/property装饰器但abstractmethod需要为第一层(最靠近实际方法层)
class IStream(metaclass=ABCMeta):
    @abstractmethod
    def read(self, maxbytes=-1):
        pass

    @abstractmethod
    def write(self, data):
        pass

# 抽象类实现一: 继承
class SocketStream(IStream):
    def read(self, maxbytes=-1):
        pass

    def write(self, data):
        pass

# 抽象类实现二: 注册
import io


IStream.register(io.IOBase)
f = open('spam.txt')
isinstance(f, IStream)
```

## 8.13 实现数据模型的类型约束

```python
# 方式一: Descriptor
class Descriptor:
    def __init__(self, name=None, **kwargs):
        self.name = name
        for k, v in kwargs.items():
            setattr(self, k, v)

    def __set__(self, instance, value):
        # 这里如果用setattr会导致RecursionError,又进到self.attr = value,死循环
        # setattr(instance, self.name, value)
        # 描述器如果只是从底层实例字典中获取某个属性值的话不需要定义__get__,默认会先从__dict__中找这个属性
        instance.__dict__[self.name] = value

class LegalType(Descriptor):
    legal_type = type(None)

    def __set__(self, instance, value):
        if isinstance(value, self.legal_type):
            super().__set__(instance, value)
        else:
            raise TypeError(f'{self.name} must be {legal_type}!')

class Sized(Descriptor):
    def __init__(self, name=None, **kwargs):
        if 'max_size' not in kwargs:
            raise TypeError('missing max_size param!')
        super().__init__(name, **kwargs)

    def __set__(self, instance, value):
        if len(value) > self.max_size:
            raise AttributeError(f"{self.name}'s length is {self.max_size} at most!")
        super().__set__(instance, value)

class SizedString(LegalType, Sized):
    legal_type = str

class Person:
    name = SizedString(name='name', max_size=10)

    def __init__(self, name):
        self.name = name


p = Person(name='nick')


# 方式二: decorator
def attributes_checker(**kwargs):
    def decorator(cls):
        for k, v in kwargs.items():
            v.name = k
            setattr(cls, k, v)
        return cls
    return decorator


@attributes_checker(name=SizedString(max_size=10))
class Student:
    def __init__(self, name):
        self.name = name


# 方式三: metaclass
class AttributesCheckMeta(type):
    def __new__(cls, clsname, bases, methods):
        for k, v in methods.items():
            if isinstance(v, Descriptor):
                # methods中含有类似__module__ / __qualname__的额外属性
                v.name = k
        return type.__new__(cls, clsname, bases, methods)


class Employee(metaclass=AttributesCheckMeta):
    name = SizedString(max_size=10)
    def __init__(self, name):
        self.name = name


# 方式四: 装饰器代替类继承,速度更快
def Unsigned(cls):
    super_set = cls.__set__
    def __set__(self, instance, value):
        if value < 0:
            raise ValueError('value must be bigger than 0!')
        super_set(self, instance, value)
    cls.__set__ = __set__
    return cls


@Unsigned
class Unsigned(Descriptor):
    pass

```

## 8.14 实现自定义容器

```python
# DeprecationWarning: Using or importing the ABCs from 'collections' instead of from 'collections.abc' is deprecated since Python 3.3, and in 3.9 it will stop working
import collections.abc


# collections实现了很多抽象基类
# 很多抽象类会为一些常见容器操作提供默认的实现
# MutableSequence就实现了append / remove / count
class Items(collections.abc.MutableSequence):
    def __init__(self, initial=None):
        self._items = list(initial) if initial is not None else []

    def __getitem__(self, index):
        print('Getting:', index)
        return self._items[index]

    def __setitem__(self, index, value):
        print('Setting:', index, value)
        self._items[index] = value

    def __delitem__(self, index):
        print('Deleting:', index)
        del self._items[index]

    def insert(self, index, value):
        print('Inserting:', index, value)
        self._items.insert(index, value)

    def __len__(self):
        print('Len')
        return len(self._items)

```

## 8.15 属性的代理访问

```python
class Proxy:
    def __init__(self, obj):
        self._obj = obj

    def __getattr__(self, name):
        # 属性不存在时才会调用__getattr__
        # 可以调用例如Proxy().__len__, 但不能调用len(Proxy()),需要对Proxy定义__len__
        print('getattr:', name)
        return getattr(self._obj, name)

    def __setattr__(self, name, value):
        if name.startswith('_'):
            super().__setattr__(name, value)
        else:
            print('setattr:', name, value)
            setattr(self._obj, name, value)

    def __delattr__(self, name):
        if name.startswith('_'):
            super().__delattr__(name)
        else:
            print('delattr:', name)
            delattr(self._obj, name)


p = Proxy([])
p.__len__()
len(p)

```

## 8.16 在类中定义多个构造器

```python
class Constructor:
    def __init__(self, name):
        self.name = name

    @classmethod
    def constructor(cls, name):
        return cls(f'Constructor({name})')


c1 = Constructor('Hello')
c2 = Constructor.constructor('Hello')

```

## 8.17 创建不调用init方法的实例

```python
class Constructor:
    def __init__(self, name):
        self.name = name

    @classmethod
    def constructor(cls, name):
        return cls(f'Constructor({name})')

    @classmethod
    def constructor_new_1(cls):
        c = Constructor.__new__(Constructor)
        c.name = 'constructor_new_1'
        return c

    @classmethod
    def constructor_new_2(cls):
        c = object.__new__(Constructor)
        c.name = 'constructor_new_2'
        return c

    @classmethod
    def constructor_new_3(cls):
        c = cls.__new__(cls)
        c.name = 'constructor_new_3'
        return c

```

## 8.18 利用Mixins扩展类功能

```python
class LoggedMappingMixin:
    # Mixin不能直接被实例化使用: 没有__init__
    # Mixin没有自己的状态信息: 没有实例属性
    __slots__ = ()

    def __getitem__(self, key):
        print('Getting ' + str(key))
        return super().__getitem__(key)

    def __setitem__(self, key, value):
        print('Setting {} = {!r}'.format(key, value))
        return super().__setitem__(key, value)

    def __delitem__(self, key):
        print('Deleting ' + str(key))
        return super().__delitem__(key)

# LoggedMappingMixin需要在前,否则先调用dict的方法
class LoggedDictDemo(LoggedMappingMixin, dict):
    pass

class LoggedDictMagic(LoggedMappingMixin, dict):
    __slots__ = ('name')


d = LoggedDictMagic()
d['name'] = 'logged_dict_magic'
d['type'] = 'LoggedDictMagic'
d['name']
d['type']
d.name = 'd.name'
d.name
d.type = 'd.type'
d.type

```

## 8.19 实现状态对象或者状态机

```python
# 为每个状态定义一个对象,避免过多的if判断
# 方式一: 通过__class__直接切换实例类型
class Connection:
    def __init__(self):
        self.new_state(ClosedConnection)

    def new_state(self, state):
        print('old self.__class__:', self.__class__)
        self.__class__ = state
        print('new self.__class__:', self.__class__)

    def read(self):
        raise NotImplementedError()

    def write(self, data):
        raise NotImplementedError()

    def open(self):
        raise NotImplementedError()

    def close(self):
        raise NotImplementedError()

class ClosedConnection(Connection):
    def read(self):
        raise RuntimeError('Not open')

    def write(self, data):
        raise RuntimeError('Not open')

    def open(self):
        self.new_state(OpenConnection)

    def close(self):
        raise RuntimeError('Already closed')

class OpenConnection(Connection):
    def read(self):
        print('reading')

    def write(self, data):
        print('writing')

    def open(self):
        raise RuntimeError('Already open')

    def close(self):
        self.new_state(ClosedConnection)

# Example
if __name__ == '__main__':
    c = Connection()
    print(c)
    try:
        c.read()
    except RuntimeError as e:
        print(e)

    print('old type:', type(c))
    c.open()
    print('new type:', type(c))
    print(c)
    c.read()
    c.close()
    print(c)

# 方式二: 代理
class Connection:
    def __init__(self):
        self.new_state(ClosedConnectionState)

    def new_state(self, newstate):
        self._state = newstate

    def read(self):
        return self._state.read(self)

    def write(self, data):
        return self._state.write(self, data)

    def open(self):
        return self._state.open(self)

    def close(self):
        return self._state.close(self)

class ConnectionState:
    @staticmethod
    def read(conn):
        raise NotImplementedError()

    @staticmethod
    def write(conn, data):
        raise NotImplementedError()

    @staticmethod
    def open(conn):
        raise NotImplementedError()

    @staticmethod
    def close(conn):
        raise NotImplementedError()

class ClosedConnectionState(ConnectionState):
    @staticmethod
    def read(conn):
        raise RuntimeError('Not open')

    @staticmethod
    def write(conn, data):
        raise RuntimeError('Not open')

    @staticmethod
    def open(conn):
        conn.new_state(OpenConnectionState)

    @staticmethod
    def close(conn):
        raise RuntimeError('Already closed')

class OpenConnectionState(ConnectionState):
    @staticmethod
    def read(conn):
        print('reading')

    @staticmethod
    def write(conn, data):
        print('writing')

    @staticmethod
    def open(conn):
        raise RuntimeError('Already open')

    @staticmethod
    def close(conn):
        conn.new_state(ClosedConnectionState)


if __name__ == '__main__':
    c = Connection()
    print(c)
    try:
        c.read()
    except RuntimeError as e:
        print(e)

    c.open()
    print(c)
    c.read()
    c.close()
    print(c)

```

## 8.20 通过字符串调用对象方法

```python
import operator


class Calculator:
    @staticmethod
    def static_add(x, y):
        return x + y

    def class_add(cls, x, y):
        return x + y

    def instance_add(self, x, y):
        return x + y

c = Calculator()
getattr_static_added_result = getattr(Calculator, 'static_add')(0, 1)
getattr_class_added_result = getattr(Calculator, 'class_add')(Calculator, 1, 2)
getattr_instance_added_result = getattr(c, 'instance_add')(2, 3)


operator_static_added_result = operator.methodcaller('static_add', 0, 1)(Calculator)
operator_class_added_result = operator.methodcaller('class_add', Calculator, 1, 2)(Calculator)
operator_instance_added_result = operator.methodcaller('instance_add', 2, 3)(c)

```

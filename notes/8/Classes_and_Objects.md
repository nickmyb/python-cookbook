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
    __slots__ = ['year', 'month', 'day'
    
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

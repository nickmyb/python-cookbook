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

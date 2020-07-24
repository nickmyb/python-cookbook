函数
===

## 7.1 可接受任意数量参数的函数

```python
# * / **: 传入任意数量的 位置参数 / 关键字参数
def any_args(*args, **kwargs):
    args_type = type(args)
    kwargs_type = type(kwargs)
    print(f'args: type[{args_type}], {args}')
    print(f'kwargs: type[{kwargs_type}], {kwargs}')

```

## 7.2 只接受关键字参数的函数

```python
def kwargs_force_1(*args, k1):
   pass

def kwargs_force_2(v1, *, k1):
    pass

# 这样写是错的
def kwargs_force_3(*args, *, k1):
    pass

```

## 7.3 给函数参数增加元信息

```python
# typing hints
def add(x: int, y: int) -> int:
    return x + y

add.__annotations__

```

## 7.4 返回多个值的函数

```python
def multi_ret():
    return 1, 2, 3

ret = multi_ret()
a, b, c = multi_ret()

```

## 7.5 定义有默认参数的函数

```python
# 可变对象使用None作为默认值
def spam_1(arg, default_mutable_arg=None):
    # 使用is做判断,不要使用if/if not判断
    if default_mutable_arg is None:
        default_mutable_arg = []


# mutable对象每次调用传入的都是同一个默认值对象
def spam_2(default_mutable_arg=[]):
    print(f'passed default_mutable_arg id: {id(default_mutable_arg)}, value: {default_mutable_arg}')
    default_mutable_arg.append(1)

for i in range(3):
    spam_2()


# 检测是否传入参数
_default_arg = object()
def check_if_default_arg_passed(default_arg=_default_arg):
    print(f'default_arg: {default_arg}')
    if default_arg is _default_arg:
        print('no default arg passed')

check_if_default_arg_passed(1)
check_if_default_arg_passed(None)
check_if_default_arg_passed()


# 默认参数在函数定义的时候赋值
_v = 1
def spam_3(arg=_v):
    print(arg)

_v = 2
spam_3()

```

## 7.6 定义匿名或内联函数

```python
# 只能使用单个表达式,并且由于":"所以typing hints也不能使用
add = lambda x, y: x + y

```

## 7.7 匿名函数捕获变量值

```python
# lambda expression is a free variable that gets bound at runtime, not definition time
# lambda表达式运行时绑定
l = lambda y: x + y
x = 10
l(10)
x = 20
l(10)

# 使用默认参数可以实现定义时绑定
x = 10
l = lambda y, x=x: x + y
l(10)

x = 20
l(10)

funcs = [lambda x: x+n for n in range(5)]
for f in funcs:
    print(f(0))

funcs = [lambda x, n=n: x+n for n in range(5)]
for f in funcs:
    print(f(0))

```

## 7.8 减少可调用对象的参数个数

```python
from functools import partial


def spam(a, b, c, d):
    print(a, b, c, d)


# partical就是依次传递, 实际传入的*args, 通过partical设置的*args, 合并的**kwargs 
partial_spam = partial(spam, 1, 2, d=4)

partial_spam(3)
print(partial_spam.func, partial_spam.args, partial_spam.keywords)

```

## 7.9 将单方法的类转换为函数

```python
class Runner:
    def __init__(self, name):
        self.name = name

    def run(self, times):
        print(f'{self.name} run for {times} times!')


# 使用闭包
def runner(name):
    def run(times):
        print(f'{name} run for {times} times!')
    return run

```

## 7.10 带额外状态信息的回调函数

```python
# 除了使用类的成员属性 / 闭包 / partical,还可以使用协程
def apply_async(func, args, *, callback):
    # Compute the result
    result = func(*args)
    # Invoke the callback with the result
    callback(result)


def add(x, y):
    return x + y


def make_handler():
    sequence = 0
    while True:
        result = yield
        sequence += 1
        print('[{}] Got: {}'.format(sequence, result))


handler = make_handler()
next(handler)  # 第一次调用生成器需要先前进到yield,等效于handler.send(None)
apply_async(add, (2, 3), callback=handler.send)
apply_async(add, ('hello', 'world'), callback=handler.send)

```

## 7.11 内联回调函数

```python
from queue import Queue
from functools import wraps


class Async:
    def __init__(self, func, args):
        self.func = func
        self.args = args


# 这就是一个建议的异步框架,实现十分的高级!
def inlined_async(f):
    @wraps(f)
    def wrapper(*args):
        # 这部分的还是都是实际runtime的时候赋值的
        generator = f(*args)
        queue = Queue()
        queue.put(None)

        while True:
            # 如果没有获取到新的内容,这里会被阻塞住,等callback塞回新的值
            result = queue.get()
            try:
                async_obj = generator.send(result)
                apply_async(async_obj.func, async_obj.args, callback=queue.put)
            except StopIteration:
                break

    return wrapper


def add(x, y):
    return x + y

@inlined_async
def test():
    print('start test')
    for i in range(5):
        ret = yield Async(add, (i, i))
        print('ret:', ret)
    print('end test')


def apply_async_1(func, args, *, callback):
    result = func(*args)
    print('apply_async_1:', result)
    callback(result)


def apply_async_2(func, args, *, callback):
    result = func(*args)
    print('apply_async_2:', result)
    callback(result)


apply_async = apply_async_1
test()
apply_async = apply_async_2
test()

```

## 7.12 访问闭包中定义的变量

```python
def closure():
    var = 0

    def f():
        print(var)

    def setter(n):
        nonlocal var
        var = n

    f.setter = setter
    return f


f = closure()
f()
f.setter(1)
f()

```

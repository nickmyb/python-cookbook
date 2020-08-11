## 9.1 在函数上添加包装器

```python
from functools import wraps
import time


def duration_decorator(f):
    # wraps 保留原始函数的元数据 
    @wraps(f)
    def decorator(*args, **kwargs):
        start = time.time()
        ret = f(*args, **kwargs)
        end = time.time()
        print(f"{f.__name__}'s duration time is: {end - start}")
    return decorator


@duration_decorator
def f():
    for i in range(2):
        time.sleep(1)


f()

```

## 9.2 创建装饰器时保留函数元信息

```python
from functools import wraps
from inspect import signature
import time


def duration_decorator(f):
    # wraps 保留原始函数的元数据 
    @wraps(f)
    def decorator(**kwargs):
        start = time.time()
        ret = f(*args, **kwargs)
        end = time.time()
        print(f"{f.__name__}'s duration time is: {end - start}")
    return decorator


@duration_decorator
def f(*args, **kwargs):
    for i in range(2):
        time.sleep(1)


original_f = f.__wrapped__
signature(f)

```

## 9.3 解除一个装饰器

```python
from functools import wraps


def print_1_decorator(f):
    @wraps(f)
    def decorator(*args, **kwargs):
        print('decorator 1 start')
        f(*args, **kwargs)
        print('decorator 1 end')
    return decorator


def print_2_decorator(f):
    @wraps(f)
    def decorator(*args, **kwargs):
        print('decorator 2 start')
        f(*args, **kwargs)
        print('decorator 2 end')
    return decorator


@print_1_decorator
@print_2_decorator
def f():
    print('f')


f()
print('-' * 89)
# 仅适用于用 wraps 包装的函数
ripped_1_f = f.__wrapped__
ripped_1_f()

# 有几层装饰器就可以扒掉几层
ripped_2_f = ripped_1_f.__wrapped__
ripped_2_f()

# @staticmethod 和 @classmethod 的原始函数保存在 __func__

```

## 9.4 定义一个带参数的装饰器

```python
def multi_params_decorator(*decorator_args):
    def decorator(f):
        def wrapper(*args, **kwargs):
            print('wrapper start')
            for index, arg in enumerate(decorator_args):
                print(index, arg)
            ret = f(*args, **kwargs)
            print('wrapper end')
            return ret
        return wrapper
    return decorator


@multi_params_decorator(1, 2, 3)
def f():
    print('f')


f()

```

## 9.5 可自定义属性的装饰器

```python
from functools import wraps


def multi_params_decorator(name=None, info=None):
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            print('name:', name)
            print('info:', info)
            ret = f(*args, **kwargs)

        def set_name(new_name):
            nonlocal name
            name = new_name

        def set_info(new_info):
            nonlocal info
            info = new_info
        
        setattr(wrapper, set_name.__name__, set_name)
        wrapper.set_info = set_info
        return wrapper

    return decorator


@multi_params_decorator(name=1, info=1)
def f():
    print('f')


f()
f.set_name(2)
f.set_info(2)
f()


def decorator(f):
    @wraps(f)
    def wrapper(*args, **kwargs):
        print('-' * 10)
        ret = f(*args, **kwargs)
        print('-' * 10)
        return ret
    return wrapper


@decorator
@multi_params_decorator(name=1, info=1)
def g():
    print('g')


# 主要就是利用wraps进行函数元数据的传递
g()
g.set_name(2)
g.set_info(2)
g()

```

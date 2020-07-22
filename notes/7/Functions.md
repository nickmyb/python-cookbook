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
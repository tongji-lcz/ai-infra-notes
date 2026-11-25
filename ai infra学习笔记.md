## 一.前置基础

### 1.核心理念

明确一个观点，所有的优化都是在计算、通信和显存的不可能三角中做取舍，学习时候始终要记住，这个技术牺牲了什么，换取了什么

### 2.编程语言基础

#### 2.1 三层语言

编排层Python、Shell，系统层C/C++，设备层CUDA C++，Triton，AI infra工程师最重要的语言能力是沿调用链跨层追踪问题

#### 2.2 常见命令工具箱

```bash
python --version       # 正在使用哪个 Python
which python           # 这个可执行文件来自哪里
python -m pip --version

g++ --version          # C++ 编译器版本
cmake --version        # 构建系统版本

uname -a               # 内核与系统信息
ps aux                 # 有哪些进程
free -h                # 主机内存使用
df -h                   # 文件系统容量

nvidia-smi             # 驱动可见的 GPU 与进程
nvcc --version         # CUDA Toolkit 中的编译器版本
```

#### 2.3 python

1.Python 变量更准确的说法是“**名字绑定到对象**”，写函数时自问一句：我这个函数是“改造了传进来的东西”，还是“做了个新东西返回出去”？这两者的语义完全不同，接口设计时要想清楚

2.常见不可变对象包括 `int`、`float`、`bool`、`str`、`bytes` 和元素均不可变时的 `tuple`；常见可变对象包括 `list`、`dict`、`set` 和大多数自定义实例。

3.|表示联合类型，int|string表达的是，int和string类型都可以

4.

```python
# 错误：同一个列表会被多次调用共享
def collect_bad(x: int, result: list[int] = []) -> list[int]:
    result.append(x)
    return result

collect_bad(1) -> [1]        id=1459917953216
collect_bad(2) -> [1, 2]     id=1459917953216
collect_bad(3) -> [1, 2, 3]  id=1459917953216     ← 三次是同一个列表！

#这里的严重问题在于局部变量 result 每次调用都是新创建的，并非同一个变量，但每次它都绑定到同一个列表对象（存在 __defaults__ 里的那个），累加发生在那个对象里，不是发生在变量里

# 正确：用 None 表示“未提供”
def collect(x: int, result: list[int] | None = None) -> list[int]:
    if result is None:
        result = []
    result.append(x)
    return result

```

这部分代码里需要注意的是，if result is None和if not result可完全不是一个东西，后者只是判断了，result为不为真，例如[]是false但是不是None

5.深拷贝和浅拷贝

```python
import copy

src = [[1], [2]]
shallow = src.copy()           # 只复制最外层列表
deep = copy.deepcopy(src)      # 递归复制内部对象

shallow[0].append(9)
print(src)                     # [[1, 9], [2]]
print(deep)                    # [[1], [2]]

shallow = src.copy()      # 新外层 S，但 S[0] 还是 B，S[1] 还是 C
deep = copy.deepcopy(src) # 新外层 + 新内层 B'、C'
```

6.Python 按 LEGB 顺序查找名字：Local（当前函数内部，函数参数、函数里赋值的变量）→ Enclosing（外层函数，闭包，内层函数看到外层函数的变量）→ Global（模块）→ Builtins（内置）。

```python
x = 1
def f():
    print(x)     # 报 UnboundLocalError！
    x = 2        # 因为函数里出现了对 x 的赋值，x 在整个函数内都被判定为 local
    
#核心思想，函数里只要给某个名字赋过值，它在整个函数里就是局部变量，和"先读到未赋值的局部变量"冲突就报错
```

在 Python 里，函数本身也是一个"值"，可以赋值给变量、放进列表、被返回；只有写出名字(...) 时才真正执行它。 

```python
from collections.abc import Callable

def make_multiplier(scale: float) -> Callable[[float], float]:
    def multiply(x: float) -> float:
        return x * scale
    return multiply

double = make_multiplier(2.0)
print(double(3.0))  # 6.0
#double就是一个函数对象，类型是function
#可以理解成：make_multiplier(2.0) 造出了一台"乘以 2"的机器，double 是这台机器的名字；double(3.0) 才是把 3.0 塞进机器里运行。再调用一次 make_multiplier(10) 会造出另一台"乘以 10"的机器，两台互不影响，各自记着自己的 scale。

-----------------------------------------------------------------------------

# 三个函数最终都读取同一个 i，结果都是 2
bad = [lambda: i for i in range(3)]

# 用默认参数在每轮定义时保存当前值
good = [lambda i=i: i for i in range(3)]
print([fn() for fn in good])  # [0, 1, 2]

```

lambda是匿名函数的简写，格式是lambda参数: 表达式，这里bad是一个三个函数对象的list列表

7.Python 的特殊方法让对象接入语言协议。例如，实现 `__len__` 后可调用 `len(obj)`，实现 `__iter__` 后可被 `for` 遍历。

8.@表示装饰器，@dataclass是类装饰器，@classmethod是方法装饰器，让方法的第一个参数变成类本身

**`@dataclass` 是「自动写代码」的机器**（省掉手写 `__init__`/`__repr__`/`__eq__`）； **`@classmethod` 是「把第一个参数换成类」的开关**（省掉先造实例才能调用的麻烦）。

9.迭代器和生成器：按需生产数据

yield是生成器函数，iterator是迭代器，生成器函数是编写迭代器的简介方式，**函数体内只要出现 `yield`，整个函数就变成生成器函数**——不管你调多少次，它都不会像普通函数那样直接跑完返回结果。

10.**装饰器就是一个普通函数：接收一个可调用对象，返回另一个可调用对象。** `@` 只是语法糖。

在调用边界附加行为

```python
from collections.abc import Callable
from functools import wraps
from time import perf_counter
from typing import ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")

def timed(fn: Callable[P, R]) -> Callable[P, R]:
    @wraps(fn)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        start = perf_counter()
        try:
            return fn(*args, **kwargs)
        finally:
            elapsed_ms = (perf_counter() - start) * 1_000
            print(f"{fn.__name__}: {elapsed_ms:.3f} ms")
    return wrapper

@timed
def preprocess(values: list[float]) -> list[float]:
    return [value * 2 for value in values]


##  @timed = 在函数调用前后加计时代码；@wraps(fn) = 保留原函数名和文档，让装饰后"看起来还是原函数"。
##  @timed 等价于 preprocess = timed(preprocess)。functools.wraps 会保留函数名、文档字符串等属性，这对日志、测试和框架反射很重要。
```

11.上下文管理器

```python
from contextlib import contextmanager
from collections.abc import Iterator
from time import perf_counter

@contextmanager
def timer(name: str) -> Iterator[None]:
    start = perf_counter()
    try:
        yield
    finally:
        elapsed_ms = (perf_counter() - start) * 1_000
        print(f"{name}: {elapsed_ms:.3f} ms")

with timer("load data"):
    data = [value for value in range(100_000)]

    
## 如果只是单个调用点需要计时，一个普通的 with timer(): 往往更直白，不必为了“高级”而使用装饰器。
```

with语句是成对的准备收尾动作，收尾一定执行，所以 `with timer("load data"):` 的意思是：进入这个块时启动计时，离开块时打印耗时。

@contextmanager装饰器相当于把一个“生成器函数”改造成上下文管理器，`yield` 之前的代码 = `__enter__` 里做的事；`yield` 本身 = 交出控制权，`with` 块在这里执行；`yield` 之后的代码 = `__exit__` 里做的事，`@contextmanager` 是把普通生成器函数变成 `with` 能用的工具。

不是 `with` 需要"装饰器"，而是 `with` 需要对象有 `__enter__`/`__exit__`；文件、锁、数据库连接天生就有，而你的生成器没有，`@contextmanager` 就是帮你把这两条"缝隙"补上，让 `with` 能驱动它，所以「任何实现了 `__enter__` 和 `__exit__` 的对象 = 上下文管理器」，而 `@contextmanager` 只是帮你少写这两个方法的快捷方式。











